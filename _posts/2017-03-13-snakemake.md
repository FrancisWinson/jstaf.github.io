---
title: Snakemake - making your work do itself
---

I love Snakemake. 
It's pretty much the greatest thing since sliced bread.
If I were to start writing a blog on the internet, it would probably be the first thing I posted about.

So what is it? (and more importantly, why should we care?)

Chances are, you've sat in front of a computer at some point and had to do a repetitive, boring task.
Why can't our computer just know what we want to do, and do it for us?
Wouldn't it be nice if we could automate our work? 
That's what Snakemake is for.
Snakemake is a pipelining tool that automates and parallelizes any command-line workflow or analysis. 
It is *hands down* the best tool in the industry. And it's free.

You might have encountered several similar tools out there (most notably GNU Make), so what make Snakemake so much better?

1. **You can learn it in 30 minutes.** Snakemake is just Python. If you know *any* Python (like at all), you already know Snakemake. No weird syntax.
2. **It is completely OS/software stack/hardware independent.** It even works on Solaris (nothing works on Solaris).
3. **Scaling is effortless.** The same workflow can be ported from a Windows laptop to a Linux supercomputing cluster with no modification. All workflows are parallel by default.
4. **Easily handles arbitrary resource requirements.** Does a particular step in your workflow need a dedicated resource like a GPU or database lock? No problem- you can make up whatever resources you need, specify how many of said resource you have available, and Snakemake will parallelize the load in such a way to maximize throughput.
5. **Good logging.** Want a pretty flow-chart to show your boss? Did your analysis completely eat it halfway through and you want to know why? Snakemake's logging and workflow plots make it easy to figure out what's going on.
6. **Installs in seconds.** No configuration/root privileges/etc. required. "pip install --user snakemake" ... done. 
7. **It's just Python.** I already said this one, but this is such a major advantage I'm saying it again. If you can do it in Python you can do it in Snakemake (no questions asked). 

---------------------------------------------------
## An example workflow

So, if you've made it this far, you're at least interested. Maybe.
So how does it work?

Although I've already mentioned that Snakemake is great for automating data analysis, it never really "clicks" with people until they see it in action.
So for the rest of this post, we are going to go through an example workflow.
This particular example involves bioinformatics, but if you don't understand the science, that's totally fine. 
The analysis itself does not matter.
Just think of it as us starting with a set of input files, and we need to run several steps of analysis to get to our output. 

Here's the problem:

I have 12 FASTQ files (paired-end reads, 2 files per sample) containing RNA-Seq sequencing reads from the mosquito *Aedes aegypi* between two conditions: fed and starved 
We want to compute differential expression between these two conditions.
Rephrased in english, I want to know which genes are turned "on" or "off" when our moquitoes are hungry. Yum!

Fortunately, there's already a ton of tools out there that can do all of our analysis for us!
We're going to use the ["new Tuxedo Suite"](http://www.nature.com/nprot/journal/v11/n9/full/nprot.2016.095.html) for this analysis.
For those not in the know, the "Tuxedo Suite" is a pretty famous/infamous set of tools for analyzing this type of data (this is the new version that's apparently a lot faster/better).
We're using it here because it's a good example of starting with a set of input files, doing several steps of analysis, and ending with a nice set of outputs.

Here is a picture of what we need to do for each of our samples:

<img src="img/snakemake/new-tuxedo.jpg" width="400">

-----------------------------------------------------
## Snakemake workflows

A Snakemake pipeline/workflow/analysis/what-have-you is just a file called "Snakefile".
The Snakefile itself contains a set of "rules".
A rule is a recipe.
It explains how to turn a set of input files in to an output file(s).

Let's look at an example:



