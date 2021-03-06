# Tips to get started with Nextflow scripting

**Error reports and suggestions welcome!**


### The working directory
- **Each execution of a process happens in its own temporary working directory.** 
- Specify the location of working directory with `workDir = '/path_to_tmp/'` in nextflow.config, or with `-w` option when running `nextflow main.nf`.
- The working directory is the folder named like `/path_to_tmp/4d9c3b333734a5b63d66f0bc0cfcdc` that Nextflow points you to when there is an error in execution. This folder contains the error log that could be useful for debugging. One can find the folder path in the .nextflow.log or in the report.html. 
- This folder only contains files (usually in form of symlinks, see below) from the input channel, so it's isolated from the rest of the file system. 
- This folder will also contain all output files (unless specifically directed elsewhere), and only those specified in the output channels and `publishDir` will be moved or copied to the `publishDir`. 
- Note that with `publishDir "path", mode: 'move'`, the output file will be moved outside of the working directory and Nextflow will not be able to use it as input for another process, so only use it when there is not a following process that uses the output file. 
- Be mindful that if the `""" (script section) """` involves changing directory, such as `cd` or `rmarkdown::render( knit_root_dir = "folder/" )`, Nextflow will still only search the working directory for output files. 
- Run `nextflow clean -f` in the excecution folder to clean up the working directories.


### Where am I?
- In Nextflow scripts (.nf files), one can use 
  - `${workflow.projectDir}` to refer where the project locates (usually the folder of main.nf). For example: `publishDir "${workflow.projectDir}/output", mode: 'copy'` or `Rscript ${workflow.projectDir}/bin/task.R`.
  - `${workflow.launchDir}` to refer to where the script is called from. 
- `$baseDir` usually refers to the same folder as `${workflow.projectDir}` but it can also be used in the config file, where `${workflow.projectDir}` and `${workflow.launchDir}` are not accessible.   
- They are much more reiable than `$PWD` or `$pwd`.


### Print - debugger's best friend
- To print a channel, use `.view()`. It's especially useful to resolve `WARN: Input tuple does not match input set cardinality declared by process`. (Don't forget to remove `.view()` after debugging) 
```
  channel_vcf
    .combine(channel_index)
    .combine(channel_chr)
    .view()
```
- To print from the script section inside the processes, add `echo true`.
```
  process test {
    echo true    // this will print the stdout from the script section on Terminal
    input: path(vcf)
    """
    head $vcf
    """
  }
```

### `Channel.fromPath("A.txt")` in channel creation
- `Channel.from( "A.txt" )` will put `A.txt` as is into the channel 
- `Channel.fromPath( "A.txt" )` will add a full path (usually current directory) and put `/path/A.txt` into the channel. 
- `Channel.fromPath( "folder/A.txt" )` will add a full path (usually current directory) and put `/path/folder/A.txt` into the channel. 
- `Channel.fromPath( "/path/A.txt" )` will put `/path/A.txt` into the channel. 
- In other words, `Channel.fromPath` will only add a full path if there isn't already one and ensure there is always a full path in the resulting channel.
- This goes hand in hand with `input: path("A.txt")` inside the process, where **Nextflow actually creates a symlink named `A.txt`** (note the path from first `/` to last `/` is stripped) **linking to `/path/A.txt` in the working directory**, so it can be accessed within the working directory by the script `cat A.txt` without specifying a path.


### `input: path("A.txt")` in the process section 
- With `input: path("A.txt")` one can refer to the file in the script as `A.txt`. Side note `A.txt` doesn't have to be the same name as in channel creation, it can be anything, `input: path("B.txt")`, `input: path("n")` etc. 
- With `input: path(A)` one can refer to the file in the script as `$A`, and the value of `$A` will be the original file name (without path, see section above). 
- `input: path("A.txt")` and `input: path "A.txt"` generally both work. Occasionally had errors that required the following (tip from [@danielecook](https://github.com/danielecook)): 
  - If not in a tuple, use `input: path "A.txt"` 
  - If in a tuple, use `input: tuple path("A.txt"), path("B.txt")`
  - This goes the same for `output`.
- From [@pditommaso](https://github.com/pditommaso): `path(A)` is almost the same as `file(A)`, however the first interprets a value of type string as the input file path (ie the location in the file system where it's stored), the latter interprets a value of type string and materialise it to a temporary files. It's recommended the use of `path` since it's less ambiguous and fits better in most use-cases.


### DSL2
- Moving to DSL2 is a one-way street. It's so intuitive with clean and readable code.
- In DSL1, each queue channel can only be used once. 
- In DSL2, a channel can be fed into multiple processes
- In DSL2, each process can only be called once. The solution is either `.concat()` the input channels so they run as parallel processes, or put the process in a module and import multiple times from the module. (One may be able to call a process in different workflows, haven't tested yet).
- DSL2 also enforces that all inputs needs to be combined into 1 channel before it goes into a process. See the [cheatsheet](https://github.com/danrlu/Nextflow_cheatsheet/blob/main/nextflow_cheatsheet.pdf) for useful operators. 
- [Simple steps to convert from original syntax to DSL2](https://github.com/danrlu/Nextflow_cheatsheet/blob/main/nextflow_convert_DSL2.pdf)
- [Deprecated operators](https://www.nextflow.io/docs/latest/dsl2.html#dsl2-migration-notes).


### Run reports
- `nextflow main.nf -with-report -with-timeline -with-dag`
- `-with-report` Nextflow html report contains resource usage for each process, and details (most useful being the status and working directory) for each process 
- `-with-timeline` How much wait time and run time each process took for the run. Very useful reference for optimizing resource allocation and improving run time.
- `-with-dag` Make a flowchart to show the relationship of channels and processes. 
- [Software dependencies](https://www.nextflow.io/docs/latest/tracing.html#execution-report) to use these features. Note the differences on Mac and Linux.
- How to set them up in the [nextflow.config](https://github.com/AndersenLab/wi-gatk/blob/master/nextflow.config) so they are automatically generated for each run. Credit [@danielecook](https://github.com/danielecook) 

### [More advanced tips](https://github.com/danrlu/Nextflow_cheatsheet/blob/main/advanced_tips.md)

### Acknowledgement
- [@danielecook](https://github.com/danielecook) for offering lots of help and advice.

