---
title: "Manual Data Processing Workflow"
teaching: 15
exercises: 15
questions:
- "How can I make my results easier to reproduce?"
objectives:
- "Understand our example problem."
keypoints:
- "Bash scripts are not an efficient way of defining a workflow."
---

Let's imagine that we're interested in understanding the frequency of words
in various books, testing how closely each book conforms to [Zipf's
Law][ref-zipf].

We've compiled our raw data (the books we want to analyze) and have prepared
several Python scripts that together make up our analysis pipeline.

Let's take quick look at one of the books using the command `head books/isles.txt`:

~~~
$ head books/isles.txt
A JOURNEY TO THE WESTERN ISLANDS OF SCOTLAND


INCH KEITH


I had desired to visit the Hebrides, or Western Islands of Scotland, so
long, that I scarcely remember how the wish was originally excited; and
was in the Autumn of the year 1773 induced to undertake the journey, by
finding in Mr. Boswell a companion, whose acuteness would help my
~~~
{: .output}

Our directory has the Python scripts and data files we we will be working
with:

~~~
|- books
|  |- abyss.txt
|  |- isles.txt
|  |- last.txt
|  |- LICENSE_TEXTS.md
|  |- sierra.txt
|- plotcount.py
|- wordcount.py
|- zipf_test.py
~~~
{: .output}

There may be some other files as well, but the ones listed above are the
ones we are concerned with at present.

The first step is to count the frequency of each word in a book. For this, we
use the `wordcound.py` script. The first argument (`books/isles.txt`) to
wordcount.py is the file to analyze, and the last argument (`isles.dat`)
specifies the output file to write:

~~~
python wordcount.py books/isles.txt isles.dat
~~~
{: .language-bash}

Let's take a quick peek at the result:

~~~
head -5 isles.dat
~~~
{: .language-bash}

This shows us the first 5 lines in the output file:

~~~
the 3822 6.7371760973
of 2460 4.33632998414
and 1723 3.03719372466
to 1479 2.60708619778
a 1308 2.30565838181
~~~
{: .output}

We can see that the file consists of one row per word. Each row shows the
word itself, the number of occurrences of that word, and the number of
occurrences as a percentage of the total number of words in the text file.
The file is sorted by descending frequency, so the first 5 lines
are the 5 most frequent words in the input text.

We can do the same thing for a different book:

~~~
python wordcount.py books/abyss.txt abyss.dat
head -5 abyss.dat
~~~
{: .language-bash}

~~~
the 4044 6.35449402891
and 2807 4.41074795726
of 1907 2.99654305468
a 1594 2.50471401634
to 1515 2.38057825267
~~~
{: .output}

Let's visualize the results. The script `plotcount.py` reads in a data file
and plots the 10 most frequently occurring words as a text-based bar plot:

~~~
python plotcount.py isles.dat ascii
~~~
{: .language-bash}

~~~
the   ########################################################################
of    ##############################################
and   ################################
to    ############################
a     #########################
in    ###################
is    #################
that  ############
by    ###########
it    ###########
~~~
{: .output}

`plotcount.py` can also create the plot as an image file:

~~~
python plotcount.py isles.dat isles.png
~~~
{: .language-bash}

Finally, let's test [Zipf's law][ref-zipf] for these two books:

~~~
python zipf_test.py abyss.dat isles.dat
~~~
{: .language-bash}

~~~
Book	First	Second	Ratio
abyss	4044	2807	1.44
isles	3822	2460	1.55
~~~
{: .output}

> ## Zipf's Law
>
> Recall that Zipf's Law predicts that the most frequent word will
> occur approximately twice as often as the second most frequent word.
{: .callout}

Together these scripts implement a common workflow:

1. Read a data file.
2. Perform an analysis on this data file.
3. Write the analysis results to a new file.
4. Plot a graph of the analysis results.
5. Save the graph as an image, so we can put it in a paper.
6. Make a summary table of the analyses, which requires aggregation of all previous results.

Running `wordcount.py` and `plotcount.py` at the shell prompt, as we have
been doing, is fine for one or two files. If, however, we had 5 or 10 or 20
text files, or if the number of steps in the pipeline were to expand, this
could turn into a lot of work. Plus, no one wants to sit and wait for a
command to finish, even just for 30 seconds.

