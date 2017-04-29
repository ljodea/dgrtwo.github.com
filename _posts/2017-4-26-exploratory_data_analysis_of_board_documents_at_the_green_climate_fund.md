---
layout: post
title:  "Exploratory Data Analysis: The Green Climate Fund"
date:   2017-4-26 17:55:01 +0900
category: r
tags: [r]
comments: true
---





Recently, I harvested over 500,000 lines of text from 500+ Green Climate Fund board documents. Because of this, I've  built a data-only R package called `gcfboardr`, so that anyone can make use of the corpus I've created.

To build the data, I used documents produced for board meetings, available [here on the GCF website](http://http://www.greenclimate.fund/boardroom/board-meetings/documents). I've read some of these documents before, and it occured me that the Fund will produce more text than anyone can read in a lifetime. So I've used my natural curiosity about the Fund as a motivating project with which to practice the tidytext approach to text analysis in R and gain deeper insight into the Fund.

In this post, I'm going to show what can be done with gcfboardr.

***

### Installation

To install the gcfboardr package you'll need to have the devtools package installed, and then install gcfboardr from github using the following code:


{% highlight r %}
# Install gcfboardr from github. Note: this step can take 1-2 minutes.
library(devtools)
install_github("ljodea/gcfboardr") 

# Load the library and the data
library(gcfboardr)
data("gcfboard_docs")
{% endhighlight %}

***

### Preview

Let's load up a few libraries and take a glimpse at the data:


{% highlight r %}
library(dplyr)
library(ggplot2)
library(tidytext)

glimpse(gcfboard_docs)
{% endhighlight %}



{% highlight text %}
## Observations: 490,537
## Variables: 3
## $ text    <chr> "Date:  March ", "Reference: ", "Sixteenth Meeting of the Board", " –  April ",...
## $ meeting <fctr> B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, B.16, ...
## $ title   <fctr> Sixteenth Meeting of the Board, Sixteenth Meeting of the Board, Sixteenth Meet...
{% endhighlight %}

We have almost 500,000 observations of three variables, in which every observation is a line from an original document. How many documents are there per meeting?

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/docspm-1.png)

We can see that B.08 and B.11 were particularly prolific, and the early meetings produced fewer docs than the later ones. B.08 was the third board meeting of 2014, the year the Fund started operations, and it produced more than double the documents from the previous meeting!

***

## EDA of One Variable: Single Words

So far we've been looking at lines, documents and meetings, but what we'd really like to look at is the words themselves. So let's unnest the words from their lines and remove some "stop words" -- words such as "and" "the" or "a" -- using an `anti_join` from the dplyr package.


{% highlight r %}
# Load a table of 1,149 common stop words
data(stop_words)

# Unnest word tokens and anti_join a table of stop words
gcf_tidy <- gcfboard_docs %>%
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

Even after we removed every instance of common stop words, the new tidy dataframe contains over 2.6 million words! Now let's look at the counts of remaining words, sorted by the number of times they appear in the text. 

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/common words-1.png)

It looks like some words appear very often, in particular words associated with the name of the fund, although the word "green" is relatively less used than are the words "climate" and "fund".

We might want to disambiguate uses of the word "climate" between uses which add important context to our analysis, and uses which just repeat the name of the fund.

To solve this problem, we're going to to see how many times the words "Green Climate Fund" appear in sequence:


{% highlight r %}
# How many times does the trigram "Green Climate Fund" appear?
gcf_name_trigram <- gcfboard_docs %>%
  unnest_tokens(trigram, text, token = "ngrams", n = 3) %>%
  separate(trigram, c("word1", "word2", "word3"), sep = " ") %>%
  filter(word1 == "green",
         word2 == "climate",
         word3 == "fund") %>%
  count(word1, word2, word3, sort = TRUE)
gcf_name_trigram
{% endhighlight %}



{% highlight text %}
## Source: local data frame [1 x 4]
## Groups: word1, word2 [1]
## 
##   word1   word2 word3     n
##   <chr>   <chr> <chr> <int>
## 1 green climate  fund  6310
{% endhighlight %}

Now we can see that 6,310 uses of each word "green", "climate" and "fund" are repetitions of the fund name. So how many uses are there in other contexts?


{% highlight r %}
gcf_name_disambiguation <- gcf_tidy %>%
  count(word, sort = TRUE) %>%
  filter(word == "green" | word == "climate" | word == "fund") %>%
  transmute(word,
         `basic count` = n,
         `name context` = 6310,
         `other contexts` = `basic count` - `name context`)
gcf_name_disambiguation
{% endhighlight %}



{% highlight text %}
## # A tibble: 3 × 4
##      word `basic count` `name context` `other contexts`
##     <chr>         <int>          <dbl>            <dbl>
## 1    fund         26004           6310            19694
## 2 climate         23673           6310            17363
## 3   green          7390           6310             1080
{% endhighlight %}

We can see that "green" occurs rarely in contexts other than the name context. The word "green" is almost never used unless the name of the fund is repeated. 

