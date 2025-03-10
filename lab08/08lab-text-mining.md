Lab 08 - Text Mining/NLP
================

# Learning goals

- Use `unnest_tokens()` and `unnest_ngrams()` to extract tokens and
  ngrams from text
- Use dplyr and ggplot2 to analyze and visualize text data
- Try a theme model using `topicmodels`

# Lab description

For this lab we will be working with the medical record transcriptions
from <https://www.mtsamples.com/> available at
<https://github.com/JSC370/JSC370-2025/tree/main/data/medical_transcriptions>.

# Deliverables

1.  Questions 1-7 answered, knit to pdf or html output uploaded to
    Quercus.

2.  Render the Rmarkdown document using `github_document` and add it to
    your github site. Add link to github site in your html.

### Setup packages

You should load in `tidyverse`, (or `data.table`), `tidytext`,
`wordcloud2`, `tm`, and `topicmodels`.

``` r
install.packages(c("tidyverse", "tidytext", "wordcloud2", "topicmodels"))
install.packages("tm", dependencies=TRUE)
install.packages("stopwords")
```

## Read in the Medical Transcriptions

Loading in reference transcription samples from
<https://www.mtsamples.com/>

``` r
library(tidytext)
library(tidyverse)
library(wordcloud2)
library(tm)
library(topicmodels)
library(stopwords)

mt_samples <- read_csv("https://raw.githubusercontent.com/JSC370/JSC370-2025/main/data/medical_transcriptions/mtsamples.csv")
mt_samples <- mt_samples |>
  select(description, medical_specialty, transcription)

head(mt_samples)
```

## Question 1: What specialties do we have?

We can use `count()` from `dplyr` to figure out how many different
medical specialties are in the data. Are these categories related?
overlapping? evenly distributed? Make a bar plot.

``` r
mt_samples |>
  count(medical_specialty, sort = TRUE) |>
  ggplot(aes(x = reorder(medical_specialty, n), y = n)) +
  geom_col(fill = "purple") +
  coord_flip() +  
  labs(
    title = "Distribution of Medical Specialties in Transcriptions",
    x = "Medical Specialty",
    y = "Number of Transcriptions"
  ) +
  theme_minimal()

mt_samples |>
  distinct(medical_specialty) |>
  nrow()
```

There are 30 different medical specialties. They don’t seem to be evenly
distributed by the bar plot, as we can see that Surgery has the
most/highest frequency with more than 900 transcriptions. The rest have
only range from 0 to more than 300 transcriptions.

## Question 2: Tokenize

- Tokenize the the words in the `transcription` column
- Count the number of times each token appears
- Visualize the top 20 most frequent words with a bar plot
- Create a word cloud of the top 20 most frequent words

### Explain what we see from this result. Does it makes sense? What insights (if any) do we get?

``` r
tokens <- mt_samples |>
  select(transcription) |>
  unnest_tokens(word, transcription) |>
  count(word, sort = TRUE)

tokens |>
  slice_max(n, n = 20) |>
  ggplot(aes(x = reorder(word, n), y = n)) +
  geom_bar(stat = "identity", fill = "purple") +
  coord_flip() +
  labs(
    title = "Top 20 Most Frequent Words in Medical Transcriptions",
    x = "Word",
    y = "Frequency"
  ) +
  theme_minimal()

tokens |>
  slice_max(n, n = 20) |>
  wordcloud2()
```

We see that top 20 words don’t make sense; “the”, “and”, “was”, “of” and
so on, they’re stopwords rather than medical terms/words. Right now, we
don’t have much insights because of the stopwords.

## Question 3: Stopwords

- Redo Question 2 but remove stopwords
- Check `stopwords()` library and `stop_words` in `tidytext`
- Use regex to remove numbers as well
- Try customizing your stopwords list to include 3-4 additional words
  that do not appear informative

### What do we see when you remove stopwords and then when you filter further? Does it give us a better idea of what the text is about?

``` r
head(stopwords("english"))
length(stopwords("english"))
head(stop_words)

custom_stopwords <- c("patient", "history", "medical", "normal", "left", "mm", "mg", "pain")  

tokens <- mt_samples |>
  select(transcription) |>
  unnest_tokens(word, transcription) |>
  filter(!word %in% stop_words$word) |>  # Remove common stopwords
  filter(!word %in% stopwords("english")) |>  # Remove additional stopwords
  filter(!str_detect(word, "^[0-9]+$")) |>  # Remove numbers
  filter(!word %in% custom_stopwords) |>  
  count(word, sort = TRUE)

tokens |>
  slice_max(n, n = 20) |>
  ggplot(aes(x = reorder(word, n), y = n)) +
  geom_bar(stat = "identity", fill = "purple") +
  coord_flip() +
  labs(
    title = "Top 20 Words (Stopwords & Numbers Removed)",
    x = "Word",
    y = "Frequency"
  ) +
  theme_minimal()

print(tokens |>
  slice_max(n, n = 20) |>
  wordcloud2()
```

After removing stopwords and numbers, the result makes more sense
because words like “the,” “and,” “was”, “of” etc. no longer dominate the
results. We can see more medical terms appear. This gives us a better
understanding of the transcriptions. Also, removing extra words like
“patient”, “history”, and I did remove some measurements (mm, mg) and
others helps highlight other important terms that provide more insight
into the text, like now we see vicryl, lateral, incision, anesthesia,
etc.

## Question 4: ngrams

Repeat question 2, but this time tokenize into bi-grams. How does the
result change if you look at tri-grams? Note we need to remove stopwords
a little differently. You don’t need to recreate the wordclouds.

