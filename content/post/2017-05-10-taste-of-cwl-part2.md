+++
date = "2017-05-10T12:00:00"
draft = false
tags = ["CWL"]
title = "A first taste of the common workflow language, part 2."
math = true
summary = """
My early experiment with common workflow language.
"""
[header]
image = "logos/CWL_banner.svg"
caption = ""

+++

# Welcome back to part 2 of my CWL series

In my previous [post]({{< ref "post/2017-05-07-taste-of-cwl-part1.md" >}}), I talked about some of things that got me excited about CWL, and
I introduced the `Command Line Tool Description`. Here, I will introduce the `Workflow Description`,
which uses the `Command Line Tool Description` as a building block. I will do this by continuing with my 
limit of detection problem. I am focusing on the first step of that problem, which was producing 
sub-sampled reads using Heng Li's `seqtk`. In particular, this workflow
will take an object that houses a pair of related FASTQ files, 
randomly sub-sample the files to several different total number of reads, with
each total number of reads retained replicated several times.


# An example Workflow description

As a quick recap, CWL provides specs for two kind of documents: `Command Line Tool Description`, 
and the `Workflow Description`. As suggested by the name, `Command Line Tool Description` documents
describe how to build a command line with some `inputs`, and describes how to handle the `output`. 
The `Workflow Description` describes the steps which connect `Command Line Tool Descriptions` in 
order to go from some raw `inputs` to the final `outputs` of interest.

The `Workflow Description` follows a similar pattern to that of the `Command Line Tool Description`, 
and is similarly readable, but in my example, it is perhaps a bit long. So, I'll break it down into parts, 
and go over it step-by-step. I decided to go with this example in spite of
its length because it is a real example of CWL in use. At the end, I'll point towards the GitHub repo where you can get the whole code.

## The general outline

The `Workflow Description` has three main sections: `header` section, an `IO` section, and a 
`steps` section. I'll go over each in turn.

## The header

First, we start with the header information:

