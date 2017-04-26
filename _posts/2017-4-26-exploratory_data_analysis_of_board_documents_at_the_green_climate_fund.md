---
layout: post
title:  "Exploratory Data Analysis of Board Documents at The Green Climate Fund"
date:   2017-4-26 17:55:01 +0900
category: r
tags: [r]
comments: true
---





# Introducing gcfboardr 

In a previous post, I introduced a package I'd written which holds Green Climate Fund (GCF) board documents, ready for for text analysis.

By looking at the most important GCF documents we might be able to get a sense of priorities at the Fund, including changes over time.

I recommend using this data with the `tidytext` R package. `tidytext` includes a set of functions which each do one thing very well, share the same syntax, reduce the need for typing, and which behave in a predictable manner (i.e. you can reason about what they're doing and they produce consistent output). Combining several smaller functions from tidytext and dplyr allows you to produce powerful transformations of text data. This is exactly what I'm going to do below.

### How to install

But first, how do you install the gcfboardr package? You'll need to have the devtools package installed, and then install gcfboardr from github using the following code:


{% highlight r %}
# Install gcfboardr from github. Note: this step can take 1-2 minutes.
library(devtools)
install_github("ljodea/gcfboardr") 

# Load the library and the data
library(gcfboardr)
data("gcfboard_docs")
{% endhighlight %}

## What's in the gcfboardr data set?

Let's load up a few libraries and take a glimpse at the data:


{% highlight r %}
library(dplyr)
library(ggplot2)

glimpse(gcfboard_docs)
{% endhighlight %}



{% highlight text %}
## Observations: 493,262
## Variables: 3
## $ text    <chr> "Date: 1 March 2017", "Reference: ", "Sixteenth Meeting of the Board", "4 – 6 A...
## $ meeting <fctr> B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, ...
## $ title   <fctr> Sixteenth Meeting of the Board, Sixteenth Meeting of the Board, Sixteenth Meet...
{% endhighlight %}

We have almost 500,000 observations of three variables, in which every observation is a line from an original document. 

How many documents are there per meeting?

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/docspm-1.png)

B.08 was the first board meeting of 2014, the year the Fund started operations.

How many lines of text are there for each meeting?

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/linespm-1.png)

We can see that B.08 and B.11 were particularly prolific, and the early meetings produced far less text than the later ones. 



### Is the board meeting a meaningful unit of analysis?

When we're first looking at this data set, we might want to know whether our data is normal for a large corpus, and Zipf's Law is one yardstick we might use. Zipf's Law states that given a corpus of natural language text, the frequency of any word is inversely proportional to its rank in the frequency table. This is a power law, and it implies that the most common word appears roughly twice as often in a corpus than the 2nd most common word, and three times as often as the 3rd most common word, and so on. 

$$f(k;s,N)=\frac{1/k^s}{\sum_{n=1}^N (1/n^s)}$$

If this relationship holds for our data, we can proceed. If it doesn't we might have some problematic documents (this was the case for B.03, which has some badly formatted text). Let's check the remaining corpus, grouping by board meeting. 


![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/zipf-1.png)

It looks like each group of board meeting documents in our data set obeys Zipf's law! This means that, without reading any documents, groups of documents belonging to each board meeting conform to out expectations about what a corpus should look like. There are no significant distortions in the text and we can move on to more interesting analysis.


## Single Word Analysis

So far we've been looking at lines, documents and meetings, but what we'd really like to look at is the words themselves. So let's use the power of `tidytext` to unnest words and remove some "stop words" -- words such as "and" "the" or "a". We'll also get rid of numbers.


{% highlight r %}
# Load stop words
data(stop_words)

# Filter out numbers, any empty lines, and unnest tokens
gcf_tidy <- gcfboard_docs %>%
  mutate(text = str_replace_all(text, "[[:digit:]]", "")) %>% 
  filter(text != "") %>% 
  unnest_tokens(word, text) %>% 
  anti_join(stop_words)
glimpse(gcf_tidy)
{% endhighlight %}



{% highlight text %}
## Observations: 2,661,944
## Variables: 3
## $ meeting <fctr> B.01, B.01, B.01, B.01, B.01, B.01, B.01, B.01, B.01, B.01, B.01, B.01, B.01, ...
## $ title   <fctr> Roles and Responsibilities of the Board, Annotations to the Provisional Agenda...
## $ word    <chr> "meritbased", "contd", "realestate", "worldclass", "spacious", "itower", "itowe...
{% endhighlight %}

Even after we removed a list of 1,149 common stop words, the new tidy dataframe contains over 2.6 million words!

Now let's look at the counts of remaining words, sorted by the number of times they appear in the text. 

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/common words-1.png)

It looks like some words appear very often, in particular words associated with the name of the fund, although the word "green" is relatively less used than are the words "climate" and "fund".

