
# How do colleges talk about diversity on social media?

Welcome! This project examines the use of Twitter by four-year colleges and universities. Specifically, I examine Tweets related to race and gender diversity. This page is focused on the analysis of the data. At the bottom of this page you will find links to tutorials to guide you through reproducing this analysis or recreating it with your own data. 

The project repository can be found at https://github.com/LaurenDahlin/colleges_on_social_media.

## Analysis

### The Data and College Tweets in General

The data in this analysis comes from all original (non-retweet) Twitter postings made by four-year colleges on their main Twitter pages and admissions Twitter pages (if they have a separate admissions page). The graph below shows the growth in tweeting by colleges from 2008-2017. 


```python
import pandas as pd
import numpy as np
import plotly
import plotly.plotly as py
import plotly.graph_objs as go
from plotly import tools
plotly.tools.set_credentials_file(username='your.username', api_key='your.api.key')
import gc, glob, re
```


```python
pkl_files = glob.glob(r'../data/clean/diversity_full_*.pkl')
append_tweets = []
for file in pkl_files:
    tw_df = pd.read_pickle(file)
    append_tweets.append(tw_df)
    del tw_df
    gc.collect()
diversity_df = pd.concat(append_tweets, ignore_index=True)
```


```python
diversity_df = diversity_df.assign(year = diversity_df.date.astype(str).str[:4].astype(int))
```


```python
year_summary = diversity_df.groupby('year').agg({'ipeds_id': lambda x: x.nunique(), 'id': 'count'})
year_summary.reset_index(level=0, inplace=True)
year_summary = year_summary[year_summary.year<2018]
year_summary.rename(index=str, columns={'ipeds_id':'School Count', 'id':'Tweet Count'}, inplace=True)
```


```python
trace1 = go.Scatter(
    x=year_summary.year,
    y=year_summary['School Count'],
    name='School Count'
)
trace2 = go.Scatter(
    x=year_summary.year,
    y=year_summary['Tweet Count'],
    name='Tweet Count',
    yaxis='y2'
)
data = [trace1, trace2]
layout = go.Layout(
    title='Total Number of Colleges and Tweets by Year',
    yaxis=dict(
        title='Number of Colleges'
    ),
    yaxis2=dict(
        title='Number of Tweets',
        overlaying='y',
        side='right'
    )
)
fig = go.Figure(data=data, layout=layout)
py.iplot(fig, filename = "tweets_over_time")
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~lauren.n.dahlin/121.embed" height="525px" width="100%"></iframe>



We are seeing a slight decline in the number of tweets produced by colleges in recent years. This likely reflects a shift in the number of platforms being used by colleges to reach students. For example, many colleges are now also posting on Instagram.

#### So what are these colleges talking about?


```python
from os import path
from PIL import Image
from wordcloud import WordCloud, STOPWORDS, ImageColorGenerator
import matplotlib.pyplot as plt
```


```python
# Data is too big for memory. Take a random sample of 10%
np.random.seed(12345)
# Create a column with random number - will be used to subset the data
diversity_df = diversity_df.assign(rand_int = np.random.randint(0, 99, diversity_df.shape[0]))
```


```python
diversity_df = diversity_df.assign(words=diversity_df.words.astype(str))
```


```python
text = " ".join(tw for tw in diversity_df[diversity_df.rand_int<10].words)
text = re.sub(r'([^\s\w]|_)+', '', text)
```


```python
# https://www.datacamp.com/community/tutorials/wordcloud-python
wordcloud = WordCloud(max_font_size=50, max_words=100, background_color="white").generate(text)
plt.figure()
plt.imshow(wordcloud, interpolation="bilinear")
plt.axis("off")
plt.show()
```


![png](index_files/index_16_0.png)


Perhaps not surprisingly, most of these words are very generic words related to college and accolades: student, learn, congrat (congratulations), thank, campu (campus), and communiti (community). Note that the words are stemmed from earlier pre-processing of the data.

### Diversity-Specific Tweeting

Now, let's look at the diversity-specific tweets. I have flagged tweets that mention "diversity" and "multicultural", as well as tweets with race ('black', 'african', 'asian', 'hispan', 'latino', 'latina') and gender ('woman', 'women','gender') -related words. How have the frequency of these words changed over time?


```python
diversity_df = diversity_df.assign(year = diversity_df.date.astype(str).str[:4].astype(int))
```


```python
year_summary2 = diversity_df.groupby('year').agg({'diversity_flag': 'sum', 'race_flag': 'sum', 'gender_flag': 'sum'})
```


```python
year_summary2.reset_index(level=0, inplace=True)
year_summary2 = year_summary2[year_summary2.year<2018]
```


```python
year_summary2
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>year</th>
      <th>diversity_flag</th>
      <th>race_flag</th>
      <th>gender_flag</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2007</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2008</td>
      <td>36.0</td>
      <td>83.0</td>
      <td>482.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2009</td>
      <td>435.0</td>
      <td>1148.0</td>
      <td>4078.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2010</td>
      <td>682.0</td>
      <td>1824.0</td>
      <td>5792.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2011</td>
      <td>911.0</td>
      <td>2733.0</td>
      <td>7482.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>2012</td>
      <td>1180.0</td>
      <td>3460.0</td>
      <td>9656.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2013</td>
      <td>1487.0</td>
      <td>3671.0</td>
      <td>10411.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>2014</td>
      <td>1903.0</td>
      <td>3721.0</td>
      <td>9665.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>2015</td>
      <td>2063.0</td>
      <td>4159.0</td>
      <td>8917.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>2016</td>
      <td>2515.0</td>
      <td>3972.0</td>
      <td>7862.0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>2017</td>
      <td>2781.0</td>
      <td>4095.0</td>
      <td>8025.0</td>
    </tr>
  </tbody>
</table>
</div>




```python
trace1 = go.Scatter(
    x=year_summary2.year,
    y=year_summary2.diversity_flag,
    name='Diversity Tweet Count'
)
trace2 = go.Scatter(
    x=year_summary2.year,
    y=year_summary2.race_flag,
    name='Race Tweet Count'
)
trace3 = go.Scatter(
    x=year_summary2.year,
    y=year_summary2.gender_flag,
    name='Gender Tweet Count'
)
data = [trace1, trace2, trace3]
layout = go.Layout(
    title='Total Number of Diversity-Related Tweets by Year',
    yaxis=dict(
        title='Number of Tweets'
    )
)
fig = go.Figure(data=data, layout=layout)
py.iplot(fig, filename = "diversity_tweets_over_time")
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~lauren.n.dahlin/123.embed" height="525px" width="100%"></iframe>



Interestingly, the number of gender-related tweets has decreased in recent years. This may reflect the fact that women are now the majority of applicants and degree-holders at U.S. universities.

#### A sample of diversity-related tweets:

Note that flagging based on keywords alone does not produce a perfect match to diversity-related tweets.


```python
pd.set_option('max_colwidth',200)
for index,row in diversity_df[(diversity_df.rand_int<1) & (diversity_df.diversity_flag==True)][:20].iterrows():
    print(row['tweet'])
