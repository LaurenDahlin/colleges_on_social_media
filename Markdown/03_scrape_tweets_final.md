
# How to Scrape Tweets of Twitter Users (Handles) and Get Around Twitter API Limits

### Objective: Get entire history of tweets for particular Twitter users using their handles. Avoid any limits put in place by the Twitter API.

In the previous step I compiled a list of all the Twitter handles from colleges in our sample on Twitter. The next step is to obtain all tweets created by each handle. This is tricky because the Twitter API [limits the number of Tweets you can pull](https://developer.twitter.com/en/docs/basics/rate-limiting.html). Thankfully, the developers of the [twint](https://github.com/twintproject/twint) Python package have figured out how to circumvent these limits by not using the API at all!

All tweets are output to a .json file for each college's main handle and admissions handle.

## Set Up


```python
import pandas as pd
import numpy as np
import os, requests, re, time
```


```python
import twint
```

#### File Locations


```python
base_path = r"C:\Users\laure\Dropbox\!research\20181026_ihe_diversity"
tw_path = os.path.join(base_path,'data','twitter_handles')
```

#### Import handles


```python
ihe_handles = pd.read_pickle(os.path.join(tw_path, "tw_df_final"))
```


```python
ihe_handles[ihe_handles.adm_handle == ''].head(2)
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
      <th>instnm</th>
      <th>main_handle</th>
      <th>adm_handle</th>
    </tr>
    <tr>
      <th>unitid</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>154022</th>
      <td>Ashford University</td>
      <td>AshfordU</td>
      <td></td>
    </tr>
    <tr>
      <th>133951</th>
      <td>Florida International University</td>
      <td>fiu</td>
      <td></td>
    </tr>
  </tbody>
</table>
</div>



## Scrape All Tweets from Username with Twint


```python
def scrape_tweets(username, csv_name):
    # Configure
    c = twint.Config()
    c.Username = username
    c.Custom = ['id', 'date', 'time', 'timezone', 'user_id', 'username', 'tweet', 'replies', 
                'retweets', 'likes', 'hashtags', 'link', 'retweet', 'user_rt', 'mentions']
    c.Store_csv = True
    c.Output = os.path.join(tw_path, csv_name)
    
    # Start search
    twint.run.Profile(c)
```


```python
for index, row in ihe_handles.iterrows():
    
    # CSV file names
    main_f_name = os.path.join(tw_path, str(index) + '_main.csv')
    adm_f_name = os.path.join(tw_path, str(index) + '_adm.csv')
    
    # Handles
    main_handle = row['main_handle']
    adm_handle = row['adm_handle']
    
    if main_handle != '':
        scrape_tweets(main_handle, main_f_name)
    
    if adm_handle != '':
        scrape_tweets(adm_handle, adm_f_name)
```
