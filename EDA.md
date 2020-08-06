---
title: "YouTube Trending Videos - Exploratory Data Analysis (EDA)"
output: github_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## GitHub Documents

This is an R Markdown format used for publishing markdown documents to GitHub. When you click the **Knit** button all R code chunks are run and a markdown file (.md) suitable for publishing to GitHub is generated.

## Including Code

You can include R code in the document as follows:

```{r cars}
summary(cars)
```

## Including Plots

You can also embed plots, for example:

```{r pressure, echo=FALSE}
plot(pressure)
```

Note that the `echo = FALSE` parameter was added to the code chunk to prevent printing of the R code that generated the plot.


Adding libraries

library(ggplot2)
library(dplyr)
library(lubridate)
library(data.table)
library(readr)
library(rjson)
library(jsonlite)
library(ggcorrplot)


#----------------------------------------------------------------------------------------------------BACKGROUND
#Why - passionate about entertainment/social media - video
#Who would benefit: content creators, marketing agencies, youtube as a business
#give an overview of these videos, further indepth analysis could follow based on outcomes (ie. trend by category)

#what are trending videos? Something that works alongside the home page to provide users with content to watch. While the Home feed is 
#HIGHLY personalised on previous views, what the user watched longest, engagement, subscriptions, the trending page is very broad 
#as it shows what many other people tend to watch. This makes it a great platform to gain a very large audience quickly


#Background: why are trending videos important/


#1. Trending data between "2017-11-14" and "2018-06-14" for US, CA and GB

#EDA: Purpose is to explore the data and 
              #1. develop understanding of the data
              #2. assess differences between english speaking countries 
                                        #(given that their culture/interests might be similar than let's say someone from JPN or RUS and data availability)

#Thus, all throughout, we will generate questions, answer them with data and then further refine these questions based on what we found out

#Knowing all these, these are the next steps I'd take (not part of current scope)
      #Collect some of the following public metrics: 
          #subscribers (if made public) as they could explain
          #other social media profiles (high fan base could contribute to this engagement)
          #reshares (reddit, facebook, etc)
          #length of video
          #SEO quality (nice but possibily not as important)
      #Would also help to have the following private metrics: usually only the individual accounts have access to these
          #watch time and % of video watched (the importance and how this related to ads/revenue) - customer satisfaction
      #Other analyses to further depth from my analysis:
          #string search and read through comments to understand sentiment/reactions
          ##See which factors contributed the most, further indepth analyses on those by segments

#Other uses, different scope:
    #Sentiment analysis in a variety of forms
    #Categorising YouTube videos based on their comments and statistics.
    #Training ML algorithms like RNNs to generate their own YouTube comments.
    #Analysing what factors affect how popular a YouTube video will be.
    #Statistical analysis over time.



#----------------------------------------------------------------------------------------------------IMPORT DATA
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

#combine into one dataset and remove previous variables from memory
raw_data <- as.data.table(rbind(gb_data, us_data, ca_data))

rm(us_data, gb_data, ca_data)
rm(US_category, GB_category, CA_category)
rm(us_cat_json, gb_cat_json, ca_cat_json)



#-----------------------------------------------------------------------------------------------------CLEAN AND TIDY DATA
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

#back up our existing data into raw_data_backup
raw_data_backup <- raw_data

#other two interesting columns are comments/rating flags so need investigating

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



#-----------------------------------------CHECK FOR NAs
raw_data %>% summarise_all(~ sum(is.na(.)))
#is this because of ratings being disabled?
raw_data %>% filter(is.na(perc_likes)) %>% group_by(ratings_disabled) %>% summarize (count = n())
#it looks like that's the case for al but 3 of them. Those are na due to not having any likes/dislikes so we will adjust the calculation slightly
raw_data <- raw_data %>% mutate(perc_likes = ifelse(ratings_disabled == "FALSE" & likes==0, 0, round(likes / (likes+dislikes), digits=2)*100))

raw_data %>% filter(is.na(perc_likes)) %>% group_by(ratings_disabled) %>% summarize (count = n())
#this error has been fixed now and NAs are only showing for those with ratings disabled