```

    UA Students to Present ‘Diversity Demonstrated’  http://bit.ly/IKzQ9I 
    The Department of English and Foreign Languages and the Office of Multicultural Affairs invite UM students,...  http://fb.me/Nj2iJJ7G
    Meaningful conversations: Black Student Union president Andrew Freed is passionate about enhancing diversity at #UAA  http://ow.ly/KvFHJ .
    #SoCaltech is an occasional series celebrating the diverse individuals who give Caltech its spirit of excellence, ambition, and ingenuity. Know someone we should profile? Send nominations to magazine@caltech.edu. pic.twitter.com/BplE0rwLgA
    Stay strong and stop bullying--Join the Office of Diversity for a group photo in front of Kendall Hall @ 12pm today! pic.twitter.com/Cbg7EHLDRp
    Cross Cultural Week! Check out the multicultural sorority & fraternity booths till 1p #FresWOW  http://bit.ly/FSXCultural  pic.twitter.com/oVqYnBdQ3T
    #inaug2013: Garcia: It is important that  as our country becomes more diverse that we live up to our promise of equal education for all.
    Is #highered a right or a privilege? Join the discussion at the @asi_csueb Diversity Center on March 5 at 4pm. Details -->...
    #UCDavis celebrates diversity in children's books on Mar 5 w/ presentations by author Belle Yang  http://ow.ly/18Meg
    @USNewsEducation ranked UCLA as one of the top 10 most diverse universities in the U.S.  http://bit.ly/1UXSiBD  #BruinBound
    Community. Diversity. Inclusion. That's why #UCRStandsTogether. Continue to share your photos! #Highlanderpride #ucr #ucriverside https://twitter.com/natnoota/status/826684680074260480 …
    50th entering class distinguished by academic achievement, diversity  http://news.ucsc.edu/2015/09/fall2015-start.html … #UCSC pic.twitter.com/7chTE21Qha
    UCSC Research: Low extinction rates made California a refuge for diverse plant species.  http://tinyurl.com/ax38p32 
    October is DiversAbility Month at SDSU. Here's how you can get involved: http://ow.ly/T2Xe4 
    SDSU Diversity Director got a behind the scenese look at U.S. Marine Corps Base Quantico  http://budurl.com/w7vk Brings lessons back to SDSU
    Inside @UofSanDiego: @USDufmc Multicultural Night is One for All  http://www.sandiego.edu/insideusd/?p=34181 … #insideusd @USDCID @torerolife @usdgradlawlife
    #SFSU professors of cinema have mixed opinions on whether progress has been made in making the #Oscars more diverse.  http://bit.ly/2kaOcwl  pic.twitter.com/VzkWJkOyRT
    The Office of Diversity Engagement and Community Outreach is hosting drop-in hours for faculty and staff this week: http://ow.ly/hSZS30663lu 
    #USFCA ranks 7th in the nation of most ethnically diverse student population @USNews  http://ow.ly/S6hZo  pic.twitter.com/k6BniPBtgY
    MT @uscannenberg: New @MDSCInitiative study #CARD2016 grades Hollywood on diversity:  http://bit.ly/1RWAzeS 
    

### What kinds of images are associated with diversity-related tweets?


```python
from IPython.display import Image
from IPython.core.display import HTML 
```


```python
for index,row in diversity_df[(diversity_df.rand_int<1) & (diversity_df.diversity_flag==True)][:50].iterrows():
    if row['photos'] != []:
        display(Image(row['photos'][0], unconfined=True))
```


![jpeg](index_files/index_31_0.jpeg)



![jpeg](index_files/index_31_1.jpeg)



![jpeg](index_files/index_31_2.jpeg)



![jpeg](index_files/index_31_3.jpeg)



![jpeg](index_files/index_31_4.jpeg)



![jpeg](index_files/index_31_5.jpeg)



![jpeg](index_files/index_31_6.jpeg)



![jpeg](index_files/index_31_7.jpeg)



![jpeg](index_files/index_31_8.jpeg)



![jpeg](index_files/index_31_9.jpeg)



![jpeg](index_files/index_31_10.jpeg)



![jpeg](index_files/index_31_11.jpeg)



![jpeg](index_files/index_31_12.jpeg)



![jpeg](index_files/index_31_13.jpeg)



![jpeg](index_files/index_31_14.jpeg)



![jpeg](index_files/index_31_15.jpeg)



![jpeg](index_files/index_31_16.jpeg)



![jpeg](index_files/index_31_17.jpeg)



![jpeg](index_files/index_31_18.jpeg)


There are many kinds of images associated with diversity in these tweets. They range from flyers for diversity-related events to stereotypical photos of college students of different backgrounds working and studying together.

## Tutorials and Code

- **Tutorial: How to Search for Webpages on Google and Scrape Social Media URLS**
    - Objective: Use Python to search for an entity (e.g. a school or company) on Google. Navigate to the first Google search result and scrape any Twitter, Facebook, and Instagram pages.
    - [Link](https://laurendahlin.github.io/colleges_on_social_media/Html/01_scrape_social_handles_final)
- **Code: Review Twitter Handles from Search in a Semi-Automated Fashion**
    - Objective: Convert Twitter URLs to handles. Inspect the URLs and check for quality of match. If webpage did not contain a Twitter URL, search for the Twitter page on Google.
    - [Link](https://laurendahlin.github.io/colleges_on_social_media/Html/02_twitter_username_etl_final)
- **Tutorial: How to Scrape Tweets of Twitter Users (Handles) and Get Around Twitter API Limits**
    - Objective: Get entire history of tweets for particular Twitter users using their handles. Avoid any limits put in place by the Twitter API.
    - [Link](https://laurendahlin.github.io/colleges_on_social_media/Html/03_scrape_tweets_final)
- **Tutorial: How to Process Tweets for Text Analysis**
    - Objective: Process Tweets for text analysis including topic modelling and wordcloud visualization. Processing includes separating words within hashtags, removing entity-specific words, stop words, and lemmatization.
    - [Link](https://laurendahlin.github.io/colleges_on_social_media/Html/04_tweet_processing)
