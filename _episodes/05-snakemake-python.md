---
title: "Snakefiles are Python code"
teaching: 30
exercises: 30
questions:
- "How can I automatically manage dependencies and outputs?"
- "How can I use Python code to add features to my pipeline?"
objectives:
- "Use Python variables, functions, and imports in a Snakefile."
- "Learn to use the `run` action to execute Python code as an action."
keypoints:
- "Snakefiles are Python code."
- "The entire Snakefile is executed whenever you run `snakemake`."
- "All actual work should be done by rules."
- "A `shell` action executes a command-line instruction."
- "A `run` action executes Python code."
---

Despite our efforts, our pipeline still has repeated content,
for instance the names of input and output files
([dependencies][ref-dependency] and [targets][ref-target]).
Our `zipf_test` rule, for instance, is extremely clunky.
What happens if we want to analyze `books/sierra.txt` as well?
We'd have to update everything!

~~~
rule zipf_test:
    input:  'zipf_test.py', 'abyss.dat', 'last.dat', 'isles.dat'
    output: 'results.txt'
    shell:  'python {input[0]} {input[1]} {input[2]} {input[3]} > {output}'
~~~
{: .language-python}

Let's try to improve this rule. One thing you've probably noticed is that all
of our rules are using Python strings. Other data structures work too - let's
try a list:

~~~
rule zipf_test:
    input:
        cmd='zipf_test.py',
        dats=['abyss.dat', 'last.dat', 'isles.dat']
    output: 'results.txt'
    shell: 'python {input.cmd} {input.dats} > {output}'
~~~
{: .language-python}

After updating your rule, run `snakemake clean` and `snakemake -p` to confirm
that the pipeline still works.

> ## Named Dependencies
>
> Note that we also had to switch to using named dependencies.  This was required
> since the first input, `zipf_text.py`, **should not** be in the list of input
> files.
{: .callout}

> ## Inputs: named vs indexed?
>
> Having seen the use of both named and indexed dependencies, which
> approach do you prefer?
>
> Which approach do you think leads to
> Snakefiles that are easier to read and maintain?
{: .discussion}

The use of a list for the input files illustrates a key feature of Snakemake:
**Snakefiles are just Python code.**

We can make our list into a variable to demonstrate this. Let's create the
global variable DATS and use it in our `zipf_test` and `dats` rules:

~~~
DATS=['abyss.dat', 'last.dat', 'isles.dat']

# generate summary table
rule zipf_test:
    input:
        cmd='zipf_test.py',
        dats=DATS
    output: 'results.txt'
    shell: 'python {input.cmd} {input.dats} > {output}'

rule dats:
    input: DATS
~~~
{: .language-python}

Great! One more step towards reducing code duplication. Now there is just
one place to update the list of files to process.

> ## Update your Snakefile
>
> Update your Snakefile with the `DATS` global variable.
>
> Try recreating both the `dats` and `results.txt` targets
> (run `snakemake clean` in between).
>
> > ## Solution
> >
> > See `.solutions/snakefiles_are_python/Snakefile_dats_list` for a full Snakefile.
> > Otherwise, just refer to the code extracts above and modify your own file.
>{:.solution}
{:.challenge}

## When are Snakefiles executed?

The last example illustrated that we can use arbitrary Python code in our Snakefile. It's
important to understand when this code gets executed. Let's add a print
statement to the top of our Snakefile:

~~~
print('Snakefile is being executed!')

DATS=['abyss.dat', 'last.dat', 'isles.dat']

# generate summary table
rule zipf_test:
    input:
# more output below
~~~
{: .language-python}

Now let's clean up our workspace with `snakemake clean`:

~~~
snakemake clean
~~~
{: .language-bash}

~~~
Snakefile is being executed!
Provided cores: 1
Rules claiming more threads will be scaled down.
Job counts:
	count	jobs
	1	clean
	1

rule clean:
    jobid: 0

Finished job 0.
1 of 1 steps (100%) done
~~~
{: .output}

Now let's re-run the pipeline...

~~~
snakemake
~~~
{: .language-bash}

~~~
Snakefile is being executed!
Provided cores: 1
Rules claiming more threads will be scaled down.
Job counts:
	count	jobs
	3	count_words
	1	zipf_test
	4

rule count_words:
    input: wordcount.py, books/last.txt
    output: last.dat
    jobid: 3
    wildcards: file=last

Finished job 3.
1 of 4 steps (25%) done

rule count_words:
    input: wordcount.py, books/abyss.txt
    output: abyss.dat
    jobid: 1
    wildcards: file=abyss

Finished job 1.
2 of 4 steps (50%) done

rule count_words:
    input: wordcount.py, books/isles.txt
    output: isles.dat
    jobid: 2
    wildcards: file=isles

Finished job 2.
3 of 4 steps (75%) done

rule zipf_test:
    input: zipf_test.py, abyss.dat, last.dat, isles.dat
    output: results.txt
    jobid: 0

Finished job 0.
4 of 4 steps (100%) done
~~~
{: .output}

Let's do a dry-run:

~~~
snakemake -n
~~~
{: .language-bash}

~~~
Snakefile is being executed!
Nothing to be done.
~~~
{: .output}

In every case, the `print()` statement ran before any of the actual pipeline
code. What we can take away from this is that Snakemake executes the entire
Snakefile every time we run `snakemake`, even for a dry-run. Because of this
we need to be careful and only put tasks that do "real work" (changing files
on disk) inside rules.

Common tasks, such as building lists of input files that will be reused in
multiple rules are a good fit for Python code that lives outside the rules.

