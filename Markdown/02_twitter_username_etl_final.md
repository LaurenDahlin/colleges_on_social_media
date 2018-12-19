
# Review Twitter Handles from Search in a Semi-Automated Fashion

### Objective: Convert Twitter URLs to handles. Inspect the URLs and check for quality of match. If webpage did not contain a Twitter URL, search for the Twitter page on Google.

The previous step returned Twitter URLs from the university's webpages. Mostly through manual inspection I now determine: (1) if there were any errors in the URL collection and (2) whether there are missing URLs. If there was an error or the URL is missing, I search for the twitter link on Google and return the first two results. I manually determine what handles are for the college's main page and which is for the college's admissions page. Note that many colleges also have pages for their sports teams and other centers on campus. These URLs are discarded during the ETL process.

Note: This reconcilliation is a fairly manual process and this code is not a tutorial in the usual sense. Use it to guide your own ETL process.

The final result is a dataset of all the primary and admissions Twitter handles for each college in the Brookings data.

### Set Up

As before, the main packages we are using are: pandas, BeautifulSoup, and selenium. Pandas is used for data management. Selenium is used to implement the Google searches and get around Google's bot detection. BeautifulSoup and requests fetch data from the webpages. 


```python
import pandas as pd
from bs4 import BeautifulSoup
import numpy as np
import os, requests, re, time
```


```python
from selenium import webdriver
from selenium.webdriver.firefox.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support.ui import Select
from selenium.common.exceptions import NoSuchElementException
from selenium.common.exceptions import NoAlertPresentException
```

#### File Locations


```python
base_path = r"C:\Users\laure\Dropbox\!research\20181026_ihe_diversity"
sm_path = os.path.join(base_path,'data','social_media_links')
sm_py_path = os.path.join(base_path,'python_scraping','scrape_handles')
tw_path = os.path.join(base_path,'data','twitter_handles')
```

#### Import cross-notebook funtions


```python
social_media_py = str(os.path.join(sm_py_path, 'social_media_urls.py'))
%run $social_media_py
```

## Import social media file scraped previously


```python
scraped_urls = pd.read_csv(os.path.join(sm_path, 'ihe_sm_pages_v2.csv'), engine="python", index_col=False, header=0, sep=';', skipinitialspace=True)
# Strip any unncessary whitespace
scraped_urls = scraped_urls.apply(lambda x: x.str.strip() if x.dtype == "object" else x)
```


```python
scraped_urls.columns
```




    Index(['unitid', 'instnm', 'url_main', 'url_admissions', 'twitter_links',
           'facebook_links', 'instagram_links'],
          dtype='object')




```python
# Two things to correct: errors from Googling and schools 
# that did not list Twitter URLs on homepage
scraped_urls.twitter_links.value_counts().head(2)
```




    ['No Twitter found']    37
    Error                   22
    Name: twitter_links, dtype: int64



## Search again for  universities with errors

Schools to search for again.


```python
re_google = scraped_urls.loc[scraped_urls.twitter_links=='Error']
```

Redo the search like in the previous step.


```python
# Write csv header
if not os.path.isfile(os.path.join(sm_path, "re_google.csv")):
    with open(os.path.join(sm_path, "re_google.csv"), 'w') as f:
        print('unitid; instnm; url_main; url_admissions; twitter_links; facebook_links; instagram_links', file=f)

# Initialize headless Firefox browser
options = Options()
options.headless = True
driver = webdriver.Firefox(options=options)

for index, row in re_google.iterrows():
    try:
        # Get URL from base webpage
        ihe_url = feeling_lucky(row['instnm'], driver=driver)

        # Get Twitter links
        main_twitter_links, main_facebook_links, main_instagram_links = get_handles(url=ihe_url, driver=driver)

        # Also search for admissions page to see if Twitter is different
        admissions_url = feeling_lucky(row['instnm'] + ' admissions', driver=driver)

        # Get Twitter links for admissions page
        adm_twitter_links, adm_facebook_links, adm_instagram_links = get_handles(url=admissions_url, driver=driver)

        # If no links found, add NA
        # Twitter
        if not main_twitter_links and not adm_twitter_links:
            main_twitter_links = ['No Twitter found']
        if not main_facebook_links and not adm_facebook_links:
            main_facebook_links = ['No Facebook found']
        if not main_instagram_links and not adm_instagram_links:
            main_instagram_links = ['No Instagram found']

        # Remove any duplicate links
        twitter_links = rm_dup(main_twitter_links + adm_twitter_links)
        facebook_links = rm_dup(main_facebook_links + adm_facebook_links)
        instagram_links = rm_dup(main_instagram_links + adm_instagram_links)

        # Pring on page and write to csv file
        print(str(row['unitid']) + ',' + row['instnm'] + ',' + ihe_url + ',' 
                  + admissions_url + ',' + str(twitter_links[0]) + ',' 
                  + str(facebook_links[0]) + ',' + str(instagram_links[0]))
        with open(os.path.join(sm_path, "re_google.csv"), 'a') as f:
            print(str(row['unitid']) + ';' + row['instnm'] + ';' + ihe_url + ';' 
                + admissions_url + ';' + str(twitter_links) + ';' 
                + str(facebook_links) + ';' + str(instagram_links), file=f)
    
    except:
        print("Error: " + str(row['unitid']) + ' - ' + row['instnm'])
        with open(os.path.join(sm_path, "re_google.csv"), 'a') as f:
            print(str(row['unitid']) + ';' + row['instnm'] + ';' + "Error" + ';' + "Error" + ';' + "Error" + ';' + "Error"+ ';' + "Error", file=f)
            
driver.quit()
```


```python
re_google_df = pd.read_csv(os.path.join(sm_path, 're_google.csv'), engine="python", index_col=False, header=0, sep=';', skipinitialspace=True)
# Strip any unncessary whitespace
re_google_df = re_google_df.apply(lambda x: x.str.strip() if x.dtype == "object" else x)
```

Schools that were not found in re-google will be fixed manually.


```python
manual_fix = re_google_df.loc[re_google_df.twitter_links=='Error']
```


```python
# Only three schools
manual_fix
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
      <th>unitid</th>
      <th>instnm</th>
      <th>url_main</th>
      <th>url_admissions</th>
      <th>twitter_links</th>
      <th>facebook_links</th>
      <th>instagram_links</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>104717</td>
      <td>Grand Canyon University</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
    </tr>
    <tr>
      <th>6</th>
      <td>232265</td>
      <td>Hampton University</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
    </tr>
    <tr>
      <th>7</th>
      <td>100654</td>
      <td>Alabama A &amp; M University</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
      <td>Error</td>
    </tr>
  </tbody>
</table>
</div>



## Google Search for Twitter Pages of Colleges without Twitter URLs on Main Website


```python
find_twitter = scraped_urls.loc[scraped_urls.twitter_links=="['No Twitter found']"]
```


```python
# Initialize headless Firefox browser
options = Options()
options.headless = True
driver = webdriver.Firefox(options=options)

for index, row in find_twitter.iterrows():
    try:
        # Search for institution + twitter on Google
        ihe_tw_url = feeling_lucky('twitter ' + row['instnm'], driver=driver)
        ihe_tw_url = ihe_tw_url.split('?')[0].split('#')[0]
        
        # Also search for admissions page to see if Twitter is different
        admissions_tw_url = feeling_lucky('twitter ' + row['instnm'] + ' admissions', driver=driver)
        admissions_tw_url = admissions_tw_url.split('?')[0].split('#')[0]
        
        # Put urls in list
        twitter_list = [ihe_tw_url, admissions_tw_url]
        
        # Remove any duplicate links
        twitter_links = rm_dup(twitter_list)

        # Print out URLs for manual checking
        print(row['instnm'] + ' ' + ihe_tw_url + ' ' + admissions_tw_url)
        
        # Update df
        find_twitter.at[index, 'twitter_links'] = twitter_links
        
    except:
        continue
            
driver.quit()
```

## Combine original csv with new results from updated google search and twitter search


```python
# Remove errors and twitter not found entries
updated_urls = scraped_urls.loc[(scraped_urls.twitter_links != 'Error') & (scraped_urls.twitter_links!="['No Twitter found']")]
```


```python
# Append new data
updated_urls = updated_urls.append(re_google_df)
updated_urls = updated_urls.append(find_twitter)
```


```python
updated_urls = updated_urls.loc[(updated_urls.twitter_links != 'Error') & (updated_urls.twitter_links!="['No Twitter found']")]
```

### Manually review Twitter links

Remove links related to athletics, news, duplicates with different spellings.


```python
# Print links to csv for manual review
for index, row in updated_urls.iterrows():
    show_links = str(row['twitter_links']).strip('[]')
    show_links = show_links.replace("'", '')
    show_links = show_links.replace(',', ';')
    with open(os.path.join(tw_path, "review_twitter.csv"), 'a') as f:
        print(str(row['unitid']) + ';' + row['instnm'] + ';' + show_links, file=f)
```

## Google search for missing admissions pages

Unfortunately many schools appear not to put their admissions twitters directly on their admissions page. Do a google search for missing schools - manually examine the top two google results for the admissions page.


```python
def google_search(search_text, result_num, driver):
    driver.get("http://www.google.com/")
    time.sleep(1)
    driver.find_element_by_name("q").click
    driver.find_element_by_name("q").send_keys(search_text)
    time.sleep(0.5)
    driver.find_element_by_name("q").send_keys(Keys.TAB)
    time.sleep(0.5)
    driver.find_element_by_name("q").send_keys(Keys.ENTER)
    time.sleep(1)
    # Result 
    result_text = driver.find_element_by_xpath(r'(.//*[@class="g"][' + str(result_num) + r']//descendant::a)').text
    result_link = driver.find_element_by_xpath(r'(.//*[@class="g"][' + str(result_num) + r']//descendant::a' + r'//descendant::div[1])').text
    return result_text, result_link  
```


```python
if 1==2:
    options = Options()
    options.headless = False
    driver = webdriver.Firefox(options=options)
    result_text, result_link = google_search(search_text="twitter admissions University of Washington", result_num=1, driver=driver)
    print(result_text + " "  + result_link)
    driver.quit()
```


```python
 driver.quit()
```


```python
manual_review = pd.read_csv(os.path.join(tw_path, "manual_review.csv"), engine="python", header=0, sep=',')
```


```python
find_adm_tw = manual_review.loc[manual_review.admissions_link.isnull()].copy()
```


```python
pd.set_option('display.max_rows', 1000)
#find_adm_tw
```


```python
# Create new cols in df
for new_col in ['likely_adm_link', 'likely_adm_text', 'google_text1', 'google_link1', 'google_text2', 'google_link2']:
    find_adm_tw[new_col] = ''
```


```python
# Initialize headless Firefox browser
options = Options()
options.headless = True
driver = webdriver.Firefox(options=options)

for index, row in find_adm_tw.iterrows():
    # Search for twitter admissions + institution on Google

    # First result
    google_text1, google_link1 = google_search(search_text='twitter admissions ' + row['instnm'], result_num=1, driver=driver)

    # Second result
    google_text2, google_link2 = google_search(search_text='twitter admissions ' + row['instnm'], result_num=2, driver=driver)

    # Update df
    find_adm_tw.at[index, 'google_text1'] = google_text1
    find_adm_tw.at[index, 'google_link1'] = google_link1
    find_adm_tw.at[index, 'google_text2'] = google_text2
    find_adm_tw.at[index, 'google_link2'] = google_link2
    
    # If text contains admissions, put in likely link column
    if 'admission' in google_text1.lower():
        find_adm_tw.at[index, 'likely_adm_link'] = google_link1
        find_adm_tw.at[index, 'likely_adm_text'] = google_text1
    if 'admission' in google_text2.lower():
        find_adm_tw.at[index, 'likely_adm_link'] = google_link2
        find_adm_tw.at[index, 'likely_adm_text'] = google_text2
    
    # If no likely admission link, print google search results for review
    if ('admission' not in google_text1.lower()) and ('admission' not in google_text2.lower()):
        print("1: " + str(row['unitid']) + " " + row['instnm'] + " " + google_text1 + " " + google_link1)
        print("2: " + str(row['unitid']) + " " + row['instnm'] + " " + google_text2 + " " + google_link2)
```


```python
driver.quit()
```


```python
# Save in case of needing to reload
find_adm_tw.to_pickle(os.path.join(tw_path, "df_google_adm_tw"))
```

### Review likely Admissions Twitter Pages and Make Replacements


```python
# find_adm_tw[find_adm_tw.likely_adm_link != '']
```


```python
find_adm_tw = pd.read_pickle(os.path.join(tw_path, "df_google_adm_tw"))
```


```python
tw_df = manual_review.copy()
```


```python
find_adm_tw = find_adm_tw[find_adm_tw.unitid.notnull()]
```


```python
tw_df.unitid = tw_df.unitid.astype(int)
find_adm_tw.unitid = find_adm_tw.unitid.astype(int)
```


```python
tw_df.set_index('unitid', inplace=True)
```


```python
find_adm_tw.set_index('unitid', inplace=True)
```


```python
find_adm_tw.head(2)
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
      <th>main_link</th>
      <th>admissions_link</th>
      <th>likely_adm_link</th>
      <th>likely_adm_text</th>
      <th>google_text1</th>
      <th>google_link1</th>
      <th>google_text2</th>
      <th>google_link2</th>
    </tr>
    <tr>
      <th>unitid</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>154022</th>
      <td>Ashford University</td>
      <td>https://twitter.com/AshfordU</td>
      <td>NaN</td>
      <td></td>
      <td></td>
      <td>Ashford University (@AshfordU) | Twitter\nhttp...</td>
      <td>https://twitter.com/ashfordu?lang=en</td>
      <td>BridgepointEducation (@Bridgepoint_Ed) | Twitt...</td>
      <td>https://twitter.com/bridgepoint_ed?lang=en</td>
    </tr>
    <tr>
      <th>236948</th>
      <td>University of Washington-Seattle Campus</td>
      <td>https://twitter.com/UW</td>
      <td>NaN</td>
      <td>https://twitter.com/seattleuadm?lang=en</td>
      <td>Seattle U Admissions (@SeattleUAdm) | Twitter\...</td>
      <td>U. of Washington (@UW) | Twitter\nhttps://twit...</td>
      <td>https://twitter.com/uw?lang=en</td>
      <td>Seattle U Admissions (@SeattleUAdm) | Twitter\...</td>
      <td>https://twitter.com/seattleuadm?lang=en</td>
    </tr>
  </tbody>
</table>
</div>




```python
tw_df.head(5)
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
      <th>main_link</th>
      <th>admissions_link</th>
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
      <td>https://twitter.com/AshfordU</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>132903</th>
      <td>University of Central Florida</td>
      <td>https://www.twitter.com/ucf</td>
      <td>https://twitter.com/ucfadmissions</td>
    </tr>
    <tr>
      <th>134130</th>
      <td>University of Florida</td>
      <td>https://twitter.com/UF/</td>
      <td>https://twitter.com/UFAdmissions</td>
    </tr>
    <tr>
      <th>193900</th>
      <td>New York University</td>
      <td>https://twitter.com/nyuniversity</td>
      <td>https://twitter.com/meetnyu</td>
    </tr>
    <tr>
      <th>232557</th>
      <td>Liberty University</td>
      <td>https://twitter.com/libertyu</td>
      <td>https://twitter.com/ExperienceLU</td>
    </tr>
  </tbody>
</table>
</div>




```python
# False positive admissions pages - Google didn't find the admissions twitter for the correct college
false_positive = [212054, 196413, 144740, 243744, 202480, 117946, 140951, 
    188429, 130226, 181002, 366711, 151306, 166452, 182670, 164739, 151102, 216010,
    160038, 219709, 201104, 195544, 196042, 194541, 176053, 209056, 154688, 130697,
    221971, 209825, 130183, 141097, 190770, 162007, 220613, 164173, 216694, 191676,
    153108, 159993]

for index, row in tw_df.iterrows():
        
    # Replace the admissions link with the likely admissions link, as long as 
    # the school is not in the falso positive list
    if (index not in false_positive) and (index in find_adm_tw.index):
        tw_df.at[index, 'admissions_link'] = find_adm_tw.at[index, 'likely_adm_link']
        
```


```python
# Admissions pages without 'admissions' in the twitter page title 
# Manually found from first two google results
in1 = [110583, 122755, 186867]
in2 = [229027, 207388, 204024, 126562, 169716]

for index, row in find_adm_tw.iterrows():
        
    # If index is in the first list, replace with the first google result
    if (index in in1):
        tw_df.at[index, 'admissions_link'] = find_adm_tw.at[index, 'google_link1']
        
    # If index is in the second list, replace with the second google result
    if (index in in1):
        tw_df.at[index, 'admissions_link'] = find_adm_tw.at[index, 'google_link2']

```


```python
# Found from totally manual google search
tw_found_manually = pd.DataFrame([
    [151379, 'https://twitter.com/iusadmissions'],
    [144962, 'https://twitter.com/ec_admissions'],
    [212054, 'https://twitter.com/DrexelAdmission'],
    [196413, 'https://twitter.com/GoSyracuseU'],
    [144740, 'https://twitter.com/DePaulAdmission'],
    [243744, 'https://twitter.com/engagestanford?lang=en'],
    [202480, 'https://twitter.com/DaytonAdmission'],
    [181002, 'https://twitter.com/ChooseCreighton'],
    [164739, 'https://twitter.com/UGABentley'],
    [176053, 'https://twitter.com/mc_admissions'],
    ], columns = ['unitid', 'tw_adm_link'])

all_found_manually = pd.DataFrame([
    [104717,'https://twitter.com/gcu', ''],
    [232265,'https://twitter.com/_hamptonu','https://twitter.com/applyhamptonu'],
    [100654, 'https://twitter.com/aamuedu', ''],
    [221838, 'https://twitter.com/TSUedu', 'https://twitter.com/tsuadmissions']
    ], columns = ['unitid', 'tw_main_link', 'tw_adm_link'])