Now we can find out the relative importance of the word "climate", disambiguated from usage which is just a repetition of the fund's name. 

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/tidycount2-1.png)

Wow! It turns out that the word "climate", independent of usage as part of the name of the Fund, is the third most common word. 

You might wonder why we didn't remove the word "project" since it is so common. This is because frequency of usage has changed a lot over time, unlike the other words we removed.

***

### How has word usage changed over time?




Let's look at changes in usage for a few words which might be interesting to us: "secretariat", "project" and "risk".

Plotting statistical transformations of word frequency can help us see these changes over time. Below you can see the bare frequenciess on top, followed by a log10 scale in the middle, and finally a squre root coordinate transform beneath.


![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/changes-1.png)

All plots show that since the Fund became operational in 2014, use of the word "project" has surged. 

At earlier meetings, establishing a secretariat was priority number one, and we can see usage declining over time. "Risk" begins to become much more important at B.07, which is when the Fund adopted an investment framework, a financial risk management framework, and a results management framework.

You'll notice that the bare frequency plot emphasises the high-frequency changes to the words "secretariat" and "project", whereas the log10 plot of word frequency really helps us see what's happening at the low-end of the frequency range. The square-root coordinate transform preserves some of the perspective of both the other plots, and might be the most useful plot of the three.

***

### Word Frequency

We might want to look at board meetings as a facet of our single word analysis. Which words are particularly associated with specific meeetings? To find this out we can use common rules-of-thumb: term frequency and it's cousin, inverse-document frequency.

Computing these with the `bind_tf_idf` function from the tidytext package, we get a data table which shows the words which are associatd in particular with one meeting. 


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

That's a lot of acronyms! If we look into the board documents themselves, we can see that these acronyms tend to denote real organizations or operational units at the fund. For example:

* At B.16, the top term is "geeref", which is the ["Global Energy Efficiency and Renewable Energy Fund"](http://geeref.com/), a Public-Private Partnership which makes equity investments in clean energy in developing countries.
* At B.06, where the top term is "iiu", this refers to the ["Independent Integrity Unit" (IIU)](http://www.greenclimate.fund/independent-integrity-unit), which is one of three accountability units at the fund. 

This means that relative to other board meetings, the IIU and GEEREF were particularly important topics at the sixth and sixteenth meetings of the board respectively.

***

### Word Correlation

We can also look at correlation in word frequency between pairs of meetings. Words that are common to one meeting are common to another, and cancel each other out, leaving interesting words at the margins.

Let's use a 45 degree dashed line to denote equal frequency in word usage. This means that:

* words above the line appear more commonly in B.16 board documents than in the comparison group, while 
* words below the line appear more frequently in the comparison group than at B.16 (check the label above the plot to find which board meeting is used as the comparison). 

For example, in the first plot below we see (below the line) that "co-chairs" was a big issue at B.01, while it wasn't at B.16. In contrast, B.16 (above the line) had a greater focus on "women", "water" and "projects" than did B.01.

![center](/figs/2017-4-26-exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/frequencies-1.png)

We can also see that:

* B.01 was focused on finding a "host" country and establishing a "secretariat",
* B.10 was relatively more concerned with "accountability" and "accreditation" than was B.16,
* B.15 was relatively more concerned with "applicants" to the fund and contains many references to "samoa", where the meeting was held.

By looking at the spread of the data, we can also see that correlation between word frequencies at B.16 and at earlier meetings increases over time. Between B.16 and B.01 the word frequencies are dispered, while the B.16-B.10 pair is a little less dispersed, and the frequencies for B.15 appear to converge towards those for B.16.

We can confirm this intuition by looking at Pearson's product-moment correlations between B.16 and previous meetings:

* between **B.01 and B.16**: correlation is about 32%

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

* between **B.10 and B.16**: correlation is about 62%

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

* between **B.15 and B.16**: correlation is about 92%

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

That's enough for one post! For more ideas about how to analyze this data set, and [for a lot more on correlation, see my post on ngrams](http://state.gy/r/ngrams_correlation_green_climate_fund/). 

***

### Notes

* **Term frequency (tf)**, measures how frequently a word occurs in a document. However, even after we remove stop words, there are always some words which will occur far more often than others, so it's not much use on its own.

$$tf(\text{word}) = (\frac{n_{\text{word}}}{n_{\text{total words in document}}})$$  

* **Inverse document frequency (idf)** weights the frequency of a word by how rarely it occurs in a set of documents, and increases the weight for words that are seldom seen in the set. You'll notice that since this is a logarithm, an extremely common word would get a rating of zero.  

$$idf(\text{word}) = {\ln{\left(\frac{n_{\text{documents}}}{n_{\text{documents containing term}}}\right)}}$$  

* **Term frequency inverse document frequency (tf-idf)** combines the above two concepts. You can think about this as the frequency of a term in a document weighted by how rarely it is used among a group of documents, with higher weights going to rarely used words, and lower weights to commonly-used words.   
  
$$tfidf(\text{word}) = tf(\text{word}) * idf(\text{word})$$  




