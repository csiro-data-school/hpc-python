---
title: "Completing the Pipeline"
teaching: 10
exercises: 30
questions:
- "How do I move generated files into a subdirectory?"
- "How do I add new processing rules to a Snakefile?"
- "What are some common practices for Snakemake?"
- "How can I get my workflow to clean up generated files?"
- "What is a default rule?"
objectives:
- "Update existing rules so that `dat` files are created in a subdirectory."
- "Add a rule to your Snakefile that generates PNG plots of word frequencies."
- "Add an `all` rule to your Snakefile."
- "Make `all` the default rule."
keypoints:
- "Keeping output files in the top-level directory can get messy. One solution is to put files into
subdirectories."
- "It is common practice to have a `clean` rule that deletes all intermediate and generated files, taking your workflow back to a blank slate."
- "A default rule is the rule that Snakemake runs if you don't specify a rule on the command line. It is simply the first rule in a Snakefile."
- "Many Snakefiles define a default target called `all` as first target in the file. This runs by default and typically executes the entire workflow."
---

## Moving Output Files into a Subdirectory

Currently our workflow is generating a lot of files in the main directory. This is not so bad with small numbers of files, but it can get messy as the file count grows. One approach to this is to generate outputs into their own directories, named after the file types. For example:

~~~
.
├── books
│   ├── abyss.txt
│   ├── isles.txt
│   ├── last.txt
│   ├── LICENSE_TEXTS.md
│   └── sierra.txt
├── dats
│   ├── abyss.dat
│   ├── isles.dat
│   ├── last.dat
│   └── sierra.dat
├── Pipfile
├── plotcount.py
├── requirements.txt
├── results.txt
├── Snakefile
├── wordcount.py
├── zipf_analysis.tar.gz
└── zipf_test.py
~~~
{:.output}

There are many potential arrangements, so you are free to choose whatever makes sense
for your project. Snakemake is not prescriptive, it will put files wherever
you tell it. So here we will learn how to move the `dat` files into a `dats`
directory.

> ## Moving the `dat` files
>
> Alter the rules in your Snakefile so that the `dat` files are created in
> their own `dats/` folder.
> Note that creating this folder beforehand is unnecessary.
> Snakemake automatically create any folders for you, as needed.
>
> > ## Hint
> >
> > * Make sure your `Snakefile` is up to date with the end of
> > the preceeding lesson. Use the provided solution files if necessary.
> > * Look for all the locations that reference the `dat` files and update to add
> > the `dats/` directory.
> >
> {:.solution}
>
> > ## Solution
> >
> > First update the `DATS` variable with the `dats` directory:
> >~~~
> >DATS = expand('dats/{file}.dat', file=glob_wildcards('./books/{book}.txt').book)
> >~~~
> >{:.language-python}
> >
> > Then update `count_words` so the dat files get created in the same place:
> >~~~
> >rule count_words:
> >    input:
> >        cmd='wordcount.py',
> >        book='books/{file}.txt'
> >    output: 'dats/{file}.dat'
> >    shell: 'python {input.cmd} {input.book} {output}'
> >~~~
> >{:.language-python}
> >
> > Finally, update the `clean` rule to remove the `dats` directory:
> >~~~
> > rule clean:
> >     shell: 'rm -rf dats/ *.dat results.txt'
> >~~~
> >{:.language-python}
> >
> > Note that in the clean rule there is no harm from keeping the `*.dat` pattern in
> > the `rm` command even though no new files will be created in that location. It will
> > help clean up if you forgot to run `snakemake clean` before updating the Snakefile.
> >
> > See `.solutions/completing_the_pipeline/Snakefile_move_dats`.
>{:.solution}
{:.challenge}

> ## Windows Note
>
> At the time of writing, there is an open bug in Snakemake (version 5.8.2) on Windows
> systems that prevents requesting specific files from the command line when those files
> are in a subdirectory.
>
> For example, before moving the `dat` files to the `dats` directory, you could request that Snakemake
> build a specific file with a command like:
>
> ~~~
> snakemake last.dat
> ~~~
> {:.language-bash}
>
> After moving the location of `dat` files, the correct command is:
>
> ~~~
> snakemake dats/last.dat
> ~~~
> {:.language-bash}
>
> On Windows systems this command produces an error. However, Snakemake can still build the files
> correctly when processing inputs for other rules (such as the `dats` rule). The bug only affects
> files requested from the command line.
>
> Later in this episode we will see one way around this issue when we introduce the `all` rule.
{:.callout}

## Generating Plots

> ## Creating PNGs
>
> Your challenge is to update your Snakefile so that it can create `.png`
> files from `dat` files using `plotcount.py`.
>
> * The new rule should be called `make_plot`.
> * All `.png` files should be created in a directory called `plots`. If you are using a
> Windows system, you could create the plots in the top-level directory instead in order
> to avoid the Windows subdirectory bug. You may need to change back to the `plots` directory
> after we introduce the `all` rule.
>
> As well as a new rule you may also need to update existing rules.
>
> Remember that when testing a pattern rule, you can't just ask Snakemake to
> execute the rule by its name, as this will result in a `WorkflowError`.
> Instead, you will need to ask Snakemake to build a specific file,
> e.g. if you wanted to generate the `dats/last.dat`, you would have to execute `snakemake dats/last.dat`
>
> > ## Solution
> >
> > Modify the `clean` rule and add a new pattern rule `make_plot`:
> > ~~~
> > # delete everything so we can re-run things
> > rule clean:
> >     shell: 'rm -rf dats/ plots/ *.dat results.txt'
> >
> > # plot one word count dat file
> > rule make_plot:
> >     input:
> >         cmd='plotcount.py',
> >         dat='dats/{file}.dat'
> >     output: 'plots/{file}.png'
> >     shell: 'python {input.cmd} {input.dat} {output}'
> > ~~~
> > {:.language-python}
> {: .solution}
{: .challenge}