#-----------------------------------------CHECK FOR NAs
raw_data %>% summarise_all(~ sum(is.null(.)))
#no issues here



#----------------------------------------------------------------------------------------------CORR PLOT
raw_data_corr<- raw_data %>% select(views,likes,dislikes, comment_count, days_diff)

# Compute a correlation matrix
corr <- round(cor(raw_data_corr), 2)
corr

# Compute a matrix of correlation p-values
pmat <- cor_pmat(raw_data_corr)
pmat

# Visualize the correlation matrix
ggcorrplot(corr, method = "square", 
           ggtheme = ggplot2::theme_minimal, 
           title = "Correlation plot",
           outline.col = "black",
           colors = c("blue","white", "red"),
           lab = TRUE,
           digits = 2)

#highest correlation between views and likes - however, as with all correlations, this does not explain which caused which
#high correlation between likes and comment count, meaning that people engaged a lot on the videos they liked BUT
#also high correlation between dislikes and comment count, meaning people engaged in comments also on videos they disliked/controversial videos

#remove variables
rm(corr, pmat, raw_data_corr)



#-----------------------------------------------------------------------------------#TRENDING DATE

#1. Let's have a look at the overall stats for the trending videos
trending_summary <- raw_data %>%
  select(country,video_id) %>%
  group_by(country) %>%
  summarize(count = n(),
            unique = n_distinct(video_id))

ggplot(trending_summary)+
  geom_col(aes(country, count, fill = country), show.legend = FALSE)+
  geom_line(aes(country,unique), group = 1, lwd = 4, col = "black", lty = 3)+
  geom_label(aes(country, unique, label = unique), fill = "black", col = "white")+
  annotate("text", x = 2.5, y = 7000, label = "Unique Trending Videos", angle = 7)+
  labs(y = "Total Trending Videos",
       x = "Country",
       title = "Total Trending Videos vs Unique Trending Videos by Country")+
  theme_gray()
#we can see that while there is a similar number of total observations/ total videos between the datasets, CA has 
#a significantly different profile where the videos tend to be unique versus US and GB.



#2 How long does a video typically trend for?  #split in days + comparison of means as lines to expand on the trend we've seen previously
trending_duration <- raw_data %>%
  select(country, video_id, trending_date) %>%
  group_by(country, video_id) %>%
  summarize(count = n())

ggplot(trending_duration, aes(count, fill = country))+
  geom_bar()+
  facet_grid(.~country)+
  labs(y = "Total Trending Videos",
       x = "Max Trending Days",
       fill = "Country",
       title = "Total Trending Videos by Max Trending Days by Country")+
  theme_gray()
#Due to the different profile regarding unique videos, CA has a significantly shorter range of trending days 
#where most videos trend for a short span while the UK has a wider range 1-38 days. US is somewhere in the middle



#3 How long does it take for a video to become trending for the first time?
#create a new dataset to help answer this question
trending_data <- raw_data %>%
  select(country, video_id, views, days_diff) %>%
  group_by(country, video_id) %>%
  summarize(count = n_distinct(video_id),
            views = max(views),
            days_diff = min(days_diff))

#let's look at a basic spread
summary(trending_data$days_diff)
#not as detailed as we'd like so let's see this in increments of 1%
quantile(trending_data$days_diff, probs = seq(0, 1, length.out=101))
#99% of the data is between 0-18 days from publish to trending; we will use this 99th percentile as a max limit for our graphs for clarity
max_limit <- quantile(trending_data$days_diff, probs = c(0.99))

#Overall, how long does it take before a video reaches the trending page?
ggplot(trending_data, aes(as.factor(days_diff), count))+
  geom_col(fill = "dark red")+
  coord_cartesian(xlim=c(0,max_limit))+
  labs(y = "Total Trending Videos",
       x = "Days Until Trending",
       title = "Total Trending Videos by Days Until Trending")+
  theme_gray()
#as seen before, most of the videos (~75%) make it to trending within 1 day or less of publishing, and ~95% make it within 5 days. 
#Changes are that if it hasn't made it to trending within the first 10 days, then it's highly unlikely to do, although some take slightly longer (~2%)
  
#Is there a difference between countries?
ggplot(trending_data, aes(days_diff, count, fill = country))+
  geom_col()+
  coord_cartesian(xlim=c(0,max_limit))+ #zooming in on the area of interest (99% of the data)
  facet_grid(.~country)+
  labs(y = "Total Trending Videos",
       x = "Days Until Trending",
       fill = "Country",
       title = "Total Trending Videos by Days Until Trending")+
  theme_gray()
#mostly, it looks like videos in Canada videos tend to become trending very quickly, as opposed to US where this spread is wider, and GB where is widest



#4. How did they trend in time? Monthly differences?
trending_by_day <- raw_data %>%
  select(country, trending_date) %>%
  group_by(country, day=floor_date(trending_date, "day")) %>%
  summarize(count = n())

ggplot(trending_by_day, aes(day, count, col = country)) + 
  geom_smooth(lwd = 2, se = FALSE)+
  scale_x_date(date_breaks = "15 days", date_labels = "%d-%b")+
  labs(y = "Total Trending Videos",
       x = "Date",
       fill = "Country",
       title = "Total Trending Videos by Days Until Trending")+
  theme_gray()
#no significant difference up until March for all countries (trending ~200max), GB has gone down to around 160 Apr-June-no clarification as to why)

#remove unnecessary variables from memory
rm(trending_summary, trending_data, trending_duration, max_limit, trending_by_day)



#------------------------------------------------------------------------------------------------------------VIEWS

#1. Let's have a look at the overall stats for the trending videos
views_summary <- raw_data %>%
  select(country,video_id,views) %>%
  group_by(country, video_id) %>%
  summarize(count = n(),
            unique = n_distinct(video_id),
            views = max(views)) %>%
  group_by(country) %>%
  summarize(count = sum(count),
            unique = round(sum(unique)/1000),
            views = sum(views)/1000000000)

ggplot(views_summary)+
  geom_col(aes(country, views, fill = country))+
  geom_line(aes(country,unique), group = 1, lwd = 2, col = "black", lty = 3)+
  geom_label(aes(country, unique, label = unique), fill = "black", col = "white")+
  annotate("text", x = 2.5, y = 6.35, label = "Unique Trending Videos (thousands)", angle = 12)+
  labs(y = "Total Trending Videos (billions)",
       x = "Country",
       fill = "Country",
       title = "Total Trending Videos vs Unique Trending Videos by Country")+
  theme_gray()
#while it's not a surprise that CA is ranking 1st due to having many more unique videos, it's surprising to see
#that GB has more views than US, despite having around half as many unique videos



#2. spread of views by country by unique videos
views_data <- raw_data %>%
  select(country,video_id,views) %>%
  group_by(country, video_id) %>%
  summarize(count = n(),
            unique = n_distinct(video_id),
            views = max(views),
            views_m_reached = floor(views/1000000),
            views_rounded = round(views/1000000))
            

#let's look at a basic spread
summary(views_data$views_m_reached)
#dive into more detail
table(quantile(views_data$views_m_reached, probs = seq(0, 1, length.out=101)))
#76% did not make it past 1 mil
#only 5% go over the 5 million mark but not as detailed as we'd like so let's see by 1% increments
quantile(views_data$views_m_reached, probs = seq(0, 1, length.out=101))
#99% of the data is between 0 days from publish to trending and 18 days so we will use this 99th percentile as a max limit for our graphs
max_limit <- quantile(views_data$views_m_reached, probs = c(0.99))


#Overall, what's the spread of views (in million) for those that reached trending
ggplot(views_data, aes(views_m_reached, count))+
  geom_col(fill="dark red")+
  coord_cartesian(xlim=c(0,max_limit))+
  labs(y = "Total Trending Videos",
       x = "Views (millions)",
       title = "Total Trending Videos by Views reached while Trending")+
  theme_gray()+
  geom_vline(xintercept=0.5, lwd = 1, col = "black", lty = 1)+
  annotate("text", x = 0.7, y = 28000, label = "76% of data", angle = 90)+
  geom_vline(xintercept=4.5, lwd = 1, col = "black", lty = 5)+
  annotate("text", x = 4.7, y = 28000, label = "95% of data", angle = 90)+
  geom_vline(xintercept=max_limit+0.5, lwd = 1, col = "black", lty = 2)+
  annotate("text", x = max_limit-0.3+0.5, y = 28000, label = "99% of data", angle = 90)
#less than 62% of videos ever reach 1m views and <5% go over 5m
#only 1% go over 19m with the highest being 425m


#Segmentation by country
ggplot(views_data)+
geom_bar(aes(views_m_reached, ..prop.., fill = country))+
facet_grid(~country)+
coord_cartesian(xlim=c(0,max_limit))+
theme_gray()+
labs(y = "% of Total Trending Videos",
       x = "Views (millions)",
       fill = "Country",
       title = "% of Trending Videos by Views reached while Trending - Segmentation by Country")
#switched to % view because there is a significant difference in volume betwee nthe countries. We can see a wider distirbution in GB and US vs CA



#3. Because a significant part of the trending videos never reached 1m, it would be helpful to look at this subset in particular
views_data_below1m <- views_data %>%
  filter (views_m_reached < 1)

#Overall, what's the spread of views for those that reached trending with under 1m views
ggplot(views_data_below1m, aes(views))+
  geom_histogram(fill="dark red", binwidth = 10000)+
  labs(y = "Total Trending Videos",
       x = "Views",
       title = "Total Trending Videos by Max Views (under 1 million) while Trending")+
  theme_gray()
#we can see that this group is skewed to the right - most videos tend to have up to 100,000 after which there's a steep decline

#Segmentation by country
ggplot(views_data_below1m)+
  geom_histogram(aes(views, fill = country), binwidth = 10000)+
  facet_grid(~country)+
  theme_gray()+
  labs(y = "Total Trending Videos",
       x = "Views",
       fill = "Country",
       title = "Total Trending Videos by Max Views (under 1 million) while Trending - Segmentation by Country")
#no significant differences between the three countries



#4. Do you have to trend for more than one day for best results?
views_by_video <- raw_data %>% 
  select(video_id, views, country, trending_date) %>%
  group_by(country, video_id, trending_date) %>%
  summarize (views = max(views),
             count = n()) %>%
  group_by(country, video_id) %>%
  summarize(views = max(views)/1000000,
            count = n())

count_by_days_trending <- views_by_video %>%
    group_by(count) %>%
    summarize(total_count = n())

#quick summary of spread
quantile(views_by_video$views)

ggplot(views_by_video, aes(as.factor(count), views)) +
  geom_boxplot(outlier.shape = NA, fill = "dark red", col = "black", size = 0.1)+
  geom_smooth(aes())+
  coord_cartesian(ylim = c(0, 80))+
  labs(y = "Max Views (million)",
       x = "Total Trending Days",
       title = "Total Trending Days by Max Views per Video")+
  theme_gray()
#we can see that the median tends to increase as the number of trending days increases

ggplot(count_by_days_trending, aes(count, total_count))+
  geom_col(fill = "dark red")+
  labs(y = "Number of Videos",
       x = "Total Trending Days",
       title = "Total Trending Days by Number of Videos")+
  theme_gray()
#however, the number of "samples" in each category decreases (much fewer have 30m views than 3m views) 
#which means that the data can be skewed by outliers

#by country
ggplot(views_by_video, aes(as.factor(count), views)) +
  geom_boxplot(aes(fill = country), outlier.shape = NA, col = "black", size = 0.1)+
  coord_cartesian(ylim = c(0, 100))+
  scale_x_discrete(breaks = seq(0,38,5))+
  labs(y = "Views (million)",
       x = "Total Trending Days",
       title = "Total Vies by amount of Trending Days")+
  theme_gray()+
  facet_wrap(~country)
#same trend, except for the difference the amount of days a video is trending between the three countries that we previously covered
 
#clear variables
rm(views_by_video, views_data, views_data_below1m, views_summary, count_by_days_trending ,max_limit)



#----------------------------------------------------------------------------------------------CATEGORY

#1. How do categories differ in terms of the amount of videos that made it on the trending page
raw_data_cat <- raw_data %>%
  select(country, category_title, video_id, views) %>%
  group_by(country, category_title, video_id) %>%
  summarize (count= n(),
             unique_count = n_distinct(video_id),
             views = max(views)) %>%
  group_by(country, category_title) %>%
  summarize (count = sum(count),
             unique_count = sum(unique_count),
             views = sum(views)/1000000) %>%
  arrange(country,desc(count))

#let's visualise (line indicates unique count)
ggplot(raw_data_cat, aes(x=reorder(category_title, count), y = count, fill = category_title)) + 
  geom_bar(stat = "identity") +
  geom_line(aes(reorder(category_title,count), unique_count), group = 1, lwd = 0.5, lty = 1) +
  facet_grid(~country) +
  theme_gray() +
  theme(legend.position = "none") + 
  labs(y = "Number of Videos",
       x = "Category",
       title = "Frequency of Number of Videos by Category (line graph - videos that trended once)")+
  coord_flip()
#we can see some different trends:
#in CA, entertainment is the only "outlier" while in the US music is also significantly higher than others
#in the UK, music takes first place , followed by entertainment
#we can also see that irrespective of category, UK has a lot of videos that trend more than once


#2. How do categories differ in terms of views
ggplot(raw_data_cat) + 
  geom_bar(aes(reorder(category_title, views), views, fill = category_title), stat = "identity") +
  facet_grid(~country) +
  theme() +
  theme(legend.position = "none") + 
  labs(y = "Number of Views (millions)",
       x = "Video Category",
       title = "Total Views by Category")+
  coord_flip()
#in CA, despite having almost 3x more videos trending within entertainment, there is fairly equal split of views within entertainment and music
#a similar trend where music videos have more views than entertainment is present in the other countries (particularly in the UK)


#3. What about the spread of views within each category?
raw_data_cat_detail <- raw_data %>%
  select(country, category_title, video_id, views) %>%
  group_by(country, category_title, video_id) %>%
  summarize (count= n(),
             unique_count = n_distinct(video_id),
             views = max(views)/1000000) %>%
  arrange(country,desc(count))
        
#excluding outliers        
ggplot(raw_data_cat_detail, aes(reorder(category_title, count), views, fill = category_title)) + 
  geom_boxplot(outlier.shape = NA) +
  facet_grid(~country) +
  theme_gray()+
  theme(legend.position = "none") +
  labs(y = "Number of Views (millions)",
       x = "Video Category",
       title = "Spread of Views (in millions) by Video Category - Excluding Outliers")+
  coord_flip(ylim=c(0,25))
#Although Music has the largest range of views across all countries, this is the most evident in the UK
#other categories that have a relatively larger range across all countries are Film and animation and comedy
#gaming seems to only have a high range in the US

#outliers analysis
ggplot(raw_data_cat_detail, aes(reorder(category_title, count), views, fill = category_title)) + 
  geom_boxplot(size = 0.5, outlier.shape = 1, outlier.colour = "red", outlier.alpha = 0.3) +
  facet_grid(~country) +
  theme_gray()+
  theme(legend.position = "none") + 
  labs(y = "Number of Views (millions)",
       x = "Video Category",
       title = "Spread of Views (in millions) by Video Category - Outliers")+
  coord_flip()
#across all countries, outliers are present predominantly within Music and Entertainment (in GB they're as high as ~425m)
#which supports the previous findings

rm(raw_data_cat, raw_data_cat_detail)

#----------------------------------------------------------------------------------------------ENGAGEMENT
#1. General stats by country
raw_data_eng_overall <- raw_data %>%
  select(country, category_title, video_id, trending_date, likes, comment_count, dislikes, views) %>%
  group_by (country) %>%
  summarize (perc_engagement = ((sum(likes)+sum(dislikes)+sum(comment_count))/sum(views))*100,
             perc_dislikes = (sum(dislikes)/(sum(likes)+sum(dislikes)))*100,
             perc_comments_to_views = (sum(comment_count)/sum(views))*100)


ggplot(raw_data_eng_overall) + 
  geom_col(aes(country, perc_dislikes, fill = country), stat = "identity") +
  geom_line(aes(country, perc_engagement), group = 1, lwd = 1, lty = 3) +
  theme_grey() +
  theme(legend.position = "none") + 
  labs(y = "% dislikes / % engagement",
       x = "Country",
       title = "% Dislikes by Country; dotted line represents engagement (likes+dislikes+comments) ratio to Views")

#let's create a dataset at category level that we can use for further analysis
         
raw_data_eng <- raw_data %>%
  select(country, category_title, video_id, trending_date, likes, comment_count, dislikes, views) %>%
  group_by(country, category_title, video_id) %>%
  summarize (trending_date = max(trending_date),
             likes = max(likes),
             comments = max(comment_count),
             dislikes = max(dislikes),
             views = max(views),
             count= n(),
             unique_count = n_distinct(video_id))%>%
  group_by(country, category_title) %>%
  summarize (perc_engagement = round((sum(likes)+sum(dislikes)+sum(comments))/sum(views)*100),
             perc_likes = round((sum(likes)/(sum(likes)+sum(dislikes)))*100),
             perc_dislikes = round((sum(dislikes)/(sum(likes)+sum(dislikes)))*100),
             perc_comments = round((sum(comments)/sum(views))*100),
             likes = mean(likes)/1000,
             views = sum(views)/1000000000)%>%
  filter(!category_title %in% c("Shows", "Nonprofits & Activism", "Movies"))


#2. Analysis by Percent Dislikes
ggplot(raw_data_eng) + 
  geom_bar(aes(reorder(category_title, perc_dislikes), perc_dislikes, fill = category_title), stat = "identity") +
  geom_line(aes(reorder(category_title, perc_dislikes), views), group = 1, lwd = 1, lty = 3) +
  facet_grid(~country) +
  theme_grey() +
  theme(legend.position = "none") + 
  labs(y = "% dislikes",
       x = NULL,
       title = "% Dislikes by Video Category (dotted line represents Total Views (billions)")+
  coord_flip()
#we can see that between countries, there is a similar trend where News & Politics is most controversial
#This is followed by entertainment, people & blogs
#for all countries pets&animals was the least disliked (in fact, who doesn't log a dog/cat video?) I know I definitely watched a lot of golden retriever videos

#3. Betwen countries
ggplot(raw_data_eng) + 
  geom_col(aes(country, perc_dislikes, fill = country)) +
  facet_wrap(~category_title, as.table = FALSE) +
  theme_grey() +
  theme(legend.position = "none") + 
  labs(y = "% Dislikes",
       x = NULL,
       title = "% Dislikes by Video Category")
#it's interesting to see that news are much more controversial in the US (20% dislikes) than GB (~14%) and CA (~8%)

#4.Percent engagement difference by category
#Users are more likely to engage(like, dislike and comment) in which category?
ggplot(raw_data_eng) + 
  geom_bar(aes(reorder(category_title, perc_engagement), perc_engagement, fill = category_title), stat = "identity") +
  geom_line(aes(reorder(category_title, perc_engagement), views), group = 1, lwd = 1, lty = 3) +
  facet_grid(~country) +
  theme_grey() +
  theme(legend.position = "none") + 
  labs(y = "% Engagement, defined as (comments+likes+dislikes) / Total Views",
       x = NULL,
       title = "% Engagement by Video Category (dotted line represents total views (billions)")+
  coord_flip()
#no significant differences between categories