> ## Is your `print` output appearing last?
>
> On some systems, output is buffered. This means that nothing is actually output
> until the buffer is full. While this is more efficient, it can delay the output
> from the `print` command.
>
> In my testing on Windows using the combination of Git Bash and Anaconda, the
> `print` statement is buffered, resulting in the text printing to the terminal
> ***after*** all the Snakemake output. If this is happening to you, tell the
> `print` statement to force a flush of the output buffer:
>
> ~~~
> print("Snakefile is being executed!", flush=True)
> ~~~
> {:.language-python}
>
> You should then see the printed text before the Snakemake output, confirming
> that this code executes first.
{:.callout}

## Using functions in Snakefiles

In our example here, we only have 4 books (and just 3 are being processed).
But what if we had 700 books to be processed? It would be a massive effort to
update our `DATS` variable to add the name of every single book's
corresponding `.dat` filename.

Fortunately, Snakemake ships with several functions that make working with
large numbers of files much easier. The two most helpful ones are
`glob_wildcards()` and `expand()`. Let's start a Python session to see how
they work.

> ## This can be done in any Python environment
>
> You can use any Python environment for the following code exploring `expand()`
> and `glob_wildcards()`. The standard Python interpreter, ipython, or
> a Jupyter Notebook. It's up to personal preference and what you have installed.
>
> On Windows, calling `python` from Git Bash does not always work. It is better
> to use the Anaconda start menu entries to run a Python prompt and then run
> `python` from there.
>
> Make sure you change to your Snakefile directory before launching Python.
{: .callout}

In this example, we will import these Snakemake functions directly in our
Python session.

> ## Importing is not required in a Snakefile
>
> You don't need to import the Snakemake utility functions within your Snakefile - they are
> always imported for you.
{: .callout}

So in your chosen Python environment, run the following:

~~~
from snakemake.io import *
~~~
{: .language-python}

### Generating file names with expand()

The first function we'll use is `expand()`. `expand()` is used quite
literally, to expand snakemake wildcards into a set of filenames:

~~~
expand('folder/{wildcard1}_{wildcard2}.txt', wildcard1=['a', 'b', 'c'], wildcard2=[1, 2, 3])
~~~
{: .language-python}

~~~
['folder/a_1.txt',
 'folder/a_2.txt',
 'folder/a_3.txt',
 'folder/b_1.txt',
 'folder/b_2.txt',
 'folder/b_3.txt',
 'folder/c_1.txt',
 'folder/c_2.txt',
 'folder/c_3.txt']
~~~
{: .output}

In this case, `expand()` created every possible combination of filenames from
the two wildcards. Nice! Of course, this still leaves us needing to get the
values for `wildcard1` and `wildcard2` in the first place.

### Get wildcard values with glob_wildcards()

To get a set of wildcards from a list of files, we can use the
`glob_wildcards()` function. It matches the given pattern against files
on the file system, returning a named tuple containing all the matches. Let's
try grabbing all of the book titles in our `books` folder:

~~~
glob_wildcards('books/{example}.txt')
~~~
{: .language-python}

~~~
Wildcards(example=['isles', 'last', 'abyss', 'sierra'])
~~~
{: .output}

In this case, there is only one wildcard, `{example}`.
We can extract the values for name by getting the `example`
property from the output of `glob_wildcards()`:

~~~
glob_wildcards('books/{example}.txt').example
~~~
{: .language-python}

~~~
['isles', 'last', 'abyss', 'sierra']
~~~
{: .output}

> ## Putting it all together
>
> Using the `expand()` and `glob_wildcards()` functions,
> modify the pipeline so that it automatically detects and analyzes
> all the files in the `books/` folder.
>
> > ## Hint
> >
> > Use `expand()` and `glob_wildcards()` together to create the value of `DATS`.
> {:.solution}
>
> > ## Solution
> >
> > The critical change is to the assignment of `DATS`, building it dynamically from
> > the input `*.txt` file names.
> >
> >~~~
> >DATS = expand('{book}.dat', book=glob_wildcards('./books/{book}.txt').book)
> >~~~
> >{:.language-python}
> >
> > See `.solutions/snakefiles_are_python/Snakefile_glob_dats` for a full Snakefile using
> > this approach.
> {: .solution}
{: .challenge}

## Using Python code as actions

One very useful feature of Snakemake is the ability to execute Python code
instead of just shell commands. Instead of `shell:` as an action, we can use
`run:` instead.

Add the following to your snakefile:

~~~
# at the top of the file
import glob
import os

# add as the last rule (we don't want it to be the default)
rule print_book_names:
    run:
        print('These are all the book names:')
        for book in glob.glob('books/*.txt'):
            print(book)
~~~
{: .language-python}

Upon execution of the corresponding rule, Snakemake runs our Python code
in the `run:` block:

~~~
snakemake print_book_names --quiet
~~~
{: .language-bash}

~~~
Provided cores: 1
Rules claiming more threads will be scaled down.
Job counts:
	count	jobs
	1	print_book_names
	1

rule print_book_names:
    jobid: 0

These are all the book names:
books/isles.txt
books/last.txt
books/abyss.txt
books/sierra.txt
Finished job 0.
1 of 1 steps (100%) done
~~~
{: .output}

> ## Note the `--quiet` option
>
> `--quiet` or `-q` suppresses a lot of the rule progress output from Snakemake.
> This can be useful when you just want to see your own output.
{:.callout}

[ref-dependency]: ../reference#dependency
[ref-target]: ../reference#target
[ref-action]: ../reference#action

{% include links.md %}
