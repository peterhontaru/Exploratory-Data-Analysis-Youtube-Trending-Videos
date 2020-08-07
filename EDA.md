# YouTube Trending Videos

## Table of contents:
 1. Background
 2. Data tidy
 3. Analysis
        3.1 Trending date
                3.1.1 How many videos?
        3.2 Views
        

# 1) Background

### Why this dataset?
YouTube has been a massive influence and source of knowledge both in my professional life as well in my personal goals/ambitions. Thus, I thought it would be interesting to explore this dataset and see what insights I could come up with

### What are Trending Videos and why are they important? 
Trending videos work alongside the home page to provide users with content to watch. 

While the Home feed is **highly** personalised on previous views, what the user watched longest, engagement, subscriptions, the trending page is very broad and is the same across all accounts.

Since it displays this across hundreds of thousands of accounts, this makes it a great source of views for content creators.

### Purpose of this analysis:
1. Develop an understanding of the YouTube Trending Videos
2. Assess differences between English-speaking countries (Canada, Great Britain, USA)
                                      
**Thus, all throughout, we will generate questions, answer them with data and then further refine these questions based on what we found out.**

### To whom might this be helpful?
*content creators / marketing agencies can get a better understanding of the audience which would help tailor content
*data science enthusiasts as they might get new ideas for how to use R (for beginners) or see other people's analyses (for intermediates)
*people that are generally interested in YouTube





## Key Insights:
* CA tends to have videos that only trend for a very short period, while in the GB/US they typically trend significantly longer (up to 38 days)





## Dataset information:
* Trending data between "2017-11-14" and "2018-06-14" for US, CA and GB
* Contains > 120,000 videos information in the tree countries


#videos by day                    #total videos




# 2) Data import, tidying, cleaning

The following packages were needed to produce the output:
``` r
library(ggplot2)
library(dplyr)
library(lubridate)
library(data.table)
library(readr)
library(rjson)
library(jsonlite)
library(ggcorrplot)
```

Download data:

All the data is downloaded from <https://www.kaggle.com/datasnaek/youtube-new>.

Raw data files are available within the "Datasets" folder.

If you'd like to skip the data importing, manipulating and cleaning, a csv file of cleaned and parsed data is also included under "raw_data.csv".

### 2.1. Import

``` r
#get datasets for the countries we're interested in
gb_data <- read_csv("~/DS/YouTube - EDA/Datasets/GBvideos.csv")
us_data <- read_csv("~/DS/YouTube - EDA/Datasets/USvideos.csv")
ca_data <- read_csv("~/DS/YouTube - EDA/Datasets/CAvideos.csv")

#add a flag so that we know which country belongs to which dataset
gb_data$country <- "GB"
us_data$country <- "US"
ca_data$country <- "CA"

#get category data
us_cat_json <- fromJSON("~/DS/YouTube/US_category_id.json")
gb_cat_json <- fromJSON("~/DS/YouTube/GB_category_id.json")
ca_cat_json <- fromJSON("~/DS/YouTube/CA_category_id.json")
summary(us_cat_json)

#bind together
US_category <-  as.data.frame(cbind(us_cat_json[["items"]][["id"]], us_cat_json[["items"]][["snippet"]][["title"]]))
GB_category <-  as.data.frame(cbind(gb_cat_json[["items"]][["id"]], gb_cat_json[["items"]][["snippet"]][["title"]]))
CA_category <-  as.data.frame(cbind(ca_cat_json[["items"]][["id"]], ca_cat_json[["items"]][["snippet"]][["title"]]))

#change column names
names(US_category) <- c("category_id","category_title")
names(GB_category) <- c("category_id","category_title")
names(CA_category) <- c("category_id","category_title")

#merge data
us_data <- merge(x = us_data, y = US_category, by = "category_id")
gb_data <- merge(x = gb_data, y = GB_category, by = "category_id")
ca_data <- merge(x = ca_data, y = CA_category, by = "category_id")

#combine into one dataset and remove previous variables from memory to keep the environment clean
raw_data <- as.data.table(rbind(gb_data, us_data, ca_data))

rm(us_data, gb_data, ca_data)
rm(US_category, GB_category, CA_category)
rm(us_cat_json, gb_cat_json, ca_cat_json)

```

### 2.2. Clean and Tidy Data

```r
#I usually like to save a backup of the raw data to make it easier to revert changes so let's do that
raw_data_backup <- raw_data

#let's check the structure of the data
str(raw_data)
#we can spot a few class problems as well as add some additional columns

#clean and format dates/times
raw_data$trending_date <- ydm(raw_data$trending_date)
raw_data$publish_date <- ymd(substr(raw_data$publish_time,start = 1,stop = 10))
raw_data$hour <- format(strptime(raw_data$publish_time,"%Y-%m-%d %H:%M:%S"),'%H')

raw_data$days_diff <- as.numeric(raw_data$trending_date-raw_data$publish_date)

#remove unnecessary data
raw_data <- raw_data %>%
        select(-description, -tags, -category_id, -publish_time)

#add new columns for further analysis
raw_data <- raw_data %>%
        mutate(perc_engagement = round((likes + dislikes + comment_count) / views, digits = 2)*100,
               perc_likes = round(likes / (likes+dislikes), digits=2)*100,
               perc_comments = round(comment_count/views, digits = 2) * 100)

#we also want to change hour and country fields to factors
raw_data$hour <- as.factor(raw_data$hour)
raw_data$country <- as.factor(raw_data$country)

#remove video_error_or_removed videos as we do not want these
table(raw_data$video_error_or_removed)

#it's a very small dataset so we will just remove without looking into these too much as they were deleted/errors/copyright violations
video_error_or_removed <- raw_data %>% 
        filter (video_error_or_removed == "TRUE")%>%
        select(country, channel_title, title, video_id) %>%
        group_by(country, channel_title, title, video_id) %>%
        summarize(count = n()) %>% #count = days_trending
        arrange(desc(count)) %>%
        print()

raw_data <- raw_data %>%
        filter (video_error_or_removed == "FALSE") %>%
        select (-video_error_or_removed)
        
rm(video_error_or_removed)

#-----------------------------------------CHECK FOR NAs
raw_data %>% summarise_all(~ sum(is.na(.)))
#is this because of ratings being disabled?
raw_data %>% filter(is.na(perc_likes)) %>% group_by(ratings_disabled) %>% summarize (count = n())
#it looks like that's the case for al but 3 of them. Those are na due to not having any likes/dislikes so we will adjust the calculation slightly
raw_data <- raw_data %>% mutate(perc_likes = ifelse(ratings_disabled == "FALSE" & likes==0, 0, round(likes / (likes+dislikes), digits=2)*100))

raw_data %>% filter(is.na(perc_likes)) %>% group_by(ratings_disabled) %>% summarize (count = n())
#this error has been fixed now and NAs are only showing for those with ratings disabled

#-----------------------------------------CHECK FOR NULLS
raw_data %>% summarise_all(~ sum(is.null(.)))
#no issues here

#back up our existing data into raw_data_backup
raw_data_backup <- raw_data
```


### 2.3. Data understanding

At this point, there are a few columns that we need to look into further as I am not fully confident I understand: 
* comments_disabled
* ratings_disabled
* hour

