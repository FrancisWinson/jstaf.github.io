---
title: Snakemake - making your work do itself
---

I love Snakemake. 
It's pretty much the greatest thing since sliced bread.
If I were to hypothetically start writing a blog on the internet, it would probably be the first thing I posted about.

So what is it? (and more importantly, why should we care?)

Chances are, you've sat in front of a computer at some point and had to do a repetitive, boring task.
Wouldn't it be nice if we could just have our computer do our work for us? 
That's what Snakemake is for.
Snakemake is a pipelining tool that automates and parallelizes any command-line workflow or analysis. 
It is *hands down* the best tool in the industry. And it's free.

You might have encountered several similar tools out there - what makes Snakemake so much better?

1. **You can learn it in 30 minutes.** Snakemake is just Python. If you know *any* Python (like at all), you already know Snakemake. No weird syntax.
2. **It is completely OS/software stack/hardware independent.** It even works on Solaris (nothing works on Solaris).
3. **Scaling is effortless.** The same workflow can be ported from a Windows laptop to a Linux supercomputing cluster with no modification. All workflows are parallel by default.
4. **Handles arbitrary resource requirements.** Does a particular step in your workflow need a weird compute resource like a GPU or database lock? No problem- just make up whatever resources you need and tell Snakemake how many of them you have.
5. **Good logging.** Want a pretty flow-chart to show your boss? Did your analysis completely eat it halfway through and you want to know why? Snakemake's logging and workflow plots make it easy to figure out what's going on.
6. **Installs in seconds.** No configuration/root privileges/etc. required. "pip install --user snakemake" ... done. 
7. **It's just Python.** If you can do it in Python you can do it in Snakemake (no questions asked). 

---------------------------------------------------
## Learning by example
<br/>
Although I've already mentioned that Snakemake is great for automating data analysis, it never really "clicks" with people until they see it in action.
So for the rest of this post, we are going to go through an example workflow.
This particular example involves bioinformatics, but if you don't understand the science, that's totally fine. 
Just think of it as us starting with a set of input files, and we need to run several steps of analysis to get to our output. 

### Here's the problem:

I have 6 samples' worth of paired-end RNA sequencing reads from the mosquito *Aedes aegypi* between two conditions: fed and starved.
We want to compute differential expression between these two conditions.
Rephrased in english, I want to know which genes are turned "on" or "off" when our moquitoes are hungry. Yum!

Fortunately, there's already a ton of tools out there that can do all of our analysis for us!
In this case, we are going to visualize our sample QC data with [FastQC](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/)+[MultiQC](http://multiqc.info), align and quantify our reads with [Kallisto](https://pachterlab.github.io/kallisto/), then calculate differential expression with [edgeR](https://bioconductor.org/packages/release/bioc/html/edgeR.html).
Why did I pick these tools? 
I'll be honest - it runs the analysis really fast/easily and fits nicely within a blog post.
It just also happens to give us valid results.

Here is a picture of what we need to do:

INSERT DAG HERE

-----------------------------------------------------
## Snakemake workflows
<br/>
A Snakemake pipeline/workflow/analysis/what-have-you is just a file called "Snakefile".
The Snakefile itself contains a set of "rules".
A rule is a recipe.
It explains how to turn a set of input files in to an output file(s).

Again, this is best visualized as an example. 
Let's start by running FastQC on one of my FASTQ files (read: starting data).

* I have a file called in my `fastq` folder called `Control2_GGCTAC_R1.fastq.gz`
* To get my results, I need to run `fastqc -o qc Control2_GGCTAC_R1.fastq.gz`
* This command results in a file in a `qc` folder called `Control2_GGCTAC_R1_fastqc.html`

So we have an input file, a command we want to run on it, and an output filename.
This is all we need to make a rule. Let's add this to our Snakefile:

{% highlight python %}
rule fastqc:
	input:	'fastq/Control2_GGCTAC_R1.fastq.gz'
	output:	'qc/Control2_GGCTAC_R1_fastqc.html'
	shell:	'fastqc -o qc Control2_GGCTAC_R1.fastq.gz'
{% endhighlight %}

We can then "make" our output file by typing `snakemake Control2_GGCTAC_R1_fastqc.html`. We should see the following:

{% highlight bash %}
Provided cores: 1
Rules claiming more threads will be scaled down.
Job counts:
	count	jobs
	1	fastqc
	1

rule fastqc:
    input: fastq/Control2_GGCTAC_R1.fastq.gz
    output: qc/Control2_GGCTAC_R1_fastqc.html
    jobid: 0

Started analysis of Control2_GGCTAC_R1.fastq.gz
Approx 5% complete for Control2_GGCTAC_R1.fastq.gz
Approx 10% complete for Control2_GGCTAC_R1.fastq.gz

... more stuff ...

Analysis complete for Control2_GGCTAC_R1.fastq.gz
Finished job 0.
1 of 1 steps (100%) done
{% endhighlight %}

Nice... it worked! Let's create a rule to delete all of our hard-earned work so we can redo things more efficiently.
This one has no input or output, it just runs the command `rm -rf qc` (delete the "qc" folder with extreme prejudice).

{% highlight python %}
rule clean:
	shell: 'rm -rf qc`
{% endhighlight %}

Now if we run `snakemake clean` (you'll notice I used the name of the rule here, not the output filename), it nukes our `qc` folder. Nice. Let's redo our last step, just... smarter.

### Wildcards

Snakemake lets us define "wildcards", or placeholders for filenames.
Wildcards let us run the same command on multiple files.
A wildcard can be named anything, and simply looks like `{name_of_wildcard}`.
Let's rewrite our "fastq" rule to use wildcards:

{% highlight python %}
rule fastqc:
	input:	'fastq/{sample}.fastq.gz'
	output:	'qc/{sample}_fastqc.html'
	shell:	'fastqc -o qc {input}'
{% endhighlight %}

This rule uses two wildcards: `{sample}` and `{input}`.
For input and output, `{sample}` takes the place of the sample name, minus the directory name and file extension.
The "shell" part of our command uses `{input}` (which is equivalent to the entire "input:" part of our rule).
Let's try running our original command again with `snakemake Control2_GGCTAC_R1_fastqc` (it works!).

Now, what if we wanted to run our "fastqc" rule on all of our FASTQ files, not just the one.
As it happens, I can make another rule (I'm going to call it "multiqc") that expects all of our FastQC samples as input.
But how do we turn this into a Snakemake rule?

My `fastq` folder looks like the following:

{% highlight bash %}
$ ls fastq
Control2_GGCTAC_R1.fastq.gz  Starved2_GATCAG_R1.fastq.gz
Control2_GGCTAC_R2.fastq.gz  Starved2_GATCAG_R2.fastq.gz
Control4_CTTGTA_R1.fastq.gz  Starved3_AGTTCC_R1.fastq.gz
Control4_CTTGTA_R2.fastq.gz  Starved3_AGTTCC_R2.fastq.gz
Control5_AGTCAA_R1.fastq.gz  Starved4_TAGCTT_R1.fastq.gz
Control5_AGTCAA_R2.fastq.gz  Starved4_TAGCTT_R2.fastq.gz
{% endhighlight %}



