---
categories:
- tidytext
- sentiment
date: "2019-09-14"
showPagination: false
tags:
- tidytext
- twitter
- sentiment analysis
thumbnailImage: /img/serge.jpg
thumbnailImagePosition: left
title: Text sentiment analysis - Twitter and VAR
---

Detailing sentiment analysis on how VAR is portrayed on Twitter

Note: Because of the nature of the tweets there are some swear words within the charts. If you really don't like seeing that sort of thing then this blog post might not be for you!
<!--more-->

Yesterday Leicester City beat Spurs 2-1, but not before two VAR decisions, one being very controversial. 

After 37 minutes Spurs were up 1-0 with a unique goal scored by Harry Kane. In the second half Leicester had a really bright start until Aurier scored for Spurs at 63 minutes. Here's where things get a bit spicy! Auriers goal was disallowed by a VAR check which found Son to be 0.0000001 centimetres offside in the build-up to the goal.

2 minutes later a rejuvinated Leicester score the equaliser, and at 84 minutes Maddison scored the winning goal. 

![maddison](/img/maddison.jpg)

I decided to do some sentiment analysis of twitter reactions to try and understand how the VAR decision in the Leicester V Spurs match was receieved. 

# What's VAR?

VAR stands for Video Assistant Referee and is football's first use of video technology. It was used on the 2018 World Cup as well as the 2019 Women's World Cup. It was introduced to the Premier League at the start of the current 2019-20 season. 

The VAR team consists of three people situated in a central control room who have direct communication with the referee on the pitch. If the referee on the pitch makes a 'clear error' in on awarding goals, penalties, red cards or though providing mistaken identity in awarding a card then VAR can be used to overturn the ruling. 

# Analysis of VAR on twitter 

First load the following libaries:

```
dplyr, twitteR, tidytext, ggplot2
```

Set up a [twitter](https://developer.twitter.com/en/dashboard) application so that you can connect R to Twitter. You'll need a twitter account in order to be able to do this. 

Remember when connecting R to Twitter and retreiving tweets that you are accessing information that twitter users have made public, however not ever twitter user will know that scraping technology exists. Because of this I think it's important to bear in mind the following statement from twitter regarding the use of twitter data:

'Be careful about using Twitter data to derive or infer potentially sensitive 
characteristics about Twitter users (ie. health, political or religious
affiliation, ethnic origin, sexual orientation, and more)'. 

# Using twitteR

Keys can be accessed by going into the twitter app and then selecting 'keys and tokens'.

```
consumer_key <- "your key"
consumer_secret <- "your key"
access_token <- "your key"
access_secret <- "your key"

setup_twitter_oauth(consumer_key, consumer_secret, access_token, access_secret)
```

The foolowing line searches for tweets that have '#VAR' in them and takes the first 3,000 tweets in english.

```
VAR_twitter <- searchTwitter("#VAR",n=3000,lang="en")

VAR_twitter_df <- twListToDF(VAR_twitter) # Convert list to data frame
```
I'm only interested in tweets with '#VAR' for the 22nd September to 23rd September, as I'd like to limit the data to around the same time the Leicester match took place.

Next I'm going to use the `un_nest` function. This seperates the tweets into seperate words, instead of having a whole sentence. 

```
tweet_words <- VAR_twitter_df %>% select(id, text) %>% unnest_tokens(word,text)
```

I want to take some stop words out of my data set so that words that don't add any value to the sentiment analysis like 'the' and 'is' are excluded. 

```
data(stop_words)

tweet_words <- tweet_words %>%
  anti_join(stop_words)
```
There are some stop words that I think are unique to this subject
that aren't worth including as they wont give any sentiment value, like 't.co' so I'm going to add these words to the stop words data.

```
my_stop_words <- stop_words %>% 
  select(-lexicon) %>% 
  bind_rows(data.frame(word = c("https", "t.co", "rt", "i'm","isn't", "var", 
                                "it's", "2", "1", "0", "i'm", "don't")))
```

Now I can apply these additional stop words to my data.

```
tweet_words <- tweet_words %>% anti_join(my_stop_words)
```

# Applying sentiment analysis

I now have the data shaped in a tidy format and I've taken out any words that wont benefit the analysis. Next I'm going to assign sentiment to the words to define if the words are positive or negative. Note that applying sentiments to words in isolation like this isn't foolproof. For example the word 'wow' appears in this data and is assigned a positive sentiment, however this kind of assignment doesn't take into account irony or sarcarm. 

So the sentiment could actually be negative: 'wow don't you just love VAR'. 

There are a few different sentiment datasets, but here I'm going to use 'bing'.

```
bing <- bind_rows(tweet_words %>% 
                            inner_join(get_sentiments("bing")) %>%# join to bing data
                            mutate(method = "Bing et al.")) %>% 
  group_by(word, sentiment) %>% 
  summarise(n = n())
```

Now I have assigned sentiment to each word in the data I can develop a couple of plots to understand how people feel about VAR. 

Firstly let's look at the spread of negative comments against positive comments.

```
bing %>%
  group_by(sentiment) %>% 
  summarise(n = n()) %>%
  mutate(freq = (round(n/sum(n)*100))) %>% 
  ggplot(aes(x = sentiment, y=freq,
             label = (round(freq,1)))) + 
  geom_col(position = "dodge", fill="#FDAE61") +
  geom_text(position = position_dodge(width = .9), vjust=-0.5)+
  theme_classic()+
  #ggtitle("Highest ranking states by proportion of total killed: 2017")+
  xlab("Sentiment")+
  ylab("Proportion") +
  theme(text = element_text(size=15), 
        axis.text.x = element_text(angle = 75, hjust = 1)) 
  ```
![negpos](/img/negpos.jpeg)

The chart shows that 65% of the words have been assigned a negative sentiment. Let's have a closer look at those negative words to get an idea of the kind of things people were saying about VAR over the last few days!

```
bing %>% 
  filter(n >= 20) %>% # words appears more than 20 times
  filter(sentiment == "negative") %>% 
  ggplot(aes(x = reorder(word, -n), y=n,
             label = (round(n,1)))) +
  geom_col(position = "dodge", fill="#4d52e8") +
  geom_text(position = position_dodge(width = .9), vjust=-0.5)+
  theme_classic()+
  #ggtitle("Highest ranking states by proportion of total killed: 2017")+
  xlab("word")+
  ylab("") +
  theme(text = element_text(size=18), 
        axis.text.x = element_text(angle = 75, hjust = 1)) 
```
![Negative](/img/Negative.jpeg)

Here we can see that the word 'joke' appears the most in the selected tweets, followed by 'fucking' and 'ruining'. 