tw_found_manually.set_index('unitid', inplace=True)
all_found_manually.set_index('unitid', inplace=True)
```


```python
for index, row in tw_df.iterrows():
        
    # If index is in the manual twitter df, replace with the link from there
    if (index in tw_found_manually.index):
        tw_df.at[index, 'admissions_link'] = tw_found_manually.at[index, 'tw_adm_link']
        
    # If index is in the all manual df, replace with the links from there
    if (index in all_found_manually.index):
        tw_df.at[index, 'main_link'] = all_found_manually.at[index, 'tw_main_link']
        tw_df.at[index, 'admissions_link'] = all_found_manually.at[index, 'tw_adm_link']
```

## Extract Handles from URLS


```python
ex1 = "https://twitter.com/somehandle?somequery"
ex2 = "http://twitter.com/somehandle?somequery"
ex3 = ""
```


```python
def extract_handle(url):
    no_query = str(url).split('?')[0]
    try:
        handle = re.search(r'.*twitter.com/([^/?]+)', no_query).group(1)
    except:
        handle = ''
    return handle
    
```


```python
if 1 == 1:
    print(extract_handle(ex1))
    print(extract_handle(ex2))
    print(extract_handle(ex3))
```

    somehandle
    somehandle
    
    


```python
# Extract main and admissions handles
tw_df['main_handle'] = tw_df.main_link.apply(extract_handle)
tw_df['adm_handle'] = tw_df.admissions_link.apply(extract_handle)
```


```python
tw_df[['instnm','main_handle','adm_handle']]
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
      <th>132903</th>
      <td>University of Central Florida</td>
      <td>ucf</td>
      <td>ucfadmissions</td>
    </tr>
    <tr>
      <th>134130</th>
      <td>University of Florida</td>
      <td>UF</td>
      <td>UFAdmissions</td>
    </tr>
    <tr>
      <th>193900</th>
      <td>New York University</td>
      <td>nyuniversity</td>
      <td>meetnyu</td>
    </tr>
    <tr>
      <th>232557</th>
      <td>Liberty University</td>
      <td>libertyu</td>
      <td>ExperienceLU</td>
    </tr>
    <tr>
      <th>228778</th>
      <td>The University of Texas at Austin</td>
      <td>UTAustin</td>
      <td>bealonghorn</td>
    </tr>
    <tr>
      <th>204796</th>
      <td>Ohio State University-Main Campus</td>
      <td>OhioState</td>
      <td>beavervip</td>
    </tr>
    <tr>
      <th>170976</th>
      <td>University of Michigan-Ann Arbor</td>
      <td>umich</td>
      <td>UMichAdmissions</td>
    </tr>
    <tr>
      <th>174066</th>
      <td>University of Minnesota-Twin Cities</td>
      <td>UMNews</td>
      <td>uofmadmissions</td>
    </tr>
    <tr>
      <th>104151</th>
      <td>Arizona State University-Tempe</td>
      <td>asu</td>
      <td>FutureSunDevils</td>
    </tr>
    <tr>
      <th>123961</th>
      <td>University of Southern California</td>
      <td>usc</td>
      <td>USCAdmission</td>
    </tr>
    <tr>
      <th>236948</th>
      <td>University of Washington-Seattle Campus</td>
      <td>UW</td>
      <td>seattleuadm</td>
    </tr>
    <tr>
      <th>228723</th>
      <td>Texas A &amp; M University-College Station</td>
      <td>tamu</td>
      <td>aggiebound</td>
    </tr>
    <tr>
      <th>214777</th>
      <td>Pennsylvania State University-Main Campus</td>
      <td>penn_state</td>
      <td>PSU_Admissions</td>
    </tr>
    <tr>
      <th>145637</th>
      <td>University of Illinois at Urbana-Champaign</td>
      <td>Illinois_Alma</td>
      <td>hashtag</td>
    </tr>
    <tr>
      <th>134097</th>
      <td>Florida State University</td>
      <td>floridastate</td>
      <td>fsuadmissions</td>
    </tr>
    <tr>
      <th>110635</th>
      <td>University of California-Berkeley</td>
      <td>UCBerkeley</td>
      <td>CalAdmissions</td>
    </tr>
    <tr>
      <th>240444</th>
      <td>University of Wisconsin-Madison</td>
      <td>uwmadison</td>
      <td>UWAdmissions</td>
    </tr>
    <tr>
      <th>133951</th>
      <td>Florida International University</td>
      <td>fiu</td>
      <td></td>
    </tr>
    <tr>
      <th>110662</th>
      <td>University of California-Los Angeles</td>
      <td>ucla</td>
      <td>uclaadmission</td>
    </tr>
    <tr>
      <th>190150</th>
      <td>Columbia University in the City of New York</td>
      <td>columbia</td>
      <td>hamiltonhall</td>
    </tr>
    <tr>
      <th>151351</th>
      <td>Indiana University-Bloomington</td>
      <td>IUBloomington</td>
      <td>IUAdmissions</td>
    </tr>
    <tr>
      <th>163286</th>
      <td>University of Maryland-College Park</td>
      <td>UofMaryland</td>
      <td>ApplyMaryland</td>
    </tr>
    <tr>
      <th>171100</th>
      <td>Michigan State University</td>
      <td>michiganstateu</td>
      <td>msu_admissions</td>
    </tr>
    <tr>
      <th>137351</th>
      <td>University of South Florida-Main Campus</td>
      <td>USouthFlorida</td>
      <td>AdmissionsUSF</td>
    </tr>
    <tr>
      <th>243780</th>
      <td>Purdue University-Main Campus</td>
      <td>lifeatpurdue</td>
      <td>PurdueAdmission</td>
    </tr>
    <tr>
      <th>110644</th>
      <td>University of California-Davis</td>
      <td>ucdavis</td>
      <td>ucd_admissions</td>
    </tr>
    <tr>
      <th>139959</th>
      <td>University of Georgia</td>
      <td>universityofga</td>
      <td>ugaadmissions</td>
    </tr>
    <tr>
      <th>163204</th>
      <td>University of Maryland-University College</td>
      <td>umuc</td>
      <td>applymaryland</td>
    </tr>
    <tr>
      <th>215293</th>
      <td>University of Pittsburgh-Pittsburgh Campus</td>
      <td>PittTweet</td>
      <td>pittadmissions</td>
    </tr>
    <tr>
      <th>104179</th>
      <td>University of Arizona</td>
      <td>uofa</td>
      <td>uaadmissions</td>
    </tr>
    <tr>
      <th>186380</th>
      <td>Rutgers University-New Brunswick</td>
      <td>rutgersnb</td>
      <td>apply2rutgers</td>
    </tr>
    <tr>
      <th>228769</th>
      <td>The University of Texas at Arlington</td>
      <td>utarlington</td>
      <td>utarlington</td>
    </tr>
    <tr>
      <th>164988</th>
      <td>Boston University</td>
      <td>BU_Tweets</td>
      <td>applytobu</td>
    </tr>
    <tr>
      <th>131469</th>
      <td>George Washington University</td>
      <td>Gwtweets</td>
      <td>gwadmissions</td>
    </tr>
    <tr>
      <th>199120</th>
      <td>University of North Carolina at Chapel Hill</td>
      <td>unc</td>
      <td>UNCAdmissions</td>
    </tr>
    <tr>
      <th>110583</th>
      <td>California State University-Long Beach</td>
      <td>csulb</td>
      <td>csulboutreach</td>
    </tr>
    <tr>
      <th>215062</th>
      <td>University of Pennsylvania</td>
      <td>Penn</td>
      <td>previewingpenn</td>
    </tr>
    <tr>
      <th>110608</th>
      <td>California State University-Northridge</td>
      <td>csunorthridge</td>
      <td></td>
    </tr>
    <tr>
      <th>232186</th>
      <td>George Mason University</td>
      <td>georgemasonu</td>
      <td>MasonAdmissions</td>
    </tr>
    <tr>
      <th>227216</th>
      <td>University of North Texas</td>
      <td>untsystem</td>
      <td>UNTadmissions</td>
    </tr>
    <tr>
      <th>201885</th>
      <td>University of Cincinnati-Main Campus</td>
      <td>UofCincy</td>
      <td>ucadmissions</td>
    </tr>
    <tr>
      <th>216339</th>
      <td>Temple University</td>
      <td>templeuniv</td>
      <td>admissionsTU</td>
    </tr>
    <tr>
      <th>225511</th>
      <td>University of Houston</td>
      <td>UHouston</td>
      <td>UHadmissions</td>
    </tr>
    <tr>
      <th>199193</th>
      <td>North Carolina State University at Raleigh</td>
      <td>ncstate</td>
      <td>ncstate</td>
    </tr>
    <tr>
      <th>110680</th>
      <td>University of California-San Diego</td>
      <td>ucsandiego</td>
      <td>ucsdadmissions</td>
    </tr>
    <tr>
      <th>153658</th>
      <td>University of Iowa</td>
      <td>uiowa</td>
      <td>IowaAdmissions</td>
    </tr>
    <tr>
      <th>433387</th>
      <td>Western Governors University</td>
      <td>wgu</td>
      <td></td>
    </tr>
    <tr>
      <th>110653</th>
      <td>University of California-Irvine</td>
      <td>ucirvine</td>
      <td>uciadmission</td>
    </tr>
    <tr>
      <th>204857</th>
      <td>Ohio University-Main Campus</td>
      <td>ohiou</td>
      <td>ohiouadmissions</td>
    </tr>
    <tr>
      <th>126614</th>
      <td>University of Colorado Boulder</td>
      <td>cuboulder</td>
      <td>futurebuffs</td>
    </tr>
    <tr>
      <th>167358</th>
      <td>Northeastern University</td>
      <td>Northeastern</td>
      <td>nuadmissions</td>
    </tr>
    <tr>
      <th>178396</th>
      <td>University of Missouri-Columbia</td>
      <td>mizzou</td>
      <td>MizzouAdmission</td>
    </tr>
    <tr>
      <th>230764</th>
      <td>University of Utah</td>
      <td>UUtah</td>
      <td>utahadmissions</td>
    </tr>
    <tr>
      <th>230038</th>
      <td>Brigham Young University-Provo</td>
      <td>byu</td>
      <td>byuadmissions</td>
    </tr>
    <tr>
      <th>129020</th>
      <td>University of Connecticut</td>
      <td>uconn</td>
      <td>uconnadmissions</td>
    </tr>
    <tr>
      <th>122409</th>
      <td>San Diego State University</td>
      <td>sdsu</td>
      <td>sdsuadmissions</td>
    </tr>
    <tr>
      <th>233921</th>
      <td>Virginia Polytechnic Institute and State Unive...</td>
      <td>virginia_tech</td>
      <td>followmetovt</td>
    </tr>
    <tr>
      <th>196088</th>
      <td>University at Buffalo</td>
      <td>UBuffalo</td>
      <td>UBAdmissions</td>
    </tr>
    <tr>
      <th>122597</th>
      <td>San Francisco State University</td>
      <td>SFSU</td>
      <td></td>
    </tr>
    <tr>
      <th>166629</th>
      <td>University of Massachusetts-Amherst</td>
      <td>umassamherst</td>
      <td>UMassAdmissions</td>
    </tr>
    <tr>
      <th>122755</th>
      <td>San Jose State University</td>
      <td>sjsu</td>
      <td>fireweatherlab</td>
    </tr>
    <tr>
      <th>162928</th>
      <td>Johns Hopkins University</td>
      <td>JohnsHopkins</td>
      <td>jhu_admissions</td>
    </tr>
    <tr>
      <th>100751</th>
      <td>The University of Alabama</td>
      <td>UofAlabama</td>
      <td>ua_admissions</td>
    </tr>
    <tr>
      <th>228459</th>
      <td>Texas State University</td>
      <td>txst</td>
      <td>txstadmissions</td>
    </tr>
    <tr>
      <th>139940</th>
      <td>Georgia State University</td>
      <td>georgiastateu</td>
      <td></td>
    </tr>
    <tr>
      <th>234030</th>
      <td>Virginia Commonwealth University</td>
      <td>VCU</td>
      <td>vcuadmissions</td>
    </tr>
    <tr>
      <th>166027</th>
      <td>Harvard University</td>
      <td>harvard</td>
      <td>applytoharvard</td>
    </tr>
    <tr>
      <th>136215</th>
      <td>Nova Southeastern University</td>
      <td>nsuflorida</td>
      <td>nsuuga</td>
    </tr>
    <tr>
      <th>229115</th>
      <td>Texas Tech University</td>
      <td>texastech</td>
      <td>TxTechAdmission</td>
    </tr>
    <tr>
      <th>234076</th>
      <td>University of Virginia-Main Campus</td>
      <td>uva</td>
      <td>uvaadmission</td>
    </tr>
    <tr>
      <th>133669</th>
      <td>Florida Atlantic University</td>
      <td>FloridaAtlantic</td>
      <td>FAUAdmissions</td>
    </tr>
    <tr>
      <th>209807</th>
      <td>Portland State University</td>
      <td>portland_state</td>
      <td>go2psu</td>
    </tr>
    <tr>
      <th>145600</th>
      <td>University of Illinois at Chicago</td>
      <td>thisisuic</td>
      <td>apply2uic</td>
    </tr>
    <tr>
      <th>126818</th>
      <td>Colorado State University-Fort Collins</td>
      <td>coloradostateu</td>
      <td>admissionscsu</td>
    </tr>
    <tr>
      <th>190415</th>
      <td>Cornell University</td>
      <td>Cornell</td>
      <td>cornelluao</td>
    </tr>
    <tr>
      <th>153603</th>
      <td>Iowa State University</td>
      <td>IowaStateU</td>
      <td>BeACyclone</td>
    </tr>
    <tr>
      <th>151111</th>
      <td>Indiana University-Purdue University-Indianapolis</td>
      <td>iupui</td>
      <td>IUPUI_Admits</td>
    </tr>
    <tr>
      <th>144740</th>
      <td>DePaul University</td>
      <td>depaulu</td>
      <td>DePaulAdmission</td>
    </tr>
    <tr>
      <th>196097</th>
      <td>Stony Brook University</td>
      <td>stonybrooku</td>
      <td></td>
    </tr>
    <tr>
      <th>155317</th>
      <td>University of Kansas</td>
      <td>KUnews</td>
      <td>BeAJayhawk</td>
    </tr>
    <tr>
      <th>179894</th>
      <td>Webster University</td>
      <td>websteru</td>
      <td></td>
    </tr>
    <tr>
      <th>105330</th>
      <td>Northern Arizona University</td>
      <td>nau</td>
      <td></td>
    </tr>
    <tr>
      <th>236939</th>
      <td>Washington State University</td>
      <td>wsupullman</td>
      <td>wsuadmissions</td>
    </tr>
    <tr>
      <th>110705</th>
      <td>University of California-Santa Barbara</td>
      <td>ucsantabarbara</td>
      <td>ucsbadmissions</td>
    </tr>
    <tr>
      <th>110617</th>
      <td>California State University-Sacramento</td>
      <td>sacstate</td>
      <td></td>
    </tr>
    <tr>
      <th>209551</th>
      <td>University of Oregon</td>
      <td>uoregon</td>
      <td>UOAdmissions</td>
    </tr>
    <tr>
      <th>203517</th>
      <td>Kent State University at Kent</td>
      <td>kentstate</td>
      <td></td>
    </tr>
    <tr>
      <th>212054</th>
      <td>Drexel University</td>
      <td>drexeluniv</td>
      <td>DrexelAdmission</td>
    </tr>
    <tr>
      <th>157085</th>
      <td>University of Kentucky</td>
      <td>universityofky</td>
      <td>ukyadmission</td>
    </tr>
    <tr>
      <th>196413</th>
      <td>Syracuse University</td>
      <td>SyracuseU</td>
      <td>GoSyracuseU</td>
    </tr>
    <tr>
      <th>169248</th>
      <td>Central Michigan University</td>
      <td>CMUniversity</td>
      <td>cmuadmissions</td>
    </tr>
    <tr>
      <th>207500</th>
      <td>University of Oklahoma-Norman Campus</td>
      <td>UofOklahoma</td>
      <td>go2ou</td>
    </tr>
    <tr>
      <th>198464</th>
      <td>East Carolina University</td>
      <td>EastCarolina</td>
      <td>ECUAdmissions</td>
    </tr>
    <tr>
      <th>238032</th>
      <td>West Virginia University</td>
      <td>WestVirginiaU</td>
      <td>WVUAdmissions</td>
    </tr>
    <tr>
      <th>131496</th>
      <td>Georgetown University</td>
      <td>georgetown</td>
      <td>ugrdadmissiongu</td>
    </tr>
    <tr>
      <th>199139</th>
      <td>University of North Carolina at Charlotte</td>
      <td>UNCCharlotte</td>
      <td>unccadmissions</td>
    </tr>
    <tr>
      <th>100858</th>
      <td>Auburn University</td>
      <td>auburnu</td>
      <td>auburnadmission</td>
    </tr>
    <tr>
      <th>170082</th>
      <td>Grand Valley State University</td>
      <td>gvsu</td>
      <td></td>
    </tr>
    <tr>
      <th>240453</th>
      <td>University of Wisconsin-Milwaukee</td>
      <td>uwm</td>
      <td>uwmadmit</td>
    </tr>
    <tr>
      <th>229027</th>
      <td>The University of Texas at San Antonio</td>
      <td>utsa</td>
      <td></td>
    </tr>
    <tr>
      <th>151801</th>
      <td>Indiana Wesleyan University</td>
      <td>IndWes</td>
      <td></td>
    </tr>
    <tr>
      <th>164076</th>
      <td>Towson University</td>
      <td>TowsonU</td>
      <td>AdmissionsatTU</td>
    </tr>
    <tr>
      <th>172699</th>
      <td>Western Michigan University</td>
      <td>WesternMichU</td>
      <td>wmuadmissions</td>
    </tr>
    <tr>
      <th>228787</th>
      <td>The University of Texas at Dallas</td>
      <td>ut_dallas</td>
      <td></td>
    </tr>
    <tr>
      <th>172644</th>
      <td>Wayne State University</td>
      <td>waynestate</td>
      <td></td>
    </tr>
    <tr>
      <th>187985</th>
      <td>University of New Mexico-Main Campus</td>
      <td>UNM</td>
      <td></td>
    </tr>
    <tr>
      <th>209542</th>
      <td>Oregon State University</td>
      <td>oregonstate</td>
      <td>beavervip</td>
    </tr>
    <tr>
      <th>145813</th>
      <td>Illinois State University</td>
      <td>IllinoisStateU</td>
      <td>ISUAdmissions</td>
    </tr>
    <tr>
      <th>139755</th>
      <td>Georgia Institute of Technology-Main Campus</td>
      <td>georgiatech</td>
      <td>gtadmission</td>
    </tr>
    <tr>
      <th>147703</th>
      <td>Northern Illinois University</td>
      <td>NIUlive</td>
      <td>niu_admissions</td>
    </tr>
    <tr>
      <th>190594</th>
      <td>CUNY Hunter College</td>
      <td>hunter_college</td>
      <td></td>
    </tr>
    <tr>
      <th>232982</th>
      <td>Old Dominion University</td>
      <td>odu</td>
      <td></td>
    </tr>
    <tr>
      <th>230728</th>
      <td>Utah State University</td>
      <td>USUAggies</td>
      <td></td>
    </tr>
    <tr>
      <th>200800</th>
      <td>University of Akron Main Campus</td>
      <td>uakron</td>
      <td>akronadmissions</td>
    </tr>
    <tr>
      <th>102368</th>
      <td>Troy University</td>
      <td>TROYUNews</td>
      <td>troyuadmissions</td>
    </tr>
    <tr>
      <th>130943</th>
      <td>University of Delaware</td>
      <td>UDelaware</td>
      <td>UDAdmissions</td>
    </tr>
    <tr>
      <th>181464</th>
      <td>University of Nebraska-Lincoln</td>
      <td>UNLincoln</td>
      <td>unladmissions</td>
    </tr>
    <tr>
      <th>149222</th>
      <td>Southern Illinois University-Carbondale</td>
      <td>SIUC</td>
      <td>SIUCAdmissions</td>
    </tr>
    <tr>
      <th>198419</th>
      <td>Duke University</td>
      <td>DukeU</td>
      <td>duke_admissions</td>
    </tr>
    <tr>
      <th>182281</th>
      <td>University of Nevada-Las Vegas</td>
      <td>UNLV</td>
      <td>unlvadmissions</td>
    </tr>
    <tr>
      <th>110671</th>
      <td>University of California-Riverside</td>
      <td>UCRiverside</td>
      <td>ucradmissions</td>
    </tr>
    <tr>
      <th>220978</th>
      <td>Middle Tennessee State University</td>
      <td>MTSUNews</td>
      <td>MTAdmissions</td>
    </tr>
    <tr>
      <th>146719</th>
      <td>Loyola University Chicago</td>
      <td>LoyolaChicago</td>
      <td>lucadmission</td>
    </tr>
    <tr>
      <th>207388</th>
      <td>Oklahoma State University-Main Campus</td>
      <td>okstate</td>
      <td></td>
    </tr>
    <tr>
      <th>150136</th>
      <td>Ball State University</td>
      <td>ballstate</td>
      <td></td>
    </tr>
    <tr>
      <th>217882</th>
      <td>Clemson University</td>
      <td>clemsonuniv</td>
      <td>beaclemsontiger</td>
    </tr>
    <tr>
      <th>243744</th>
      <td>Stanford University</td>
      <td>stanford</td>
      <td>engagestanford</td>
    </tr>
    <tr>
      <th>137032</th>
      <td>Saint Leo University</td>
      <td>SaintLeoUniv</td>
      <td></td>
    </tr>
    <tr>
      <th>106397</th>
      <td>University of Arkansas</td>
      <td>uarkansas</td>
      <td>uofaadmissions</td>
    </tr>
    <tr>
      <th>232423</th>
      <td>James Madison University</td>
      <td>JMU</td>
      <td></td>
    </tr>
    <tr>
      <th>190664</th>
      <td>CUNY Queens College</td>
      <td>QC_News</td>
      <td></td>
    </tr>
    <tr>
      <th>155399</th>
      <td>Kansas State University</td>
      <td>KState</td>
      <td>KStateAdmission</td>
    </tr>
    <tr>
      <th>144050</th>
      <td>University of Chicago</td>
      <td>thisisuic</td>
      <td>apply2uic</td>
    </tr>
    <tr>
      <th>164924</th>
      <td>Boston College</td>
      <td>BostonCollege</td>
      <td>bc_admission</td>
    </tr>
    <tr>
      <th>204024</th>
      <td>Miami University-Oxford</td>
      <td>miamiuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>119605</th>
      <td>National University</td>
      <td>natuniv</td>
      <td>cuaadmission</td>
    </tr>
    <tr>
      <th>126562</th>
      <td>University of Colorado Denver</td>
      <td>cudenver</td>
      <td></td>
    </tr>
    <tr>
      <th>135726</th>
      <td>University of Miami</td>
      <td>univmiami</td>
      <td>umadmission</td>
    </tr>
    <tr>
      <th>157289</th>
      <td>University of Louisville</td>
      <td>uofl</td>
      <td>UofLadm</td>
    </tr>
    <tr>
      <th>169798</th>
      <td>Eastern Michigan University</td>
      <td>EasternMichU</td>
      <td></td>
    </tr>
    <tr>
      <th>110592</th>
      <td>California State University-Los Angeles</td>
      <td>calstatela</td>
      <td></td>
    </tr>
    <tr>
      <th>110714</th>
      <td>University of California-Santa Cruz</td>
      <td>ucsc</td>
      <td>ucsc_admission</td>
    </tr>
    <tr>
      <th>196060</th>
      <td>SUNY at Albany</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <th>141574</th>
      <td>University of Hawaii at Manoa</td>
      <td>UHManoa</td>
      <td>manoaadmissions</td>
    </tr>
    <tr>
      <th>230782</th>
      <td>Weber State University</td>
      <td>WeberStateU</td>
      <td></td>
    </tr>
    <tr>
      <th>190512</th>
      <td>CUNY Bernard M Baruch College</td>
      <td>baruchcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>110529</th>
      <td>California State Polytechnic University-Pomona</td>
      <td>calpolypomona</td>
      <td>cpp_admissions</td>
    </tr>
    <tr>
      <th>139658</th>
      <td>Emory University</td>
      <td>emoryuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>110556</th>
      <td>California State University-Fresno</td>
      <td>Fresno_State</td>
      <td></td>
    </tr>
    <tr>
      <th>230737</th>
      <td>Utah Valley University</td>
      <td>UVU</td>
      <td></td>
    </tr>
    <tr>
      <th>228796</th>
      <td>The University of Texas at El Paso</td>
      <td>utepnews</td>
      <td></td>
    </tr>
    <tr>
      <th>196592</th>
      <td>Touro College</td>
      <td>WeAreTouro</td>
      <td></td>
    </tr>
    <tr>
      <th>179867</th>
      <td>Washington University in St Louis</td>
      <td>wustl</td>
      <td>WashUAdmissions</td>
    </tr>
    <tr>
      <th>100663</th>
      <td>University of Alabama at Birmingham</td>
      <td>chooseuab</td>
      <td></td>
    </tr>
    <tr>
      <th>185590</th>
      <td>Montclair State University</td>
      <td>montclairstateu</td>
      <td></td>
    </tr>
    <tr>
      <th>197869</th>
      <td>Appalachian State University</td>
      <td>appstate</td>
      <td></td>
    </tr>
    <tr>
      <th>221999</th>
      <td>Vanderbilt University</td>
      <td>vanderbiltu</td>
      <td></td>
    </tr>
    <tr>
      <th>220862</th>
      <td>University of Memphis</td>
      <td>uofmemphis</td>
      <td>membound</td>
    </tr>
    <tr>
      <th>211440</th>
      <td>Carnegie Mellon University</td>
      <td>carnegiemellon</td>
      <td>CM_Admission</td>
    </tr>
    <tr>
      <th>130794</th>
      <td>Yale University</td>
      <td>yale</td>
      <td></td>
    </tr>
    <tr>
      <th>195809</th>
      <td>St John's University-New York</td>
      <td>StJohnsU</td>
      <td></td>
    </tr>
    <tr>
      <th>140164</th>
      <td>Kennesaw State University</td>
      <td>kennesawstate</td>
      <td></td>
    </tr>
    <tr>
      <th>179566</th>
      <td>Missouri State University-Springfield</td>
      <td>missouristate</td>
      <td></td>
    </tr>
    <tr>
      <th>183044</th>
      <td>University of New Hampshire-Main Campus</td>
      <td>uofnh</td>
      <td></td>
    </tr>
    <tr>
      <th>227881</th>
      <td>Sam Houston State University</td>
      <td>samhoustonstate</td>
      <td></td>
    </tr>
    <tr>
      <th>157951</th>
      <td>Western Kentucky University</td>
      <td>wku</td>
      <td>wkuadmissions</td>
    </tr>
    <tr>
      <th>176080</th>
      <td>Mississippi State University</td>
      <td>msstate</td>
      <td>MSStateAdmit</td>
    </tr>
    <tr>
      <th>136172</th>
      <td>University of North Florida</td>
      <td>UofNorthFlorida</td>
      <td>unfadmissions</td>
    </tr>
    <tr>
      <th>199148</th>
      <td>University of North Carolina at Greensboro</td>
      <td>UNCG</td>
      <td></td>
    </tr>
    <tr>
      <th>196079</th>
      <td>SUNY at Binghamton</td>
      <td>binghamtonu</td>
      <td></td>
    </tr>
    <tr>
      <th>131159</th>
      <td>American University</td>
      <td>AmericanU</td>
      <td></td>
    </tr>
    <tr>
      <th>152080</th>
      <td>University of Notre Dame</td>
      <td>NotreDame</td>
      <td></td>
    </tr>
    <tr>
      <th>110538</th>
      <td>California State University-Chico</td>
      <td>chicostate</td>
      <td>ChicoAdmissions</td>
    </tr>
    <tr>
      <th>127060</th>
      <td>University of Denver</td>
      <td>uofdenver</td>
      <td></td>
    </tr>
    <tr>
      <th>106458</th>
      <td>Arkansas State University-Main Campus</td>
      <td>ArkansasState</td>
      <td></td>
    </tr>
    <tr>
      <th>110510</th>
      <td>California State University-San Bernardino</td>
      <td>csusbnews</td>
      <td></td>
    </tr>
    <tr>
      <th>142115</th>
      <td>Boise State University</td>
      <td>boisestatelive</td>
      <td></td>
    </tr>
    <tr>
      <th>110574</th>
      <td>California State University-East Bay</td>
      <td>CalStateEastBay</td>
      <td></td>
    </tr>
    <tr>
      <th>139931</th>
      <td>Georgia Southern University</td>
      <td>georgiasouthern</td>
      <td>GS_Admissions</td>
    </tr>
    <tr>
      <th>228246</th>
      <td>Southern Methodist University</td>
      <td>smu</td>
      <td></td>
    </tr>
    <tr>
      <th>171571</th>
      <td>Oakland University</td>
      <td>oaklandu</td>
      <td>OUAdmissions</td>
    </tr>
    <tr>
      <th>217484</th>
      <td>University of Rhode Island</td>
      <td>universityofri</td>
      <td></td>
    </tr>
    <tr>
      <th>202134</th>
      <td>Cleveland State University</td>
      <td>CLE_State</td>
      <td></td>
    </tr>
    <tr>
      <th>237011</th>
      <td>Western Washington University</td>
      <td>WWU</td>
      <td></td>
    </tr>
    <tr>
      <th>110422</th>
      <td>California Polytechnic State University-San Lu...</td>
      <td>calpoly</td>
      <td></td>
    </tr>
    <tr>
      <th>201441</th>
      <td>Bowling Green State University-Main Campus</td>
      <td>bgsu</td>
      <td>BGSUAdmissions</td>
    </tr>
    <tr>
      <th>229179</th>
      <td>Texas Woman's University</td>
      <td>txwomans</td>
      <td></td>
    </tr>
    <tr>
      <th>195003</th>
      <td>Rochester Institute of Technology</td>
      <td>RITNEWS</td>
      <td>RITAdmissions</td>
    </tr>
    <tr>
      <th>190549</th>
      <td>CUNY Brooklyn College</td>
      <td>BklynCollege411</td>
      <td></td>
    </tr>
    <tr>
      <th>168148</th>
      <td>Tufts University</td>
      <td>TuftsUniversity</td>
      <td>tuftsadmissions</td>
    </tr>
    <tr>
      <th>169910</th>
      <td>Ferris State University</td>
      <td>FerrisState</td>
      <td>FerrisAdmission</td>
    </tr>
    <tr>
      <th>216764</th>
      <td>West Chester University of Pennsylvania</td>
      <td>wcuofpa</td>
      <td></td>
    </tr>
    <tr>
      <th>166638</th>
      <td>University of Massachusetts-Boston</td>
      <td>umassboston</td>
      <td></td>
    </tr>
    <tr>
      <th>206604</th>
      <td>Wright State University-Main Campus</td>
      <td>WrightState</td>
      <td></td>
    </tr>
    <tr>
      <th>227368</th>
      <td>The University of Texas-Pan American</td>
      <td>utpa</td>
      <td></td>
    </tr>
    <tr>
      <th>182290</th>
      <td>University of Nevada-Reno</td>
      <td>unevadareno</td>
      <td></td>
    </tr>
    <tr>
      <th>188030</th>
      <td>New Mexico State University-Main Campus</td>
      <td>nmsu</td>
      <td></td>
    </tr>
    <tr>
      <th>127918</th>
      <td>Regis University</td>
      <td>RegisUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>166683</th>
      <td>Massachusetts Institute of Technology</td>
      <td>mit</td>
      <td></td>
    </tr>
    <tr>
      <th>216597</th>
      <td>Villanova University</td>
      <td>VillanovaU</td>
      <td>VUAdmission</td>
    </tr>
    <tr>
      <th>166513</th>
      <td>University of Massachusetts-Lowell</td>
      <td>umasslowell</td>
      <td></td>
    </tr>
    <tr>
      <th>213020</th>
      <td>Indiana University of Pennsylvania-Main Campus</td>
      <td>iupedu</td>
      <td></td>
    </tr>
    <tr>
      <th>156620</th>
      <td>Eastern Kentucky University</td>
      <td>eku</td>
      <td>EKUAdmissions</td>
    </tr>
    <tr>
      <th>196264</th>
      <td>SUNY Empire State College</td>
      <td>SUNYEmpire</td>
      <td></td>
    </tr>
    <tr>
      <th>179159</th>
      <td>Saint Louis University</td>
      <td>SLU_Official</td>
      <td></td>
    </tr>
    <tr>
      <th>231174</th>
      <td>University of Vermont</td>
      <td>uvmvermont</td>
      <td></td>
    </tr>
    <tr>
      <th>176372</th>
      <td>University of Southern Mississippi</td>
      <td>southernmiss</td>
      <td></td>
    </tr>
    <tr>
      <th>174783</th>
      <td>Saint Cloud State University</td>
      <td>stcloudstate</td>
      <td></td>
    </tr>
    <tr>
      <th>194310</th>
      <td>Pace University-New York</td>
      <td>PaceUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>155061</th>
      <td>Fort Hays State University</td>
      <td>forthaysstate</td>
      <td></td>
    </tr>
    <tr>
      <th>226091</th>
      <td>Lamar University</td>
      <td>LamarUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>199218</th>
      <td>University of North Carolina Wilmington</td>
      <td>uncwilmington</td>
      <td>UNCW_Admissions</td>
    </tr>
    <tr>
      <th>180489</th>
      <td>The University of Montana</td>
      <td>umontana</td>
      <td></td>
    </tr>
    <tr>
      <th>178402</th>
      <td>University of Missouri-Kansas City</td>
      <td>umkansascity</td>
      <td></td>
    </tr>
    <tr>
      <th>110547</th>
      <td>California State University-Dominguez Hills</td>
      <td>dominguezhills</td>
      <td></td>
    </tr>
    <tr>
      <th>191649</th>
      <td>Hofstra University</td>
      <td>hofstrau</td>
      <td></td>
    </tr>
    <tr>
      <th>224554</th>
      <td>Texas A &amp; M University-Commerce</td>
      <td>tamuc</td>
      <td></td>
    </tr>
    <tr>
      <th>131113</th>
      <td>Wilmington University</td>
      <td>theWilmU</td>
      <td></td>
    </tr>
    <tr>
      <th>173920</th>
      <td>Minnesota State University-Mankato</td>
      <td>MNSUMankato</td>
      <td></td>
    </tr>
    <tr>
      <th>195030</th>
      <td>University of Rochester</td>
      <td>UofR</td>
      <td>URAdmissions</td>
    </tr>
    <tr>
      <th>154095</th>
      <td>University of Northern Iowa</td>
      <td>northerniowa</td>
      <td></td>
    </tr>
    <tr>
      <th>220075</th>
      <td>East Tennessee State University</td>
      <td>etsu</td>
      <td>etsuadmissions</td>
    </tr>
    <tr>
      <th>217235</th>
      <td>Johnson &amp; Wales University-Providence</td>
      <td>JWUProvidence</td>
      <td></td>
    </tr>
    <tr>
      <th>234827</th>
      <td>Central Washington University</td>
      <td>CentralWashU</td>
      <td></td>
    </tr>
    <tr>
      <th>127565</th>
      <td>Metropolitan State University of Denver</td>
      <td>msudenver</td>
      <td></td>
    </tr>
    <tr>
      <th>149231</th>
      <td>Southern Illinois University-Edwardsville</td>
      <td>siue</td>
      <td></td>
    </tr>
    <tr>
      <th>163268</th>
      <td>University of Maryland-Baltimore County</td>
      <td>umbc</td>
      <td></td>
    </tr>
    <tr>
      <th>142285</th>
      <td>University of Idaho</td>
      <td>uidaho</td>
      <td></td>
    </tr>
    <tr>
      <th>181394</th>
      <td>University of Nebraska at Omaha</td>
      <td>unomaha</td>
      <td></td>
    </tr>
    <tr>
      <th>190567</th>
      <td>CUNY City College</td>
      <td>citycollegeny</td>
      <td></td>
    </tr>
    <tr>
      <th>180814</th>
      <td>Bellevue University</td>
      <td>BellevueU</td>
      <td></td>
    </tr>
    <tr>
      <th>149772</th>
      <td>Western Illinois University</td>
      <td>WesternILUniv</td>
      <td>WIUAdmissions</td>
    </tr>
    <tr>
      <th>178420</th>
      <td>University of Missouri-St Louis</td>
      <td>umsl</td>
      <td></td>
    </tr>
    <tr>
      <th>193654</th>
      <td>The New School</td>
      <td>thenewschool</td>
      <td></td>
    </tr>
    <tr>
      <th>156125</th>
      <td>Wichita State University</td>
      <td>wichitastate</td>
      <td></td>
    </tr>
    <tr>
      <th>186399</th>
      <td>Rutgers University-Newark</td>
      <td>rutgers_newark</td>
      <td></td>
    </tr>
    <tr>
      <th>206941</th>
      <td>University of Central Oklahoma</td>
      <td>UCOBronchos</td>
      <td></td>
    </tr>
    <tr>
      <th>177968</th>
      <td>Lindenwood University</td>
      <td>lindenwoodu</td>
      <td></td>
    </tr>
    <tr>
      <th>122612</th>
      <td>University of San Francisco</td>
      <td>usfca</td>
      <td>usfca_admission</td>
    </tr>
    <tr>
      <th>190600</th>
      <td>CUNY John Jay College of Criminal Justice</td>
      <td>JohnJayCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>240727</th>
      <td>University of Wyoming</td>
      <td>uwyonews</td>
      <td></td>
    </tr>
    <tr>
      <th>102094</th>
      <td>University of South Alabama</td>
      <td>UofSouthAlabama</td>
      <td></td>
    </tr>
    <tr>
      <th>157447</th>
      <td>Northern Kentucky University</td>
      <td>nkuedu</td>
      <td></td>
    </tr>
    <tr>
      <th>174914</th>
      <td>University of St Thomas</td>
      <td>UofStThomasMN</td>
      <td></td>
    </tr>
    <tr>
      <th>212106</th>
      <td>Duquesne University</td>
      <td>duqedu</td>
      <td></td>
    </tr>
    <tr>
      <th>184782</th>
      <td>Rowan University</td>
      <td>RowanUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>160658</th>
      <td>University of Louisiana at Lafayette</td>
      <td>ULLafayette</td>
      <td>raginrecruiters</td>
    </tr>
    <tr>
      <th>200332</th>
      <td>North Dakota State University-Main Campus</td>
      <td>ndsu</td>
      <td>ndsuadmission</td>
    </tr>
    <tr>
      <th>201645</th>
      <td>Case Western Reserve University</td>
      <td>cwru</td>
      <td>cwruadmission</td>
    </tr>
    <tr>
      <th>202480</th>
      <td>University of Dayton</td>
      <td>univofdayton</td>
      <td>DaytonAdmission</td>
    </tr>
    <tr>
      <th>200280</th>
      <td>University of North Dakota</td>
      <td>UofNorthDakota</td>
      <td>ndadmissions</td>
    </tr>
    <tr>
      <th>144892</th>
      <td>Eastern Illinois University</td>
      <td>eiu</td>
      <td>eiu_admissions</td>
    </tr>
    <tr>
      <th>127741</th>
      <td>University of Northern Colorado</td>
      <td>UNC_Colorado</td>
      <td></td>
    </tr>
    <tr>
      <th>235097</th>
      <td>Eastern Washington University</td>
      <td>ewueagles</td>
      <td></td>
    </tr>
    <tr>
      <th>231624</th>
      <td>College of William and Mary</td>
      <td>williamandmary</td>
      <td>WM_Admission</td>
    </tr>
    <tr>
      <th>109785</th>
      <td>Azusa Pacific University</td>
      <td>azusapacific</td>
      <td></td>
    </tr>
    <tr>
      <th>196130</th>
      <td>Buffalo State SUNY</td>
      <td>buffalostate</td>
      <td></td>
    </tr>
    <tr>
      <th>138354</th>
      <td>The University of West Florida</td>
      <td>UWF</td>
      <td>uwfadmissions</td>
    </tr>
    <tr>
      <th>141264</th>
      <td>Valdosta State University</td>
      <td>valdostastate</td>
      <td></td>
    </tr>
    <tr>
      <th>117946</th>
      <td>Loyola Marymount University</td>
      <td>loyolamarymount</td>
      <td></td>
    </tr>
    <tr>
      <th>178721</th>
      <td>Park University</td>
      <td>parkuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>122436</th>
      <td>University of San Diego</td>
      <td>uofsandiego</td>
      <td>usdadmissions</td>
    </tr>
    <tr>
      <th>217156</th>
      <td>Brown University</td>
      <td>BrownUniversity</td>
      <td>brownuadmission</td>
    </tr>
    <tr>
      <th>176965</th>
      <td>University of Central Missouri</td>
      <td>ucentralmo</td>
      <td></td>
    </tr>
    <tr>
      <th>228431</th>
      <td>Stephen F Austin State University</td>
      <td>SFASU</td>
      <td></td>
    </tr>
    <tr>
      <th>237525</th>
      <td>Marshall University</td>
      <td>marshallu</td>
      <td>marshalluapply</td>
    </tr>
    <tr>
      <th>217819</th>
      <td>College of Charleston</td>
      <td>CofC</td>
      <td>cofcadmissions</td>
    </tr>
    <tr>
      <th>165024</th>
      <td>Bridgewater State University</td>
      <td>BridgeStateU</td>
      <td></td>
    </tr>
    <tr>
      <th>240189</th>
      <td>University of Wisconsin-Whitewater</td>
      <td>uwwhitewater</td>
      <td>uwwadmissions</td>
    </tr>
    <tr>
      <th>121150</th>
      <td>Pepperdine University</td>
      <td>pepperdine</td>
      <td>seaveradmission</td>
    </tr>
    <tr>
      <th>122931</th>
      <td>Santa Clara University</td>
      <td>SantaClaraUniv</td>
      <td>scuadmission</td>
    </tr>
    <tr>
      <th>168005</th>
      <td>Suffolk University</td>
      <td>Suffolk_U</td>
      <td>ApplyToSuffolk</td>
    </tr>
    <tr>
      <th>186584</th>
      <td>Seton Hall University</td>
      <td>setonhall</td>
      <td></td>
    </tr>
    <tr>
      <th>106467</th>
      <td>Arkansas Tech University</td>
      <td>arkansastech</td>
      <td>go2ATu</td>
    </tr>
    <tr>
      <th>128771</th>
      <td>Central Connecticut State University</td>
      <td>CCSU</td>
      <td></td>
    </tr>
    <tr>
      <th>140951</th>
      <td>Savannah College of Art and Design</td>
      <td>SCADdotedu</td>
      <td></td>
    </tr>
    <tr>
      <th>228875</th>
      <td>Texas Christian University</td>
      <td>tcu</td>
      <td>tcuadmission</td>
    </tr>
    <tr>
      <th>145725</th>
      <td>Illinois Institute of Technology</td>
      <td>illinoistech</td>
      <td>iitugadmission</td>
    </tr>
    <tr>
      <th>180461</th>
      <td>Montana State University</td>
      <td>montanastate</td>
      <td>AdmissionsMSU</td>
    </tr>
    <tr>
      <th>130493</th>
      <td>Southern Connecticut State University</td>
      <td>scsu</td>
      <td></td>
    </tr>
    <tr>
      <th>219356</th>
      <td>South Dakota State University</td>
      <td>sdstate</td>
      <td></td>
    </tr>
    <tr>
      <th>215770</th>
      <td>Saint Joseph's University</td>
      <td>saintjosephs</td>
      <td>sjuadmissions</td>
    </tr>
    <tr>
      <th>190637</th>
      <td>CUNY Lehman College</td>
      <td>LehmanCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>132471</th>
      <td>Barry University</td>
      <td>BarryUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>174233</th>
      <td>University of Minnesota-Duluth</td>
      <td>umnduluth</td>
      <td>UMDuluthAdmis</td>
    </tr>
    <tr>
      <th>188429</th>
      <td>Adelphi University</td>
      <td>AdelphiU</td>
      <td></td>
    </tr>
    <tr>
      <th>211361</th>
      <td>California University of Pennsylvania</td>
      <td>CalUofPA</td>
      <td>calu_admissions</td>
    </tr>
    <tr>
      <th>108232</th>
      <td>Academy of Art University</td>
      <td>academy_of_art</td>
      <td></td>
    </tr>
    <tr>
      <th>102553</th>
      <td>University of Alaska Anchorage</td>
      <td>uaanchorage</td>
      <td></td>
    </tr>
    <tr>
      <th>160612</th>
      <td>Southeastern Louisiana University</td>
      <td>oursoutheastern</td>
      <td></td>
    </tr>
    <tr>
      <th>240268</th>
      <td>University of Wisconsin-Eau Claire</td>
      <td>uweauclaire</td>
      <td>uwadmissions</td>
    </tr>
    <tr>
      <th>228705</th>
      <td>Texas A &amp; M University-Kingsville</td>
      <td>JavelinaNation</td>
      <td>beajavelina</td>
    </tr>
    <tr>
      <th>240365</th>
      <td>University of Wisconsin-Oshkosh</td>
      <td>uwoshkosh</td>
      <td>uwoadmissions</td>
    </tr>
    <tr>
      <th>225432</th>
      <td>University of Houston-Downtown</td>
      <td>UHDowntown</td>
      <td>uhdadmissions</td>
    </tr>
    <tr>
      <th>193016</th>
      <td>Mercy College</td>
      <td>mercycollege</td>
      <td></td>
    </tr>
    <tr>
      <th>240329</th>
      <td>University of Wisconsin-La Crosse</td>
      <td>uwlacrosse</td>
      <td>uwladmissions</td>
    </tr>
    <tr>
      <th>130226</th>
      <td>Quinnipiac University</td>
      <td>quinnipiacu</td>
      <td></td>
    </tr>
    <tr>
      <th>235316</th>
      <td>Gonzaga University</td>
      <td>GonzagaU</td>
      <td>zagadmissions</td>
    </tr>
    <tr>
      <th>196121</th>
      <td>SUNY College at Brockport</td>
      <td>brockport</td>
      <td>bportadmissions</td>
    </tr>
    <tr>
      <th>106245</th>
      <td>University of Arkansas at Little Rock</td>
      <td>ualr</td>
      <td>ualradmissions</td>
    </tr>
    <tr>
      <th>142276</th>
      <td>Idaho State University</td>
      <td>IdahoStateU</td>
      <td></td>
    </tr>
    <tr>
      <th>236595</th>
      <td>Seattle University</td>
      <td>seattleu</td>
      <td>seattleuadm</td>
    </tr>
    <tr>
      <th>117140</th>
      <td>University of La Verne</td>
      <td>ULaVerne</td>
      <td></td>
    </tr>
    <tr>
      <th>140447</th>
      <td>Mercer University</td>
      <td>MercerYou</td>
      <td>mercernow</td>
    </tr>
    <tr>
      <th>200004</th>
      <td>Western Carolina University</td>
      <td>wcu</td>
      <td>WCUadmission</td>
    </tr>
    <tr>
      <th>216038</th>
      <td>Slippery Rock University of Pennsylvania</td>
      <td>sruofpa</td>
      <td></td>
    </tr>
    <tr>
      <th>229780</th>
      <td>Wayland Baptist University</td>
      <td>waylandbaptist</td>
      <td></td>
    </tr>
    <tr>
      <th>196176</th>
      <td>State University of New York at New Paltz</td>
      <td>newpaltz</td>
      <td></td>
    </tr>
    <tr>
      <th>187444</th>
      <td>William Paterson University of New Jersey</td>
      <td>WPUNJ_EDU</td>
      <td>wpunj_admission</td>
    </tr>
    <tr>
      <th>199847</th>
      <td>Wake Forest University</td>
      <td>WakeForest</td>
      <td>WFUadmissions</td>
    </tr>
    <tr>
      <th>186867</th>
      <td>Stevens Institute of Technology</td>
      <td>FollowStevens</td>
      <td>followstevens</td>
    </tr>
    <tr>
      <th>190558</th>
      <td>College of Staten Island CUNY</td>
      <td>csinews</td>
      <td></td>
    </tr>
    <tr>
      <th>163851</th>
      <td>Salisbury University</td>
      <td>salisburyu</td>
      <td>flocktosu</td>
    </tr>
    <tr>
      <th>181002</th>
      <td>Creighton University</td>
      <td>creighton</td>
      <td>ChooseCreighton</td>
    </tr>
    <tr>
      <th>221847</th>
      <td>Tennessee Technological University</td>
      <td>tennesseetech</td>
      <td>followme2ttu</td>
    </tr>
    <tr>
      <th>366711</th>
      <td>California State University-San Marcos</td>
      <td>csusm</td>
      <td></td>
    </tr>
    <tr>
      <th>151324</th>
      <td>Indiana State University</td>
      <td>indianastate</td>
      <td>choosestate</td>
    </tr>
    <tr>
      <th>144281</th>
      <td>Columbia College-Chicago</td>
      <td>ColumbiaChi</td>
      <td>columadmit</td>
    </tr>
    <tr>
      <th>151306</th>
      <td>University of Southern Indiana</td>
      <td>USIedu</td>
      <td></td>
    </tr>
    <tr>
      <th>141334</th>
      <td>University of West Georgia</td>
      <td>UnivWestGa</td>
      <td>uwgadmissions</td>
    </tr>
    <tr>
      <th>147776</th>
      <td>Northeastern Illinois University</td>
      <td>NEIU</td>
      <td></td>
    </tr>
    <tr>
      <th>178411</th>
      <td>Missouri University of Science and Technology</td>
      <td>MissouriSandT</td>
      <td>SandTadmissions</td>
    </tr>
    <tr>
      <th>165015</th>
      <td>Brandeis University</td>
      <td>BrandeisU</td>
      <td></td>
    </tr>
    <tr>
      <th>174020</th>
      <td>Metropolitan State University</td>
      <td>msudenver</td>
      <td>choose_metro</td>
    </tr>
    <tr>
      <th>221740</th>
      <td>The University of Tennessee-Chattanooga</td>
      <td>UTChattanooga</td>
      <td></td>
    </tr>
    <tr>
      <th>161253</th>
      <td>University of Maine</td>
      <td>UMaine</td>
      <td>GoUMaine</td>
    </tr>
    <tr>
      <th>219471</th>
      <td>University of South Dakota</td>
      <td>usd</td>
      <td>usd_admissions</td>
    </tr>
    <tr>
      <th>206695</th>
      <td>Youngstown State University</td>
      <td>youngstownstate</td>
      <td>ysuadmissions</td>
    </tr>
    <tr>
      <th>185828</th>
      <td>New Jersey Institute of Technology</td>
      <td>NJIT</td>
      <td></td>
    </tr>
    <tr>
      <th>169479</th>
      <td>Davenport University</td>
      <td>davenportu</td>
      <td></td>
    </tr>
    <tr>
      <th>186131</th>
      <td>Princeton University</td>
      <td>Princeton</td>
      <td>applyprinceton</td>
    </tr>
    <tr>
      <th>131520</th>
      <td>Howard University</td>
      <td>HowardU</td>
      <td>HowardAdmission</td>
    </tr>
    <tr>
      <th>157401</th>
      <td>Murray State University</td>
      <td>murraystateuniv</td>
      <td>raceradmissions</td>
    </tr>
    <tr>
      <th>159939</th>
      <td>University of New Orleans</td>
      <td>UofNO</td>
      <td>uno_admissions</td>
    </tr>
    <tr>
      <th>240480</th>
      <td>University of Wisconsin-Stevens Point</td>
      <td>UWStevensPoint</td>
      <td></td>
    </tr>
    <tr>
      <th>166452</th>
      <td>Lesley University</td>
      <td>lesley_u</td>
      <td></td>
    </tr>
    <tr>
      <th>186876</th>
      <td>The Richard Stockton College of New Jersey</td>
      <td>Stockton_edu</td>
      <td></td>
    </tr>
    <tr>
      <th>111948</th>
      <td>Chapman University</td>
      <td>chapmanu</td>
      <td>chapmanadmit</td>
    </tr>
    <tr>
      <th>196194</th>
      <td>SUNY College at Oswego</td>
      <td>sunyoswego</td>
      <td>oswegoadmission</td>
    </tr>
    <tr>
      <th>167729</th>
      <td>Salem State University</td>
      <td>salemstate</td>
      <td></td>
    </tr>
    <tr>
      <th>182670</th>
      <td>Dartmouth College</td>
      <td>dartmouth</td>
      <td></td>
    </tr>
    <tr>
      <th>194091</th>
      <td>New York Institute of Technology</td>
      <td>nyit</td>
      <td>nyitadmissions</td>
    </tr>
    <tr>
      <th>106704</th>
      <td>University of Central Arkansas</td>
      <td>ucabears</td>
      <td>ucaadmissions</td>
    </tr>
    <tr>
      <th>164739</th>
      <td>Bentley University</td>
      <td>bentleyu</td>
      <td>UGABentley</td>
    </tr>
    <tr>
      <th>179557</th>
      <td>Southeast Missouri State University</td>
      <td>SEMissouriState</td>
      <td></td>
    </tr>
    <tr>
      <th>151102</th>
      <td>Indiana University-Purdue University-Fort Wayne</td>
      <td>iufortwayne</td>
      <td></td>
    </tr>
    <tr>
      <th>213349</th>
      <td>Kutztown University of Pennsylvania</td>
      <td>KutztownU</td>
      <td>admissions_ku</td>
    </tr>
    <tr>
      <th>161554</th>
      <td>University of Southern Maine</td>
      <td>USouthernMaine</td>
      <td>usmadmissions</td>
    </tr>
    <tr>
      <th>219602</th>
      <td>Austin Peay State University</td>
      <td>austinpeay</td>
      <td>apsu_admissions</td>
    </tr>
    <tr>
      <th>213543</th>
      <td>Lehigh University</td>
      <td>LehighU</td>
      <td>lehighadmission</td>
    </tr>
    <tr>
      <th>123572</th>
      <td>Sonoma State University</td>
      <td>ssu_1961</td>
      <td></td>
    </tr>
    <tr>
      <th>185572</th>
      <td>Monmouth University</td>
      <td>monmouthu</td>
      <td></td>
    </tr>
    <tr>
      <th>227757</th>
      <td>Rice University</td>
      <td>riceuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>196149</th>
      <td>SUNY College at Cortland</td>
      <td>suny_cortland</td>
      <td>cortlandsuny</td>
    </tr>
    <tr>
      <th>147536</th>
      <td>National Louis University</td>
      <td>NationalLouisU</td>
      <td></td>
    </tr>
    <tr>
      <th>110495</th>
      <td>California State University-Stanislaus</td>
      <td>stan_state</td>
      <td>stanadmissions</td>
    </tr>
    <tr>
      <th>240417</th>
      <td>University of Wisconsin-Stout</td>
      <td>UWStout</td>
      <td>stoutadmissions</td>
    </tr>
    <tr>
      <th>152248</th>
      <td>Purdue University-Calumet Campus</td>
      <td>PurdueNorthwest</td>
      <td>purdueadmission</td>
    </tr>
    <tr>
      <th>175272</th>
      <td>Winona State University</td>
      <td>winonastateu</td>
      <td></td>
    </tr>
    <tr>
      <th>187134</th>
      <td>The College of New Jersey</td>
      <td>tcnj</td>
      <td>TCNJ_Admissions</td>
    </tr>
    <tr>
      <th>129941</th>
      <td>University of New Haven</td>
      <td>unewhaven</td>
      <td></td>
    </tr>
    <tr>
      <th>206622</th>
      <td>Xavier University</td>
      <td>XavierUniv</td>
      <td>xulaadmissions</td>
    </tr>
    <tr>
      <th>191968</th>
      <td>Ithaca College</td>
      <td>IthacaCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>216010</th>
      <td>Shippensburg University of Pennsylvania</td>
      <td>shippensburgU</td>
      <td></td>
    </tr>
    <tr>
      <th>120883</th>
      <td>University of the Pacific</td>
      <td>UOPacific</td>
      <td>uopadmission</td>
    </tr>
    <tr>
      <th>224147</th>
      <td>Texas A &amp; M University-Corpus Christi</td>
      <td>IslandCampus</td>
      <td>futureislander</td>
    </tr>
    <tr>
      <th>126580</th>
      <td>University of Colorado Colorado Springs</td>
      <td>uccs</td>
      <td></td>
    </tr>
    <tr>
      <th>159647</th>
      <td>Louisiana Tech University</td>
      <td>latech</td>
      <td>techadmissions</td>
    </tr>
    <tr>
      <th>110486</th>
      <td>California State University-Bakersfield</td>
      <td>csubakersfield</td>
      <td></td>
    </tr>
    <tr>
      <th>202806</th>
      <td>Franklin University</td>
      <td>FranklinU</td>
      <td></td>
    </tr>
    <tr>
      <th>196246</th>
      <td>SUNY College at Plattsburgh</td>
      <td>sunyplattsburgh</td>
      <td>plattslife</td>
    </tr>
    <tr>
      <th>172051</th>
      <td>Saginaw Valley State University</td>
      <td>SVSU</td>
      <td></td>
    </tr>
    <tr>
      <th>171456</th>
      <td>Northern Michigan University</td>
      <td>NorthernMichU</td>
      <td></td>
    </tr>
    <tr>
      <th>207263</th>
      <td>Northeastern State University</td>
      <td>NSURiverhawks</td>
      <td></td>
    </tr>
    <tr>
      <th>160038</th>
      <td>Northwestern State University of Louisiana</td>
      <td>nsula</td>
      <td></td>
    </tr>
    <tr>
      <th>213367</th>
      <td>La Salle University</td>
      <td>lasalleuniv</td>
      <td>lasalle_admiss</td>
    </tr>
    <tr>
      <th>167987</th>
      <td>University of Massachusetts-Dartmouth</td>
      <td>umassd</td>
      <td></td>
    </tr>
    <tr>
      <th>184603</th>
      <td>Fairleigh Dickinson University-Metropolitan Ca...</td>
      <td>FDUWhatsNew</td>
      <td>fduadmissions</td>
    </tr>
    <tr>
      <th>148487</th>
      <td>Roosevelt University</td>
      <td>RooseveltU</td>
      <td></td>
    </tr>
    <tr>
      <th>225627</th>
      <td>University of the Incarnate Word</td>
      <td>uiwcardinals</td>
      <td>uiw_admissions</td>
    </tr>
    <tr>
      <th>199102</th>
      <td>North Carolina A &amp; T State University</td>
      <td>ncatsuaggies</td>
      <td>ncatadmissions</td>
    </tr>
    <tr>
      <th>101480</th>
      <td>Jacksonville State University</td>
      <td>JSUNews</td>
      <td>jsuadmissions</td>
    </tr>
    <tr>
      <th>146612</th>
      <td>Lewis University</td>
      <td>LewisUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>129525</th>
      <td>University of Hartford</td>
      <td>UofHartford</td>
      <td></td>
    </tr>
    <tr>
      <th>174817</th>
      <td>Saint Mary's University of Minnesota</td>
      <td>smumn</td>
      <td>smc_admission</td>
    </tr>
    <tr>
      <th>131283</th>
      <td>Catholic University of America</td>
      <td>CatholicUniv</td>
      <td>cuaadmission</td>
    </tr>
    <tr>
      <th>214041</th>
      <td>Millersville University of Pennsylvania</td>
      <td>millersvilleu</td>
      <td>villeadmissions</td>
    </tr>
    <tr>
      <th>230603</th>
      <td>Southern Utah University</td>
      <td>suutbirds</td>
      <td></td>
    </tr>
    <tr>
      <th>115755</th>
      <td>Humboldt State University</td>
      <td>humboldtstate</td>
      <td></td>
    </tr>
    <tr>
      <th>238616</th>
      <td>Concordia University-Wisconsin</td>
      <td>CUWisconsin</td>
      <td></td>
    </tr>
    <tr>
      <th>229814</th>
      <td>West Texas A &amp; M University</td>
      <td>wtamu</td>
      <td></td>
    </tr>
    <tr>
      <th>218724</th>
      <td>Coastal Carolina University</td>
      <td>CCUchanticleers</td>
      <td>ccu_admissions</td>
    </tr>
    <tr>
      <th>189705</th>
      <td>Canisius College</td>
      <td>canisiuscollege</td>
      <td></td>
    </tr>
    <tr>
      <th>228802</th>
      <td>The University of Texas at Tyler</td>
      <td>uttyler</td>
      <td>applyutt</td>
    </tr>
    <tr>
      <th>148335</th>
      <td>Robert Morris University Illinois</td>
      <td>RobertMorrisU</td>
      <td></td>
    </tr>
    <tr>
      <th>219709</th>
      <td>Belmont University</td>
      <td>BelmontUniv</td>
      <td></td>
    </tr>
    <tr>
      <th>139861</th>
      <td>Georgia College and State University</td>
      <td>georgiacollege</td>
      <td>GCAdmissions</td>
    </tr>
    <tr>
      <th>199157</th>
      <td>North Carolina Central University</td>
      <td>NCCU</td>
      <td>NCCUadmissions</td>
    </tr>
    <tr>
      <th>100706</th>
      <td>University of Alabama in Huntsville</td>
      <td>uahuntsville</td>
      <td>uahadmissions</td>
    </tr>
    <tr>
      <th>212160</th>
      <td>Edinboro University of Pennsylvania</td>
      <td>edinboro</td>
      <td>boroadmissions</td>
    </tr>
    <tr>
      <th>169716</th>
      <td>University of Detroit Mercy</td>
      <td>UDMDetroit</td>
      <td></td>
    </tr>
    <tr>
      <th>153269</th>
      <td>Drake University</td>
      <td>DrakeUniversity</td>
      <td>drakeadmission</td>
    </tr>
    <tr>
      <th>185129</th>
      <td>New Jersey City University</td>
      <td>njcuniversity</td>
      <td>njcuadmissions</td>
    </tr>
    <tr>
      <th>163046</th>
      <td>Loyola University Maryland</td>
      <td>loyolamaryland</td>
      <td>loyoladmission</td>
    </tr>
    <tr>
      <th>167783</th>
      <td>Simmons College</td>
      <td>simmonscollege</td>
      <td>applysimmons</td>
    </tr>
    <tr>
      <th>159717</th>
      <td>McNeese State University</td>
      <td>mcneese</td>
      <td>becomeacowboy</td>
    </tr>
    <tr>
      <th>171128</th>
      <td>Michigan Technological University</td>
      <td>MichiganTech</td>
      <td></td>
    </tr>
    <tr>
      <th>137847</th>
      <td>The University of Tampa</td>
      <td>UofTampa</td>
      <td></td>
    </tr>
    <tr>
      <th>240277</th>
      <td>University of Wisconsin-Green Bay</td>
      <td>uwgb</td>
      <td>admissions</td>
    </tr>
    <tr>
      <th>221838</th>
      <td>Tennessee State University</td>
      <td>TSUedu</td>
      <td>tsuadmissions</td>
    </tr>
    <tr>
      <th>230995</th>
      <td>Norwich University</td>
      <td>norwichnews</td>
      <td></td>
    </tr>
    <tr>
      <th>201104</th>
      <td>Ashland University</td>
      <td>Ashland_Univ</td>
      <td></td>
    </tr>
    <tr>
      <th>155681</th>
      <td>Pittsburg State University</td>
      <td>pittstate</td>
      <td>psuadmission</td>
    </tr>
    <tr>
      <th>198516</th>
      <td>Elon University</td>
      <td>elonuniversity</td>
      <td>ElonAdmissions</td>
    </tr>
    <tr>
      <th>168263</th>
      <td>Westfield State University</td>
      <td>westfieldstate</td>
      <td></td>
    </tr>
    <tr>
      <th>186371</th>
      <td>Rutgers University-Camden</td>
      <td>Rutgers_Camden</td>
      <td>rucamdenapply</td>
    </tr>
    <tr>
      <th>217420</th>
      <td>Rhode Island College</td>
      <td>RICNews</td>
      <td>pc_perspectives</td>
    </tr>
    <tr>
      <th>196185</th>
      <td>SUNY Oneonta</td>
      <td>SUNY_Oneonta</td>
      <td></td>
    </tr>
    <tr>
      <th>216931</th>
      <td>Wilkes University</td>
      <td>wilkesU</td>
      <td>wilkesuadm</td>
    </tr>
    <tr>
      <th>141644</th>
      <td>Hawaii Pacific University</td>
      <td>hpu</td>
      <td>gotohpu</td>
    </tr>
    <tr>
      <th>199999</th>
      <td>Winston-Salem State University</td>
      <td>WSSURAMS</td>
      <td></td>
    </tr>
    <tr>
      <th>178624</th>
      <td>Northwest Missouri State University</td>
      <td>nwmostate</td>
      <td></td>
    </tr>
    <tr>
      <th>168421</th>
      <td>Worcester Polytechnic Institute</td>
      <td>WPI</td>
      <td>WPIAdmissions</td>
    </tr>
    <tr>
      <th>171146</th>
      <td>University of Michigan-Flint</td>
      <td>UMFlint</td>
      <td></td>
    </tr>
    <tr>
      <th>139366</th>
      <td>Columbus State University</td>
      <td>ColumbusState</td>
      <td></td>
    </tr>
    <tr>
      <th>212115</th>
      <td>East Stroudsburg University of Pennsylvania</td>
      <td>esuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>240471</th>
      <td>University of Wisconsin-River Falls</td>
      <td>UWRiverfalls</td>
      <td></td>
    </tr>
    <tr>
      <th>195544</th>
      <td>Saint Joseph's College-New York</td>
      <td>SJCNY</td>
      <td></td>
    </tr>
    <tr>
      <th>143358</th>
      <td>Bradley University</td>
      <td>bradleyu</td>
      <td>Apply2Bradley</td>
    </tr>
    <tr>
      <th>196042</th>
      <td>Farmingdale State College</td>
      <td>farmingdalesc</td>
      <td></td>
    </tr>
    <tr>
      <th>240462</th>
      <td>University of Wisconsin-Platteville</td>
      <td>uwplatteville</td>
      <td></td>
    </tr>
    <tr>
      <th>192819</th>
      <td>Marist College</td>
      <td>Marist</td>
      <td></td>
    </tr>
    <tr>
      <th>174358</th>
      <td>Minnesota State University-Moorhead</td>
      <td>MSUMoorhead</td>
      <td>admissionsmsum</td>
    </tr>
    <tr>
      <th>196167</th>
      <td>SUNY College at Geneseo</td>
      <td>SUNYGeneseo</td>
      <td>geneseoua</td>
    </tr>
    <tr>
      <th>133881</th>
      <td>Florida Institute of Technology</td>
      <td>floridatech</td>
      <td></td>
    </tr>
    <tr>
      <th>196158</th>
      <td>SUNY at Fredonia</td>
      <td>FredoniaU</td>
      <td></td>
    </tr>
    <tr>
      <th>175856</th>
      <td>Jackson State University</td>
      <td>jacksonstateU</td>
      <td></td>
    </tr>
    <tr>
      <th>173665</th>
      <td>Hamline University</td>
      <td>HamlineU</td>
      <td>Admission_HU</td>
    </tr>
    <tr>
      <th>186201</th>
      <td>Ramapo College of New Jersey</td>
      <td>ramapocollegenj</td>
      <td>rcnjadmissions</td>
    </tr>
    <tr>
      <th>193973</th>
      <td>Niagara University</td>
      <td>niagarauniv</td>
      <td>nuadmit</td>
    </tr>
    <tr>
      <th>198561</th>
      <td>Gardner-Webb University</td>
      <td>gardnerwebb</td>
      <td>futuredawgs</td>
    </tr>
    <tr>
      <th>229063</th>
      <td>Texas Southern University</td>
      <td>TexasSouthern</td>
      <td>txsu_admissions</td>
    </tr>
    <tr>
      <th>194541</th>
      <td>Polytechnic Institute of New York University</td>
      <td>nyutandon</td>
      <td></td>
    </tr>
    <tr>
      <th>198136</th>
      <td>Campbell University</td>
      <td>campbelledu</td>
      <td>cuadmissions</td>
    </tr>
    <tr>
      <th>227526</th>
      <td>Prairie View A &amp; M University</td>
      <td>PVAMU</td>
      <td>thejaguarway</td>
    </tr>
    <tr>
      <th>156082</th>
      <td>Washburn University</td>
      <td>washburnuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>224226</th>
      <td>Dallas Baptist University</td>
      <td>dbupatriots</td>
      <td>dbuadmissions</td>
    </tr>
    <tr>
      <th>181215</th>
      <td>University of Nebraska at Kearney</td>
      <td>UNKearney</td>
      <td></td>
    </tr>
    <tr>
      <th>210429</th>
      <td>Western Oregon University</td>
      <td>wounews</td>
      <td>wouadmissions</td>
    </tr>
    <tr>
      <th>147828</th>
      <td>Olivet Nazarene University</td>
      <td>olivetnazarene</td>
      <td></td>
    </tr>
    <tr>
      <th>176053</th>
      <td>Mississippi College</td>
      <td>misscollege</td>
      <td>mc_admissions</td>
    </tr>
    <tr>
      <th>161457</th>
      <td>University of New England</td>
      <td>unetweets</td>
      <td>UNEadmissions</td>
    </tr>
    <tr>
      <th>177214</th>
      <td>Drury University</td>
      <td>DruryUniversity</td>
      <td>druryadmission</td>
    </tr>
    <tr>
      <th>128744</th>
      <td>University of Bridgeport</td>
      <td>UBridgeport</td>
      <td></td>
    </tr>
    <tr>
      <th>102049</th>
      <td>Samford University</td>
      <td>SamfordU</td>
      <td>choosesamford</td>
    </tr>
    <tr>
      <th>233374</th>
      <td>University of Richmond</td>
      <td>urichmond</td>
      <td>uradmission</td>
    </tr>
    <tr>
      <th>138789</th>
      <td>Armstrong Atlantic State University</td>
      <td>georgiasouthern</td>
      <td>appadmissions</td>
    </tr>
    <tr>
      <th>178615</th>
      <td>Truman State University</td>
      <td>TrumanState</td>
      <td>trumanadmission</td>
    </tr>
    <tr>
      <th>222831</th>
      <td>Angelo State University</td>
      <td>AngeloState</td>
      <td>goangelostate</td>
    </tr>
    <tr>
      <th>183080</th>
      <td>Plymouth State University</td>
      <td>PlymouthState</td>
      <td></td>
    </tr>
    <tr>
      <th>232681</th>
      <td>University of Mary Washington</td>
      <td>marywash</td>
      <td></td>
    </tr>
    <tr>
      <th>194578</th>
      <td>Pratt Institute-Main</td>
      <td>prattinstitute</td>
      <td>prattadmissions</td>
    </tr>
    <tr>
      <th>129242</th>
      <td>Fairfield University</td>
      <td>fairfieldu</td>
      <td>fairfield_admis</td>
    </tr>
    <tr>
      <th>221768</th>
      <td>The University of Tennessee-Martin</td>
      <td>utmartin</td>
      <td>ut_admissions</td>
    </tr>
    <tr>
      <th>143118</th>
      <td>Aurora University</td>
      <td>AuroraU</td>
      <td></td>
    </tr>
    <tr>
      <th>102614</th>
      <td>University of Alaska Fairbanks</td>
      <td>uafairbanks</td>
      <td></td>
    </tr>
    <tr>
      <th>171492</th>
      <td>Northwood University-Michigan</td>
      <td>northwoodu</td>
      <td></td>
    </tr>
    <tr>
      <th>186283</th>
      <td>Rider University</td>
      <td>RiderUniversity</td>
      <td>rideruadmission</td>
    </tr>
    <tr>
      <th>101879</th>
      <td>University of North Alabama</td>
      <td>north_alabama</td>
      <td>UNA_Admissions</td>
    </tr>
    <tr>
      <th>218964</th>
      <td>Winthrop University</td>
      <td>winthropu</td>
      <td></td>
    </tr>
    <tr>
      <th>148654</th>
      <td>University of Illinois at Springfield</td>
      <td>UISedu</td>
      <td>uisadmissions</td>
    </tr>
    <tr>
      <th>165820</th>
      <td>Fitchburg State University</td>
      <td>Fitchburg_State</td>
      <td></td>
    </tr>
    <tr>
      <th>226833</th>
      <td>Midwestern State University</td>
      <td>msutexas</td>
      <td>msu_admissions</td>
    </tr>
    <tr>
      <th>168430</th>
      <td>Worcester State University</td>
      <td>worcesterstate</td>
      <td></td>
    </tr>
    <tr>
      <th>159966</th>
      <td>Nicholls State University</td>
      <td>nichollsstate</td>
      <td>nsuadmissions</td>
    </tr>
    <tr>
      <th>155025</th>
      <td>Emporia State University</td>
      <td>emporiastate</td>
      <td>ESUadmissions</td>
    </tr>
    <tr>
      <th>110097</th>
      <td>Biola University</td>
      <td>biolau</td>
      <td></td>
    </tr>
    <tr>
      <th>235167</th>
      <td>The Evergreen State College</td>
      <td>EvergreenStCol</td>
      <td></td>
    </tr>
    <tr>
      <th>183062</th>
      <td>Keene State College</td>
      <td>kscadmissions</td>
      <td>KSCAdmissions</td>
    </tr>
    <tr>
      <th>148627</th>
      <td>Saint Xavier University</td>
      <td>SaintXavier</td>
      <td>sxuadmission</td>
    </tr>
    <tr>
      <th>107044</th>
      <td>Harding University</td>
      <td>HardingU</td>
      <td></td>
    </tr>
    <tr>
      <th>110413</th>
      <td>California Lutheran University</td>
      <td>callutheran</td>
      <td></td>
    </tr>
    <tr>
      <th>129215</th>
      <td>Eastern Connecticut State University</td>
      <td>EasternCTStateU</td>
      <td>WhyEastern</td>
    </tr>
    <tr>
      <th>199281</th>
      <td>University of North Carolina at Pembroke</td>
      <td>uncpembroke</td>
      <td>uncpadmissions</td>
    </tr>
    <tr>
      <th>112075</th>
      <td>Concordia University-Irvine</td>
      <td>ConcordiaIrvine</td>
      <td></td>
    </tr>
    <tr>
      <th>126775</th>
      <td>Colorado School of Mines</td>
      <td>coschoolofmines</td>
      <td>MinesAdmissions</td>
    </tr>
    <tr>
      <th>173160</th>
      <td>Bethel University</td>
      <td>BethelU</td>
      <td></td>
    </tr>
    <tr>
      <th>163453</th>
      <td>Morgan State University</td>
      <td>MorganStateU</td>
      <td>morganadmission</td>
    </tr>
    <tr>
      <th>210146</th>
      <td>Southern Oregon University</td>
      <td>souashland</td>
      <td>souadmissions</td>
    </tr>
    <tr>
      <th>209056</th>
      <td>Lewis &amp; Clark College</td>
      <td>lewisandclark</td>
      <td></td>
    </tr>
    <tr>
      <th>165866</th>
      <td>Framingham State University</td>
      <td>FraminghamU</td>
      <td>framstateadmit</td>
    </tr>
    <tr>
      <th>162584</th>
      <td>Frostburg State University</td>
      <td>frostburgstate</td>
      <td></td>
    </tr>
    <tr>
      <th>134945</th>
      <td>Jacksonville University</td>
      <td>JacksonvilleU</td>
      <td></td>
    </tr>
    <tr>
      <th>139311</th>
      <td>Clayton  State University</td>
      <td>ClaytonState</td>
      <td>claytonadm</td>
    </tr>
    <tr>
      <th>191931</th>
      <td>Iona College</td>
      <td>ionacollege</td>
      <td>icadmissions</td>
    </tr>
    <tr>
      <th>123554</th>
      <td>Saint Mary's College of California</td>
      <td>stmarysca</td>
      <td>smc_admission</td>
    </tr>
    <tr>
      <th>154688</th>
      <td>Baker University</td>
      <td>BakerUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>231712</th>
      <td>Christopher Newport University</td>
      <td>cnucaptains</td>
      <td>cnu_admission</td>
    </tr>
    <tr>
      <th>151263</th>
      <td>University of Indianapolis</td>
      <td>uindy</td>
      <td>uindyadmissions</td>
    </tr>
    <tr>
      <th>151379</th>
      <td>Indiana University-Southeast</td>
      <td>IUSoutheast</td>
      <td>iusadmissions</td>
    </tr>
    <tr>
      <th>165662</th>
      <td>Emerson College</td>
      <td>EmersonCollege</td>
      <td>emersonugadmis</td>
    </tr>
    <tr>
      <th>212133</th>
      <td>Eastern University</td>
      <td>easternU</td>
      <td>eiu_admissions</td>
    </tr>
    <tr>
      <th>195720</th>
      <td>Saint John Fisher College</td>
      <td>FisherNews</td>
      <td></td>
    </tr>
    <tr>
      <th>160621</th>
      <td>Southern University and A &amp; M College</td>
      <td>SouthernU_BR</td>
      <td></td>
    </tr>
    <tr>
      <th>173045</th>
      <td>Augsburg College</td>
      <td>augsburgu</td>
      <td></td>
    </tr>
    <tr>
      <th>189228</th>
      <td>Berkeley College-New York</td>
      <td>berkeleycollege</td>
      <td></td>
    </tr>
    <tr>
      <th>216852</th>
      <td>Widener University-Main Campus</td>
      <td>wideneruniv</td>
      <td>choosewidener</td>
    </tr>
    <tr>
      <th>219976</th>
      <td>Lipscomb University</td>
      <td>lipscomb</td>
      <td>golipscomb</td>
    </tr>
    <tr>
      <th>130776</th>
      <td>Western Connecticut State University</td>
      <td>westconn</td>
      <td>wcsuadmissions</td>
    </tr>
    <tr>
      <th>165334</th>
      <td>Clark University</td>
      <td>clarkuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>228149</th>
      <td>St Mary's University</td>
      <td>StMarysU</td>
      <td>stmuadmission</td>
    </tr>
    <tr>
      <th>172334</th>
      <td>Spring Arbor University</td>
      <td>springarboru</td>
      <td></td>
    </tr>
    <tr>
      <th>207458</th>
      <td>Oklahoma City University</td>
      <td>okcu</td>
      <td></td>
    </tr>
    <tr>
      <th>130697</th>
      <td>Wesleyan University</td>
      <td>wesleyan_u</td>
      <td></td>
    </tr>
    <tr>
      <th>221971</th>
      <td>Union University</td>
      <td>UnionUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>201195</th>
      <td>Baldwin Wallace University</td>
      <td>BaldwinWallace</td>
      <td>bwadmission</td>
    </tr>
    <tr>
      <th>218742</th>
      <td>University of South Carolina-Upstate</td>
      <td>USCUpstate</td>
      <td></td>
    </tr>
    <tr>
      <th>213613</th>
      <td>Lock Haven University</td>
      <td>LockHavenUniv</td>
      <td>LHU_Admissions</td>
    </tr>
    <tr>
      <th>197045</th>
      <td>Utica College</td>
      <td>uticacollege</td>
      <td></td>
    </tr>
    <tr>
      <th>198543</th>
      <td>Fayetteville State University</td>
      <td>uncfsu</td>
      <td>fsu_admissions</td>
    </tr>
    <tr>
      <th>211088</th>
      <td>Arcadia University</td>
      <td>arcadia1853</td>
      <td>arcadiabound</td>
    </tr>
    <tr>
      <th>152600</th>
      <td>Valparaiso University</td>
      <td>valpou</td>
      <td>valpoadmission</td>
    </tr>
    <tr>
      <th>236577</th>
      <td>Seattle Pacific University</td>
      <td>seattlepacific</td>
      <td></td>
    </tr>
    <tr>
      <th>154235</th>
      <td>Saint Ambrose University</td>
      <td>stambrose</td>
      <td></td>
    </tr>
    <tr>
      <th>211291</th>
      <td>Bucknell University</td>
      <td>BucknellU</td>
      <td></td>
    </tr>
    <tr>
      <th>197151</th>
      <td>School of Visual Arts</td>
      <td>sva_news</td>
      <td>svaadmissions</td>
    </tr>
    <tr>
      <th>222178</th>
      <td>Abilene Christian University</td>
      <td>acuedu</td>
      <td>acuadmissions</td>
    </tr>
    <tr>
      <th>217518</th>
      <td>Roger Williams University</td>
      <td>myrwu</td>
      <td>rwuadmission</td>
    </tr>
    <tr>
      <th>234155</th>
      <td>Virginia State University</td>
      <td>VSUTrojans</td>
      <td>vsu_admissions</td>
    </tr>
    <tr>
      <th>214713</th>
      <td>Pennsylvania State University-Penn State Harri...</td>
      <td>PSUHarrisburg</td>
      <td></td>
    </tr>
    <tr>
      <th>209825</th>
      <td>University of Portland</td>
      <td>UPortland</td>
      <td></td>
    </tr>
    <tr>
      <th>178341</th>
      <td>Missouri Southern State University</td>
      <td>mosolions</td>
      <td></td>
    </tr>
    <tr>
      <th>150163</th>
      <td>Butler University</td>
      <td>butleru</td>
      <td></td>
    </tr>
    <tr>
      <th>214591</th>
      <td>Pennsylvania State University-Penn State Erie-...</td>
      <td>psbehrend</td>
      <td>psu_admissions</td>
    </tr>
    <tr>
      <th>130183</th>
      <td>Post University</td>
      <td>Postuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>232706</th>
      <td>Marymount University</td>
      <td>marymountu</td>
      <td>mugradadmission</td>
    </tr>
    <tr>
      <th>164580</th>
      <td>Babson College</td>
      <td>babson</td>
      <td>babsonadmission</td>
    </tr>
    <tr>
      <th>137546</th>
      <td>Stetson University</td>
      <td>stetsonu</td>
      <td>stetsonadmit</td>
    </tr>
    <tr>
      <th>211352</th>
      <td>Cabrini College</td>
      <td>CabriniUniv</td>
      <td>Cabrini_Apply</td>
    </tr>
    <tr>
      <th>167835</th>
      <td>Smith College</td>
      <td>smithcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>151342</th>
      <td>Indiana University-South Bend</td>
      <td>IUSouthBend</td>
      <td></td>
    </tr>
    <tr>
      <th>193292</th>
      <td>Molloy College</td>
      <td>MolloyCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>153001</th>
      <td>Buena Vista University</td>
      <td>buenavistauniv</td>
      <td></td>
    </tr>
    <tr>
      <th>141097</th>
      <td>Southern Polytechnic State University</td>
      <td>kennesawstate</td>
      <td></td>
    </tr>
    <tr>
      <th>240107</th>
      <td>Viterbo University</td>
      <td>viterbo_univ</td>
      <td></td>
    </tr>
    <tr>
      <th>208822</th>
      <td>George Fox University</td>
      <td>georgefox</td>
      <td></td>
    </tr>
    <tr>
      <th>133553</th>
      <td>Embry-Riddle Aeronautical University-Daytona B...</td>
      <td>ERAU_Daytona</td>
      <td>EmbryRiddle</td>
    </tr>
    <tr>
      <th>190770</th>
      <td>Dowling College</td>
      <td>hofstrau</td>
      <td></td>
    </tr>
    <tr>
      <th>164748</th>
      <td>Berklee College of Music</td>
      <td>BerkleeCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>180179</th>
      <td>Montana State University-Billings</td>
      <td>msubillings</td>
      <td>admissionsmsu</td>
    </tr>
    <tr>
      <th>162007</th>
      <td>Bowie State University</td>
      <td>bowiestate</td>
      <td></td>
    </tr>
    <tr>
      <th>149781</th>
      <td>Wheaton College</td>
      <td>WheatonCollege</td>
      <td>wheatonadmiss</td>
    </tr>
    <tr>
      <th>173328</th>
      <td>Concordia University-Saint Paul</td>
      <td>concordiastpaul</td>
      <td></td>
    </tr>
    <tr>
      <th>128106</th>
      <td>Colorado State University-Pueblo</td>
      <td>csupueblo</td>
      <td></td>
    </tr>
    <tr>
      <th>212601</th>
      <td>Gannon University</td>
      <td>gannonu</td>
      <td></td>
    </tr>
    <tr>
      <th>230807</th>
      <td>Westminster College</td>
      <td>westminsterslc</td>
      <td></td>
    </tr>
    <tr>
      <th>202763</th>
      <td>The University of Findlay</td>
      <td>ufindlay</td>
      <td>oileradmissions</td>
    </tr>
    <tr>
      <th>207865</th>
      <td>Southwestern Oklahoma State University</td>
      <td>swosu</td>
      <td></td>
    </tr>
    <tr>
      <th>193584</th>
      <td>Nazareth College</td>
      <td>nazarethcollege</td>
      <td>nazadmissions</td>
    </tr>
    <tr>
      <th>178059</th>
      <td>Maryville University of Saint Louis</td>
      <td>maryvilleu</td>
      <td>maryvilleuadmit</td>
    </tr>
    <tr>
      <th>198695</th>
      <td>High Point University</td>
      <td>HighPointU</td>
      <td>HPUadmissions</td>
    </tr>
    <tr>
      <th>156286</th>
      <td>Bellarmine University</td>
      <td>bellarmineu</td>
      <td>bellarmineadmit</td>
    </tr>
    <tr>
      <th>151290</th>
      <td>Indiana Institute of Technology</td>
      <td>indianatech</td>
      <td>tech_admissions</td>
    </tr>
    <tr>
      <th>172264</th>
      <td>Siena Heights University</td>
      <td>sienaheights</td>
      <td></td>
    </tr>
    <tr>
      <th>209612</th>
      <td>Pacific University</td>
      <td>pacificu</td>
      <td>pacu_admissions</td>
    </tr>
    <tr>
      <th>144962</th>
      <td>Elmhurst College</td>
      <td>elmhurstcollege</td>
      <td>ec_admissions</td>
    </tr>
    <tr>
      <th>199069</th>
      <td>Mount Olive College</td>
      <td>OfficialUMO</td>
      <td>mountadmissions</td>
    </tr>
    <tr>
      <th>215099</th>
      <td>Philadelphia University</td>
      <td>Penn</td>
      <td>lasalle_admiss</td>
    </tr>
    <tr>
      <th>174844</th>
      <td>St Olaf College</td>
      <td>stolaf</td>
      <td></td>
    </tr>
    <tr>
      <th>169080</th>
      <td>Calvin College</td>
      <td>calvincollege</td>
      <td></td>
    </tr>
    <tr>
      <th>217165</th>
      <td>Bryant University</td>
      <td>BryantUniv</td>
      <td>bryantadmission</td>
    </tr>
    <tr>
      <th>168254</th>
      <td>Western New England University</td>
      <td>wneuniversity</td>
      <td>wneadmissions</td>
    </tr>
    <tr>
      <th>196219</th>
      <td>SUNY at Purchase College</td>
      <td>SUNY_Purchase</td>
      <td></td>
    </tr>
    <tr>
      <th>148584</th>
      <td>University of St Francis</td>
      <td>UofStFrancis</td>
      <td></td>
    </tr>
    <tr>
      <th>192323</th>
      <td>Le Moyne College</td>
      <td>lemoyne</td>
      <td>lemoynecollege</td>
    </tr>
    <tr>
      <th>204501</th>
      <td>Oberlin College</td>
      <td>oberlincollege</td>
      <td>ObieAdmissions</td>
    </tr>
    <tr>
      <th>201548</th>
      <td>Capital University</td>
      <td>Capital_U</td>
      <td>capadmission</td>
    </tr>
    <tr>
      <th>213011</th>
      <td>Immaculata University</td>
      <td>immaculatau</td>
      <td></td>
    </tr>
    <tr>
      <th>220613</th>
      <td>Lee University</td>
      <td>leeu</td>
      <td></td>
    </tr>
    <tr>
      <th>224004</th>
      <td>Concordia University-Texas</td>
      <td>concordiatx</td>
      <td>ctxadmissions</td>
    </tr>
    <tr>
      <th>155089</th>
      <td>Friends University</td>
      <td>friendsu</td>
      <td></td>
    </tr>
    <tr>
      <th>236230</th>
      <td>Pacific Lutheran University</td>
      <td>PLUNEWS</td>
      <td>plu_admission</td>
    </tr>
    <tr>
      <th>170675</th>
      <td>Lawrence Technological University</td>
      <td>LawrenceTechU</td>
      <td>ltuadmissions</td>
    </tr>
    <tr>
      <th>215442</th>
      <td>Point Park University</td>
      <td>pointparku</td>
      <td></td>
    </tr>
    <tr>
      <th>200217</th>
      <td>University of Mary</td>
      <td>umary</td>
      <td></td>
    </tr>
    <tr>
      <th>168227</th>
      <td>Wentworth Institute of Technology</td>
      <td>wentworthinst</td>
      <td>witadmissions</td>
    </tr>
    <tr>
      <th>190044</th>
      <td>Clarkson University</td>
      <td>ClarksonUniv</td>
      <td></td>
    </tr>
    <tr>
      <th>136950</th>
      <td>Rollins College</td>
      <td>rollinscollege</td>
      <td></td>
    </tr>
    <tr>
      <th>179964</th>
      <td>William Woods University</td>
      <td>WilliamWoodsU</td>
      <td></td>
    </tr>
    <tr>
      <th>192925</th>
      <td>Medaille College</td>
      <td>MedailleCollege</td>
      <td>medailleuga</td>
    </tr>
    <tr>
      <th>210401</th>
      <td>Willamette University</td>
      <td>willamette_u</td>
      <td>wuadmit</td>
    </tr>
    <tr>
      <th>195474</th>
      <td>Siena College</td>
      <td>SienaCollege</td>
      <td>sienaadmissions</td>
    </tr>
    <tr>
      <th>200253</th>
      <td>Minot State University</td>
      <td>Minotstateu</td>
      <td></td>
    </tr>
    <tr>
      <th>206914</th>
      <td>Cameron University</td>
      <td>CUAggies</td>
      <td></td>
    </tr>
    <tr>
      <th>147660</th>
      <td>North Central College</td>
      <td>northcentralcol</td>
      <td>NCCardinalAdmit</td>
    </tr>
    <tr>
      <th>121309</th>
      <td>Point Loma Nazarene University</td>
      <td>plnu</td>
      <td>go2plnu</td>
    </tr>
    <tr>
      <th>192703</th>
      <td>Manhattan College</td>
      <td>ManhattanEdu</td>
      <td>Admissions_MC</td>
    </tr>
    <tr>
      <th>178387</th>
      <td>Missouri Western State University</td>
      <td>missouriwestern</td>
      <td></td>
    </tr>
    <tr>
      <th>213826</th>
      <td>Marywood University</td>
      <td>marywooduadm</td>
      <td>marywooduadm</td>
    </tr>
    <tr>
      <th>236328</th>
      <td>University of Puget Sound</td>
      <td>univpugetsound</td>
      <td></td>
    </tr>
    <tr>
      <th>164173</th>
      <td>Stevenson University</td>
      <td>stevensonu</td>
      <td></td>
    </tr>
    <tr>
      <th>143048</th>
      <td>School of the Art Institute of Chicago</td>
      <td>saic_news</td>
      <td></td>
    </tr>
    <tr>
      <th>187648</th>
      <td>Eastern New Mexico University-Main Campus</td>
      <td>enmu</td>
      <td></td>
    </tr>
    <tr>
      <th>237367</th>
      <td>Fairmont State University</td>
      <td>fairmontstate</td>
      <td>fsuadmit</td>
    </tr>
    <tr>
      <th>212984</th>
      <td>Holy Family University</td>
      <td>holyfamilyu</td>
      <td></td>
    </tr>
    <tr>
      <th>221892</th>
      <td>Trevecca Nazarene University</td>
      <td>Trevecca</td>
      <td></td>
    </tr>
    <tr>
      <th>218238</th>
      <td>Limestone College</td>
      <td>at_LimestoneCo</td>
      <td></td>
    </tr>
    <tr>
      <th>166124</th>
      <td>College of the Holy Cross</td>
      <td>holy_cross</td>
      <td>hcadmission</td>
    </tr>
    <tr>
      <th>220516</th>
      <td>King University</td>
      <td>KingUnivBristol</td>
      <td></td>
    </tr>
    <tr>
      <th>190099</th>
      <td>Colgate University</td>
      <td>colgateuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>229267</th>
      <td>Trinity University</td>
      <td>Trinity_U</td>
      <td>trincolladmiss</td>
    </tr>
    <tr>
      <th>100830</th>
      <td>Auburn University at Montgomery</td>
      <td>aumontgomery</td>
      <td></td>
    </tr>
    <tr>
      <th>100724</th>
      <td>Alabama State University</td>
      <td>ASUHornetNation</td>
      <td></td>
    </tr>
    <tr>
      <th>176035</th>
      <td>Mississippi University for Women</td>
      <td>muwedu</td>
      <td>muwadmissions</td>
    </tr>
    <tr>
      <th>210739</th>
      <td>DeSales University</td>
      <td>desales</td>
      <td></td>
    </tr>
    <tr>
      <th>215743</th>
      <td>Saint Francis University</td>
      <td>SaintFrancisPA</td>
      <td></td>
    </tr>
    <tr>
      <th>179326</th>
      <td>Southwest Baptist University</td>
      <td>SBUniv</td>
      <td></td>
    </tr>
    <tr>
      <th>212832</th>
      <td>Gwynedd Mercy University</td>
      <td>GMercyU</td>
      <td></td>
    </tr>
    <tr>
      <th>119173</th>
      <td>Mount St Mary's College</td>
      <td>msmu_la</td>
      <td>mountadmissions</td>
    </tr>
    <tr>
      <th>175616</th>
      <td>Delta State University</td>
      <td>deltastate</td>
      <td></td>
    </tr>
    <tr>
      <th>211583</th>
      <td>Chestnut Hill College</td>
      <td>chestnuthill</td>
      <td></td>
    </tr>
    <tr>
      <th>179043</th>
      <td>Rockhurst University</td>
      <td>RockhurstU</td>
      <td></td>
    </tr>
    <tr>
      <th>107071</th>
      <td>Henderson State University</td>
      <td>hendersonstateu</td>
      <td></td>
    </tr>
    <tr>
      <th>163578</th>
      <td>Notre Dame of Maryland University</td>
      <td>NotreDameofMD</td>
      <td></td>
    </tr>
    <tr>
      <th>168740</th>
      <td>Andrews University</td>
      <td>andrewsuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>168342</th>
      <td>Williams College</td>
      <td>williamscollege</td>
      <td></td>
    </tr>
    <tr>
      <th>170301</th>
      <td>Hope College</td>
      <td>HopeCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>199111</th>
      <td>University of North Carolina at Asheville</td>
      <td>UncAvl</td>
      <td></td>
    </tr>
    <tr>
      <th>101189</th>
      <td>Faulkner University</td>
      <td>FaulknerEdu</td>
      <td></td>
    </tr>
    <tr>
      <th>190716</th>
      <td>D'Youville College</td>
      <td>DYouville</td>
      <td></td>
    </tr>
    <tr>
      <th>237792</th>
      <td>Shepherd University</td>
      <td>ShepherdU</td>
      <td></td>
    </tr>
    <tr>
      <th>167996</th>
      <td>Stonehill College</td>
      <td>stonehill_info</td>
      <td></td>
    </tr>
    <tr>
      <th>218070</th>
      <td>Furman University</td>
      <td>FurmanU</td>
      <td></td>
    </tr>
    <tr>
      <th>206862</th>
      <td>Southern Nazarene University</td>
      <td>FollowSNU</td>
      <td></td>
    </tr>
    <tr>
      <th>168218</th>
      <td>Wellesley College</td>
      <td>wellesley</td>
      <td></td>
    </tr>
    <tr>
      <th>175421</th>
      <td>Belhaven University</td>
      <td>belhavenu</td>
      <td></td>
    </tr>
    <tr>
      <th>186432</th>
      <td>Saint Peter's University</td>
      <td>saintpetersuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>238980</th>
      <td>Lakeland College</td>
      <td>LakelandWI</td>
      <td></td>
    </tr>
    <tr>
      <th>240374</th>
      <td>University of Wisconsin-Parkside</td>
      <td>uwparkside</td>
      <td>uwpadmissions</td>
    </tr>
    <tr>
      <th>183974</th>
      <td>Centenary College</td>
      <td>Centenary_NJ</td>
      <td></td>
    </tr>
    <tr>
      <th>214175</th>
      <td>Muhlenberg College</td>
      <td>muhlenberg</td>
      <td></td>
    </tr>
    <tr>
      <th>126669</th>
      <td>Colorado Christian University</td>
      <td>my_ccu</td>
      <td></td>
    </tr>
    <tr>
      <th>166939</th>
      <td>Mount Holyoke College</td>
      <td>mtholyoke</td>
      <td></td>
    </tr>
    <tr>
      <th>212674</th>
      <td>Gettysburg College</td>
      <td>gettysburg</td>
      <td></td>
    </tr>
    <tr>
      <th>237066</th>
      <td>Whitworth University</td>
      <td>whitworth</td>
      <td></td>
    </tr>
    <tr>
      <th>139199</th>
      <td>Brenau University</td>
      <td>BrenauU</td>
      <td></td>
    </tr>
    <tr>
      <th>217536</th>
      <td>Salve Regina University</td>
      <td>SalveRegina</td>
      <td></td>
    </tr>
    <tr>
      <th>147679</th>
      <td>North Park University</td>
      <td>npu</td>
      <td></td>
    </tr>
    <tr>
      <th>238458</th>
      <td>Carroll University</td>
      <td>carrollu</td>
      <td></td>
    </tr>
    <tr>
      <th>150534</th>
      <td>University of Evansville</td>
      <td>UEvansville</td>
      <td></td>
    </tr>
    <tr>
      <th>203775</th>
      <td>Malone University</td>
      <td>MaloneU</td>
      <td></td>
    </tr>
    <tr>
      <th>173300</th>
      <td>Concordia College at Moorhead</td>
      <td>concordia_mn</td>
      <td></td>
    </tr>
    <tr>
      <th>195526</th>
      <td>Skidmore College</td>
      <td>SkidmoreCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>204194</th>
      <td>Mount Vernon Nazarene University</td>
      <td>MVNUNews</td>
      <td></td>
    </tr>
    <tr>
      <th>226231</th>
      <td>LeTourneau University</td>
      <td>letourneauuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>150400</th>
      <td>DePauw University</td>
      <td>DePauwU</td>
      <td></td>
    </tr>
    <tr>
      <th>214272</th>
      <td>Neumann University</td>
      <td>NeumannUniv</td>
      <td></td>
    </tr>
    <tr>
      <th>218733</th>
      <td>South Carolina State University</td>
      <td>SCSTATE1896</td>
      <td></td>
    </tr>
    <tr>
      <th>238476</th>
      <td>Carthage College</td>
      <td>carthagecollege</td>
      <td></td>
    </tr>
    <tr>
      <th>164562</th>
      <td>Assumption College</td>
      <td>AssumptionNews</td>
      <td>achoundbound</td>
    </tr>
    <tr>
      <th>213385</th>
      <td>Lafayette College</td>
      <td>LafCol</td>
      <td></td>
    </tr>
    <tr>
      <th>194161</th>
      <td>Nyack College</td>
      <td>NyackCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>138947</th>
      <td>Clark Atlanta University</td>
      <td>cau</td>
      <td></td>
    </tr>
    <tr>
      <th>207582</th>
      <td>Oral Roberts University</td>
      <td>OralRobertsU</td>
      <td></td>
    </tr>
    <tr>
      <th>213987</th>
      <td>Mercyhurst University</td>
      <td>mercyhurstu</td>
      <td></td>
    </tr>
    <tr>
      <th>175078</th>
      <td>Southwest Minnesota State University</td>
      <td>smsutoday</td>
      <td></td>
    </tr>
    <tr>
      <th>150066</th>
      <td>Anderson University</td>
      <td>AndersonU</td>
      <td></td>
    </tr>
    <tr>
      <th>166850</th>
      <td>Merrimack College</td>
      <td>merrimack</td>
      <td></td>
    </tr>
    <tr>
      <th>174491</th>
      <td>University of Northwestern-St Paul</td>
      <td>NorthwesternMN</td>
      <td></td>
    </tr>
    <tr>
      <th>213996</th>
      <td>Messiah College</td>
      <td>messiahcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>227331</th>
      <td>Our Lady of the Lake University</td>
      <td>ollunivsatx</td>
      <td></td>
    </tr>
    <tr>
      <th>218061</th>
      <td>Francis Marion University</td>
      <td>FrancisMarionU</td>
      <td></td>
    </tr>
    <tr>
      <th>143084</th>
      <td>Augustana College</td>
      <td>Augustana_IL</td>
      <td></td>
    </tr>
    <tr>
      <th>161217</th>
      <td>University of Maine at Augusta</td>
      <td>umaugusta</td>
      <td></td>
    </tr>
    <tr>
      <th>173902</th>
      <td>Macalester College</td>
      <td>macalester</td>
      <td>MacAdmission</td>
    </tr>
    <tr>
      <th>193353</th>
      <td>Mount Saint Mary College</td>
      <td>msmu</td>
      <td></td>
    </tr>
    <tr>
      <th>195216</th>
      <td>St Lawrence University</td>
      <td>stlawrenceu</td>
      <td></td>
    </tr>
    <tr>
      <th>202523</th>
      <td>Denison University</td>
      <td>denisonu</td>
      <td></td>
    </tr>
    <tr>
      <th>209506</th>
      <td>Oregon Institute of Technology</td>
      <td>OregonTech</td>
      <td></td>
    </tr>
    <tr>
      <th>184694</th>
      <td>Fairleigh Dickinson University-College at Florham</td>
      <td>FDUWhatsNew</td>
      <td></td>
    </tr>
    <tr>
      <th>155520</th>
      <td>MidAmerica Nazarene University</td>
      <td>followMNU</td>
      <td></td>
    </tr>
    <tr>
      <th>184773</th>
      <td>Georgian Court University</td>
      <td>Georgiancourt</td>
      <td></td>
    </tr>
    <tr>
      <th>212577</th>
      <td>Franklin and Marshall College</td>
      <td>FandMCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>217493</th>
      <td>Rhode Island School of Design</td>
      <td>risd</td>
      <td></td>
    </tr>
    <tr>
      <th>130934</th>
      <td>Delaware State University</td>
      <td>DelStateUniv</td>
      <td></td>
    </tr>
    <tr>
      <th>189097</th>
      <td>Barnard College</td>
      <td>BarnardCollege</td>
      <td>bcadmit</td>
    </tr>
    <tr>
      <th>204936</th>
      <td>Otterbein University</td>
      <td>otterbein</td>
      <td>OtterbeinUAdmit</td>
    </tr>
    <tr>
      <th>210775</th>
      <td>Alvernia University</td>
      <td>AlverniaUniv</td>
      <td></td>
    </tr>
    <tr>
      <th>239080</th>
      <td>Marian University</td>
      <td>marian_wi</td>
      <td></td>
    </tr>
    <tr>
      <th>127185</th>
      <td>Fort Lewis College</td>
      <td>FLCDurango</td>
      <td></td>
    </tr>
    <tr>
      <th>204617</th>
      <td>Ohio Dominican University</td>
      <td>OhioDominican</td>
      <td></td>
    </tr>
    <tr>
      <th>198613</th>
      <td>Guilford College</td>
      <td>GuilfordCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>161086</th>
      <td>Colby College</td>
      <td>colbycollege</td>
      <td></td>
    </tr>
    <tr>
      <th>201654</th>
      <td>Cedarville University</td>
      <td>cedarville</td>
      <td></td>
    </tr>
    <tr>
      <th>151786</th>
      <td>Marian University</td>
      <td>marianuniv</td>
      <td>marianadmission</td>
    </tr>
    <tr>
      <th>130590</th>
      <td>Trinity College</td>
      <td>trinitycollege</td>
      <td></td>
    </tr>
    <tr>
      <th>232609</th>
      <td>Lynchburg College</td>
      <td>lynchburg</td>
      <td>go2lynchburg</td>
    </tr>
    <tr>
      <th>177418</th>
      <td>Fontbonne University</td>
      <td>FontbonneU</td>
      <td></td>
    </tr>
    <tr>
      <th>192192</th>
      <td>Keuka College</td>
      <td>keukacollege</td>
      <td></td>
    </tr>
    <tr>
      <th>226471</th>
      <td>University of Mary Hardin-Baylor</td>
      <td>umhb</td>
      <td></td>
    </tr>
    <tr>
      <th>211431</th>
      <td>Carlow University</td>
      <td>CarlowU</td>
      <td></td>
    </tr>
    <tr>
      <th>196112</th>
      <td>SUNY Institute of Technology at Utica-Rome</td>
      <td>SUNYPolyInst</td>
      <td></td>
    </tr>
    <tr>
      <th>198950</th>
      <td>Meredith College</td>
      <td>meredithcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>212009</th>
      <td>Dickinson College</td>
      <td>DickinsonCol</td>
      <td></td>
    </tr>
    <tr>
      <th>173647</th>
      <td>Gustavus Adolphus College</td>
      <td>gustavus</td>
      <td></td>
    </tr>
    <tr>
      <th>170639</th>
      <td>Lake Superior State University</td>
      <td>lifeatlssu</td>
      <td></td>
    </tr>
    <tr>
      <th>194958</th>
      <td>Roberts Wesleyan College</td>
      <td>RobertsWesleyan</td>
      <td></td>
    </tr>
    <tr>
      <th>217688</th>
      <td>Charleston Southern University</td>
      <td>csuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>107141</th>
      <td>John Brown University</td>
      <td>johnbrownuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>150145</th>
      <td>Bethel College-Indiana</td>
      <td>BethelCollegeIN</td>
      <td></td>
    </tr>
    <tr>
      <th>120254</th>
      <td>Occidental College</td>
      <td>occidental</td>
      <td></td>
    </tr>
    <tr>
      <th>197197</th>
      <td>Wagner College</td>
      <td>wagnercollege</td>
      <td></td>
    </tr>
    <tr>
      <th>182795</th>
      <td>Franklin Pierce University</td>
      <td>FPUniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>191630</th>
      <td>Hobart William Smith Colleges</td>
      <td>hwscolleges</td>
      <td></td>
    </tr>
    <tr>
      <th>204200</th>
      <td>College of Mount St Joseph</td>
      <td>MountStJosephU</td>
      <td>MSJ_Admissions</td>
    </tr>
    <tr>
      <th>154350</th>
      <td>Simpson College</td>
      <td>simpsoncollege</td>
      <td></td>
    </tr>
    <tr>
      <th>163295</th>
      <td>Maryland Institute College of Art</td>
      <td>mica</td>
      <td></td>
    </tr>
    <tr>
      <th>133492</th>
      <td>Eckerd College</td>
      <td>eckerdcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>239318</th>
      <td>Milwaukee School of Engineering</td>
      <td>MSOE</td>
      <td></td>
    </tr>
    <tr>
      <th>110404</th>
      <td>California Institute of Technology</td>
      <td>caltech</td>
      <td></td>
    </tr>
    <tr>
      <th>134079</th>
      <td>Florida Southern College</td>
      <td>flsouthern</td>
      <td></td>
    </tr>
    <tr>
      <th>128498</th>
      <td>Albertus Magnus College</td>
      <td>AlbertusSocial</td>
      <td></td>
    </tr>
    <tr>
      <th>221953</th>
      <td>Tusculum College</td>
      <td>tusculum_univ</td>
      <td></td>
    </tr>
    <tr>
      <th>126678</th>
      <td>Colorado College</td>
      <td>coloradocollege</td>
      <td></td>
    </tr>
    <tr>
      <th>191515</th>
      <td>Hamilton College</td>
      <td>hamiltoncollege</td>
      <td>HamiltonAdmssn</td>
    </tr>
    <tr>
      <th>137564</th>
      <td>Southeastern University</td>
      <td>seuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>212656</th>
      <td>Geneva College</td>
      <td>GenevaCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>177339</th>
      <td>Evangel University</td>
      <td>evangeluniv</td>
      <td></td>
    </tr>
    <tr>
      <th>225399</th>
      <td>Houston Baptist University</td>
      <td>HoustonBaptistU</td>
      <td></td>
    </tr>
    <tr>
      <th>215105</th>
      <td>The University of the Arts</td>
      <td>uarts</td>
      <td></td>
    </tr>
    <tr>
      <th>190761</th>
      <td>Dominican College of Blauvelt</td>
      <td>DominicanOburg</td>
      <td></td>
    </tr>
    <tr>
      <th>154013</th>
      <td>Mount Mercy University</td>
      <td>MountMercy</td>
      <td></td>
    </tr>
    <tr>
      <th>207324</th>
      <td>Oklahoma Christian University</td>
      <td>okchristian</td>
      <td></td>
    </tr>
    <tr>
      <th>216278</th>
      <td>Susquehanna University</td>
      <td>susquehannau</td>
      <td></td>
    </tr>
    <tr>
      <th>153375</th>
      <td>Grand View University</td>
      <td>grandviewuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>181446</th>
      <td>Nebraska Wesleyan University</td>
      <td>NEWesleyan</td>
      <td></td>
    </tr>
    <tr>
      <th>238193</th>
      <td>Alverno College</td>
      <td>alvernocollege</td>
      <td></td>
    </tr>
    <tr>
      <th>184348</th>
      <td>Drew University</td>
      <td>drewuniversity</td>
      <td></td>
    </tr>
    <tr>
      <th>145646</th>
      <td>Illinois Wesleyan University</td>
      <td>IL_Wesleyan</td>
      <td></td>
    </tr>
    <tr>
      <th>128902</th>
      <td>Connecticut College</td>
      <td>ConnCollege</td>
      <td>CC_Admission</td>
    </tr>
    <tr>
      <th>173258</th>
      <td>Carleton College</td>
      <td>CarletonCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>101709</th>
      <td>University of Montevallo</td>
      <td>Montevallo</td>
      <td>Admit_UM</td>
    </tr>
    <tr>
      <th>147244</th>
      <td>Millikin University</td>
      <td>MillikinU</td>
      <td></td>
    </tr>
    <tr>
      <th>204185</th>
      <td>University of Mount Union</td>
      <td>mountunion</td>
      <td></td>
    </tr>
    <tr>
      <th>210571</th>
      <td>Albright College</td>
      <td>albrightcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>139144</th>
      <td>Berry College</td>
      <td>berrycollege</td>
      <td>berryadmissions</td>
    </tr>
    <tr>
      <th>210669</th>
      <td>Allegheny College</td>
      <td>alleghenycol</td>
      <td>gotoallegheny</td>
    </tr>
    <tr>
      <th>152530</th>
      <td>Taylor University</td>
      <td>tayloru</td>
      <td></td>
    </tr>
    <tr>
      <th>219000</th>
      <td>Augustana College</td>
      <td>augustanasd</td>
      <td></td>
    </tr>
    <tr>
      <th>152318</th>
      <td>Rose-Hulman Institute of Technology</td>
      <td>rosehulman</td>
      <td></td>
    </tr>
    <tr>
      <th>237932</th>
      <td>West Liberty University</td>
      <td>westlibertyu</td>
      <td></td>
    </tr>
    <tr>
      <th>204909</th>
      <td>Ohio Wesleyan University</td>
      <td>OhioWesleyan</td>
      <td>OWUAdmission</td>
    </tr>
    <tr>
      <th>225247</th>
      <td>Hardin-Simmons University</td>
      <td>HSUTX</td>
      <td></td>
    </tr>
    <tr>
      <th>140553</th>
      <td>Morehouse College</td>
      <td>Morehouse</td>
      <td>mhouseadmission</td>
    </tr>
    <tr>
      <th>211468</th>
      <td>Cedar Crest College</td>
      <td>cedarcrestcolle</td>
      <td>AdmissionsCCC</td>
    </tr>
    <tr>
      <th>110370</th>
      <td>California College of the Arts</td>
      <td>CACollegeofArts</td>
      <td></td>
    </tr>
    <tr>
      <th>212197</th>
      <td>Elizabethtown College</td>
      <td>etowncollege</td>
      <td></td>
    </tr>
    <tr>
      <th>216524</th>
      <td>Ursinus College</td>
      <td>UrsinusCollege</td>
      <td>UrsinusAdmit</td>
    </tr>
    <tr>
      <th>198385</th>
      <td>Davidson College</td>
      <td>DavidsonCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>218229</th>
      <td>Lander University</td>
      <td>follow_lander</td>
      <td></td>
    </tr>
    <tr>
      <th>214157</th>
      <td>Moravian College</td>
      <td>MoravianCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>162283</th>
      <td>Coppin State University</td>
      <td>coppinstateuniv</td>
      <td></td>
    </tr>
    <tr>
      <th>215284</th>
      <td>University of Pittsburgh-Johnstown</td>
      <td>PittJohnstown</td>
      <td></td>
    </tr>
    <tr>
      <th>237330</th>
      <td>Concord University</td>
      <td>CampusBeautiful</td>
      <td></td>
    </tr>
    <tr>
      <th>211981</th>
      <td>Delaware Valley College</td>
      <td>DelVal</td>
      <td></td>
    </tr>
    <tr>
      <th>203535</th>
      <td>Kenyon College</td>
      <td>KenyonCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>140960</th>
      <td>Savannah State University</td>
      <td>savannahstate</td>
      <td></td>
    </tr>
    <tr>
      <th>174792</th>
      <td>Saint Johns University</td>
      <td>CSBSJU</td>
      <td></td>
    </tr>
    <tr>
      <th>206589</th>
      <td>The College of Wooster</td>
      <td>woosteredu</td>
      <td></td>
    </tr>
    <tr>
      <th>218973</th>
      <td>Wofford College</td>
      <td>woffordcollege</td>
      <td>whywofford</td>
    </tr>
    <tr>
      <th>154527</th>
      <td>Wartburg College</td>
      <td>WartburgCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>166674</th>
      <td>Massachusetts College of Art and Design</td>
      <td>MassArt</td>
      <td></td>
    </tr>
    <tr>
      <th>219806</th>
      <td>Carson-Newman University</td>
      <td>cneagles</td>
      <td></td>
    </tr>
    <tr>
      <th>120184</th>
      <td>Notre Dame de Namur University</td>
      <td>NDNU</td>
      <td></td>
    </tr>
    <tr>
      <th>183239</th>
      <td>Saint Anselm College</td>
      <td>saintanselm</td>
      <td></td>
    </tr>
    <tr>
      <th>167288</th>
      <td>Massachusetts College of Liberal Arts</td>
      <td>MCLA_EDU</td>
      <td></td>
    </tr>
    <tr>
      <th>106412</th>
      <td>University of Arkansas at Pine Bluff</td>
      <td>uapbinfo</td>
      <td></td>
    </tr>
    <tr>
      <th>146481</th>
      <td>Lake Forest College</td>
      <td>lfcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>152390</th>
      <td>Saint Mary's College</td>
      <td>saintmarys</td>
      <td></td>
    </tr>
    <tr>
      <th>196291</th>
      <td>SUNY Maritime College</td>
      <td>MaritimeCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>237899</th>
      <td>West Virginia State University</td>
      <td>WVStateU</td>
      <td></td>
    </tr>
    <tr>
      <th>206525</th>
      <td>Wittenberg University</td>
      <td>wittenberg</td>
      <td></td>
    </tr>
    <tr>
      <th>192864</th>
      <td>Marymount Manhattan College</td>
      <td>NYCMarymount</td>
      <td></td>
    </tr>
    <tr>
      <th>203845</th>
      <td>Marietta College</td>
      <td>MariettaCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>216667</th>
      <td>Washington &amp; Jefferson College</td>
      <td>wjcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>237057</th>
      <td>Whitman College</td>
      <td>whitmancollege</td>
      <td></td>
    </tr>
    <tr>
      <th>221351</th>
      <td>Rhodes College</td>
      <td>RhodesCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>109651</th>
      <td>Art Center College of Design</td>
      <td>artcenteredu</td>
      <td></td>
    </tr>
    <tr>
      <th>199209</th>
      <td>North Carolina Wesleyan College</td>
      <td>ncwesleyan</td>
      <td></td>
    </tr>
    <tr>
      <th>228343</th>
      <td>Southwestern University</td>
      <td>southwesternu</td>
      <td></td>
    </tr>
    <tr>
      <th>125727</th>
      <td>Westmont College</td>
      <td>westmontnews</td>
      <td></td>
    </tr>
    <tr>
      <th>120865</th>
      <td>Pacific Union College</td>
      <td>pucnow</td>
      <td></td>
    </tr>
    <tr>
      <th>209922</th>
      <td>Reed College</td>
      <td>Reed_College_</td>
      <td></td>
    </tr>
    <tr>
      <th>197984</th>
      <td>Belmont Abbey College</td>
      <td>BelmontAbbey</td>
      <td></td>
    </tr>
    <tr>
      <th>100937</th>
      <td>Birmingham Southern College</td>
      <td>FromTheHilltop</td>
      <td></td>
    </tr>
    <tr>
      <th>101435</th>
      <td>Huntingdon College</td>
      <td>HuntingdonColl</td>
      <td></td>
    </tr>
    <tr>
      <th>115409</th>
      <td>Harvey Mudd College</td>
      <td>harveymudd</td>
      <td></td>
    </tr>
    <tr>
      <th>154493</th>
      <td>Upper Iowa University</td>
      <td>upperiowa</td>
      <td></td>
    </tr>
    <tr>
      <th>157386</th>
      <td>Morehead State University</td>
      <td>moreheadstate</td>
      <td></td>
    </tr>
    <tr>
      <th>211644</th>
      <td>Clarion University of Pennsylvania</td>
      <td>clarionu</td>
      <td></td>
    </tr>
    <tr>
      <th>219718</th>
      <td>Bethel University</td>
      <td>BethelU</td>
      <td></td>
    </tr>
    <tr>
      <th>232566</th>
      <td>Longwood University</td>
      <td>longwoodu</td>
      <td>whylongwood</td>
    </tr>
    <tr>
      <th>204635</th>
      <td>Ohio Northern University</td>
      <td>ohionorthern</td>
      <td></td>
    </tr>
    <tr>
      <th>181783</th>
      <td>Wayne State College</td>
      <td>waynestcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>234207</th>
      <td>Washington and Lee University</td>
      <td>wlulex</td>
      <td>wluadmissions</td>
    </tr>
    <tr>
      <th>213783</th>
      <td>Mansfield University of Pennsylvania</td>
      <td>mansfieldU</td>
      <td>mansfieldadmiss</td>
    </tr>
    <tr>
      <th>153834</th>
      <td>Luther College</td>
      <td>luthercollege</td>
      <td>LutherAdmission</td>
    </tr>
    <tr>
      <th>216694</th>
      <td>Waynesburg University</td>
      <td>WaynesburgU</td>
      <td></td>
    </tr>
    <tr>
      <th>188641</th>
      <td>Alfred University</td>
      <td>alfredu</td>
      <td>admissions_au</td>
    </tr>
    <tr>
      <th>128391</th>
      <td>Western State Colorado University</td>
      <td>westerncolou</td>
      <td></td>
    </tr>
    <tr>
      <th>102377</th>
      <td>Tuskegee University</td>
      <td>TuskegeeUniv</td>
      <td>TUAdmissions</td>
    </tr>
    <tr>
      <th>161226</th>
      <td>University of Maine at Farmington</td>
      <td>UMFinthenews</td>
      <td></td>
    </tr>
    <tr>
      <th>191676</th>
      <td>Houghton College</td>
      <td>HoughtonCollege</td>
      <td></td>
    </tr>
    <tr>
      <th>153384</th>
      <td>Grinnell College</td>
      <td>grinnellcollege</td>
      <td>gcadmission</td>
    </tr>
    <tr>
      <th>153108</th>
      <td>Central College</td>
      <td>centralcollege</td>
      <td></td>
    </tr>
    <tr>
      <th>110565</th>
      <td>California State University-Fullerton</td>
      <td>csuf</td>
      <td></td>
    </tr>
    <tr>
      <th>218663</th>
      <td>University of South Carolina-Columbia</td>
      <td>uofsc</td>
      <td>uofscadmissions</td>
    </tr>
    <tr>
      <th>221759</th>
      <td>The University of Tennessee-Knoxville</td>
      <td>utknoxville</td>
      <td>ut_admissions</td>
    </tr>
    <tr>
      <th>450933</th>
      <td>Columbia Southern University</td>
      <td>csuedu</td>
      <td>admissionscsu</td>
    </tr>
    <tr>
      <th>191241</th>
      <td>Fordham University</td>
      <td>fordhamnyc</td>
      <td>fordham_admis</td>
    </tr>
    <tr>
      <th>206084</th>
      <td>University of Toledo</td>
      <td>utoledo</td>
      <td>utadmission</td>
    </tr>
    <tr>
      <th>223232</th>
      <td>Baylor University</td>
      <td>baylor</td>
      <td>beabaylorbear</td>
    </tr>
    <tr>
      <th>239105</th>
      <td>Marquette University</td>
      <td>marquetteu</td>
      <td>muadmissions</td>
    </tr>
    <tr>
      <th>183026</th>
      <td>Southern New Hampshire University</td>
      <td>snhu</td>
      <td></td>
    </tr>
    <tr>
      <th>228529</th>
      <td>Tarleton State University</td>
      <td>tarletonstate</td>
      <td></td>
    </tr>
    <tr>
      <th>133650</th>
      <td>Florida Agricultural and Mechanical University</td>
      <td>famu_1887</td>
      <td></td>
    </tr>
    <tr>
      <th>233277</th>
      <td>Radford University</td>
      <td>radfordu</td>
      <td>readyforradford</td>
    </tr>
    <tr>
      <th>211158</th>
      <td>Bloomsburg University of Pennsylvania</td>
      <td>bloomsburgu</td>
      <td></td>
    </tr>
    <tr>
      <th>145619</th>
      <td>Benedictine University</td>
      <td>benu1887</td>
      <td></td>
    </tr>
    <tr>
      <th>192439</th>
      <td>LIU Brooklyn</td>
      <td>liubrooklyn</td>
      <td></td>
    </tr>
    <tr>
      <th>194824</th>
      <td>Rensselaer Polytechnic Institute</td>
      <td>rpi</td>
      <td>rpiadmissions</td>
    </tr>
    <tr>
      <th>215929</th>
      <td>University of Scranton</td>
      <td>univofscranton</td>
      <td>uofsadmissions</td>
    </tr>
    <tr>
      <th>130253</th>
      <td>Sacred Heart University</td>
      <td>sacredheartuniv</td>
      <td>shuadmissions</td>
    </tr>
    <tr>
      <th>170806</th>
      <td>Madonna University</td>
      <td>madonnau</td>
      <td></td>
    </tr>
    <tr>
      <th>159993</th>
      <td>University of Louisiana at Monroe</td>
      <td>ulm_official</td>
      <td></td>
    </tr>
    <tr>
      <th>238430</th>
      <td>Cardinal Stritch University</td>
      <td>stritchu</td>
      <td>stritchadmitu</td>
    </tr>
    <tr>
      <th>215655</th>
      <td>Robert Morris University</td>
      <td>rmu</td>
      <td>rmuadmissions</td>
    </tr>
    <tr>
      <th>173124</th>
      <td>Bemidji State University</td>
      <td>bemidjistate</td>
      <td></td>
    </tr>
    <tr>
      <th>207971</th>
      <td>University of Tulsa</td>
      <td>utulsa</td>
      <td></td>
    </tr>
    <tr>
      <th>196200</th>
      <td>SUNY College at Potsdam</td>
      <td>sunypotsdam1816</td>
      <td></td>
    </tr>
    <tr>
      <th>144005</th>
      <td>Chicago State University</td>
      <td>chicagostate</td>
      <td></td>
    </tr>
    <tr>
      <th>159009</th>
      <td>Grambling State University</td>
      <td>grambling1901</td>
      <td>gramfamrecruit</td>
    </tr>
    <tr>
      <th>193645</th>
      <td>The College of New Rochelle</td>
      <td>cnr1904</td>
      <td></td>
    </tr>
    <tr>
      <th>203368</th>
      <td>John Carroll University</td>
      <td>johncarrollu</td>
      <td>jcuadmission</td>
    </tr>
    <tr>
      <th>147013</th>
      <td>McKendree University</td>
      <td>mckendreeu</td>
      <td></td>
    </tr>
    <tr>
      <th>217749</th>
      <td>Bob Jones University</td>
      <td>bjuedu</td>
      <td></td>
    </tr>
    <tr>
      <th>217864</th>
      <td>Citadel Military College of South Carolina</td>
      <td>citadel1842</td>
      <td></td>
    </tr>
    <tr>
      <th>207847</th>
      <td>Southeastern Oklahoma State University</td>
      <td>se1909</td>
      <td></td>
    </tr>
    <tr>
      <th>214069</th>
      <td>Misericordia University</td>
      <td>misericordiau</td>
      <td>misericordiauad</td>
    </tr>
    <tr>
      <th>160904</th>
      <td>Xavier University of Louisiana</td>
      <td>xula1925</td>
      <td>xulaadmissions</td>
    </tr>
    <tr>
      <th>121345</th>
      <td>Pomona College</td>
      <td>pomonacollege</td>
      <td>pomonaadmit</td>
    </tr>
    <tr>
      <th>146427</th>
      <td>Knox College</td>
      <td>knoxcollege1837</td>
      <td></td>
    </tr>
  </tbody>
</table>
</div>




```python
tw_df[['instnm','main_handle','adm_handle']].to_pickle(os.path.join(tw_path, "tw_df_final"))
```