### How many times does the phrase "Green Climate Fund" appear?

In order to work out the relative importance of the words "climate" and "fund", we might want to disambiguate uses of these words between uses which add important context to our analysis, and uses which just repeat the name of the fund.

To solve, we're going to look briefly at trigrams. 

What's a trigram? A trigram is a word triplet: three words which co-occur together in sequence. The tidytext package provides a convenient argument to their unnest_tokens function which we can use here to find every trigram in the text: 

`unnest_tokens(tbl = gcfboard_docs, output = trigram, input = text, token = "ngrams", n = 3)`. 

If we filter this output such that the first word is "green", the second is "climate" and the third is "fund", we can see how many times the Fund name appears across all board documents:


{% highlight text %}
## Source: local data frame [1 x 4]
## Groups: word1, word2 [1]
## 
##   word1   word2 word3     n
##   <chr>   <chr> <chr> <int>
## 1 green climate  fund  6310
{% endhighlight %}

Looking at the result, we can see that 6,310 uses of each word "green", "climate" and "fund" are repetitions of the name. So how many uses are there in other contexts?


{% highlight text %}
## # A tibble: 3 × 4
##      word `basic count` `name context` `other contexts`
##     <chr>         <int>          <dbl>            <dbl>
## 1    fund         26004           6310            19694
## 2 climate         23673           6310            17363
## 3   green          7390           6310             1080
{% endhighlight %}

Interesting! 

If we look at the counts of words, above, we can see that "green" occurs rarely in contexts other than the name context. The word "green" is almost never used unless the name of the fund is repeated.

Now we can use the results of this trigram analysis to find out the relative importance of the word "climate", disambiguated from usage which is just a repetition of the fund's name. 


### Which words are most common if we remove some of the noisy usage?

Let's remove the name references of those words to see how important each individual word is. Let's also remove some words such as "board" and the "gcf" acronym.

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/tidycount2-1.png)

Wow! It turns out that the word "climate", independent of usage as part of the name of the Fund, is the third most common word. It's a good job we didn't remove it out of hand. Usage of the word "fund" is less clear, because it is both a short-name for the GCF, and is also used as a verb in contexts which have nothing to do with the name.

You might also be wondering why we didn't remove the word "project" since it is so common. This is because frequency of usage has changed a lot over time, unlike the other words we removed.

### How has word usage changed over time?




Let's look at a few words which might be interesting to us: "secretariat", "project", "risk" and "private". Plotting statistical transformations of word frequency can help us see these changes over time. Below you can see the bare frequenciess on top, followed by a log10 scale in the middle, and finally a squre root coordinate transform at the bottom.


![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/changes-1.png)

All three plots show that usage has changed for some words more than others. Since the Fund became operational in 2014, use of the word "project" has surged. 

At the earliest meetings, establishing a secretariat was priority number one, and we can see usage declining over time. "Risk" begins to become much more important at B.07, which is when the Fund adopted a risk management framework. The term "private" seems to have declined a bit in usage. 

You'll notice that the bare frequency plot emphasises the high-frequency changes to the words "secretariat" and "project", whereas the log10 plot of word frequency really helps us see what's happening at the low-end of the frequency range. The square-root coordinate transform preserves some of the perspective of both the other plots, and might be the most useful plot of the three.

### What does single word analysis say about different board meetings?

We might want to look at board meetings as a facet of our single word analysis. Which words are particularly associated with specific meeetings? To find this out we can common rules-of-thumb: term frequency and it's cousin, inverse-document frequency.

* **Term frequency (tf)**, measure how frequently a word occurs in a document. However, even after we remove words such as “and”, “the”, "a”, et cetera, there are some words which will occur much more often than others.

$$tf(\text{word}) = (\frac{n_{\text{word}}}{n_{\text{total words in document}}})$$
* **Inverse document frequency (idf)** weights the frequency of a word by how rarely it occurs in a set of documents, and increases the weight for words that are seldom seen in the set. You'll notice that since this takes the logarithm, an extremely common word would get a rating of zero.

$$idf(\text{word}) = {\ln{\left(\frac{n_{\text{documents}}}{n_{\text{documents containing term}}}\right)}}$$

These two concepts can be combined to calculate a weighted frequency measure for each word: the **term frequency inverse document frequency (tf-idf)**. You can think about this as the frequency of a term in a document weighted by how rarely it is used among a group of documents, with higher weights going to rarely used words, and lower weights to commonly-used words. 

$$tfidf(\text{word}) = tf(\text{word}) * idf(\text{word})$$


Computing this with the `bind_tf_idf` function from the tidytext package, we get a data table which shows the words which are associatd in particular with one meeting. 