``` r
stopwords2 <- c(stopwords::stopwords("en"), custom_stopwords)

sw_start <- paste0("^", paste(stopwords2, collapse=" |^"), "$")
sw_end <- paste0("", paste(stopwords2, collapse="$| "), "$")

tokens_bigram <- mt_samples |>
  select(transcription) |>
  unnest_tokens(ngram, transcription, token = "ngrams", n = 2) |>
  filter(!str_detect(ngram, sw_start)) |>
  filter(!str_detect(ngram, sw_end)) |>
  filter(!str_detect(ngram, "\\d+"))

tokens_bigram |>
  count(ngram, sort = TRUE) |>
  slice_max(n, n = 20) |>
  ggplot(aes(x = reorder(ngram, n), y = n)) +
  geom_bar(stat = "identity") +
  coord_flip() +
  labs(title = "Top 20 Bigrams", x = "Bigram", y = "Count")

tokens_trigram <- mt_samples |>
  select(transcription) |>
  unnest_tokens(ngram, transcription, token = "ngrams", n = 3) |>
  filter(!str_detect(ngram, sw_start)) |>
  filter(!str_detect(ngram, sw_end)) |>
  filter(!str_detect(ngram, "\\d+"))

tokens_trigram |>
  count(ngram, sort = TRUE) |>
  slice_max(n, n = 20) |>
  ggplot(aes(x = reorder(ngram, n), y = n)) +
  geom_bar(stat = "identity") +
  coord_flip() +
  labs(title = "Top 20 Trigrams", x = "Trigram", y = "Count")
```

Bigram phrases like “year old,” “operating room,” and “preoperative
diagnosis” give general information about the patient, location, and
medical condition. Trigram phrases like “patient was prepped,” “incision
was made,” and “tolerated the procedure” offer more specific details
about what happened during the procedure. Trigrams help us understand
the flow of events, while bigrams give us a broader idea of the key
topics.

## Question 5: Examining words

Using the results from the bigram, pick a word and count the words that
appear before and after it, and create a plot of the top 20.

``` r
library(stringr)

words_before <- tokens_trigram |>
  filter(str_detect(ngram, "\\w+ preoperative diagnosis")) |>
  mutate(word = str_remove(ngram, " preoperative diagnosis.*"),
         word = str_remove_all(word, "[[:punct:]]"),
         position = "Words before") |>
  count(word, position, sort = TRUE) |>
  slice_max(n, n = 20)

words_after <- tokens_trigram |>
  filter(str_detect(ngram, "preoperative diagnosis \\w+")) |>
  mutate(word = str_remove(ngram, ".*preoperative diagnosis "),
         word = str_remove_all(word, "[[:punct:]]"),
         position = "Words after") |>
  count(word, position, sort = TRUE) |>
  slice_max(n, n = 20)

words_combined <- bind_rows(words_before, words_after)

words_combined |>
  ggplot(aes(x = reorder(word, n), y = n)) +
  geom_bar(stat = "identity") +
  coord_flip() +
  facet_wrap(~ position, scales = "free_y") +
  labs(title = "Top 20 Words Before and After 'preoperative diagnosis'", 
       x = "Word", 
       y = "Count") +
  theme_minimal()
```

## Question 6: Words by Specialties

Which words are most used in each of the specialties? You can use
`group_by()` and `top_n()` from `dplyr` to have the calculations be done
within each specialty. Remember to remove stopwords. How about the 5
most used words?

``` r
# most used in each
mt_samples |>
  unnest_tokens(word, transcription) |>
  filter(!word %in% stop_words$word) |>           
  filter(!word %in% stopwords("english")) |>     
  filter(!str_detect(word, "^[0-9]+$")) |>       
  filter(!word %in% custom_stopwords) |>         
  count(medical_specialty, word, sort = TRUE) |>
  group_by(medical_specialty) |>
  top_n(1) |>
  ungroup()

# top 5
mt_samples |>
  unnest_tokens(word, transcription) |>
  filter(!word %in% stop_words$word) |>           
  filter(!word %in% stopwords("english")) |>     
  filter(!str_detect(word, "^[0-9]+$")) |>       
  filter(!word %in% custom_stopwords) |>         
  count(medical_specialty, word, sort = TRUE) |>
  group_by(medical_specialty) |>
  top_n(5) |>
  arrange(medical_specialty, desc(n))
```

We can see some of the medical specialties has most used word
“procedure”. Others seem expected like the name of the medical specialty
(ex. Urology has “bladder” and Gynecology has “uterus).

## Question 7: Topic Models

See if there are any themes in the data by using a topic model (LDA).

- you first need to create a document term matrix
- then you can try the LDA function in `topicmodels`. Try different k
  values.
- create a facet plot of the results from the LDA (see code from
  lecture)

``` r
transcripts_dtm <- mt_samples |>
  select(transcription) |>
  unnest_tokens(word, transcription) |>
  filter(!word %in% stop_words$word) |>           
  filter(!word %in% stopwords("english")) |>     
  filter(!str_detect(word, "^[0-9]+$")) |>       
  filter(!word %in% custom_stopwords) |>          
  count(document = row_number(), word) |>         
  cast_dtm(document, word, n)

transcripts_lda <- LDA(transcripts_dtm, k = 4, control = list(seed = 1234))

lda_top_terms <- tidy(transcripts_lda, matrix = "beta") |>
  group_by(topic) |>
  slice_max(beta, n = 10) |>    # Select top 10 terms per topic
  ungroup() |>
  arrange(topic, -beta)

lda_top_terms |>
  mutate(term = reorder_within(term, beta, topic)) %>%
  ggplot(aes(beta, term, fill = factor(topic))) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~ topic, scales = "free") +
  theme_bw() +
  scale_y_reordered() +
  labs(title = "Top Terms for Each Topic", x = "Beta", y = "Term")
```

Here I do 4 topics. In each we can see the top 10 terms.
