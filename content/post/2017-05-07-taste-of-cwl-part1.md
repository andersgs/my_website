+++
date = "2017-05-07T12:00:00"
draft = false
tags = ["CWL"]
title = "A first taste of the common workflow language, part 1."
math = true
summary = """
My early experiment with common workflow language.
"""
[header]
image = "logos/CWL_banner.svg"
caption = ""

+++

# Some motivation...

In a few days, I'll be attending a workshop on the 
[Common Workflow Language]((http://www.commonwl.org/)), organised by the
great folks at [Melbourne Bioinformatics]( http://www.melbournebioinformatics.org.au), 
and to be run by one of CWL's principal developers, Michael Crusoe
([@biocrusoe](https://twitter.com/biocrusoe)). In preparation for the workshop
I thought I would write a series of three posts to outline a bit of my
experience with CWL, and get all my questions for Michael ready!

# Outline

In this first post, I'll give a quick outline of why I got excited about
CWL, and introduce the `Command Line Tool Description`, the one of two
documents you can write with CWL. In the second post, I'll go over the 
`Workflow Description` document, and then in the third post, I'll 
wrap up by giving a quick overview of the tools available to run 
CWL flows, and finish with my list of what I really like 
about CWL and what I think was not quite as intuitive as I had hoped.

# Getting excited about CWL

If you go to the CWL webpage you are greated, right at the top, by these two 
sentences:

{{% alert success %}}
The Common Workflow Language (CWL) is a specification for describing analysis workflows and tools in a way that makes them portable and scalable across a variety of software and hardware environments, from workstations to cluster, cloud, and high performance computing (HPC) environments. CWL is designed to meet the needs of data-intensive science, such as Bioinformatics, Medical Imaging, Astronomy, Physics, and Chemistry.
{{% /alert %}}

When I first read that a couple of months ago, I found it hard not to get 
very excited with these promises. The idea of 
writing a single workflow that can be easily scaled up from a single
machine to the cloud, and beyond, is very appealing! `And, it supports Docker
containers!`. I then decided to try it out 
with a real problem that I had at the time: I needed to estimate limit of 
detection of *in silico* MLST for NATA accreditation. At MDU, we use 
Torsten Seemann's ([The Genome Factory](https://thegenomefactory.blogspot.com.au/))
[MLST](https://github.com/tseemann/mlst) program, which requires an assembly.
My workflow would then consist of taking PE reads as input, sub-sampling 
them to a certain depth of coverage, assembling, running MLST, and collating
the data into a single table to be easily imported into `R`. Simple, right? 
Almost, there were a few hitches along the way, but I was still impressed 
with CWL.

{{% alert warning %}}
One important thing to remember: CWL is the spec! It is designed to be `vendor`
agnostic (a *lingua franca* or trading langauge for workflows). So, it is 
important to distinguish between perceived CWL shortcommings, and
shortcommings of any particular `vendor` that advertises CWL-support.
{{% /alert %}}

# The CWL basics

Given my experience with `make` I made the mistake of making comparisons with it, 
and had the wrong expectations. 

{{% alert info %}}
My advise is, if you have used `make`, keep an open mind while dealving into CWL.
While there are some similarities, implementation differs.
{{% /alert %}}

The general idea of CWL is that you write your workflow as a YAML or JSON 
file. These files can then be read and executed by any number of 
different software (called `vendors`). That is what makes CWL portable.
It provides a set of rules of how to write these YAML files that 
can be expected to be respected by any piece of software that claims to 
supoort CWL.

There are two kind of documents you can write: `Command Line Tool
Description`, and a `Workflow Description`. In this post, we will look 
over `Command Line Tool Description`.

## An example Command Line Tool Description

The first step of my limit of detection pipeline was to sub-sample reads
from a pair of FASTQ files. In this post, and the next, I will show you
how I did this in `CWL` using Heng Li's `seqtk`. 

The `Command Line Tool Description` is a YAML document that describes how
to build a command line string to run a certain command, and what to do
with the output of the command. To get started, let us have a look a the
description for `seqtk sample`:

```
Usage:   seqtk sample [-2] [-s seed=11] <in.fa> <frac>|<number>

Options: -s INT       RNG seed [11]
         -2           2-pass mode: twice as slow but with much reduced memory
```

By this description, `seqtk sample` needs an input sequence file (that while 
is implied to be FASTA, it will also accept FASTQ); a fraction or number of
reads to keep (in my case, I used number of reads), and then it can optionally
take a `seed` to set the random number generator, and a flag to indicate
whether to use a `2-pass mode` or not. I chose not to use the `2-pass mode`, 
so will ignore that option below. However, the `seed` option was essential
to work with PE reads. By giving the pair of sequences the same seed, I 
can guarantee that the same sequenced fragments will be selected from the
two files. What is not explicit in the description is that the output is 
streamed to `stdout`.

So, I am looking to build the equivalent of:

```
seqtk sample -s 6773 seq_R1.fq 1000 > seq_10000_R1.fq
```

Here is how we achieve it using the CLI tool description in CWL (below
I go over each line in detail):

```yaml
cwlVersion: v1.0
class: CommandLineTool
doc: >
  A command line tool to use seqtk sample to sub-sample N reads from a single
  FASTQ file. The use of the seed ensures that pairs of reads have the same
  records sampled, and ensure reproducibility of the pipeline by others.
baseCommand: ['seqtk', 'sample']
inputs:
    seed:
        type: int
        inputBinding:
            prefix: -s
            position: 1
        doc: The seed needed to ensure the same records are kept across PE files
    fastq:
        type: File
        inputBinding:
            position: 2
        doc: A single FASTQ input file
    number:
        type: int
        inputBinding:
            position: 3
        doc: An integer specifying how many records to keep
    seqid: 
        type: string
        doc: >
          A string to be used in the output name (notice no inputBinding) to 
          indicate the ID of the sample.
    read_number: 
        type: int
        doc: >
          An int to be used in the output name (notice no inputBinding) that 
          indicates if it is READ1 or READ2 in a pair
    rep: 
        type: int
        doc: >
          An int to be used in the output name (notice no inputBinding) to 
          indicate the replicate sub-sample number
outputs:
    seqtkout:
        type: stdout
stdout: $(inputs.seqid)___$(inputs.number)-$(inputs.seed)-$(inputs.rep)___R$(inputs.read_number).fq
```

The first two lines are self-evident. You must specify the version of the 
CWL specification that you are using, and that this document specifies a 
`Command Line Tool`. The `baseCommand` specifies the executable that 
should be called, in this case I gave it an array with the main executable
(`seqtk`), and the sub-command (`sample`). But, a single string with a 
command can be used when appropriate (e.g., `baseCommand: mlst`).

The `doc` strings can be used throughout the document to describe what you are
doing and why! I really like this feature, but I suspect it will seldomly be 
actually used, which is a pity.

Next I have listed the inputs to the tool under `inputs`.
In `inputs`, you can list each input the command-line tool requires, and under
each input, you need to specify the `type`, in this case there are two
examples `File` and `int`. Other types include `string`, `array`, 
`Directory`, or custom types, which I'll demonstrate in the next post. 

For those inputs that will be used to form the command string that will 
be run, you must specify the `inputBindings`. These describe any 
`prefixes` the option should have while building the command string, 
the position of each option, if there should be a space between the `prefix`
and the `value`, etc.

The `outputs` section works in a similar way. The name of a particular input
here could be used as an `input` in another tool downstream. This helps setup
`make`-like dependencies among tools in a pipeline. One cool thing about `CWL`
is that it is able to capture `stdout` of a tool, and push it to a file.

If capturing the tools `stdout`, you need to specify a filename 
under the `stdout` key. This could be a simple string (e.g., `myout.txt`), 
or it can be an expression, as I have used here. In this case, I used the
`$()` notation to access elements of the `inputs` object. The `$()` gives you 
access to a limited set of javascript notation to access different objects.
You can use it to access data in `inputs`, `self`, and `runtime` objects. 
In this case, I use it to create a filename that has the following pattern:

```
mySeqId___1000-899666-3___R1.fq
```

The values for each parameter are listed under the `inputs` section!

{{% alert info %}}
CWL lets you use a `$()` to access parameter values within different
objects. Check out the
[documentation](http://www.commonwl.org/v1.0/Workflow.html) for description
of the objects you can access automatically.
{{% /alert %}}

# That is it for now...

That is all you really need to know to get started writing your own
`Command Line Tool Descriptions`. It is fairly simple, and I fount it to be
highly readable. If you want some more examples, check out the 
[Wrapping Command Line Tools](http://www.commonwl.org/v1.0/UserGuide.html#Wrapping_Command_Line_Tools) 
section of the Gentle Introduction to CWL.

As I said before, in part 2 of this post I'll go over the basic 
of writing a `Workflow Description` to produce untold number of
sub-sampled reads that use the above CLI description as a building
block.