#we need to create a dataset at video level now to look at spread within categories
raw_data_eng_detail <- raw_data %>%
  filter (comments_disabled == FALSE, ratings_disabled == FALSE) %>%
  select(country, category_title, video_id, trending_date, likes, comment_count, dislikes, views) %>%
  group_by(country, category_title, video_id) %>%
  summarize (trending_date = max(trending_date),
             likes = max(likes),
             comments = max(comment_count),
             dislikes = max(dislikes),
             views = max(views),
             count= n(),
             unique_count = n_distinct(video_id),
             perc_engagement = round((sum(likes)+sum(dislikes)+sum(comments))/sum(views)*100),
             perc_likes = round((sum(likes)/(sum(likes)+sum(dislikes)))*100),
             perc_dislikes = round((sum(dislikes)/(sum(likes)+sum(dislikes)))*100),
             perc_comments = round((sum(comments)/sum(views))*100)) %>%
  filter(!category_title %in% c("Shows", "Nonprofits & Activism", "Movies"))%>%
  mutate(engagement_status = ifelse(perc_dislikes > 10, "negative", "positive"))


#5. Spread of perc_dislikes by video
ggplot(raw_data_eng_detail) + 
  geom_boxplot(aes(reorder(category_title, dislikes), perc_dislikes, fill = category_title), outlier.alpha = 0.1, outlier.colour = "grey45") +
  facet_grid(~country) +
  theme_grey() +
  theme(legend.position = "none") + 
  coord_flip()+
  labs(title="% difference in Dislikes by Category", 
       x=NULL, 
       y="% Dislikes")
  

#6. Are negative vidoes more likely to have more comments or the other way around?
ggplot(raw_data_eng_detail) + 
  geom_boxplot(aes(comments, engagement_status, fill = engagement_status), outlier.shape = NA)+
  facet_grid(~country) +
  theme(legend.position = "none") + 
  coord_flip(xlim=c(0,15000))+
  labs(title="Comments by Engagement Status", 
       x="Comments", 
       y="Engagement Status")

#----------------------------------------------------------------------------------------------CHANNEL
#total views, total videos, unique videos, total trendings days?

raw_data_channels <- raw_data %>%
  select(country, channel_title, video_id, views) %>%
  group_by(country, channel_title, video_id) %>%
  summarize (count= n(),
             unique_count = n_distinct(video_id),
             views = max(views)) %>%
  group_by(country, channel_title) %>%
  summarize (count = sum(count),
             unique_count = sum(unique_count),
             views = round(sum(views)/1000000)) %>%
  arrange(country,desc(count)) %>%
  mutate(mean_us = mean(count[country == "US"]),
         mean_ca = mean(count[country == "CA"]),
         mean_gb = mean(count[country == "GB"]),
         mean = ifelse(is.na(mean_us),ifelse(is.na(mean_ca),mean_gb,mean_ca),mean_us)) %>%
  top_n(views, n = 10)



ggplot(raw_data_channels, aes(x = channel_title, y = views, fill = country)) + 
  geom_bar(stat = "identity") +
  geom_line(aes(channel_title, count), group = 1, lwd = 1, lty = 3) +
  geom_line(aes(channel_title, mean_us), group = 1, lwd = 1, lty = 2) +
  geom_line(aes(channel_title, mean_ca), group = 1, lwd = 1, lty = 2) +
  geom_line(aes(channel_title, mean_gb), group = 1, lwd = 1, lty = 2) +
  theme() +
  theme(axis.text.x = element_text(angle = 70,hjust = 1), legend.position = "none") + 
  scale_x_discrete(name = "Video category ") + 
  scale_y_continuous(name = "Number of videos") + 
  ggtitle ("Frequency of Total Trending Days by Unique Videos")+
  coord_flip()+
  facet_wrap(~country, nrow=3, ncol = 3, scales = "free_y") +
  geom_text(aes(label = views, size = 3, hjust = 1.3))

#this is in line with what we expected where 



#-----------------------------------------------------------------------------------------------------------GRAPH IDEAS

#overlay multiple histograms with geom_freqpoly and set color = dimension

#(ALT+-) results in <-

#one to show Thumbnail of each top 10 by country

#boxplot width
varwidth = TRUE

#day on day increase
lag/lead within mutate

#do not understand hour format

#add top 10 thumbnails and put that in readme too
