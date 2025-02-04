

# The first task
Before we write any sort of WDL -- whether it is for somatic mutation calling like we will be going over, or any other bioinformatics task -- we need to understand the building blocks of WDL: Tasks!

As mentioned in the first part of this course, every WDL workflow is made up of at least one task. A task typically has inputs, outputs, runtime attributes, and a command section. You can think of a task as a discrete step in a workflow. It can involve a single call to a single bioinformatics tool, a sequence of bash commands, an inline Python script... almost anything you can do non-interactively in a terminal, you can do in a WDL task. In this section, we will go over the parts of a WDL task in more detail to help us write a task for somatic mutation calling.

## Inputs
The inputs of a task are the files and/or variables you will passing into your task's command section. Typically, you will want to include at least one File input in a task, but that isn't a requirement. You can pass most WDL variable types into a task. In our example workflow, we are starting with a single fastq file per sample, and we know we will need to convert it into a sam file. A sam file is an alignment, so we will need a reference genome to align our fastqs to. We also want to be able to control the threading for this task. Our first task's inputs will therefore start out looking like this:
```
task some_aligner {
  input {
    File input_fastq
    File ref_fasta
    Int threads
  }
[...]
}
```

For some aligners, this would be a sufficient set of inputs, but we have decided to use bwa mem in particular to take us from fastq to sam. bwa mem requires a lot of index files, which we will also need to input. This can be done via an array, but for now we'll list everything separately to make sure nothing is being left out.

We also want to define a default value for `threads` so that someone who does not know much about threading can still use the workflow. We want to use this workflow on human data, so we'll go a little high for the default number of threads and set it to sixteen. In WDL, we do this by declaring `Int threads = 16`. Make sure to put this in the task (or workflow) inputs section -- if you put it elsewhere, that variable cannot be changed from its default value, so it will always be 16.

```
task BwaMem {
  input {
    # main input
    File input_fastq

    # options
    Int threads = 16

    # reference files
    File ref_fasta
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
  }
  [...]
}
```

### Referencing inputs in the command section

