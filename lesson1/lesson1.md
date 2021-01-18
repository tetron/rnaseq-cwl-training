# Turning a bash script into a workflow using existing tools

In this lesson we will turn `rnaseq_analysis_on_input_file.sh` into a workflow.

# Setting up

We will create a new git repository and import a library of existing
tool definitions that will help us build our workflow.

1. Select "Terminal->New terminal"

2. Create a new git repository to hold our workflow with this command:

## Arvados

```
git clone https://github.com/arvados/arvados-vscode-cwl-template.git rnaseq-cwl-training-exercises
```

## Generic

```
git init rnaseq-cwl-training-exercises
```


3. Go to File->Open Folder and select rnaseq-cwl-training-exercises

4. Go to the terminal window

5. Import bio-cwl-tools with this command:

```
git submodule add https://github.com/common-workflow-library/bio-cwl-tools.git
```

# Writing the workflow

1. Create a new file "main.cwl"

2. Start with this header.


```
cwlVersion: v1.2
class: Workflow
label: RNAseq CWL practice workflow
```

3. Workflow Inputs

The purpose of a workflow is to consume some input parameters, run a
series of steps, and produce output values.

For this analysis, the input parameters are the fastq file and the reference data required by STAR.

In CWL, these are declared in the `inputs` section.

The inputs section lists each input parameter and its type.  Valid
types include `File`, `Directory`, `string`, `boolean`, `int`, and
`float`.

In this case, the fastq and gene annotation file are individual files.  The STAR index is a directory.  We can describe these inputs in CWL like this:

```
inputs:
  fq: File
  genome: Directory
  gtf: File
```

4. Workflow Steps

A workflow consists of one or more steps.  This is the `steps` section.

Now we need to describe the first step of the workflow.  This step is to run `fastqc`.

A workflow step consists of the name of the step, the tool to `run`,
the input parameters to be passed to the tool in `in`, and the output
parameters expected from the tool in `out`.

The value of `run` references the tool file.  Tip: while typing the
file name, you can get suggestions and auto-completion on a partial
name using control+space.

The `in` block lists input parameters to the tool and the workflow
parameters that will be assigned to those inputs.

The `out` block lists output parameters to the tool that are used
later in the workflow.

You need to know which input and output parameters are available for
each tool.  In vscode, click on the value of `run` and select "Go to
definition" to open the tool file.  Look for the `inputs` and
`outputs` sections of the tool file to find out what parameters are
defined.

```
steps:
  fastqc:
    run: bio-cwl-tools/fastqc/fastqc_2.cwl
    in:
	  reads_file: fq
    out: [html_file, summary_file]
```

5. Running alignment with STAR

STAR has more parameters.  Sometimes we want to provide input values
to a step without making them as workflow-level inputs.  We can do
this with `{default: N}`


```
  STAR:
    requirements:
      ResourceRequirement:
        ramMin: 6000
    run: bio-cwl-tools/STAR/STAR-Align.cwl
    in:
      RunThreadN: {default: 4}
      GenomeDir: genome
      ForwardReads: fq
      OutSAMtype: {default: BAM}
      OutSAMunmapped: {default: Within}
    out: [alignment]
```

6. Running samtools

The third step is to generate an index for the aligned BAM.

For this step, we need to use the output of a previous step as input
to this step.  We refer the output of a step by with name of the step
(STAR), a slash, and the name of the output parameter (alignment), e.g. `STAR/alignment`

This creates a dependency between steps.  This means the `samtools`
step will not run until the `STAR` step has completed successfully.

```
  samtools:
    run: bio-cwl-tools/samtools/samtools_index.cwl
    in:
      bam_sorted: STAR/alignment
    out: [bam_sorted_indexed]
```

7. featureCounts

As of this writing, the `subread` package that provides
`featureCounts` is not available in bio-cwl-tools (and if it has been
added since writing this, let's pretend that it isn't there.)  We will
dive into how to write a CWL wrapper for a command line tool in
lesson 2.  For now, we will leave off the final step.

8. Workflow Outputs

The last thing to do is declare the workflow outputs in the `outputs` section.

For each output, we need to declare the type of output, and what
parameter has the output value.

Output types are the same as input types, valid types include `File`,
`Directory`, `string`, `boolean`, `int`, and `float`.

The `outputSource` field refers the a step output in the same way that
the `in` block does, the name of the step, a slash, and the name of
the output parameter.

For our final outputs, we want the results from fastqc and the
aligned, sorted and indexed BAM file.

```
outputs:
  qc_html:
    type: File
    outputSource: fastqc/html_file
  qc_summary:
    type: File
    outputSource: fastqc/summary_file
  bam_sorted_indexed:
    type: File
    outputSource: samtools/bam_sorted_indexed
```