```yaml
cwlVersion: v1.0
class: Workflow

requirements:
 - class: ScatterFeatureRequirement
 - class: InlineJavascriptRequirement
 - class: StepInputExpressionRequirement
 - $import: readPair.yml
 ```
 
 The first two lines are self-explanatory. You have to specify the 
 version of the spec, and that this document refers to a Workflow.
 
 The `requirements` outline some additional information that is required
 to run the Workflow. Three of the listed requirements are provided by 
 the spec: `ScatterFeatureRequirement`, `InlineJavascriptRequirement`, and
 `StepInputExpressionRequirement`. Their names already hint at what 
 functionality they enable, but I'll address them individually when I 
 get to the appropriate portion of the description. This is only a subset
 of the additional features you get out of the box. You can find the 
 description to others [here](http://www.commonwl.org/v1.0/Workflow.html#WorkflowStep).
 
 The last item in the requirements section imports a `readPair.yml` file. 
 Because my workflow works on pairs of FASTQ files, I needed to 
 create an object that kept the pairs of files together, and passed them
 along through the workflow.
 
 {{% alert info %}}
 While I am specifying the `requirements` at the start of my `Workflow Description`,
 they can be specified at each step individually, or within the individual
 `Command Line Tool Descriptions`. So, there is plenty of flexibility here. 
 I like to think of these as modules in `Python`, or libraries in `R`. They
 add functionality!
 {{% /alert %}}
 
 
## My custom object: `readPair.yml`
 
 Before we continue, I will digress and show you the `readPair.yml` file to 
 demonstrate the creation of a custom `type` of input. The motivation to 
 create one was bourne out of the necessity of keeping PE FASTQ files together
 while pushing them through the pipeline. It is an unfortunate standard, but 
 it is the standard at the moment:
 
 ```yaml
class: SchemaDefRequirement
types:
 - name: FilePair
   type: record
   fields:
     - name: forward
       type: File
     - name: reverse
       type: File
     - name: seed
       type: int[]
     - name: number
       type: int[]
     - name: rep
       type: int[]
     - name: seqid
       type: string
 ```

The `class` just tells CWL that what is coming next is the definition of 
a new `schema` or `type` of object. The `types` field allows you to 
specify more then one new object per file. For each new type, you 
need to specify a `name`, a `type` (which is always `record` for now, 
this may change in future versions of CWL), 
and then the `fields` that are the individual components of the object. 
Each field has a `name`, and 
`type`, which are usually the base types (e.g., `int`, `string`, `array`).

Note that I am using some syntactic sugar here to define `int` arrays: `int[]`.
That notation can be used with any of the defined types. So, after defining 
a `FilePair` type, I can use `FilePair[]` to specify an array of `FilePair`s. 

In short, my `FilePair` object has two `File`s: `forward` and `reverse`; it has
an array of `int`s that set the seeds to be used with `seqtk sample`; it has
an array of `int`s that have the `number` of reads I want to save, an array
of `int`s for the `rep`lication number, and a `string` for the sequence ID. 
This object describes all the information needed to process a single pair of isolates. Each
array is as long as the number of sub-samples I will produce for that pair. For example, if
I wanted two replicates each of 10,000 and 50,000 reads (a total of four sets of sub-sampled reads), the
relevant section of my input file would look like this:

```yaml
     seed: [42, 52, 62, 72]
     number: [10000, 10000, 50000, 50000]
     rep: [1, 2, 1, 2]
```

So, one run of `seqtk sample` would use `seed` = 42; `number` = 10000, and `rep` = 1, 
and so forth. This arrangement is what makes it possible for me to take advantage
of the `ScatterFeatureRequirement` requirement named in the header, which I will 
discuss below. 


{{% alert success %}}
Before we move on, I would just like to acknowledge that inspiration for
my `readPair.yml` came from [h3abionet16S analysis package](https://github.com/h3abionet/h3abionet16S).
The project describes quite an extensive CWL flow to deal with 16S data, and
it a good place to get some examples and inspiration. And, of course, a CWL flow 
for 16S data.
{{% /alert %}}

## The IO section

The IO section is not formally called the IO section anywhere. I just
called it that because it specifies the `inputs` and `outputs` of the
overall workflow.

Below, is the IO section of my `seqtk sample PE` workflow. 

```yaml
inputs:
    forward: File
    reverse: File
    seqid: string
    seed: int[]
    number: int[]
    rep: int[]

outputs:
    resampled_fastq:
        type: "readPair.yml#FilePair"
        outputSource: collect_output/fastq_pair_out
```

As you can see, there are two parts. In the `inputs` section, 
you specify all the inputs you will need to 
start your workflow, or that will be needed during the workflow but
are not generated as part of one of the intermediary steps (e.g., 
the path to a DB you might use for an intermediary BLAST). The
elements of the `inputs` have the same name of the 
elements in the in the `FilePair` object created in the 
`readPair.yml` file. That is how you connect information in your input 
file to the `Workflow`.

In the `outputs` section, we specify what the `Workflow` will
output. In this case, there is a single output, named `resampled_fastq`.
It is of type `FilePair`. Here, a special notation is used that 
identifies the particular schema we want from the `readPair.yml` file
by using the `#`. A deeper explanation of this notation can be
found in the [`Schema SALAD specification`](http://www.commonwl.org/v1.0/SchemaSalad.html)
that underlies the whole object notation used in CWL.

Finally, we specify the `source` of the output with a notation
`step_name/step_output`. This output will come from the `collect_output`
step, specifically, the `fastq_pair_out` output from that step.

## The steps section

The `steps` section outlines the steps that need to be taken to 
go from the `inputs` to the `outputs` specified in the IO section. In
my `seqtk PE sample` workflow, there are three steps: `subsample_1`, 
`subsample_2`, and `collect_output`. Because `seqtk sample` only 
takes one file at a time, we need to run `seqtk sample` twice, one
for each file in the pair, with the same `seed`. The first two 
steps take care of sub-sampling the two files, the last step
collects the pairs of files, and rebuilds a `FilePair` object to
return. The reason I want to return a `FilePair` object from this workflow
is that this workflow is a sub-workflow of my larger limit of detection
workflow. The following step in that larger workflow is an assembly step, 
which takes as input PE FASTQ files. Returning a `FilePair` object is therefore
convenient for the next step.

Because steps 1 and 2 are essentially the same, I'll present 
both below, but only go over the first step.

```yaml
steps:
    subsample_1:
        in:
            fastq: forward
            seed: seed
            number: number
            rep: rep
            seqid: seqid
            read_number:
                valueFrom: ${return 1;}
        scatter: [seed, number, rep]
        scatterMethod: dotproduct
        out: [seqtkout]
        run: seqtk_sample.cwl
    subsample_2:
        in:
            fastq: reverse
            seed: seed
            number: number
            rep: rep
            seqid: seqid
            read_number:
                valueFrom: ${return 2;}
        scatter: [seed, number, rep]
        scatterMethod: dotproduct
        out: [seqtkout]
        run: seqtk_sample.cwl
```

For each `step`, some keywords are necessary: `in`, `out`, and `run`. 
The `run` key can take either the name of a file that 
contains a `Command Line Tool Description`, the name of a file 
with another `Workflow Description` (allowing you to nest workflows), 
or you could specify what to run within the document 
(i.e., I could just copy-paste the
`Command Line Tool Description` from the previous 
[post]({{< ref "post/2017-05-07-taste-of-cwl-part1.md" >}}) directly
here). This, however, is not desirable. By keeping the definitions
separate, I am free to use it in other workflows. Below, I'll give
a cool example where I run some custom JavaScript, and explicitly 
provide the definition of what to run rather than naming a file. 

The `in` and `out` sections define what goes in to the step, and 
what comes out. As might be expected in a workflow, there is a lot of IO that 
happens at different levels (e.g., at the step level, or at the workflow level, 
or at the command line tool level). Getting the plumbing right
is crucial for your `Workflow` to work properly. 

In the `in`, we use a series of `key`:`value` pairs. The `keys` are the values
expected by the tool being run in the current step. If you
look back at the previous
[post]({{< ref "post/2017-05-07-taste-of-cwl-part1.md" >}}) 
you will notice that the inputs for `seqtk_sample.cwl` are
named `fastq`, `seed`, `rep`, etc. Under `read_number` I
use the notation `${}` to write a very small JavaScript script
to return a default `int` value of `1`. In step 2, I use the same 
artifice to return a default `int` value of `2`. These make the `R1` and
`R2` bit of the sub-sampled PE filenames. I can use the notation `${}` to 
embedded JavaScript into my workflow because I added
the requirement `InlineJavascriptRequirement` to the header
section.

{{% alert info %}}
By including the `InlineJavascriptRequirement` it is possible 
to write JavaScript scripts using the `${}` notation.
{{% /alert %}}

The ability to take an input by using the keyword `valueFrom` is 
possible because I included the `StepInputExpressionRequirement` in 
the header section.

{{% alert info %}}
The `StepInputExpressionRequirement` allows one to use the keyword
`valueFrom` to set default values or use JavaScript expressions to 
change inputs depending on other runtime variables.
{{% /alert %}}

The `values` associated with each `key` either reference 
the `inputs` to the workflow directly
(those defined in the IO section), or outputs from other steps.
When referencing outputs from other steps, we use the same notation
used in the overall `outputs` section of `step_name/step_output`.
An example of this is in the third and final step of this `Workflow`. 
When using outputs from other steps as inputs, this creates a
dependency between the steps, and dictates in what order 
steps can be carried-out. If you have worked with `make`, this will sound
familiar.

The first two steps also include the following lines:

```yaml
        scatter: [seed, number, rep]
        scatterMethod: dotproduct
```

The `scatter` allows one to break up the input in to a number of 
smaller jobs. In this case, I am specifying that
`seed`, `number`, and `rep` need to be combined to form individual
jobs. The `scatterMethod` specifies how we combine them. 
In particular, the `dotproduct` method means that the first element of 
each of those arrays are joined to make a job, then the second
element of each array are joined to make another job, and so on. 
It is a prerequisite, therefore, that the arrays all be of the 
same length. Other methods include `nested_crossproduct` and 
`flat_crossproduct` - read more about
them [here](http://www.commonwl.org/v1.0/Workflow.html#WorkflowStep).
This is possible because I added the `ScatterFeatureRequirement` in
the header section.

{{% alert info %}}
The `ScatterFeatureRequirement` gives you the ability to apply or map
a command over input arrays. 
{{% /alert %}}

The third step (`collect_output`), provides two additional examples of
what is possible in CWL: (1) you can write the code for a step 
directly into the `Workflow Description`; (2) there is a third class of
object called `ExpressionTool` that allows one to use JavaScript to 
accomplish what is needed. This gives CWL great flexibility to 
build some complex workflows!

```yaml
    collect_output:
        run:
            class: ExpressionTool
            inputs:
                seq_1:
                    type:
                        type: array
                        items: File
                seq_2:
                    type:
                        type: array
                        items: File
                seqid: string
            outputs:
                fastq_pair_out: "readPair.yml#FilePair"
            expression: >
                ${
                    var ret=[];
                    for (var i = 0; i < inputs.seq_1.length; ++i) {
                        var tmp = {}
                        tmp['forward'] = inputs.seq_1[i];
                        tmp['reverse'] = inputs.seq_2[i];
                        tmp['seqid'] = inputs.seqid;
                        tmp['seed'] = [inputs.seed[i]];
                        tmp['number'] = [inputs.number[i]];
                        tmp['rep'] = [inputs.rep[i]];
                        ret.push(tmp);
                    }
                    return { 'fastq_pair_out' : ret }
                }
        in:
            seq_1: subsample_1/seqtkout
            seq_2: subsample_2/seqtkout
            seqid: seqid
            seed: seed
            number: number
            rep: rep
        out:
            [ fastq_pair_out ]
```

In this step, I use the `ExpressionTool` to collect the 
outputs from steps 1 and 2 and build `FilePair` objects, 
and generate an array of `FilePair` objects to be the final
output of this `Workflow`. 

The principles from the previous steps apply here too. The one
gotcha of writing the step in this way, is that it can be a bit
confusing with all the `inputs` and `outputs`. I should also mention again 
that this is probably not very desirable way of organising your code. If 
you would like to reuse this `ExpressionTool` in other workflows, you would 
be better off writing it out to a separate file, as we demonstrated in 
steps 1 and 2 of this workflow.

There are two things I want to call attention to here. First, to how `array`s are specified: 

```yaml
                seq_1:
                    type:
                        type: array
                        items: File
```

If you follow all the input plumbing, you should figure out that
`seq_1` is the array of `Files` outputted by step 1. But, 
remember that when writing the `inputs` section, we need to 
specify the `type` of each input. Because the elements of an 
`array` can be of many different types, we need to specify what 
is the type of the elements of this particular `array`. Another way 
of doing this would be to use the `File[]` notation. The reason for 
this odd notation (i.e., repetition of `type`) is because `array`s
are represented internally as `{"type": "array", "items": <T>}`.

The second point I wish to make relates to how `ExpressionTool`s are 
written. Note that there is **no** `baseCommand` keyword here, but rather the 
`expression` key is used. I also used the standard `YAML` notation `expression: >` to 
indicate that the `value` for that `key` will span multiple lines, and new lines
should be ignored (if new lines were important, then I would use the `expression: |`
notation). The actual expression presented here just loops over the 
arrays outputted by steps 1 and 2, and builds individual `FilePair` objects
for each of the sub-sampled pairs created at those steps. 

# Wrapping up...

That is it! You now should have the basics to build your own
workflows in CWL. What you need are individual `Command Line Tool Description`s 
or `ExpressionTools` that will form the building blocks for the steps of a workflow, 
and then put them together using the `Workflow Description`. You can nest sub-workflows into larger workflows
by making individual steps in a workflow call other `Workflow Description`s. 
The examples I have provided in these two posts are, for instance, only a small sub-workflow
of my larger limit of detection workflow. The full workflow code can 
be found on my `GitHub` in the [cwl_flows](https://github.com/andersgs/cwl_flows) repo (more 
detailed documentation still to be added).

I would like to finish this post by thanking Michael Crusoe
([@biocrusoe](https://twitter.com/biocrusoe)) for help setting up some of this
workflow (check out my question and answers on [BioStars](https://www.biostars.org/p/245032/)). 
This highlighted the incredible support available to build CWL flows currently 
available. I'll talk more about this in the next 
post. In the meantime, I highly recommend checking out the 
[BioStars](https://www.biostars.org/t/cwl/) CWL tag if you 
ever get stuck!

{{% alert info %}}
You can get great support with your workflows through the 
CWL tag on [BioStars](https://www.biostars.org/t/cwl/)
{{% /alert %}}