The command section of a WDL task is a bash script that will be run [non-interactively](https://tldp.org/LDP/abs/html/intandnonint.html) by the WDL executor. Although it is helpful to think of tasks as discrete steps in a workflow, that does not mean each task needs to be a single line. You could, for example, call a bioinformatics tool and then reprocess the outputs in the same WDL task. 

Within the command section, we refer to those variables using `~{this}` syntax. For instance, if the user sets `threads` to 8, then the `-t ~{threads}` part of the command section below will be interpreted as `-t 8`.

A WDL task's input variables are generally referred to in the command section using a tilde (~) and curly braces, using heredoc syntax. 

<details>
<summary> <b>Why we use heredox syntax.</b></summary>

You may see WDLs that use this notation for the command section in a task:

```
task do_something_curly_braces {
  input {
    String some_string
  }
  command {
    some_other_string="FOO"
    echo ${some_string}
    echo $some_other_string
  }
}
```

We recommend using heredoc-style syntax instead:
```
task do_something_carrots {
  input {
    String some_string
  }
  command <<<
    some_other_string="FOO"
    echo ~{some_string}
    echo $some_other_string
  >>>
}
```

Heredoc-style syntax for command sections can be clearer than the alternative, as it makes a clearer distinction between bash variables and WDL variables. This is especially helpful for complicated bash scripts. Heredoc-style syntax is also what the WDL 1.1 spec recommends using in most cases. However, the older non-heredoc style is still perfectly functional for a lot of use cases.

</details>


To prevent issues with spaces in String and File types, it is often a good idea to put quotation marks around a String or File variabls, like so:

```
task cowsay {
  input {
    String some_string
  }
  command <<<
    cowsay -t "~{some_string}"
  >>>
}
```

<details>
<summary><b>Why we put quotation marks around a String or File variables in Commands.</b></summary>

If `some_string` is "hello world" then the command section of this task is interpreted as the following:

```
cowsay -t "hello world"
```

What happens if we had not wrapped `~{some_string}` in quotation marks? If `some_string` was just "hello", it wouldn't matter. But because `some_string` is two words with a space in between, then the script would be interpreted as `cowsay -t hello world` and cause an error, because the cowsay program thinks `world` is another argument. By including quotation marks, `cowsay -t "~{some_string}"` can be interpreted as `cowsay -t "hello world"` and you will correctly get a cow's greeting instead of an error.

</details>

Let's see how we can reference our inputs in the command section of our task. 

```
task BwaMem {
  input {
    File input_fastq
    File ref_fasta
    Int threads = 16

    # these variables may look as though they are unused... but bwa mem needs them!
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
  }
  command <<<
  # warning: this will not run on all backends! see below for an explanation!
  bwa mem \
      -p -v 3 -t ~{threads} -M -R '@RG\tID:foo\tSM:foo2' \
      "~{ref_fasta}" "~{input_fastq}" > my_nifty_output.sam 
  >>>
}
```

If we were to run this task in a workflow as-is, we might expect it to run on any backend that can handle the hardware requirements. Those hardware requirements are a bit steep -- the `-t 16` part specifically requests 16 threads, for example -- but besides that, it may look like a perfectly functional task. Unfortunately, even on backends that can provide the necessary computing power, it is quite likely this task will not run as expected. This is because of how inputs work in WDL -- or, more specifically, how input files get localized when working with WDL.

### File localization
When running a WDL, a WDL executor will typically place duplicates of the input files in a brand-new subfolder of the task's [working directory](https://www.ibm.com/docs/en/zos/3.1.0?topic=directories-working-directory). Typically, you don't know the name of the directory before runtime -- they vary depending on the backend you are running and the WDL executor itself. Thankfully, at runtime, File-type variables such as `~{input_fastq}` and `~{ref_fasta}` will be replaced with paths to their respective files.

For example, if you were to run this workflow on a laptop using miniwdl, `~{ref_fasta}` would likely end up turning into `./_miniwdl_inputs/0/ref.fa` at runtime. On the other hand, if you were running the exact same workflow with Cromwell, `~{ref_fasta}` would turn into something like `/cromwell-executions/BwaMem/97c9341e-9322-9a2f-4f54-4114747b8fff/call-test_localization/inputs/-2022115965/ref.fa`. Keep in mind that these are the paths of *copies* of the input files, and that sometimes input files can be in different subfolders. For example, it's possible `~{input_fastq}` would be `./_miniwdl_inputs/0/sample.fastq` while `~{ref_fasta}` may be `./_miniwdl_inputs/1/ref.fa`.

For many programs, an input file being at `./ref.fa` versus `/_miniwdl_inputs/0/ref.fa` is inconsequential. However, this aspect of WDL can occasionally cause issues. bwa mem is a great example of the type of command where this sort of thing can go haywire without proper planning, due to the program making an assumption about some of your input files. Specifically, bwa mem assumes that the reference fasta that you pass in shares the same folder as the other reference files (ref_amb, ref_ann, ref_bwt, etc), and it does not allow you to specify otherwise.

<details>
<summary><b>Another example of file localization issue.</b></summary>

bwa is not the only program that makes assumptions about where files are located, and assumptions being made do not only affect reference genome files. Bioinformatics programs that take in some sort of index file requently assume that index file is located in the same directory as the non-index input. For example, if you were to pass in `SAMN1234.bam` into [covstats](https://github.com/brentp/goleft/tree/master/covstats), it would expect an index file named `SAMN1234.bam.bai` or `SAMN1234.bai` in the same directory as the bam file, [as seen in the source code here](https://github.com/brentp/goleft/blob/fa6b00d20d1f73a068ffbab49a5769d173cae56d/covstats/covstats.go#L239). As there is no way to specify that the index file manually, you need to take that into consideration when writing WDLs involving covstats, bwa, and other similar tools.

</details>

Thankfully, the solution here is simple: Move all of the input files directly into the working directory.

```
task BwaMem {
  input {
    File input_fastq
    File ref_fasta
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
    Int threads = 16
  }

  command <<<
    set -eo pipefail

    # This can also be done by creating an array and then looping that array,
    # but we'll do it one line at a time or clarity's sake.
    mv "~{ref_fasta}" .
    mv "~{ref_fasta_index}" .
    mv "~{ref_dict}" .
    mv "~{ref_amb}" .
    mv "~{ref_ann}" .
    mv "~{ref_bwt}" .
    mv "~{ref_pac}" .
    mv "~{ref_sa}" .

    bwa mem \
    [...]
  >>>
}
```
::: {.notice data-latex="warning"}
Some backends/executors do not support `mv` acting on input files. If you are running into problems with this and are working with miniwdl, the `--copy-input-files` flag will usually allow `mv` to work. You could also simply use `cp` to copy the files instead of move them, although this may not be an efficient use of disk space, so consider using `mv` if your target backends and executors can handle it.
:::

With our files now all in the working directory, we can turn our attention to the bwa task itself. We can no longer directly pass in `~{ref_fasta}` or any of the other files we mved into the working directory, because those variables will point to a non-existent file in a now-empty input directory. There are several ways to solve this problem:

* Assuming the filename of an input is constant, which might be a safe assumption for reference files

* Using the bash built-in basename function

* Using the WDL built-in basename() function along with private variables

We recommend using the last option, as it works for essentially any input and may be more intuitive than the bash basename function. [OpenWDL explains](https://docs.openwdl.org/en/stable/WDL/basename/) how `basename()` works. The next section will provide an example of using it alongside private variables.

### Private variables
Is there a variable you wish to use in your task section that is based on another input variable, or do not want people using your workflow to be able to directly overwrite? You can define variables outside the `input {}` section to create variables that function like private variables. In our case, we create `String ref_fasta_local` as `ref_fasta`'s file base name to refer to the files we have moved to the working directory. We also create `String base_file_name` as `input_fastq`'s file base name and use it to name our output files, such as `"~{base_file_name}.sorted_query_aligned.bam"`. 

```
task BwaMem {
  input {
    File input_fastq
    File ref_fasta
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
    Int threads = 16
  }
  
  # basename() is a built-in WDL function that acts like bash's basename
  String base_file_name = basename(input_fastq, ".fastq")
  String ref_fasta_local = basename(ref_fasta)

  command <<<
    set -eo pipefail

    mv "~{ref_fasta}" .
    mv "~{ref_fasta_index}" .
    mv "~{ref_dict}" .
    mv "~{ref_amb}" .
    mv "~{ref_ann}" .
    mv "~{ref_bwt}" .
    mv "~{ref_pac}" .
    mv "~{ref_sa}" .


    bwa mem \
      -p -v 3 -t ~{threads} -M -R '@RG\tID:foo\tSM:foo2' \
      "~{ref_fasta_local}" "~{input_fastq}" > "~{base_file_name}.sam"
    samtools view -1bS -@ 15 -o "~{base_file_name}.aligned.bam" "~{base_file_name}.sam"
    samtools sort -n -@ 15 -o "~{base_file_name}.sorted_query_aligned.bam" "~{base_file_name}.aligned.bam"

  >>>
}
```

## Runtime attributes
The runtime attributes of a task tell the WDL executor important information about how to run the task. For a bwa mem task, we want to make sure we have plenty of hardware resources available. We also need to include a reference to the docker image we want the task to actually run in.

```
  runtime {
    memory: "48 GB"
    cpu: 16
    docker: "fredhutch/bwa:0.7.17"
    disks: "local-disk 100 SSD"
  }
```

In WDL 1.0, the interpretation of runtime attributes by different executors and backends is extremely varied. The [WDL 1.0 spec](https://github.com/openwdl/wdl/blob/main/versions/1.0/SPEC.md#runtime-section) allows for arbitrary values here:

> Individual backends will define which keys they will inspect so a key/value pair may or may not actually be honored depending on how the task is run. Values can be any expression and it is up to the engine to reject keys and/or values that do not make sense in that context.

This can lead to some pitfalls: 

* Some of the attributes in your task's `runtime` section may be silently ignored, such as the `memory` attribute when running Cromwell on the Fred Hutch HPC (as of Feb 2024)

* Some runtime attributes that are unique to particular backends, such as the Fred Hutch HPC's `walltime` attribute

* The same runtime attribute working differently on different backends, such as `disks` acting differently on Cromwell depending on whether it is running on AWS or GCP

When writing WDL 1.0 workflows with specific hardware requirements, keep in mind what your backend and executor is able to interpret. It is also helpful to consider that other people running your workflow may be doing so on different backends and executors. More information can be found in the appendix, where we talk about designing WDLs for specific backends. For now, we will stick with `memory`, `cpu`, `docker`, and `disks` as this group of four runtime attributes will help us run this workflow on the majority of backends and executors. Even though the Fred Hutch HPC will ignore the `memory` and `disks` attributes, for instance, their inclusion will not cause the workflow to fail, but they will allow the workflow to run on Terra.

<details>
<summary><b>Some differences between WDL 1.0 and 1.1 on Runtime attributes.</b></summary>

Although the focus of this course is on WDL 1.0, it is worth noting that in the [WDL 1.1 spec](https://github.com/openwdl/wdl/blob/main/versions/1.1/SPEC.md#runtime-section), a very different approach to runtime attributes is taken:

> There are a set of reserved attributes (described below) that must be supported by the execution engine, and which have well-defined meanings and default values. Default values for all optional standard attributes are directly defined by the WDL specification in order to encourage portability of workflows and tasks; execution engines should NOT provide additional mechanisms to set default values for when no runtime attributes are defined.

If you are writing WDLs under the WDL 1.1 standard, you may have more flexibility with runtime attributes. Be aware that as of February 2024, Cromwell does not support WDL 1.1.

</details>

### Docker images and containers
WDL is built to make use of Docker as it makes handling software dependencies much simpler. Docker images can help address all of these situations: 

* Some software is difficult to install or compile on certain systems

* Some programs have conflicting dependencies

* You may not want to directly install software on your system to prevent it from breaking existing software

* You may not have permission to install software if you are using an institute HPC or other shared resource

When you run a WDL task that has a `docker` runtime attribute, your task will be executed in a Docker container sandbox environment. This container sandbox is derived from a template called a Docker image, which packages installed software in a special filesystem. This is one of the main features of a Docker image -- because a Docker image packages the software you need, you can skip much of the installation and dependency issues associated with using new software, and because you take actions within a Docker container sandbox, it's unlikely for you to "mess up" your main system's files. Although a Docker container is, strictly speaking, not the same as a virtual machine, it is helpful to think of it as one if you are new to Docker. Docker containers are managed by Docker Engine, and the official Docker GUI is called Docker Desktop.

<details>
<summary><b>More information on finding and developing Docker images. </b></summary>

Although you will generally need to be able to run Docker in order to run WDLs, you do not need to know how to create Dockerfiles -- plaintext files which compile Docker images when run via `docker build` -- to write your own WDLs. Most popular bioinformatic software packages already have ready-to-use Docker images available, which you can typically find on [Docker Hub](https://hub.docker.com/search?q=). Other registries include quay.io and the Google Container Registry. With that being said, if you would like to create your own Docker images, there are many tutorials and [guidelines](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) available online. You can also learn more about the details of Docker (and why they technically aren't virtual machines) in [Docker's official curriculum](https://docker-curriculum.com/#introduction).

</details>




## Outputs

The outputs of a task are defined in the `output` section of your task. Typically, this will take the form of directly outputting a file that was created in the command section. When these file outputs are referenced in the `output` section, you can refer to their path in the Docker container directly. You can also make outputs a function of input variables, including private input variables. This can be helpful if you intend on running this WDL on many different files -- each one will get a unique filename based on the input fastq, instead of every sample ending up being named something generic like "converted.sam". For our bwa mem task, one way to write the output section would be as follows:

```
  output {
    File analysisReadyBam = "~{base_file_name}.aligned.bam"
    File analysisReadySorted = "~{base_file_name}.sorted_query_aligned.bam"
  }
```

Another way of writing this is with string concatenation. This is equivalent to what we wrote above -- choose whichever version you prefer.

```
  output {
    File analysisReadyBam = base_file_name + ".aligned.bam"
    File analysisReadySorted = base_file_name + ".sorted_query_aligned.bam"
  }
```

If the output was not in the working directory, we would need to change the output to point to the file's path relative to the working directory, such as `File analysisReadyBam = "some_folder/~{base_file_name}.aligned.bam"`.

Below are some some additional ways you can handle task outputs.

<details>
<summary><b>Ouputs as functions of other outputs in the same task.</b></summary>

Outputs can (generally, see warning below) also be functions of other outputs in the same task, as long as those outputs are declared first.

```
task add_one {
  input {
    Int some_integer
  }
  command <<<
    echo ~{some_integer} > a.txt
    echo "1" > b.txt
  >>>
  output {
    Int a = read_int("a.txt")
    Int b = read_int("b.txt")
    Int c = a + b
  }
}
```

::: {.notice data-latex="warning"}
Cromwell does not fully support outputs being a function of the same task's other outputs. On the Terra backend, the above code example would cause an error.
:::

</details>


<details>
<summary><b>Grabbing multiple outputs at the same time</b></summary>

To grab multiple outputs at the same time, use glob() to create an array of files. We'll also take this opportunity to demonstrate iterating through a bash array created from an Array[String] input -- for more information on this data type, see chapter six of this course.

```
task one_word_per_file {
  input {
    Array[String] a_big_sentence
  }
  command <<<
    ARRAY_OF_WORDS=(~{sep=" " a_big_sentence})
    i=0
    for word in "${!ARRAY_OF_WORDS[@]}"
    do
      i=$((i+1))
      echo $word >> $i.txt
    done
  >>>
  output {
    Array[File] several_words = glob("*.txt")
  }
}
```

`glob()` can also be used to grab just one file via `glob("*.txt")[0]` to grab the first thing that matches the glob. This is usually only necessary if you know the extension of an output, but do not have a way of predicting the rest of its filename. Be aware that if anything else in the working directory has the extension you are searching for, you might accidentally grab that one instead of the one you are looking for!

</details>


## The whole task

We've now designed a bwa mem task that can run on essentially any backend that supports WDL and can handle the hardware requirements. Issues involving bwa mem expecting reference files to be in the same folder and/or putting output files into input folders have been sidestepped thanks to careful design and consideration. The runtime section clearly defines the expected hardware requirements, and the outputs section defines what we expect the task to give us when all is said and done. We're now ready to continue with the rest of our workflow.

```         
task BwaMem {
  input {
    File input_fastq
    File ref_fasta
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
    Int threads = 16
  }
  
  String base_file_name = basename(input_fastq, ".fastq")
  String ref_fasta_local = basename(ref_fasta)

  command <<<
    set -eo pipefail

    mv "~{ref_fasta}" .
    mv "~{ref_fasta_index}" .
    mv "~{ref_dict}" .
    mv "~{ref_amb}" .
    mv "~{ref_ann}" .
    mv "~{ref_bwt}" .
    mv "~{ref_pac}" .
    mv "~{ref_sa}" .

    bwa mem \
      -p -v 3 -t ~{threads} -M -R '@RG\tID:foo\tSM:foo2' \
      "~{ref_fasta_local}" "~{input_fastq}" > "~{base_file_name}.sam"
    samtools view -1bS -@ 15 -o "~{base_file_name}.aligned.bam" "~{base_file_name}.sam"
    samtools sort -n -@ 15 -o "~{base_file_name}.sorted_query_aligned.bam" "~{base_file_name}.aligned.bam"

  >>>
  output {
    File analysisReadyBam = "~{base_file_name}.aligned.bam"
    File analysisReadySorted = "~{base_file_name}.sorted_query_aligned.bam"
  }
  runtime {
    memory: "48 GB"
    cpu: 16
    docker: "fredhutch/bwa:0.7.17"
    disks: "local-disk 100 SSD"
  }
}
```

## Putting the workflow together

A workflow is needed to run the `BwaMem` task we just built. The workflow's input variables are defined by the workflow JSON metadata, and are then passed on as inputs in our `BwaMem` call. When the `BwaMem` call is complete, the workflow's output File variable is defined based on the task's output. Lastly, we have a parameter_meta component in our workflow that describes each workflow input variable as documentation.

For the workflow to actually "see" the task, the task will either need to be imported at the top of the workflow (just under the `version 1.0` string), or included in the same file as the workflow. For simplicity, we will put the workflow and the task in the same file.
```         
version 1.0

workflow minidata_test_alignment {
  input {
    # Sample info
    File sampleFastq
    # Reference Genome information
    File ref_fasta
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
    #Optional BwaMem threading variable
    Int? bwa_mem_threads
  }

  #  Map reads to reference
  call BwaMem {
    input:
      input_fastq = sampleFastq,
      ref_fasta = ref_fasta,
      ref_fasta_index = ref_fasta_index,
      ref_dict = ref_dict,
      ref_amb = ref_amb,
      ref_ann = ref_ann,
      ref_bwt = ref_bwt,
      ref_pac = ref_pac,
      ref_sa = ref_sa,
      threads = bwa_mem_threads

  }
   
  # Outputs that will be retained when execution is complete
  output {
    File alignedBamSorted = BwaMem.analysisReadySorted
  }

  parameter_meta {
    sampleFastq: "Sample .fastq (expects Illumina)"
    ref_fasta: "Reference genome to align reads to"
    ref_fasta_index: "Reference genome index file (created by bwa index)
    ref_dict: "Reference genome dictionary file (created by bwa index)"
    ref_amb: "Reference genome non-ATCG file (created by bwa index)"
    ref_ann: "Reference genome ref seq annotation file (created by bwa index)"
    ref_bwt: "Reference genome binary file (created by bwa index)"
    ref_pac: "Reference genome binary file (created by bwa index)"
    ref_sa: "Reference genome binary file (created by bwa index)"
  }
# End workflow
}

task BwaMem {
  input {
    File input_fastq
    File ref_fasta
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
    Int threads = 16
  }
  
  String base_file_name = basename(input_fastq, ".fastq")
  String ref_fasta_local = basename(ref_fasta)

  command <<<
    set -eo pipefail

    mv "~{ref_fasta}" .
    mv "~{ref_fasta_index}" .
    mv "~{ref_dict}" .
    mv "~{ref_amb}" .
    mv "~{ref_ann}" .
    mv "~{ref_bwt}" .
    mv "~{ref_pac}" .
    mv "~{ref_sa}" .

    bwa mem \
      -p -v 3 -t ~{threads} -M -R '@RG\tID:foo\tSM:foo2' \
      "~{ref_fasta_local}" "~{input_fastq}" > "~{base_file_name}.sam"
    samtools view -1bS -@ 15 -o "~{base_file_name}.aligned.bam" "~{base_file_name}.sam"
    samtools sort -n -@ 15 -o "~{base_file_name}.sorted_query_aligned.bam" "~{base_file_name}.aligned.bam"

  >>>
  output {
    File analysisReadyBam = "~{base_file_name}.aligned.bam"
    File analysisReadySorted = "~{base_file_name}.sorted_query_aligned.bam"
  }
  runtime {
    memory: "48 GB"
    cpu: 16
    docker: "fredhutch/bwa:0.7.17"
    disks: "local-disk 100 SSD"
  }
}
```

## Testing your first task

To test your first task and your workflow, you should have expectation of output is. For this first `BwaMem` task, we just care that the BAM file is created with aligned reads. You can use `samtools view output.sorted_query_aligned.bam` to examine the reads and pipe it to wordcount `wc` to get the number of total reads. This number should be almost identical as the number of reads from your input FASTQ file if you run `wc input.fastq`. In other tasks, we might have a more precise expectation of what the output file should be, such as containing the specific somatic mutation call that we have curated.
