---
layout: post
title:  "gcfboardr: A Text Data Set Ready For Analysis"
date:   2017-4-25 12:16:01 -0600
category: r
tags: [r]
comments: true
---




A new R package, `tidytext`, caught my eye recently. It extends the principles of tidy data to text-mining. Tidy data is data in which:

* every row is an observation
* every column is a variable
* every table is a distinct observational unit

In the context of text analysis, tidy data is data in which every row contains a single *token*. What is a token? 

A token is a meaningul unit of analysis in a text, such as a word, a bigram (which is a pair of words), or a sentence.

Every other text mining package in R does not produce tidy data. So when I saw tidytext, I knew I wanted to take it for a test drive.

I've also been looking for an opportunity to practice text harvesting in R with the `rvest` package. So it made sense to produce my own corpus. To do that, I needed a big stash of text which nobody had really looked at. 

***
##### Why the GCF Board Documents?

Lately I've been thinking a lot about the Green Climate Fund, a fascinating organization. Almost every decision the Fund takes at board level has been documented and published. For an organization which goes to such lengths in the name of transparency, they also get hit with a lot of flak. So it made sense to ask some questions about this huge corpus they've produced, and let the data speak for itself.

To build the data, I used documents produced for board meetings, available [here on the GCF website](http://http://www.greenclimate.fund/boardroom/board-meetings/documents). I've read some of these documents before, and it occured me that the Fund will produce more text than anyone can read in a lifetime. So I've used my natural curiosity about the Fund as a motivating project with which to practice the tidytext approach to text mining and gain deeper insight into the Fund.

***
##### A new R package: `gcfboardr`

In the course of my analysis, I harvested over 500,000 lines of text from 500+ GCF board documents. 

These operations are computationally time-intensive, and it's a pain to repeat them. Because of this, I've  built a data-only R package called `gcfboardr`, so that anyone can make use of the corpus I've created without having to go through the process of sourcing all the pdf documents, extracting text from them, and then cleaning it.

***
##### Who is this package for? 

This is a large corpus of text which concerns finance, development, infrastructure, climate, risk and international negotiations. If you're not interested in these subjects, then this dataset is probably not for you!

If you are interested, however, you can use the gcfboardr package to learn how to use tidytext, or you can use it for machine-learning practice with topic modelling algorithms.

***
##### Installing the gcfboardr package and loading the data 

Data in this package was designed to work with `tidytext` `dplyr`, `ggplot2` and the rest of the tidyverse. However, it does require a little pre-processing before it is *tidy*.

Let's load the data, and print out a few observations in order to judge whether it is tidy data.


{% highlight r %}
# Load the libraries we'll use for pre-processing
library(dplyr)
library(tidytext)
library(stringr)
library(devtools)
library(ggplot2)

# Install the gcfboardr package from github
install_github("ljodea/gcfboardr") 

# Load the library and the data
library(gcfboardr)
data("gcfboard_docs")

# Print out the first 5 observations
head(gcfboard_docs, n = 5L)
{% endhighlight %}


{% highlight text %}
## # A tibble: 5 × 3
##                                                   text meeting                          title
##                                                  <chr>  <fctr>                         <fctr>
## 1                                   Date: 1 March 2017    B.16 Sixteenth Meeting of the Board
## 2                                          Reference:     B.16 Sixteenth Meeting of the Board
## 3                       Sixteenth Meeting of the Board    B.16 Sixteenth Meeting of the Board
## 4                                     4 – 6 April 2017    B.16 Sixteenth Meeting of the Board
## 5 GCF Headquarters, Songdo, Incheon, Republic of Korea    B.16 Sixteenth Meeting of the Board
{% endhighlight %}

Each observation in the `text` column is a line, and you can see which document it belongs to by looking at the `title` column. This is a useful visual confirmation that the text matches the text found in the body of board documents. 
 
However, it comes at the expense of tidiness. The data "as-is" contains multiple tokens per-row, because each observation is composed of both words and a line from an original document. Un-nesting these tokens is straightforward, however.


***
##### Turning the data into tidy data

To get tidy data, all we need is the `unnest_tokens` function from the `tidytext` package.


{% highlight r %}
gcfboard_docs %>% 
  unnest_tokens(word, text) %>% 
  count(word, sort = TRUE) 
{% endhighlight %}



{% highlight text %}
## # A tibble: 44,122 × 2
##     word      n
##    <chr>  <int>
## 1    the 388953
## 2    and 213281
## 3     of 200375
## 4     to 144181
## 5     in  97765
## 6    for  72539
## 7      a  68431
## 8     on  40802
## 9     as  39289
## 10  that  39265
## # ... with 44,112 more rows
{% endhighlight %}

Now the data is tidy. But what else do you notice? 

A few banal words occur very often. We'd like to stop using them because they don't tell us anything. They're called "stop words" and `tidytext` contains a convenient list of stop words. There are also a few words in the `gcfboard_docs` data which occur too often to be of interest. For example "green", "climate", "fund" and "board". Finally, there are numbers in the text, and we'd rather not have numbers recorded as words.

The easiest way to remove stopwords is to use an `anti_join` from the dplyr package. The easiest way to remove numbers is to use a regular expression.


{% highlight r %}
# Load stop words from the tidytext package and add custom stop words
data(stop_words)

custom_stop_words <- bind_rows(stop_words, 
                           data_frame(word = c("fund", "green", "climate", "board", "gcf", "gcfb"), 
                                      lexicon = rep("custom", 6)))

# Remove stop words using an anti_join
gcf_tidy <- gcfboard_docs %>%
  mutate(text = str_replace_all(text, "[[:digit:]]", "")) %>% 
  unnest_tokens(word, text) %>%
  anti_join(custom_stop_words)
{% endhighlight %}

Now let's visualize counts of the remaining words:

![center](/figs/2017-4-25-GCF_board_documents_to_tidy_data/unnamed-chunk-5-1.png)

That's better! It looks like the GCF is very project-focused. 

***
#### Previewing the tidy data set

Let's verify that our data set is now tidy, by pulling out some rows at random.


{% highlight r %}
# Look at some random rows in the data
 gcf_tidy[4997:5003, ]
{% endhighlight %}



{% highlight text %}
## # A tibble: 7 × 3
##   meeting              title       word
##    <fctr>             <fctr>      <chr>
## 1    B.08 Outcome of the IRM   figueres
## 2    B.08 Outcome of the IRM christiana
## 3    B.08 Outcome of the IRM      annan
## 4    B.08 Outcome of the IRM      annan
## 5    B.08 Outcome of the IRM       kofi
## 6    B.08 Outcome of the IRM       kofi
## 7    B.08 Outcome of the IRM    chair's
{% endhighlight %}


Every observation is a now a single token, which is what we wanted. Great! We're done here and can move on to more interesting analysis of the data.

For ideas on how to use the data set for analysis, see my post on [exploratory data analysis of the gcfboardr dataset](http://state.gy/r/exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/).  
  

***
##### Notes

1. Users should be aware that there are some issues with the data from B.03, a board meeting which was documented using an unknown process for creating Pdf documents. This set of Pdf documents caused some parsing problems for the `pdftools` package, resulting in garbled word tokens. As such, gcfboardr does not contain any documents from the B.03 board meeting.  

2. On github, I've also included the R script documenting how I harvested all the files from the GCF website and pulled the text out of them. The script includes a few functions I wrote, such as the pdf_date function, which makes it easier to pull "creation date" metadata out of pdfs, using a method which works well with the map family of functions from purrr. This means that unlike the pdf_info function in the pdftools package, it can be used when building a data frame using dplyr. You can find that script [here](https://github.com/ljodea/gcfboardr/blob/master/data-raw/prep_data.R).