## Default Rules

The default rule is the rule that Snakemake runs if you don't specify a rule
on the command-line (e.g.: if you just run `snakemake`).

The default rule is simply the first rule in a Snakefile. While the default
rule can be anything you like, it is common practice to call the default rule
`all`, and have it run the entire workflow.

> ## Add an `all` rule
>
> Add an `all` rule to your Snakefile.
>
> Note that `all` rules often don't need to do any processing of their own.
> It is suffient to make them depend on all the final outputs from other rules.
> In this case, the outputs are `results.txt` and all the PNG files.
>
> > ## Hint
> >
> > It is easiest to use `glob_wildcards` and `expand` to build the list of
> > all expected `.png` files.
> {:.solution}
> > ## Solution
> >
> > First, we modify the existing code that builds `DATS` to first extract the
> > list of book names, and then to build `DATS` and a new global variable `PLOTS`
> > listing all plots:
> > ~~~
> > # Build the list of book names. We need to use it multiple times when building
> > # the lists of files that will be built in the workflow
> > BOOK_NAMES = glob_wildcards('./books/{book}.txt').book
> >
> > # The list of all dat files
> > DATS = expand('dats/{file}.dat', file=BOOK_NAMES)
> >
> > # The list of all plot files
> > PLOTS = expand('plots/{file}.png', file=BOOK_NAMES)
> > ~~~
> > {:.language-python}
> >
> > Now we can define the `all` rule:
> > ~~~
> > # pseudo-rule that tries to build everything.
> > # Just add all the final outputs that you want built.
> > rule all:
> >     input: 'results.txt', PLOTS
> > ~~~
> > {:.language-python}
> {:.solution}
{:.challenge}

## Cleaning House

It is common practice to have a `clean` rule that deletes all intermediate
and generated files, taking your workflow back to a blank slate.

We already have a `clean` rule, so now is a good time to check that it
removes all intermediate and output files. First do a `snakemake all` followed
by `snakemake clean`. Then check to see if any output files remain and add them
to the clean rule if required.

## Creating an Archive

Let's add a processing rule that depends on all previous stages of the workflow.
In this case, we will create an archive tar file.

> ## Windows Note
>
> If you are using a Windows system, make sure you have followed the
> [setup instructions][lesson-setup] regarding the use of Git Bash.
> This should provide the required environment for the `tar` command to work.
{:.callout}

> ## Creating an Archive
>
> Update your pipeline to:
>
> * Create an archive file called `zipf_analysis.tar.gz`
> * The archive should contain all `dat` files, plots, and the
> Zipf summary table (`results.txt`).
> * Update `all` to expect `zipf_analysis.tar.gz` as input.
> * Remove the archive when `snakemake clean` is called.
>
> The syntax to create an archive is:
> ~~~
> tar -czvf zipf_analysis.tar.gz file1 directory2 file3 etc
> ~~~
> {: .language-bash}
>
> > ## Solution
> >
> > First the `create_archive` rule:
> > ~~~
> > # create an archive with all results
> > rule create_archive:
> >     input: 'results.txt', DATS, PLOTS
> >     output: 'zipf_analysis.tar.gz'
> >     shell: 'tar -czvf {output} {input}'
> > ~~~
> > {:.language-python}
> >
> > Then the update to the `clean target`:
> > ~~~
> > # delete everything so we can re-run things
> > rule clean:
> >     shell: 'rm -rf dats/ plots/ *.dat results.txt zipf_analysis.tar.gz'
> > ~~~
> > {:.language-python}
> >
> > Then the change to `all`. The workflow would still be correct without this
> > step, but since `create_archive` requires building everything, it is simpler
> > to just get `all` to depend on `create_archive`. This means we have just one
> > rule to maintain if we add new outputs later on.
> > ~~~
> > # pseudo-rule that tries to build everything.
> > # Just add all the final outputs that you want built.
> > rule all:
> >     input: 'zipf_analysis.tar.gz'
> > ~~~
> > {:.language-python}
> {:.solution}
{: .challenge}

After these exercises our final workflow should look something like the following:
![Final directed acyclic graph][fig-final-dag]

> ## Adding more books
>
> We can now do a better job at testing Zipf's rule by adding more books.
> The books we have used come from the [Project Gutenberg](http://www.gutenberg.org/) website.
> Project Gutenberg offers thousands of free ebooks to download.
>
>  **Exercise instructions:**
>
> * go to [Project Gutenberg](http://www.gutenberg.org/) and use the search box to find another book,
> for example ['The Picture of Dorian Gray'](https://www.gutenberg.org/ebooks/174) from Oscar Wilde.
> * download the 'Plain Text UTF-8' version and save it to the `books` folder;
> choose a short name for the file
> * optionally, open the file in a text editor and remove extraneous text at the beginning and end
> (look for the phrase `End of Project Gutenberg's [title], by [author]`)
> * run `snakemake` and check that the correct commands are run
> * check the results.txt file to see how this book compares to the others
{: .challenge}

[fig-final-dag]: ../fig/06-final-dag.svg

{% include links.md %}
