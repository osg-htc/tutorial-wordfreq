---
ospool:
    path: software_examples/python/tutorial-wordfreq/README.md
---

# A Wordcount Tutorial for Submitting Multiple Jobs

## Introduction

This tutorial shows you how to submit multiple jobs from a single submit file. This enables you to submit your workload without having to create one submit file per task. Use this guide as an introduction to learning the different ways to submit multiple jobs.

## Our list of tasks

Imagine you want to analyze how word usage varies from book to book or author to author over a collection of books. The type of workflow covered in this tutorial can be used to describe workflows that take have different input files or parameters from job to job.

1. Log in to your OSPool Access Point.
1. Download the materials for this tutorial using the following command:

```
git clone https://github.com/osg-htc/tutorial-wordfreq
```

## Submit a single task: analyze one book

### Test the command

Let's test our analysis. We can analyze the book `Alice_in_Wonderland.txt` by running the `wordcount.py` script: 

```
./wordcount.py Alice_in_Wonderland.txt
```

Run the `ls` command. You should see a new file, `counts.Alice_in_Wonderland.tsv`, which has the results of this python script. This is the output we want, and the output we expect HTCondor to return to us when our job finishes. For now, remove the output: 

```
rm counts.Alice_in_Wonderland.tsv
```

### Create a submit file

Let's translate this analysis into a form that HTCondor understands, so it can run the analysis for you. We describe this analysis in the form of a submit file.

Two key components of our analysis are

1. The command (`./wordcount.py`)
2. The input file(s) (`Alice_in_Wonderland.txt`)

In HTCondor's submit file syntax, we describe these with the `executable` and `arguments` options:

```
executable = wordcount.py
arguments = Alice_in_Wonderland.txt	
```

The `executable` is the script that we want to run, and the `arguments` is 
everything else that follows the script when we run it, like the test above.

The input file for this job is the `Alice_in_Wonderland.txt` 
text file. While we provided the name as in the `arguments`, we *also* need
to explicitly tell HTCondor to transfer the corresponding file.
We include the file name in the following submit file option: 

```
transfer_input_files = Alice_in_Wonderland.txt
```

There are other submit file options that control other aspects of the job, like 
where to save error and logging information, and how many resources to request per 
job.

This tutorial has a sample submit file (`wordcount.sub`) with these submit file options filled in: 

```
executable = wordcount.py
arguments = Alice_in_Wonderland.txt

log = log/job.$(Cluster).$(Process).log
error = err/job.$(Cluster).$(Process).err
output = out/job.$(Cluster).$(Process).out

transfer_input_files = Alice_in_Wonderland.txt

request_cpus   = 1
request_memory = 1GB
request_disk   = 1GB

queue 1
``` 

Confirm the contents of the submit file.

```
cat wordcount.sub
```

We will now submit our one-book analysis to the OSPool using HTCondor.

### Submit and monitor the job

After confirming the submit file, submit the job: 

```
condor_submit wordcount.sub
```

HTCondor will then print out a unique job ID.

Check the job's progress using `condor_q`.

```
condor_q
```

You can also use the command `condor_watch_q` to monitor the
queue in real time (use the keyboard shortcut `Ctrl` + `C` to exit).

Once the job finishes, you should see the same `counts.Alice_in_Wonderland.tsv` output when you enter `ls`.

## Analyze multiple books

Now suppose you want to analyze multiple books - more than one at a time. 
You could create a separate submit file for each book, and submit all of the
files manually, but you'd have a lot of file lines to modify each time
(specifically, the `arguments` and `transfer_input_files` lines from the 
previous submit file). 

This would be tedious! HTCondor has options that make it easy to 
submit many jobs from one submit file. 

### Make a list

HTCondor can loop through a list and submit one job per line of that list - this is perfect for running the same analysis for different books!

First, let's make a list of items that are different for each task we want to run.

In this example, the main difference is the books we want to analyze. So, our list should contain the names of these books.

We can easily create this list by using an `ls` command and sending the output to a text file that we'll call `book.list`: 

```
ls *.txt > book.list
```

The `book.list` file now contains each of the `.txt` file names in the current directory.

```
cat book.list
```

returns

```
Alice_in_Wonderland.txt
Dracula.txt
Huckleberry_Finn.txt
Pride_and_Prejudice.txt
Ulysses.txt
```

### Modify the submit file

Next, we will make changes to our submit file so that it submits a job for 
each book title in our list (seen in the `book.list` file). 

Create a copy of our existing submit file, which we will use for this job submission. 

```
cp wordcount.sub many-wordcount.sub
```

We need to tell HTCondor which list to loop over, and what variable we want to assign that value. Open the `many-wordcount.sub` file with a text editor (i.e., `vi`, `nano`) and go to the end.

```
queue book from book.list 
```

This statement works like a `for` loop:

1. HTcondor looks for the text file, `book.list`
1. HTCondor looks at the first line of `book.list` and assigns that value to the variable `book`.
1. In the submit file, every occurrence of `$(book)` is replaced by the value assigned to `book`.
1. HTCondor submits one job.
1. HTCondor then reads the second line of `book.list`, and repeats steps 2-4 until it reaches the end of the list.

> The syntax `$(variablename)` represents a submit variable whose value
> will be substituted at the time of submission.

Now, let's edit the rest of the submit file. Replace the name of the book `Alice_in_Wonderland.txt` in our submit file with the variable `$(book)`.

So, the following lines in the submit file should be changed to use the variable `$(book)`: 

```
arguments = $(book)

transfer_input_files = $(book)
```

### Submit and monitor the Job

Let's submit all of our jobs. 

```
condor_submit many-wordcount.sub
```

This will now submit five jobs (one for each book on our list). Once all five 
have finished running, we should see five "counts" files, one for each book in the directory. 

If you don't see all five "counts" files, consider investigating the log files and see if
you can identify what caused that to happen.
