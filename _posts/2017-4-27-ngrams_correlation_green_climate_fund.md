---
layout: post
title:  "N-grams: The Green Climate Fund"
date:   2017-4-27 12:55:01 +0900
category: r
tags: [r]
comments: true
---






Last time we looked at a [single-word analysis](http://state.gy/r/exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/) of the gcfboardr dataset. However, more interesting avenues for analysis open up when we consider relationships between words, in particular between pairs of words.

In this post, we'll look at bigrams and use network graphs to plot relationships among many words.

***

### N-grams

A bigram is a pair of words which occur together in sequence, and a trigram is the same concept extended to word triplets. "N-grams" are a generalization of this concept, although in practice most analyses restrict the "n" to a maximum of three. 

Let's load the dataset:


{% highlight r %}
# Load libraries and the gcfboardr data set
library(dplyr)
library(tidytext)
library(ggplot2)
library(gcfboardr)
data("gcfboard_docs")
{% endhighlight %}

And let's unnest bigrams from the gcfboardr dataset using the `unnest_tokens` function from the tidytext package. 


{% highlight r %}
# Unnest bigrams
bigrams_filtered <- gcfboard_docs %>%
  unnest_tokens(bigram, text, token = "ngrams", n = 2) %>% 
  separate(bigram, c("word1", "word2"), sep = " ") %>% 
  filter(!word1 %in% stop_words$word) %>%
  filter(!word2 %in% stop_words$word)

# Count co-occurences of the two separated words
bigram_counts <- bigrams_filtered %>% 
  count(word1, word2, sort = TRUE)

bigram_counts 
{% endhighlight %}



{% highlight text %}
## Source: local data frame [315,893 x 3]
## Groups: word1 [23,216]
## 
##       word1     word2     n
##       <chr>     <chr> <int>
## 1   climate    change  7277
## 2   climate      fund  6433
## 3     green   climate  6430
## 4   private    sector  5260
## 5   funding  proposal  4884
## 6   project programme  4029
## 7    agenda      item  3351
## 8      fund   funding  2969
## 9  proposal      page  2840
## 10  interim   trustee  2542
## # ... with 315,883 more rows
{% endhighlight %}

It looks like the most common bigram in the gcfboardr data set is (surprise surprise!) "climate change". Now that we've removed stop words, we can unite the word pairs back into a single column, using the `unite` function from the tidyr package:


{% highlight r %}
# Unite the counts into bigram terms
bigrams_united <- bigrams_filtered %>%
  unite(bigram, word1, word2, sep = " ") %>% 
  select(bigram, meeting, title)
bigrams_united
{% endhighlight %}



{% highlight text %}
## # A tibble: 1,276,503 × 3
##                  bigram meeting                                      title
## *                 <chr>  <fctr>                                     <fctr>
## 1      additional rules    B.01 Additional Rules of Procedure of the Board
## 2             board gcf    B.01 Additional Rules of Procedure of the Board
## 3        august meeting    B.01 Additional Rules of Procedure of the Board
## 4          board august    B.01 Additional Rules of Procedure of the Board
## 5         august geneva    B.01 Additional Rules of Procedure of the Board
## 6    geneva switzerland    B.01 Additional Rules of Procedure of the Board
## 7    switzerland agenda    B.01 Additional Rules of Procedure of the Board
## 8           agenda item    B.01 Additional Rules of Procedure of the Board
## 9              item gcf    B.01 Additional Rules of Procedure of the Board
## 10 recommended decision    B.01 Additional Rules of Procedure of the Board
## # ... with 1,276,493 more rows
{% endhighlight %}

Now out data is tidy!

***

### Bigram Frequencies

As well as for single words frequencies, we can also use Term-Frequency-Inverse-Document-Frequency (see the [notes here for definitions](http://state.gy/r/exploratory_data_analysis_of_board_documents_at_the_green_climate_fund/)) as a rule of thumb to work out which bigrams are particularly associated with particular meetings.


{% highlight r %}
# Which bigrams are particularly associated with a specific board meeting?
bigram_tf_idf <- bigrams_united %>%
  count(meeting, bigram) %>%
  bind_tf_idf(bigram, meeting, n) %>%
  arrange(desc(tf_idf))

plot_bigram_tf_idf <- bigram_tf_idf %>%
  arrange(desc(tf_idf)) %>%
  mutate(bigram = factor(bigram, levels = rev(unique(bigram))))
{% endhighlight %}

Let's plot top bigrams by frequency faceting to select a few meetings:  



![center](/figs/2017-4-27-ngrams_correlation_green_climate_fund/faceted_bigram_tfidf-1.png)

We see that B.02 was particularly concerned with finding a "host city", while B.12 contained a lot of references to "comments/inputs" and B.16 considered a lot of proposals and projects!

***

### "Risk" Bigrams

We can also use bigrams to home in an a single word of interest and look at many similar bigrams which include that word. For example, which risks does the GCF write about the most?

Let's start by looking at bigram counts in which the second word is "risk":

![center](/figs/2017-4-27-ngrams_correlation_green_climate_fund/riskcount-1.png)

It looks like the Fund cares about "ES risk" -- environmental and social risk -- almost twice as much as any other risk, or at least they have devoted considerable space to such risks in their board documents. We could even make a case that bigrams such as "social risk" and "climate risk" are a subset of broader discussions about "ES risk", which would make "ES Risk" over three times more important than the second most discussed risk: financial risk. 

What about bigrams in which risk is the first word? 

![center](/figs/2017-4-27-ngrams_correlation_green_climate_fund/riskbigrams2-1.png)

Interesting! Risk bigrams in which the word "risk" comes first tend to refer to aspects of risk management at the fund, whereas when "risk" is the second word, the bigram generally refers to a specific type of risk.

***

### Correlation 

Knowing how often words appear together is neat but not very useful because some words, such as "project", appear so often that it's not clear whether a bigram involving them can tell us anything. One example you may have noticed above is "project page" -- it occurs often and it is meaningless.

Avoiding such problematic words when we look at bigrams involves asking how often words appear together relative to how often they appear separately. 

We can provide an answer by checking correlation between word pairs using the `pairwise_cor` function from the widyr package:


{% highlight r %}
# load widyr
library(widyr)

# hack a lookup table with documents as sections
lookup <- gcfboard_docs %>% 
  group_by(meeting, title) %>%
  summarise(key = n()) %>% 
  ungroup()
lookup$key <- rep(1:nrow(lookup))

# create a tidy df of words in each document
gcf_section_words <- gcfboard_docs %>%
  left_join(lookup) %>%
  select(-meeting, -title) %>% 
  unnest_tokens(word, text) %>%
  filter(!word %in% stop_words$word)
{% endhighlight %}

`pairwise_cor` checks how often a word appears in any single section of a corpus -- in this case, we're using documents to represent sections -- *with* other words, compared to how often it appears in any single section *without* those other words.


{% highlight text %}
## # A tibble: 1,328,234 × 3
##         item1      item2 correlation
##         <chr>      <chr>       <dbl>
## 1  immunities privileges   0.9816426
## 2  privileges immunities   0.9816426
## 3     society      civil   0.9609286
## 4       civil    society   0.9609286
## 5       shift   paradigm   0.9465226
## 6    paradigm      shift   0.9465226
## 7     learned    lessons   0.8952116
## 8     lessons    learned   0.8952116
## 9      sector    private   0.8855799
## 10    private     sector   0.8855799
## # ... with 1,328,224 more rows
{% endhighlight %}

It looks like the words "paradigm" and "shift" occur in the same document together 95% of the time, and only occur apart about 5% of the time. 

Now that we have a word correlation table, we can look up specific correlations among words. For example, let's look at pairwise correlations for the words "risk", "social" and "private".

![center](/figs/2017-4-27-ngrams_correlation_green_climate_fund/wordcors-1.png)

"Private" and "sector" are strongly correlated and tend to appear together about 85% of the time.

***

### Plotting Correlation with Network Graphs

We can also look at pairwise correlation among words using network graphs. In the following network graphs, every point is a "node" -- and in this case they represent words or bigrams. Every line or arc represents an "edge", which represents a relationship between two nodes.

To see what this means, let's plot pairwise correlations using the word correlations data table we built previously. Let's represent the strength of the correlation between two words by both the thickness and colour of the "edges" -- in this case blue lines -- connecting each word. In the following graph, a thick dark blue edge denotes, such as exists between the words "priveleges" and "immunities" denotes strong correlation (above 95%). Thinner lighter-blue edges represent relatively weaker correlation.


{% highlight r %}
library(ggraph)
set.seed(6789)
word_cors %>%
  filter(correlation > .75) %>%
  graph_from_data_frame() %>%
  ggraph(layout = "fr") +
  geom_edge_link(aes(edge_alpha = correlation, edge_width = correlation), edge_colour = "royalblue") +
  geom_node_point(size = 3) +
  geom_node_text(aes(label = name), repel = TRUE,
                 point.padding = unit(0.6, "lines")) +
  theme_void()
{% endhighlight %}

![center](/figs/2017-4-27-ngrams_correlation_green_climate_fund/networkgraph-1.png)

It looks like the word "ecosystem" is strongly related to a cluster of many other words (in the bottom left of the diagram above), which relate to the core work of the Fund. No other word has so many edges emerging from it.

***

Now let's use network graphs to look at pairwise correlation among different bigrams.




{% highlight r %}
set.seed(1234)
bigram_cors %>%
  filter(correlation > .79) %>%
  graph_from_data_frame() %>%
  ggraph(layout = "fr") +
  geom_edge_link(aes(edge_alpha = correlation, edge_density = correlation), edge_colour = "royalblue") +
  geom_node_point(size = 2) +
  geom_node_text(aes(label = name), repel = TRUE,
                 point.padding = unit(0.5, "lines")) +
  theme_void()
{% endhighlight %}

![center](/figs/2017-4-27-ngrams_correlation_green_climate_fund/networkgraph2-1.png)

Fascinating! It looks like we have two distinct clusters among these pairwise correlations, indicating that some bigrams belong to documents of a specific type. Can you guess what they are? 

One cluster above includes bigrams such as:

* "detailed project",
* "appraisal summary",
* "expected performance" and
* "results monitoring".

These bigrams match the kind of terms we might expect to find in a cluster of documents relating to **decisions on project funding proposals**. 

The other cluster contains bigrams such as:

* "accreditation assessment",
* "es risk",
* "financial exposure" and
* "applicant provided".

These are the kind of terms we'd expect to see in a cluster relating to **decisions on accreditation**.

There are also a couple of smaller clusters: one relating to bigrams such as "fiduciary standard" and "funding allocation". Another one contains bigrams such as "climate resilient", "paradigm shift" and "sustainable development". It's a little harder to tell what these clusters represent because we have fewer "edges" to reason with.

***

### Wrap Up

We looked at bigrams in the gcfboardr data set and discovered that the most common bigram is "climate change", while the most commonly discussed risk is "ES Risk" (environmental and social risk). The word "ecosystem" appears to have a greater number of strongly correlated edges than any other word, and we see it connecting to nodes such as "livelihoods", "rural", "agriculture", "infrastructure", "conservation" and "resilience".

Meanwhile, creating network graphs from pairwise bigram correlations uncovered two distinct clusters of nodes: one corresponding to accreditation decisions and another to funding decisions. 

There were a few smaller clusters and we'll get onto them in the next post, when we'll practice topic modelling algorithms. Standby for the link!

For now, here are the top 2,400+ bigram edges in the gcfboardr dataset:

![center](/figs/2017-4-27-ngrams_correlation_green_climate_fund/bcors2-1.png)