The most common solution to the tedium of data processing is to write
a shell script that runs the whole pipeline from start to finish.

Using your text editor of choice (e.g. nano), add the following to a new file named
`run_pipeline.sh`.

~~~
# USAGE: bash run_pipeline.sh
# to produce plots for isles and abyss
# and the summary table for the Zipf's law tests

python wordcount.py books/isles.txt isles.dat
python wordcount.py books/abyss.txt abyss.dat

python plotcount.py isles.dat isles.png
python plotcount.py abyss.dat abyss.png

# Generate summary table
python zipf_test.py abyss.dat isles.dat > results.txt
~~~
{: .language-bash}

Run the script and check that the output is the same as before:

~~~
bash run_pipeline.sh
cat results.txt
~~~
{: .language-bash}

This shell script solves several problems in computational reproducibility:

1. It explicitly documents our pipeline,
    making communication with colleagues (and our future selves) more efficient.
2. It allows us to type a single command, `bash run_pipeline.sh`, to
    reproduce the full analysis.
3. It prevents us from _repeating_ typos or mistakes.
    You might not get it right the first time, but once you fix something
    it'll stay fixed.

Despite these benefits it has a few shortcomings.

Let's adjust the width of the bars in our plot produced by `plotcount.py`.

Edit `plotcount.py` so that the bars are 0.8 units wide instead of 1 unit.
(Hint: replace `width = 1.0` with `width = 0.8` in the definition of
`plot_word_counts`.)

Now we want to recreate our figures. We _could_ just `bash run_pipeline.sh`
again. That would work, but it could also be a big pain if counting words
takes more than a few seconds. The word counting routine hasn't changed; we
shouldn't need to recreate those files.

Alternatively, we could manually rerun the plotting for each word-count file.
(Experienced shell scripters can make this easier on themselves using a
for-loop.)

~~~
for book in abyss isles; do
    python plotcount.py $book.dat $book.png
done
~~~
{: .language-bash}

Another popular option is to comment out a subset of the lines in
`run_pipeline.sh`:

~~~
# USAGE: bash run_pipeline.sh
# to produce plots for isles and abyss
# and the summary table

# These lines are commented out because they don't need to be rerun.
#python wordcount.py books/isles.txt isles.dat
#python wordcount.py books/abyss.txt abyss.dat

python plotcount.py isles.dat isles.png
python plotcount.py abyss.dat abyss.png

# This line is also commented out because it doesn't need to be rerun.
#python zipf_test.py abyss.dat isles.dat > results.txt
~~~
{: .language-bash}

Then, we would run our modified shell script using `bash run_pipeline.sh`.

But commenting out these lines, and subsequently uncommenting them, can be a
hassle and source of errors in complicated pipelines. What happens if we have
hundreds of input files? No one wants to enter the same command a hundred
times, and then edit the result.

What we really want is an executable _description_ of our pipeline that
allows software to do the tricky part for us: figuring out what tasks need to
be run where and when, then perform those tasks for us.

> ## Thinking about updating results
>
> 1. Make some small adjustment to `plotcount.py` that changes how the information is presented in the graph (hint: just like how we earlier adjusted the `width`).
> 2. Run the pipeline script to update all graph files with the new style of graph.
>
> Then consider the following questions:
>
> 1. How do you know which graphs need to be updated?
> 2. What would you have to do if one of the books' contents changed at a later date, but you didn't know which book had changed?
> 3. Would this be a problem if it took much longer (e.g. hours) to do the book-specific analysis that is required to draw the corresponding graph?
> 
> > ## Solution
> > 1. Since you made a change to the graph drawing logic, which affects *all* graphs, you can reason that *all* graphs must be updated. Assuming you made no changes to the pipeline script, running it will cause all graphs to be updated to the new style.
> > 2. You would have to issue the command to re-draw all of the graphs again, even though only one of them actually needs updating.
> > 3. Compared to our demonstration workflow which can be run in seconds, a real-life or large-scale workflow could take much longer to complete. So having to re-run the _entire_ workflow because only _one part_ of the inputs has changed could mean taking up considerable time and compute resources in order to update _all_ of the results.
> {:.solution}
{:.challenge}

[ref-zipf]: ../reference#zipfs-law

{% include links.md %}