{% highlight text %}
## # A tibble: 120,189 × 6
##    meeting     word     n           tf       idf       tf_idf
##     <fctr>    <chr> <int>        <dbl>     <dbl>        <dbl>
## 1     B.16   geeref   794 0.0015003099 2.7080502 0.0040629145
## 2     B.14    ugeap   587 0.0008643819 1.6094379 0.0013911690
## 3     B.11     lged   223 0.0004362046 2.7080502 0.0011812639
## 4     B.16    saïss   225 0.0004251508 2.7080502 0.0011513297
## 5     B.13 bandesal   248 0.0004181448 2.7080502 0.0011323571
## 6     B.10      tbd   450 0.0013617588 0.7621401 0.0010378509
## 7     B.15      cis   462 0.0006954741 1.3217558 0.0009192470
## 8     B.13      eba   399 0.0006727410 1.3217558 0.0008891994
## 9     B.11      waf   164 0.0003207962 2.7080502 0.0008687322
## 10    B.16     siea   167 0.0003155564 2.7080502 0.0008545425
## # ... with 120,179 more rows
{% endhighlight %}

We can then plot this data, faceting by meeting and picking the top 5 terms associated with a particular meeting:

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/plotwbm2-1.png)

That's a lot of acronyms! If we look into the board documents themselves, we can see that these acronyms tend to denote real organizations or operational units at the fund. For example, let's look at B.16:

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/b16-1.png)

GEEREF is the Global Energy Efficiency and Renewable Energy Fund. Great!

And let's look at B.06:

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/b06-1.png)

The "iiu" is in fact the "Independent Integrity Unit" (IIU), which is one of three accountability units at the fund. Similarly, the "ieu" is the Independent Evaluation Unit" (IEU) and the "irm" is the Independent Redress Mechanism (IRM).

We can see that, relative to other board meetings, the IIU and the other accountability units were a particularly important topic at the sixth meeting of the board.

### How similar are individual board meetings?

Using purely dplyr and tidyr functions, we can also transform the data to look at frequencies of words grouped by meeting and plot them by meeting pairs. This gives us a different visualization, since words that are common to one meeting are common to another, and cancel each other out, leaving interesting words at the margins.

In the following plots, a 45 degree dashed line denotes equal frequency between word usage. Any words above the line appear more commonly in B.16 board documents, while any words below the line appear more frequently at an earlier meeting (which specific meeting is denoted by a facet label above the plot). 

So for example in the following, we can see that "co-chairs" was a big issue at B.01, while it wasn't important at B.16. In contrast, B.16 had a greater focus on "women", "water" and "projects" than did B.01.

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/frequencies-1.png)

We can see that at B.16 the EBRD was a greater topic of discussion than it was at some previous board meetings. B.15 was particularly concerned with "applicants" to the fund and it was held in "samoa", while B.10 was relatively more concerned with "applicants" and "accountability" than was B.16.

By looking at the spread of the data, we can also see that correlation between word frequencies at B.16 and at earlier meetings converges over time. Between B.16 and B.01 the word frequencies are dispered, while the B.16-B.10 pair is a little less dispersed, and the frequencies for B.15 appear to converge towards those for B.16.

We can confirm this intuition by looking at Pearson's product-moment correlations between B.16 and previous meetings:

* First between B.01 and B.16: correlation is about 32%

{% highlight text %}
## 
## 	Pearson's product-moment correlation
## 
## data:  proportion and B.16
## t = 15.422, df = 2141, p-value < 2.2e-16
## alternative hypothesis: true correlation is not equal to 0
## 95 percent confidence interval:
##  0.2775762 0.3538086
## sample estimates:
##       cor 
## 0.3162027
{% endhighlight %}

* Next between B.10 and B.16: correlation is about 62%

{% highlight text %}
## 
## 	Pearson's product-moment correlation
## 
## data:  proportion and B.16
## t = 53.63, df = 4562, p-value < 2.2e-16
## alternative hypothesis: true correlation is not equal to 0
## 95 percent confidence interval:
##  0.6037163 0.6393166
## sample estimates:
##       cor 
## 0.6218376
{% endhighlight %}

* Last, between B.15 and B.16: correlation is about 92%

{% highlight text %}
## 
## 	Pearson's product-moment correlation
## 
## data:  proportion and B.16
## t = 196.81, df = 7462, p-value < 2.2e-16
## alternative hypothesis: true correlation is not equal to 0
## 95 percent confidence interval:
##  0.9119396 0.9192719
## sample estimates:
##       cor 
## 0.9156819
{% endhighlight %}

Great! This confirms our intuition that B.16 is more similar to the meetings which immediately preceded it than to other earlier meetings of the board.

### Wrap Up

That's enough for one post! For more ideas about how to analyze this data set beyond single-word analysis, and for a lot more on correlation, standy by for my next post on ngrams. I will post the link here as soon as it's available.

***