``` r

#comments_disabled
table(raw_data$comments_disabled)
table(quantile(raw_data$comments_disabled, probs = seq(0, 1, length.out=101)))
#~1% of the data has comments disabled

#ratings_disabled
table(raw_data$ratings_disabled)
table(quantile(raw_data$ratings_disabled, probs = seq(0, 1, length.out=101)))
#<1% of the data has ratings disabled

#although they're not a significant chunk of the dataset, we won't remove these from the raw data, as these are genuine videos
#it is likely that these might be more controversial and we will look at this later in the data exploration process
#the dataset is before the rule with videos aimed to children will not be allowed comments so that's not a factor
#however, they will be removed from engagement-specific analysis as non existent data (0s would act as outliers)


#-----------------------------------------COMMENTS DISABLED ANALYSIS
#1
table(raw_data$comments_disabled)

#2
#facet by country but not by category
comments_disabled_by_country <- raw_data %>%
  select (country, likes, dislikes, comments_disabled) %>%
  group_by (country, comments_disabled) %>%
  summarize(count = n(),
            dislikes_perc = round(sum(dislikes)/(sum(likes)+sum(dislikes))*100,0)) %>%
  print()

ggplot(comments_disabled_by_country, aes(country, dislikes_perc , fill = comments_disabled)) + 
  geom_col(position = "dodge")

#potentially some controversial videos choose to take comments down to avoid backlash (particularly in news and entertainment)
#these look to be more dislikes, on average than likes (but sample size is also much smaller so exceptions are likely to skew data)


#-----------------------------------------RATINGS DISABLED ANALYSIS

ratings_disabled <- raw_data %>% 
  filter (ratings_disabled == "TRUE") %>%
  select (category_title, comment_count) %>%
  group_by (category_title) %>%
  summarize(count = n()) %>%
  arrange(desc(count)) %>%
  print

#remove from memory
rm(ratings_disabled, comments_disabled_by_country)
```

Part 2 Analysis
---------------

If the Tweets from the UK has already been extracted and saved as 'tweets.csv', then start the project from there. Analyze the sentiment of the tweets using qdap package, each tweet is broken into sentences and fed to QDAP to perform sentiment analysis. QDAP returns a number which represent the sentiment of the text. A positive value represents positive sentiment and vice versa for negative value. The absolute value of the sentiment value represents the strength of the sentiment.

``` r
require(qdap)
tweets=read.csv("tweets.csv", header = TRUE, stringsAsFactors = FALSE )
#take a look at the first few lines of tweets
head(tweets$text)
```

    ## [1] "Used car recently added: #NISSAN #NAVARA only £5980 http://t.co/MYU4CFeG"                            
    ## [2] "Hey Monday"                                                                                          
    ## [3] "Horrible, horrible nightmare. Also, my alarm is a bastard."                                          
    ## [4] "@BeeStrawbridge a day to disconnect and reconnect to what's important - an excellent idea. Thank you"
    ## [5] "Used car recently added: #VOLKSWAGEN #GOLF only £695 http://t.co/lsGgIVEN"                           
    ## [6] "http://t.co/whsQJLMQ"

Here are the first few tweets of the day. Good, We spotted one negative(\#3) and one positive(\#4) tweets already

``` r
#create data frame
tweets=data.frame(time=tweets$created_at, text=tweets$text, sentiment=NA, postw=0, negtw=0, tottw=1)

#Detect sentiment for each tweet
for(i in 1:nrow(tweets)){

  }
}

#remove tweets that qdap() cannot distinguish
tweets=tweets[!is.nan(tweets$sentiment)&!is.na(tweets$sentiment),]
```



 further datasets
 
 
 
## Knowing all these, these are the next steps I'd take (not part of current scope)
### 1) Collect some of the following public metrics: 
* subscribers (if made public) as they could explain
* other social media profiles (high fan base could contribute to this engagement)
* reshares (reddit, facebook, etc)
* length of video
* SEO quality (nice but possibily not as important)
### 2) Would also help to have the following private metrics: usually only the individual accounts have access to these
* watch time and % of video watched (the importance and how this related to ads/revenue)
* % click through rate


## Other uses for this dataset:
* Sentiment analysis on the comment section
* Clustering analyses to analyse different types of videos
* Analysing what factors affect how popular a YouTube video will be
* Statistical analysis to understand which factors contributed to increased engagement (views/likes)