---
layout: post
title: "Kaggle and machine learning flow with Luigi"
date:   2016-10-13 16:48:00
---

Introduction
------------

I used to have a lot of trouble keeping track of code when I was doing Kaggle competitions. My workflow was simple:

1. Write code in a notebook
2. Execute it
3. _while I wait for results, fiddle with the code_
4. See that the run from step 2 went really well
5. Realize I couldn't remember what changes I'd made :(

I also had a lot of problems with larger-dataset kaggle comps, where I wanted to store intermediate results: I found I'd sometimes leave over old files, or not know how to re-create intermediate results from the past.

Luigi to the rescue
-------------------

[Luigi](https://luigi.readthedocs.io/en/stable/) is a spotify tool for batch processing jobs. It lets you declare dependencies and outputs and so on. The things I like about it:

1. Really nice handling of task prerequisites
2. Parallel processing
3. All in python, so I can access jobs from other jobs.

Common workflows
----------------
Some common problems I solve:

 - Fetch data from S3. Great for AWS spot workers, because it automates the otherwise tedious s3 fetch
 - Splitting train and test sets, and writing them into cache files. Particularly useful for...
 - Making multiple features in different files with the same test and train sets. It's very important to build different features from the same test and train sets, otherwise data leakage will happen. With luigi I'll just have a job that splits the train and test set and saves them as CSV files, then I can use those files as inputs to the various feature jobs. Note that if you do this, be sure to set a stable RNG seed so your task is repeatable.
 - Running multiple feature generators in parallel. Super easy, just set the `--workers=n` option for Luigi.

Quick tips
----------

 - `--local-scheduler` means you don't need to run the luigi server.
 - make the RNG seed an argument to each task, but one with a sane default. Now all your work will be totally repeatable. (perhaps pull the default from a config file, so you can change all of them at once).
 - A luigi task should produce only one official result file. Therefore, if I have a task that produces multiple outputs, I'll produce a single output file as the "result" at the end. To make loading things a little simpler, I'll create task classes that also include loading, for example:

{%highlight python %}
class Task(luigi.Task):
   def output(self):
      return luigi.LocalTarget('./done')
   def run(self):
      # ...
      with open('result1.csv') as f:
          pass
      with open('result2.csv') as f:
          pass
      with self.output().open('w') as f:
            f.write('Done!')

   def load1(self):
      return pandas.read_csv('./result1.csv')
   def load2(self):
      return pandas.read_csv('./result2.csv')
{%endhighlight%}

If you want to see an example project, I'm currently doing the *outbrain* click prediction kaggle competition. My code [is available](https://github.com/ririw/outbrain), and uses luigi for most of the task management.
