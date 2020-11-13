Exploratory Data Analysis - Youtube Trending Videos
================
by Peter Hontaru

<center>

![YouTube logo](_support%20files/YT%20logo.png)

</center>

# Introduction

## Problem statement:

**Are there differences between YouTube Trending videos in
English-speaking countries (Canada, Great Britain, United States)? If
yes, which aspects of the videos cause these differences?**

All throughout this analysis we will generate questions, answer them
with data and then further refine these questions based on what was
discovered.

## Who is this project intended for?

  - Content Creators / Marketing Agencies who want to better understand
    their audience and tailor their content effectively
  - Data Science enthusiasts, as they might get new ideas for how to use
    R (for beginners) or see other people’s analyses (for intermediates)
  - those generally interested in YouTube

## Why this dataset?

YouTube has been an influential source of knowledge both in my
professional life as well in my personal goals/ambitions. Thus, I
thought it would be interesting to **explore** this dataset and gather
**insights** in order to better understand the platform.

## Key Insights:

  - there were medium to high correlations between our main variables

![Correlation](_support%20files/1%20-%20Correlation%20Plot-1.png)

  - despite a similar number of total trending videos, **Canada has more
    unique videos** (60%) versus Great Britain (8%) and The United
    States of America (16%)

![Overall
stats](_support%20files/2%20-%20overall%20videos%20by%20country-1.png)

  - **most videos in CA only trend for 1-2 days** (80%) versus GB and US
    where it isn’t uncommon to trend for up to 10 days (highest 38
    days), showing a need for consistent innovation

![Max Trending
Days](_support%20files/3%20-%20Trending%20Days%20Timespan-1.png)

  - generally, **75% of videos trend within their first day** and **95%
    of videos trend within the first 5 days**. In CA, videos usually
    take 1-2 days to reach trending, while in the US and UK it could
    take up to 5-7 days
  - **only 25% of all videos receive over 1 million views** and **5%
    receive over 5 million**

![Views](_support%20files/4-%20View%20Spread-1.png)

  - **Music** and **Entertainment** are the most common trending
    categories
  - **News & Politics** is the most controversial category (as judged by
    % of dislikes), while **Pets & Animals** and **Comedy** are the
    least disliked

![Category](_support%20files/5%20-%20Views%20by%20Category-1.png)

## What are Trending Videos and why are they important?

Trending videos work alongside the home page to provide users with
content to watch. While the home page is **highly personalised** (via
the YouTube algorithm) based on previous views, what the user watched
longest, engagement, subscriptions, the trending page is **very broad
and identical across all accounts**. Since it shows this feed to
millions of users, it serves as a great source of views for content
creators (think viral videos).

![YouTube Trending Page](_support%20files/Trending%20Example.png)

## Where did the data come from?:

  - contains \>120,000 videos across three countries (Canada, Great
    Britain, United States of America)
  - 8 months of Daily Trending data between **2017-11-14** and
    **2018-06-14** (approx 200 videos/day/country)
  - all the data is downloaded from
    <https://www.kaggle.com/datasnaek/youtube-new> - *Raw data files are
    available within the “raw data” folder*

# 3\. Extended analysis:

Full analysis available:

  - at the following
    [link](http://htmlpreview.github.io/?https://github.com/peterhontaru/Exploratory-Data-Analysis-Youtube-Trending-Videos/blob/master/Exploratory-Data-Analysis.html),
    in HTML format (**recommended**)
  - in the **Exploratory-Data-Analysis.md** of this repo (however, I
    recommend previewing it at the above link since it was originally
    designed as a html document)
