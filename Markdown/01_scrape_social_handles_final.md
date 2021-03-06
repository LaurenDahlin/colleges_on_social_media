
# How to Search for Webpages on Google and Scrape Social Media URLS

### Objective: Use Python to search for an entity (e.g. a school or company) on Google. Navigate to the first Google search result and scrape any Twitter, Facebook, and Instagram pages.

The sample of schools I use for the college social media dataset comes from Brookings, a thinktank with extensive research about education and other public policy topics. In the data, the authors calculate "value added" for all major universities in the United States. Their value added measures represent how effective colleges are at increasing student earnings. (Learn more [here](https://www.brookings.edu/research/using-earnings-data-to-rank-colleges-a-value-added-approach-updated-with-college-scorecard-data/)) The data underlying the Brookings value added measures comes from the Integrated Postsecondary Education Data System ([IPEDS](https://nces.ed.gov/ipeds/)), which is maintained by the U.S. Department of education. 

## Set Up

#### Import Packages
The main packages we are using are: pandas, BeautifulSoup, and selenium. Pandas is used for data management. Selenium is used to implement the Google searches and get around Google's bot detection. BeautifulSoup and requests fetch data from the webpages. 


```python
import pandas as pd
from bs4 import BeautifulSoup 
import os, requests, re, time, sys, inspect
```


```python
from selenium import webdriver
from selenium.webdriver.firefox.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.common.keys import Keys
```

#### File Locations


```python
# Update with your own file structure
base_path = r"C:\Users\laure\Dropbox\!research\20181026_ihe_diversity"
# Path to the Brooking's data with the names of the universities
brookings_path = os.path.join(base_path,'data','brookings_value_added')
sm_path = os.path.join(base_path,'data','social_media_links')
sm_py_path = os.path.join(base_path,'python_scraping','scrape_handles')
```

#### Import functions from .py
The functions in this file are used in the next step of this project as well. Because of this, I added the functions to ihe_scrape.py file, which I import below. To make it easier to see how these functions work, I use the inspect package to show the code for the functions in this notebook. 


```python
social_media_py = str(os.path.join(sm_py_path, 'social_media_urls.py'))
%run $social_media_py
```


```python
def print_func(func):
    lines = inspect.getsource(func)
    print(lines)
```

## Base Data on Colleges to Search

The base IHE data comes from the Brookings college value-added rankings.


```python
b_df = pd.read_csv(os.path.join(brookings_path,'brookings_4yr_value_added.csv'))
b_df.head(2)
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
      <th>ccbasic</th>
      <th>pct_baplus_awards</th>
      <th>primarily_four_year</th>
      <th>city</th>
      <th>omb_cbsa</th>
      <th>omb_title</th>
      <th>stabbr</th>
      <th>omb_state_name</th>
      <th>...</th>
      <th>BSALg_z_pct_STEM_award_HIGH_ANY</th>
      <th>BSALg_z_lnMed_inc_field</th>
      <th>BSALg_z_grad_rate200</th>
      <th>BOCC_z_Retention_Rate2009</th>
      <th>BOCC_z_avg_inst_aid</th>
      <th>BOCC_z_lnavg_inst_sal</th>
      <th>BOCC_z_lnsch_skill_val</th>
      <th>BOCC_z_pct_STEM_award_HIGH_ANY</th>
      <th>BOCC_z_lnMed_inc_field</th>
      <th>BOCC_z_grad_rate200</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>372213</td>
      <td>University of Phoenix-Online Campus</td>
      <td>Doctoral/Research Universities</td>
      <td>64%</td>
      <td>1</td>
      <td>Phoenix</td>
      <td>38060.0</td>
      <td>Phoenix-Mesa-Scottsdale, AZ</td>
      <td>AZ</td>
      <td>Arizona</td>
      <td>...</td>
      <td>0.3%</td>
      <td>4.8%</td>
      <td>-2.6%</td>
      <td>-0.6%</td>
      <td>0.1%</td>
      <td>-1.1%</td>
      <td>2.9%</td>
      <td>0.2%</td>
      <td>0.3%</td>
      <td>-0.1%</td>
    </tr>
    <tr>
      <th>1</th>
      <td>154022</td>
      <td>Ashford University</td>
      <td>Master's Colleges and Universities (larger pro...</td>
      <td>90%</td>
      <td>1</td>
      <td>Clinton</td>
      <td>17540.0</td>
      <td>Clinton, IA</td>
      <td>IA</td>
      <td>Iowa</td>
      <td>...</td>
      <td>-1.6%</td>
      <td>1.6%</td>
      <td>-1.8%</td>
      <td>-0.4%</td>
      <td>-0.2%</td>
      <td>0.2%</td>
      <td>-0.7%</td>
      <td>-0.9%</td>
      <td>0.1%</td>
      <td>-0.1%</td>
    </tr>
  </tbody>
</table>
<p>2 rows × 122 columns</p>
</div>



Keep only institutions with non-missing Value-Added data


```python
b_df = b_df.dropna(subset=['VA_sal', 'VA_repay', 'VA_occ', 'VA_repay_broad', 'VA_sal_GRAD'])
```

Create list of schools concatenated with state. These will be implemented in Google search.


```python
ihe_search = b_df.instnm + " " + b_df.omb_state_name
b_df = b_df.assign(ihe_search=ihe_search)
```

## Scrape Twitter URLS from Webpages


```python
print_func(get_handles)
```

    def get_handles(url, driver):
        
        # Get all text on page using Selenium
        driver.get(url)
        # Wait for page load
        for i in range(120):
            page_state = driver.execute_script('return document.readyState;')
            try:
                if page_state=='complete': break
            except: pass
            time.sleep(0.5)
        else: self.fail("time out")    
        text = driver.page_source
        
        # Get all links on page that contain Twitter or Facebook using BeautifulSoup
        root = BeautifulSoup(text, "lxml")
        
        # Twitter
        twitter_links = []
        for link in root.find_all("a", href=re.compile("twitter")):
            twitter_links.append(link.get('href'))
        
        # Remove individual twitter post links by removing links from list that have two consecutive numbers
        twitter_links = [link for link in twitter_links if bool(re.search(r'\d\d', link)) == False]
        
        # Facebook
        facebook_links = []
        for link in root.find_all("a", href=re.compile("facebook")):
            facebook_links.append(link.get('href'))
            
        # Instagram
        instagram_links = []
        for link in root.find_all("a", href=re.compile("instagram")):
            instagram_links.append(link.get('href'))
        
        return twitter_links, facebook_links, instagram_links
    
    


```python
# Test
# Switch to 1==1 to turn on
if 1==2:
    options = Options()
    options.headless = True
    driver = webdriver.Firefox(options=options)
    url = r"https://www.ucf.edu/"
    twitter_links, facebook_links, instagram_links = get_handles(url=url,driver=driver)
    print(str(twitter_links) + str(facebook_links) + str(instagram_links))
    driver.quit()
```

    ['https://www.twitter.com/ucf', 'https://twitter.com/UCF']['https://www.facebook.com/ucf', 'https://www.facebook.com/UCF']['https://www.instagram.com/ucf.edu/', 'https://www.instagram.com/ucf.edu/']
    

## Use "I'm Feeling Lucky" to Search for Pages and Get URLS


```python
print_func(feeling_lucky)
```

    def feeling_lucky(search_target, driver):
        
        
        driver.get("http://www.google.com/")
        time.sleep(1)
        
        # Input university to search for
        driver.find_element_by_name("q").send_keys(search_target)
        time.sleep(1)
        driver.find_element_by_name("q").send_keys(Keys.TAB)
        time.sleep(1)
        
        # Click on get lucky button 
        driver.find_element_by_id("gbqfbb").click()
        wait = WebDriverWait(driver, 20)
        wait.until(lambda driver: 'google' not in driver.current_url)
        
        page_url = driver.current_url
        
        return page_url 
    
    


```python
# Test
if 1==1:
    options = Options()
    options.headless = True
    driver = webdriver.Firefox(options=options)
    print(feeling_lucky("University of Washington", driver=driver))
    driver.quit()
```

    https://www.washington.edu/
    

## Remove Duplicate URLs


```python
print_func(rm_dup)
```

    def rm_dup(a_list): 
        final_list = [] 
        for item in a_list: 
            if item not in final_list: 
                final_list.append(item) 
        return final_list 
    
    


```python
if 1==1:
    test_list = ['a', 'a', 'b']
    print(rm_dup(test_list))
```

    ['a', 'b']
    

## Put it togther: Find Twitter, Facebook, and Instagram URLs for Colleges


```python
# Write csv header
if not os.path.isfile(os.path.join(sm_path, "ihe_sm_pages_v2.csv")):
    with open(os.path.join(sm_path, "ihe_sm_pages_v2.csv"), 'w') as f:
        print('unitid; instnm; url_main; url_admissions; twitter_links; facebook_links; instagram_links', file=f)
```


```python
# Initialize headless Firefox browser
options = Options()
options.headless = True
driver = webdriver.Firefox(options=options)

for index, row in b_df.iterrows():
    try:
        # Get URL from base webpage
        ihe_url = feeling_lucky(row['ihe_search'], driver=driver)

        # Get Twitter links
        main_twitter_links, main_facebook_links, main_instagram_links = get_handles(url=ihe_url, driver=driver)

        # Also search for admissions page to see if Twitter is different
        admissions_url = feeling_lucky(row['ihe_search'] + ' admissions', driver=driver)

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
        with open(os.path.join(sm_path, "ihe_sm_pages_v2.csv"), 'a') as f:
            print(str(row['unitid']) + ';' + row['instnm'] + ';' + ihe_url + ';' 
                + admissions_url + ';' + str(twitter_links) + ';' 
                + str(facebook_links) + ';' + str(instagram_links), file=f)
    
    except:
        print("Error: " + str(row['unitid']) + ' - ' + row['instnm'])
        with open(os.path.join(sm_path, "ihe_sm_pages_v2.csv"), 'a') as f:
            print(str(row['unitid']) + ';' + row['instnm'] + ';' + "Error" + ';' + "Error" + ';' + "Error" + ';' + "Error"+ ';' + "Error", file=f)
            
driver.quit()
    
```

    154022,Ashford University,https://www.ashford.edu/,https://www.ashford.edu/online-admissions,https://twitter.com/AshfordU,https://www.facebook.com/ashforduniversity,https://instagram.com/ashfordu
    132903,University of Central Florida,https://www.ucf.edu/,https://www.ucf.edu/admissions/,https://www.twitter.com/ucf,https://www.facebook.com/ucf,https://www.instagram.com/ucf.edu/
    134130,University of Florida,http://www.ufl.edu/,https://admissions.ufl.edu/,https://twitter.com/UF/,http://www.facebook.com/uflorida/,https://instagram.com/uflorida/
    193900,New York University,https://www.nyu.edu/,https://www.nyu.edu/admissions/undergraduate-admissions.html,https://twitter.com/nyuniversity,https://www.facebook.com/NYU,https://www.instagram.com/p/Bkfnfy1F0Mg/
    232557,Liberty University,http://www.liberty.edu/,https://www.liberty.edu/admissions/,http://twitter.com/libertyu?utm_source=liberty.edu&utm_medium=footer&utm_campaign=lu_twitter,http://www.facebook.com/LibertyUniversity?utm_source=liberty.edu&utm_medium=footer&utm_campaign=lu_facebook,http://www.instagram.com/LibertyUniversity
    228778,The University of Texas at Austin,https://www.utexas.edu/,https://admissions.utexas.edu/,http://www.twitter.com/UTAustin,http://www.facebook.com/UTAustinTX,https://www.instagram.com/utaustintx
    204796,Ohio State University-Main Campus,https://www.osu.edu/,https://www.osu.edu/futurestudents/,http://twitter.com/OhioState,http://www.facebook.com/osu,https://www.instagram.com/theohiostateuniversity/
    170976,University of Michigan-Ann Arbor,https://umich.edu/,https://admissions.umich.edu/,http://www.twitter.com/umich,http://www.facebook.com/universityofmichigan,http://instagram.com/uofmichigan
    174066,University of Minnesota-Twin Cities,https://twin-cities.umn.edu/,https://twin-cities.umn.edu/admissions-aid,https://twitter.com/umnews,https://www.facebook.com/UofMN,https://instagram.com/umnpics
    104151,Arizona State University-Tempe,https://www.asu.edu/,https://admission.asu.edu/,http://www.twitter.com/asu,http://www.facebook.com/arizonastateuniversity,https://www.instagram.com/FutureSunDevils/
    123961,University of Southern California,https://www.usc.edu/,https://admission.usc.edu/,https://twitter.com/usc,https://www.facebook.com/usc/,http://instagram.com/uscedu
    236948,University of Washington-Seattle Campus,https://www.washington.edu/,https://www.washington.edu/admissions/,https://twitter.com/UW,https://www.facebook.com/UofWA,http://instagram.com/uofwa
    228723,Texas A & M University-College Station,https://www.tamu.edu/,https://admissions.tamu.edu/,https://twitter.com/tamu,https://www.facebook.com/tamu,https://www.instagram.com/tamu/
    214777,Pennsylvania State University-Main Campus,https://www.psu.edu/,https://admissions.psu.edu/,//twitter.com/penn_state,http://www.facebook.com/pennstate,http://instagram.com/pennstate
    145637,University of Illinois at Urbana-Champaign,https://illinois.edu/,https://illinois.edu/admissions/,https://twitter.com/share?url=https://sustainability.illinois.edu/from-field-to-library-fresh-press-and-the-illinois-library-venture-into-sustainable-conservation/,https://www.facebook.com/sharer/sharer.php?u=https://online.illinois.edu/news/2018/10/17/university-of-illinois-launches-coursera-for-illinois,http://instagram.com/illinois1867
    134097,Florida State University,https://www.fsu.edu/,https://admissions.fsu.edu/,https://twitter.com/floridastate,https://www.facebook.com/floridastate,https://instagram.com/floridastateuniversity/
    110635,University of California-Berkeley,https://www.berkeley.edu/,https://admissions.berkeley.edu/,https://twitter.com/#!/UCBerkeley,http://www.facebook.com/UCBerkeley,http://instagram.com/ucberkeleyofficial#
    240444,University of Wisconsin-Madison,https://www.wisc.edu/,https://www.wisc.edu/admissions/,https://twitter.com/uwmadison,https://facebook.com/uwmadison,https://www.instagram.com/uwmadison/
    133951,Florida International University,https://www.fiu.edu/,https://admissions.fiu.edu/,https://twitter.com/fiu,https://www.facebook.com/floridainternational,https://www.instagram.com/fiuinstagram/
    110662,University of California-Los Angeles,http://www.ucla.edu/,http://www.admission.ucla.edu/,http://twitter.com/ucla,http://www.facebook.com/uclabruins,http://www.instagram.com/ucla
    190150,Columbia University in the City of New York,https://www.columbia.edu/content/columbia-and-new-york,https://www.columbia.edu/content/admissions,https://twitter.com/columbia,https://www.facebook.com/columbia,https://www.instagram.com/columbia/
    151351,Indiana University-Bloomington,https://www.indiana.edu/,https://admissions.indiana.edu/,https://www.twitter.com/IUBloomington,https://www.facebook.com/IndianaUniversity,https://www.instagram.com/p/BpZlX6vlMdQ/
    163286,University of Maryland-College Park,https://www.umd.edu/,https://www.umd.edu/contact-us,https://twitter.com/UofMaryland,https://www.facebook.com/UnivofMaryland,https://www.instagram.com/univofmaryland/
    171100,Michigan State University,https://msu.edu/,https://admissions.msu.edu/apply/freshman/default.aspx,https://www.twitter.com/michiganstateu,https://www.facebook.com/spartans.msu/,https://www.instagram.com/michiganstateu
    Error: 104717 - Grand Canyon University
    137351,University of South Florida-Main Campus,https://www.usf.edu/,https://www.usf.edu/admissions/,http://twitter.com/USouthFlorida,http://facebook.com/USouthFlorida,http://www.instagram.com/usouthflorida
    243780,Purdue University-Main Campus,https://www.purdue.edu/,https://www.purdue.edu/purdue/admissions/index.php,https://twitter.com/lifeatpurdue,https://www.facebook.com/PurdueUniversity/,https://www.instagram.com/lifeatpurdue/
    110644,University of California-Davis,https://www.ucdavis.edu/,https://www.ucdavis.edu/admissions/,https://twitter.com/ucdavis,https://www.facebook.com/UCDavis,https://instagram.com/ucdavis
    139959,University of Georgia,https://www.uga.edu/,https://www.admissions.uga.edu/,https://twitter.com/universityofga,https://www.facebook.com/universityofga/,https://www.instagram.com/universityofga/
    163204,University of Maryland-University College,https://www.umuc.edu/,https://www.umuc.edu/admissions/index.cfm,http://twitter.com/umuc,https://www.facebook.com/UMUC,http://instagram.com/umucpix
    215293,University of Pittsburgh-Pittsburgh Campus,https://www.pitt.edu/,https://www.pitt.edu/admissions,https://twitter.com/PittTweet,http://www.facebook.com/upitt,http://www.instagram.com/pittadmissions
    104179,University of Arizona,https://www.arizona.edu/,https://admissions.arizona.edu/,http://twitter.com/uofa,http://facebook.com/uarizona,http://instagram.com/uarizona
    186380,Rutgers University-New Brunswick,https://www.rutgers.edu/,https://admissions.rutgers.edu/,https://twitter.com/RutgersU,//www.facebook.com/RutgersU?utm_source=uwide_links&utm_medium=web&utm_campaign=uwide_links,//instagram.com/RutgersU?utm_source=uwide_links&utm_medium=web&utm_campaign=uwide_links
    228769,The University of Texas at Arlington,https://www.uta.edu/uta/,https://www.uta.edu/admissions/,https://twitter.com/utarlington,https://www.facebook.com/utarlington/,https://www.instagram.com/utarlington/
    164988,Boston University,https://www.bu.edu/,https://www.bu.edu/admissions/,https://twitter.com/BU_Tweets,http://www.facebook.com/BostonUniversity,https://www.instagram.com/applytobu/
    110565,California State University-Fullerton,http://www.fullerton.edu/,http://international.fullerton.edu/admissions/,No Twitter found,No Facebook found,No Instagram found
    131469,George Washington University,https://www.gwu.edu/,https://undergraduate.admissions.gwu.edu/,https://twitter.com/GWtweets?lang=en,https://www.facebook.com/georgewashingtonuniversity/,https://www.instagram.com/gwuniversity/?hl=en
    199120,University of North Carolina at Chapel Hill,https://www.unc.edu/,https://www.unc.edu/,https://twitter.com/unc,https://www.facebook.com/uncchapelhill/,https://www.instagram.com/uncchapelhill/
    110583,California State University-Long Beach,https://www.csulb.edu/,http://www.csulb.edu/admissions,https://twitter.com/csulb,https://www.facebook.com/csulb,http://instagram.com/csulongbeach
    215062,University of Pennsylvania,https://www.upenn.edu/,https://www.upenn.edu/admissions,http://twitter.com/Penn,https://www.facebook.com/UnivPennsylvania,http://instagram.com/uofpenn
    110608,California State University-Northridge,https://www.csun.edu/,https://www.csun.edu/admissions-records,https://twitter.com/csunorthridge,https://www.facebook.com/calstatenorthridge,https://www.instagram.com/csun_edu/
    232186,George Mason University,https://www2.gmu.edu/,https://www2.gmu.edu/admissions-aid,https://twitter.com/georgemasonu,http://www.facebook.com/georgemason,https://instagram.com/georgemasonu
    227216,University of North Texas,https://www.unt.edu/,https://admissions.unt.edu/,https://twitter.com/UNTsocial,https://www.facebook.com/northtexas,https://www.instagram.com/UNT/
    201885,University of Cincinnati-Main Campus,https://www.uc.edu/,https://admissions.uc.edu/,http://twitter.com/UofCincy,http://www.facebook.com/uofcincinnati,http://instagram.com/uofcincy
    216339,Temple University,https://www.temple.edu/,https://www.temple.edu/admissions,https://twitter.com/templeuniv,https://www.facebook.com/templeu,https://instagram.com/templeuniv
    225511,University of Houston,http://www.uh.edu/,http://www.uh.edu/admissions/,http://twitter.com/UHouston,http://www.facebook.com/universityofhouston,http://instagram.com/universityofhouston
    199193,North Carolina State University at Raleigh,https://www.ncsu.edu/,https://www.ncsu.edu/admissions/,http://www.twitter.com/ncstate/,https://www.facebook.com/ncstate,http://instagram.com/ncstate
    110680,University of California-San Diego,https://ucsd.edu/,https://admissions.ucsd.edu/,https://twitter.com/ucsandiego/,https://www.facebook.com/UCSanDiego/,https://www.instagram.com/ucsandiego/
    153658,University of Iowa,https://uiowa.edu/,https://uiowa.edu/admission,https://www.twitter.com/uiowa,https://www.facebook.com/universityofiowa,https://www.instagram.com/uiowa
    433387,Western Governors University,https://www.wgu.edu/#,https://www.wgu.edu/admissions.html#,https://twitter.com/wgu,https://www.facebook.com/wgu.edu/,https://www.instagram.com/westerngovernorsu/
    110653,University of California-Irvine,https://uci.edu/,https://www.admissions.uci.edu/,https://twitter.com/ucirvine,https://www.facebook.com/UCIrvine,http://instagram.com/ucirvine
    204857,Ohio University-Main Campus,https://www.ohio.edu/,https://www.ohio.edu/admissions/,http://twitter.com/ohiou/,http://www.facebook.com/OhioUniversity/,https://instagram.com/ohiouadmissions/
    126614,University of Colorado Boulder,https://www.colorado.edu/,https://www.colorado.edu/admissions,http://www.twitter.com/cuboulder,http://www.facebook.com/cuboulder,http://instagram.com/cuboulder
    167358,Northeastern University,https://www.northeastern.edu/,https://www.northeastern.edu/admissions/,//twitter.com/Northeastern,//www.facebook.com/northeastern/,https://www.instagram.com/northeastern/
    178396,University of Missouri-Columbia,https://missouri.edu/,https://admissions.missouri.edu/,https://twitter.com/mizzou,https://facebook.com/Mizzou,https://instagram.com/mizzou
    230764,University of Utah,https://www.utah.edu/,https://admissions.utah.edu/apply/,https://twitter.com/UUtah,http://www.facebook.com/universityofutah,https://www.instagram.com/universityofutah/
    230038,Brigham Young University-Provo,https://www.byu.edu/,https://www.byu.edu/admissions,http://twitter.com/byu,https://www.facebook.com/BYU/videos/158650175080587/?__tn__=-R&__xts__%5B0%5D=68.ARA7n87LkTbSZzAxpdR9KwyWT7Adm44E9aVs-8mJZvAuvEzykXdt-O9M7jLea1zDt97OZWlLGpwsrA33IM3IbEDIYg5CbwfHync9HD2urhA-mvz27i31Mb9kkk0-fjb-BH027e8B9rmLLWRg5etD-WD_hfMJK_HDPYdcv3enTX9XZZ2HSeOl4t5EpgWJXRRco7rPSOTlgE3CGE6hL3bgyGppiwYN4GzRQvXoXkbOFw,https://www.instagram.com/brighamyounguniversity/
    129020,University of Connecticut,https://uconn.edu/,https://uconn.edu/admissions/,https://twitter.com/uconn,https://facebook.com/UConn,https://instagram.com/UConn/
    122409,San Diego State University,https://www.sdsu.edu/,https://admissions.sdsu.edu/,http://twitter.com/sdsu,http://www.facebook.com/SanDiegoState,http://instagram.com/sandiegostateuniversity#
    233921,Virginia Polytechnic Institute and State University,https://vt.edu/,https://vt.edu/admissions/undergraduate.html,https://twitter.com/virginia_tech,https://facebook.com/virginiatech,https://instagram.com/virginia.tech
    218663,University of South Carolina-Columbia,https://www.sc.edu/,https://sc.edu/about/offices_and_divisions/undergraduate_admissions/,No Twitter found,No Facebook found,No Instagram found
    196088,University at Buffalo,https://www.buffalo.edu/,https://admissions.buffalo.edu/,https://twitter.com/UBuffalo,https://www.facebook.com/universityatbuffalo,https://www.instagram.com/universityatbuffalo/
    122597,San Francisco State University,http://www.sfsu.edu/,http://bulletin.sfsu.edu/undergraduate-admissions/,https://twitter.com/SFSU,https://www.facebook.com/sanfranciscostate,http://instagram.com/sanfranciscostate
    166629,University of Massachusetts-Amherst,https://www.umass.edu/,https://www.umass.edu/admissions/,http://twitter.com/umassamherst,http://www.facebook.com/umassamherst,http://instagram.com/umass
    122755,San Jose State University,http://www.sjsu.edu/,http://www.sjsu.edu/admissions/,https://twitter.com/sjsu,http://www.facebook.com/sanjosestate,http://www.instagram.com/sjsu
    162928,Johns Hopkins University,https://www.jhu.edu/,https://www.jhu.edu/admissions/,http://www.twitter.com/JohnsHopkins,http://www.facebook.com/johnshopkinsuniversity,http://instagram.com/johnshopkinsu
    100751,The University of Alabama,https://www.ua.edu/,https://gobama.ua.edu/,https://www.twitter.com/UofAlabama,https://www.facebook.com/UniversityofAlabama,https://www.instagram.com/univofalabama
    221759,The University of Tennessee-Knoxville,https://www.utk.edu/,https://www.utk.edu/admissions/,No Twitter found,No Facebook found,No Instagram found
    228459,Texas State University,https://www.txstate.edu/,https://www.admissions.txstate.edu/,https://twitter.com/txst,https://www.facebook.com/txstateu/photos/a.313278130566/10160856974535567/?type=3,https://www.instagram.com/p/BpaWPpLlpWh/
    139940,Georgia State University,https://www.gsu.edu/,https://admissions.gsu.edu/,https://twitter.com/georgiastateu,https://www.facebook.com/GeorgiaStateUniversity,https://instagram.com/georgiastateuniversity
    234030,Virginia Commonwealth University,https://www.vcu.edu/,https://www.vcu.edu/admissions/,https://twitter.com/VCU,http://www.facebook.com/virginiacommonwealthuniversity,https://www.instagram.com/p/BpZ3yQLAxPm/
    166027,Harvard University,https://www.harvard.edu/,https://college.harvard.edu/admissions,http://twitter.com/harvartmuseums,http://facebook.com/harvard,http://www.instagram.com/harvard
    136215,Nova Southeastern University,https://www.nova.edu/,https://www.nova.edu/undergraduate/admissions/index.html,http://www.twitter.com/nsuflorida,https://www.facebook.com/NSUFlorida,https://www.instagram.com/nsuflorida/
    229115,Texas Tech University,https://www.ttu.edu/,http://www.depts.ttu.edu/admissions/,http://www.twitter.com/texastech/,http://www.facebook.com/TexasTechYou,https://www.instagram.com/texastech/
    234076,University of Virginia-Main Campus,http://www.virginia.edu/,https://admission.virginia.edu/,https://twitter.com/uva,https://www.facebook.com/UniversityofVirginia,https://www.instagram.com/p/BpaOrYzFfZ8/
    133669,Florida Atlantic University,http://www.fau.edu/,https://www.fau.edu/admissions/,https://twitter.com/FloridaAtlantic,//www.facebook.com/FloridaAtlantic,//instagram.com/FloridaAtlantic
    209807,Portland State University,https://www.pdx.edu/,https://www.pdx.edu/undergraduate-admissions/,http://www.twitter.com/portland_state,https://www.facebook.com/portlandstateu,https://www.instagram.com/portlandstate/
    145600,University of Illinois at Chicago,https://www.uic.edu/,https://admissions.uic.edu/,https://twitter.com/thisisuic,https://www.facebook.com/uic.edu,https://instagram.com/uicamiridis/
    126818,Colorado State University-Fort Collins,https://www.colostate.edu/,https://admissions.colostate.edu/,https://twitter.com/coloradostateu,https://www.facebook.com/coloradostateuniversity,https://www.instagram.com/coloradostateuniversity/
    190415,Cornell University,https://www.cornell.edu/,https://admissions.cornell.edu/,http://www.twitter.com/Cornell,https://www.facebook.com/Cornell,//instagram.com/CornellUniversity
    153603,Iowa State University,https://www.iastate.edu/,https://www.admissions.iastate.edu/,http://twitter.com/IowaStateU,http://www.facebook.com/IowaStateU,https://www.instagram.com/iowastateu/
    151111,Indiana University-Purdue University-Indianapolis,https://www.iupui.edu/,https://www.iupui.edu/,https://twitter.com/iupui,https://www.facebook.com/IUPUI/,https://www.instagram.com/p/BpZv1N_AW32/
    144740,DePaul University,https://www.depaul.edu/Pages/default.aspx,https://www.depaul.edu/admission-and-aid/Pages/default.aspx,https://twitter.com/depaulu,https://www.facebook.com/DePaulUniversity/,https://www.instagram.com/depaulu/
    196097,Stony Brook University,https://www.stonybrook.edu/,https://www.stonybrook.edu/undergraduate-admissions/,https://twitter.com/stonybrooku,https://www.facebook.com/stonybrooku,http://instagram.com/stonybrooku
    450933,Columbia Southern University,https://www.columbiasouthern.edu/,https://www.columbiasouthern.edu/admissions,No Twitter found,https://www.facebook.com/Columbia.Southern,http://instagram.com/columbiasouthernuniversity
    155317,University of Kansas,https://ku.edu/,https://admissions.ku.edu/,https://twitter.com/KUnews,https://www.facebook.com/KU/,https://www.instagram.com/universityofkansas/
    179894,Webster University,http://www.webster.edu/,http://www.webster.edu/admissions/,http://twitter.com/websteru,http://www.facebook.com/websteru,http://instagram.com/websteru
    105330,Northern Arizona University,https://nau.edu/,https://nau.edu/admissions/,https://twitter.com/nau,https://www.facebook.com/NAUFlagstaff,https://instagram.com/nauflagstaff/
    236939,Washington State University,https://wsu.edu/,https://admission.wsu.edu/,https://twitter.com/wsupullman,https://www.facebook.com/sharer/sharer.php?u=https://wsu.edu/,https://www.instagram.com/WSUPullman
    110705,University of California-Santa Barbara,https://www.ucsb.edu/,http://admissions.sa.ucsb.edu/,https://twitter.com/ucsantabarbara,https://www.facebook.com/ucsantabarbara,https://www.instagram.com/ucsantabarbara/
    110617,California State University-Sacramento,https://www.csus.edu/,https://www.csus.edu/admissions/,http://twitter.com/sacstate,http://www.facebook.com/sacstate,http://www.instagram.com/sacstate
    209551,University of Oregon,https://www.uoregon.edu/,https://admissions.uoregon.edu/,https://www.twitter.com/uoregon?utm_source=homepage&utm_campaign=socialmedia,https://www.facebook.com/universityoforegon?utm_source=homepage&utm_campaign=socialmedia,https://instagram.com/uoregon?utm_source=homepage&utm_campaign=socialmedia
    203517,Kent State University at Kent,https://www.kent.edu/,https://www.kent.edu/admissions/undergraduate,https://www.twitter.com/kentstate,https://www.facebook.com/kentstate,https://www.instagram.com/kentstate
    212054,Drexel University,https://drexel.edu/,https://drexel.edu/cnhp/academics/graduate/MHS-Physician-Assistant/,https://twitter.com/drexeluniv,http://www.facebook.com/drexeluniv,http://instagram.com/drexeluniv
    157085,University of Kentucky,http://www.uky.edu/UKHome/,http://www.uky.edu/admission/welcome-university-kentucky,http://twitter.com/universityofky,http://www.facebook.com/universityofky,https://www.instagram.com/universityofky/
    196413,Syracuse University,https://www.syracuse.edu/,https://www.syracuse.edu/admissions/,https://twitter.com/SyracuseU,https://www.facebook.com/syracuseuniversity/,https://www.instagram.com/syracuseu/
    169248,Central Michigan University,https://www.cmich.edu/Pages/default.aspx,https://www.cmich.edu/admissions/Pages/default.aspx,https://twitter.com/CMUniversity,https://www.facebook.com/cmich,http://instagram.com/cmuniversity/
    207500,University of Oklahoma-Norman Campus,http://www.ou.edu/,http://www.ou.edu/admissions,#twitter,#facebook,https://www.instagram.com/uofoklahoma/
    198464,East Carolina University,http://www.ecu.edu/,http://www.ecu.edu/cs-acad/admissions/,https://twitter.com/EastCarolina,https://www.facebook.com/EastCarolina,https://www.instagram.com/eastcarolinauniv/
    238032,West Virginia University,https://www.wvu.edu/,https://admissions.wvu.edu/,https://twitter.com/WestVirginiaU/,http://www.facebook.com/wvumountaineers,https://instagram.com/westvirginiau/
    131496,Georgetown University,https://www.georgetown.edu/,https://www.georgetown.edu/admissions,http://twitter.com/georgetown,http://www.facebook.com/georgetownuniv,https://instagram.com/georgetownuniversity/
    199139,University of North Carolina at Charlotte,https://www.uncc.edu/,https://www.uncc.edu/landing/admissions-financial-aid,https://twitter.com/UNCCharlotte,https://www.facebook.com/UNCCharlotte,https://www.instagram.com/unccharlotte/
    100858,Auburn University,http://www.auburn.edu/,http://www.auburn.edu/admissions/,http://twitter.com/auburnu,http://www.facebook.com/auburnu,https://instagram.com/auburnu
    170082,Grand Valley State University,https://www.gvsu.edu/,https://www.gvsu.edu/admissions/,http://pic.twitter.com/D9RvJKQO8O,http://www.facebook.com/grandvalley,http://www.instagram.com/gvsu
    240453,University of Wisconsin-Milwaukee,https://uwm.edu/,https://uwm.edu/admission/,http://twitter.com/uwm,http://facebook.com/uwmilwaukee,http://instagram.com/uwmilwaukee
    229027,The University of Texas at San Antonio,https://www.utsa.edu/,https://www.utsa.edu/admissions/,https://twitter.com/utsa,https://www.facebook.com/utsa,https://instagram.com/utsa
    151801,Indiana Wesleyan University,https://www.indwes.edu/,https://www.indwes.edu/admissions,https://twitter.com/IndWes,https://www.facebook.com/IndWes,No Instagram found
    164076,Towson University,https://www.towson.edu/,https://www.towson.edu/admissions/,http://twitter.com/TowsonU,https://www.facebook.com/towsonuniversity,http://instagram.com/towsonuniversity
    172699,Western Michigan University,https://wmich.edu/,https://wmich.edu/admissions,https://twitter.com/WesternMichU,https://www.facebook.com/westernmichu/,https://www.instagram.com/westernmichu
    228787,The University of Texas at Dallas,https://www.utdallas.edu/,https://www.utdallas.edu/,https://twitter.com/ut_dallas,https://www.facebook.com/utdallas,https://www.instagram.com/ut_dallas/
    172644,Wayne State University,https://wayne.edu/,https://wayne.edu/admissions,https://twitter.com/waynestate,https://www.facebook.com/waynestateuniversity,https://www.instagram.com/waynestate/
    187985,University of New Mexico-Main Campus,http://www.unm.edu/,http://admissions.unm.edu/,https://www.twitter.com/UNM,https://facebook.com/profile.php?id=21749746264,https://www.instagram.com/uofnm
    209542,Oregon State University,https://oregonstate.edu/,https://admissions.oregonstate.edu/,https://twitter.com/oregonstate,https://www.facebook.com/osubeavers,https://instagram.com/oregonstate
    145813,Illinois State University,https://illinoisstate.edu/,https://illinoisstate.edu/admissions/,https://twitter.com/IllinoisStateU,https://www.facebook.com/IllinoisStateUniversity,https://www.instagram.com/illinoisstateu/
    139755,Georgia Institute of Technology-Main Campus,https://www.gatech.edu/,http://admission.gatech.edu/,https://twitter.com/georgiatech,https://www.facebook.com/georgiatech,https://instagram.com/georgiatech
    147703,Northern Illinois University,https://www.niu.edu/index.shtml,https://www.niu.edu/admissions/,https://twitter.com/NIUlive,https://www.facebook.com/NorthernIllUniv,https://instagram.com/northern_illinois_university
    190594,CUNY Hunter College,http://www.hunter.cuny.edu/main/,http://www.hunter.cuny.edu/admissions/,http://www.twitter.com/hunter_college,https://www.facebook.com/huntercollege,http://instagram.com/huntercollege
    232982,Old Dominion University,https://www.odu.edu/,https://www.odu.edu/admission,https://twitter.com/odu,https://www.facebook.com/Old.Dominion.University,https://instagram.com/olddominionu
    230728,Utah State University,http://www.usu.edu/,https://www.usu.edu/admissions/apply/,http://twitter.com/USUAggies,http://www.facebook.com/UtahState,http://instagram.com/usuaggielife/
    200800,University of Akron Main Campus,https://www.uakron.edu/,https://www.uakron.edu/admissions/,https://twitter.com/uakron,https://www.facebook.com/UniversityofAkron,http://instagram.com/uakron
    102368,Troy University,https://www.troy.edu/,https://www.troy.edu/admissions/,https://twitter.com/TROYUNews,https://www.facebook.com/TroyUniversity/,http://instagram.com/troyuniversity
    130943,University of Delaware,https://www.udel.edu/,https://www.udel.edu/apply/undergraduate-admissions/,https://twitter.com/UDelaware,https://www.facebook.com/udelaware,https://www.instagram.com/udelaware/
    181464,University of Nebraska-Lincoln,https://www.unl.edu/,https://admissions.unl.edu/,https://twitter.com/UNLincoln,https://www.facebook.com/sharer/sharer.php?u=https%3A%2F%2Fwww.unl.edu%2F%3Futm_campaign%3Dwdn_social%26utm_medium%3Dshare_this%26utm_source%3Dwdn_facebook,https://www.instagram.com/p/BpQRkTaBES9/
    149222,Southern Illinois University-Carbondale,https://siu.edu/,https://admissions.siu.edu/,https://twitter.com/SIUCAdmissions,https://www.facebook.com/SIUC.Admissions/,https://www.instagram.com/siucadmissions/
    198419,Duke University,https://www.duke.edu/,https://admissions.duke.edu/,https://twitter.com/DukeU,https://www.facebook.com/DukeUniv,http://instagram.com/dukeuniversity
    182281,University of Nevada-Las Vegas,https://www.unlv.edu/,https://www.unlv.edu/admissions/freshman,https://twitter.com/UNLV,https://www.facebook.com/OfficialUNLV,https://www.instagram.com/unlv
    110671,University of California-Riverside,https://www.ucr.edu/,http://admissions.ucr.edu/,https://twitter.com/UCRiverside,https://www.facebook.com/UCRiverside,https://www.instagram.com/ucriversideofficial/
    220978,Middle Tennessee State University,https://www.mtsu.edu/,https://www.mtsu.edu/how-to-apply/,https://twitter.com/MTSUNews,https://www.facebook.com/mtsublueraiders,https://instagram.com/p/BpUrV5eh-6V
    146719,Loyola University Chicago,https://www.luc.edu/,https://www.luc.edu/admission.shtml,http://twitter.com/waltertangarife,http://www.facebook.com/LoyolaChicago,http://instagram.com/loyolachicago
    207388,Oklahoma State University-Main Campus,https://go.okstate.edu/,https://admissions.okstate.edu/,https://twitter.com/okstate,https://www.facebook.com/pages/Oklahoma-State-University/39013362306,http://instagram.com/okstateu
    150136,Ball State University,https://www.bsu.edu/,https://www.bsu.edu/admissions,https://twitter.com/ballstate,https://www.facebook.com/ballstate/,https://www.instagram.com/ballstateuniversity/
    217882,Clemson University,http://www.clemson.edu/,http://www.clemson.edu/admissions/,http://twitter.com/clemsonuniv,http://www.facebook.com/clemsonuniv,http://instagram.com/clemsonuniversity
    243744,Stanford University,https://www.stanford.edu/,https://www.stanford.edu/admission/,http://twitter.com/stanford,http://www.facebook.com/stanford,http://instagram.com/stanford
    137032,Saint Leo University,https://www.saintleo.edu/,https://www.saintleo.edu/university-campus-admissions,https://twitter.com/SaintLeoUniv,https://www.facebook.com/officialsaintleo,https://www.instagram.com/saintleouniv/
    106397,University of Arkansas,https://www.uark.edu/,https://admissions.uark.edu/,https://twitter.com/uarkansas,https://www.facebook.com/UofArkansas,https://instagram.com/uarkansas
    232423,James Madison University,https://www.jmu.edu/,https://www.jmu.edu/admissions/undergrad/index.shtml,https://twitter.com/JMU,https://www.facebook.com/jamesmadisonuniversity/,https://www.instagram.com/jamesmadisonuniversity/
    191241,Fordham University,https://www.fordham.edu/,https://www.fordham.edu/info/20063/undergraduate_admission,No Twitter found,No Facebook found,https://instagram.com/fordhamuniversity
    190664,CUNY Queens College,https://www.qc.cuny.edu/pages/home.aspx,https://www.qc.cuny.edu/admissions/Pages/default.aspx,https://twitter.com/QC_News,https://www.facebook.com/home.php?#!/QueensCollege,No Instagram found
    155399,Kansas State University,https://www.k-state.edu/,http://www.k-state.edu/admissions/,https://twitter.com/KState,https://www.facebook.com/KState,https://www.instagram.com/p/BpaHXM5FovA/
    144050,University of Chicago,https://www.uic.edu/,https://admissions.uic.edu/,https://twitter.com/thisisuic,https://www.facebook.com/uic.edu,https://instagram.com/uicamiridis/
    164924,Boston College,https://www.bc.edu/,https://www.bc.edu/bc-web/admission.html,https://twitter.com/BostonCollege?ref_src=twsrc%5Etfw,https://www.facebook.com/BostonCollege,https://instagram.com/bostoncollege/
    204024,Miami University-Oxford,https://www.miami.miamioh.edu/,http://miamioh.edu/admission/,https://www.twitter.com/PresGreg,https://www.facebook.com/75387987874/posts/10156188279557875,https://www.instagram.com/p/BorznObh7o5/
    119605,National University,https://www.nu.edu/,https://www.nu.edu/Admissions.html,https://twitter.com/natuniv,https://www.facebook.com/nationaluniversity,https://www.instagram.com/nationaluniversity/
    126562,University of Colorado Denver,http://www.ucdenver.edu/pages/ucdwelcomepage.aspx,http://www.ucdenver.edu/admissions/Pages/index.aspx,https://twitter.com/cudenver,https://www.facebook.com/events/286008635457845/,https://www.instagram.com/p/BozRl4dH9aK/?taken-by=cudenver
    135726,University of Miami,https://welcome.miami.edu/,https://welcome.miami.edu/admissions/index.html,https://twitter.com/univmiami,https://www.facebook.com/events/516937869134636/,https://instagram.com/univmiami
    157289,University of Louisville,http://louisville.edu/,http://louisville.edu/admissions/,https://twitter.com/uofl,https://www.facebook.com/UniversityofLouisville/,https://www.instagram.com/universityoflouisville
    169798,Eastern Michigan University,https://www.emich.edu/,https://www.emich.edu/admissions/index.php,https://twitter.com/EasternMichU/,https://www.facebook.com/EasternMichU/,https://www.instagram.com/easternmichigan/
    110592,California State University-Los Angeles,http://www.calstatela.edu/,http://www.calstatela.edu/admissions,https://twitter.com/calstatela,https://www.facebook.com/CalStateLA,http://instagram.com/calstatela
    206084,University of Toledo,http://www.utoledo.edu/,http://www.utoledo.edu/admission/,No Twitter found,No Facebook found,No Instagram found
    110714,University of California-Santa Cruz,https://www.ucsc.edu/,https://admissions.ucsc.edu/,https://www.twitter.com/ucsc,https://www.facebook.com/ucsantacruz,https://instagram.com/ucsc
    196060,SUNY at Albany,https://www.albany.edu/,https://www.albany.edu/,http://www.albany.edu/main/twitter.shtml,http://www.albany.edu/main/facebook.shtml,http://www.albany.edu/main/instagram.shtml
    141574,University of Hawaii at Manoa,https://manoa.hawaii.edu/,http://manoa.hawaii.edu/admissions/,https://twitter.com/UHManoa,https://www.facebook.com/uhmanoa,https://instagram.com/uhmanoanews
    230782,Weber State University,https://www.weber.edu/,https://www.weber.edu/admissions,https://twitter.com/WeberStateU,https://www.facebook.com/WeberState,https://instagram.com/weberstate
    190512,CUNY Bernard M Baruch College,http://www.baruch.cuny.edu/,http://www.baruch.cuny.edu/admission/,http://www.twitter.com/baruchcollege,http://www.facebook.com/baruchcollege,No Instagram found
    110529,California State Polytechnic University-Pomona,https://www.cpp.edu/,https://www.cpp.edu/~admissions/,https://twitter.com/cpp_admissions,https://www.facebook.com/cppadmissions,https://www.instagram.com/cpp_admissions/
    139658,Emory University,http://www.emory.edu/home/index.html,https://www.emory.edu/home/admission/index.html,http://twitter.com/emoryuniversity,http://www.facebook.com/EmoryUniversity,http://instagram.com/emoryuniversity
    110556,California State University-Fresno,http://www.csufresno.edu/,http://www.fresnostate.edu/home/admissions/index.html,http://twitter.com/Fresno_State,http://www.facebook.com/fresnostate,http://instagram.com/fresno_state
    230737,Utah Valley University,https://www.uvu.edu/,https://www.uvu.edu/admissions/,https://twitter.com/UVU,https://www.facebook.com/UtahValleyUniversity/videos/315719132547332/,https://www.instagram.com/utah.valley.university/
    228796,The University of Texas at El Paso,https://www.utep.edu/,https://www.utep.edu/student-affairs/admissions/,https://twitter.com/utepnews ,https://www.facebook.com/UTEPMiners/ ,https://www.instagram.com/utep_miners/ 
    196592,Touro College,https://www.touro.edu/,https://www.touro.edu/admissions/,https://twitter.com/WeAreTouro,https://www.facebook.com/WeAreTouro,https://instagram.com/WeAreTouro
    179867,Washington University in St Louis,https://wustl.edu/,https://admissions.wustl.edu/,https://twitter.com/wustl/,https://www.facebook.com/pg/wustl/photos/?tab=album&album_id=10157067258936178&__tn__=-UC-R,https://www.instagram.com/wustl_official/
    100663,University of Alabama at Birmingham,http://www.uab.edu/home/,https://www.uab.edu/students/admissions/,http://twitter.com/intent/follow?source=followbutton&variant=1.0&screen_name=chooseuab,http://www.facebook.com/chooseuab,https://www.instagram.com/uabarts
    185590,Montclair State University,https://www.montclair.edu/,https://www.montclair.edu/university-admissions/,https://twitter.com/montclairstateu,https://www.facebook.com/MontclairStateUniversity,No Instagram found
    197869,Appalachian State University,https://www.appstate.edu/,https://www.appstate.edu/admissions/,https://twitter.com/appstate,https://www.facebook.com/appalachianstateuniversity,https://instagram.com/appstate/
    221999,Vanderbilt University,https://www.vanderbilt.edu/,https://admissions.vanderbilt.edu/,http://twitter.com/vanderbiltu,http://facebook.com/vanderbilt,http://instagram.com/vanderbiltu
    220862,University of Memphis,https://www.memphis.edu/,https://www.memphis.edu/admissions/,https://twitter.com/uofmemphis,https://www.facebook.com/uofmemphis,https://www.instagram.com/uofmemphis/
    211440,Carnegie Mellon University,https://www.cmu.edu/,https://admission.enrollment.cmu.edu/,http://www.twitter.com/carnegiemellon,http://www.facebook.com/carnegiemellonu,https://instagram.com/carnegiemellon/
    130794,Yale University,https://www.yale.edu/,https://www.yale.edu/admissions,http://www.twitter.com/yale,https://www.facebook.com/YaleUniversity,//instagram.com/yale/
    195809,St John's University-New York,https://www.stjohns.edu/,https://www.stjohns.edu/admission-aid/undergraduate-admission,https://twitter.com/StJohnsU,https://www.facebook.com/stjohnsu/,https://www.instagram.com/stjohnsu/
    140164,Kennesaw State University,http://www.kennesaw.edu/,http://www.kennesaw.edu/admissions.php,https://twitter.com/kennesawstate,https://www.facebook.com/KennesawStateUniversity,https://instagram.com/kennesawstateuniversity
    179566,Missouri State University-Springfield,https://www.missouristate.edu/,https://www.missouristate.edu/admissions/,https://twitter.com/missouristate,https://www.facebook.com/missouristateu,https://instagram.com/missouristate
    183044,University of New Hampshire-Main Campus,https://www.unh.edu/,https://admissions.unh.edu/,http://www.twitter.com/uofnh,https://www.facebook.com/universityofnewhampshire,http://www.instagram.com/uofnh
    227881,Sam Houston State University,http://www.shsu.edu/,http://www.shsu.edu/admissions/,//twitter.com/samhoustonstate,//www.facebook.com/samhoustonstate,//instagram.com/samhoustonstate
    157951,Western Kentucky University,https://www.wku.edu/,https://www.wku.edu/admissions/,http://twitter.com/wkualumni,http://www.facebook.com/WKUNews,http://www.instagram.com/wku
    176080,Mississippi State University,https://www.msstate.edu/,https://www.admissions.msstate.edu/,http://www.twitter.com/msstate/,http://www.facebook.com/msstate/,https://instagram.com/msstate/
    136172,University of North Florida,http://www.unf.edu/,http://www.unf.edu/admissions/,https://twitter.com/UofNorthFlorida,https://www.facebook.com/UofNorthFlorida,http://instagram.com/uofnorthflorida/
    199148,University of North Carolina at Greensboro,https://www.uncg.edu/,https://www.uncg.edu/admissions/,https://twitter.com/UNCG,https://www.facebook.com/uncg1891/,https://www.instagram.com/uncg/
    196079,SUNY at Binghamton,https://www.binghamton.edu/,https://www.binghamton.edu/admissions/index.html,http://twitter.com/binghamtonu,http://www.facebook.com/pages/Binghamton-NY/Binghamton-University/51791915551,http://instagram.com/binghamtonu
    131159,American University,https://www.american.edu/,https://www.american.edu/admissions/,http://www.twitter.com/AmericanU,http://www.facebook.com/AmericanUniversity,http://instagram.com/americanuniversity
    152080,University of Notre Dame,https://www.nd.edu/,https://www.nd.edu/admissions/,https://twitter.com/NotreDame/,https://www.facebook.com/notredame/,https://www.instagram.com/notredame/
    223232,Baylor University,https://www.baylor.edu/,https://www.baylor.edu/admissions/,No Twitter found,No Facebook found,No Instagram found
    110538,California State University-Chico,http://www.csuchico.edu/,https://www.csuchico.edu/admissions/,https://twitter.com/chicostate,https://www.facebook.com/CaliforniaStateUniversityChico,http://instagram.com/chicostate
    127060,University of Denver,https://www.du.edu/,http://www.ucdenver.edu/admissions/Pages/index.aspx,http://www.twitter.com/uofdenver,http://www.facebook.com/uofdenver,http://www.instagram.com/uofdenver/
    106458,Arkansas State University-Main Campus,https://www.astate.edu/,https://www.astate.edu/,http://www.twitter.com/ArkansasState,http://www.facebook.com/ArkansasState,http://instagram.com/arkansasstate
    110510,California State University-San Bernardino,https://www.csusb.edu/,https://www.csusb.edu/admissions,http://www.twitter.com/csusbnews,http://www.facebook.com/CSUSB,https://instagram.com/csusb
    142115,Boise State University,https://www.boisestate.edu/,https://www.boisestate.edu/admissions/,https://twitter.com/boisestatelive,https://www.facebook.com/BoiseStateLive,https://www.instagram.com/boisestateuniversity/
    110574,California State University-East Bay,http://www.csueastbay.edu/,http://www.csueastbay.edu/admissions/,https://twitter.com/CalStateEastBay?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor,https://www.facebook.com/CalStateEastBay/,https://www.instagram.com/csueb/
    139931,Georgia Southern University,https://www.georgiasouthern.edu/,https://admissions.georgiasouthern.edu/,http://twitter.com/georgiasouthern,https://www.facebook.com/GeorgiaSouthern,http://instagram.com/georgiasouthernuniversity
    228246,Southern Methodist University,https://www.smu.edu/,https://www.smu.edu/admission,http://twitter.com/smu/,http://facebook.com/smudallas,http://www.instagram.com/smudallas
    171571,Oakland University,https://oakland.edu/,https://oakland.edu/futurestudents/,https://twitter.com/oaklandu,https://www.facebook.com/oaklandu/,https://www.instagram.com/oaklandu/
    217484,University of Rhode Island,https://www.uri.edu/,https://web.uri.edu/admission/,https://twitter.com/universityofri,https://www.facebook.com/universityofri,https://www.instagram.com/universityofri/
    202134,Cleveland State University,https://www.csuohio.edu/,https://www.csuohio.edu/admissions/admissions,https://twitter.com/CLE_State,http://www.facebook.com/clevelandstateuniversity,http://instagram.com/cle_state
    237011,Western Washington University,https://www.wwu.edu/,https://admissions.wwu.edu/freshman/freshman-requirements,https://twitter.com/WWU,http://www.facebook.com/westernwashingtonuniversity,No Instagram found
    110422,California Polytechnic State University-San Luis Obispo,https://www.calpoly.edu/,https://admissions.calpoly.edu/,https://twitter.com/calpoly,https://www.facebook.com/CalPoly,https://www.instagram.com/cal_poly
    201441,Bowling Green State University-Main Campus,https://www.bgsu.edu/,https://www.bgsu.edu/admissions.html,https://www.twitter.com/bgsu,https://www.facebook.com/OfficialBGSU/,https://www.instagram.com/officialbgsu/
    229179,Texas Woman's University,https://twu.edu/,https://twu.edu/admissions/,https://twitter.com/txwomans,https://www.facebook.com/TexasWomansUniversity/,https://www.instagram.com/txwomans/
    195003,Rochester Institute of Technology,https://www.rit.edu/,http://www.rit.edu/emcs/admissions/,https://twitter.com/RITNEWS,https://www.facebook.com/RITNews,http://instagram.com/RITTigers
    190549,CUNY Brooklyn College,http://www.brooklyn.cuny.edu/web/home.php,http://www.brooklyn.cuny.edu/web/admissions.php,http://www.brooklyn.cuny.edu/web/twitter.php,http://www.brooklyn.cuny.edu/web/facebook.php,http://www.brooklyn.cuny.edu/web/instagram.php
    168148,Tufts University,https://www.tufts.edu/,https://admissions.tufts.edu/,https://twitter.com/TuftsUniversity,https://www.facebook.com/tuftsu,http://instagram.com/tuftsuniversity
    169910,Ferris State University,https://ferris.edu/,https://ferris.edu/admissions/homepage.htm,https://twitter.com/FerrisState,https://www.facebook.com/5981304995,https://www.instagram.com/ferrisstateu/
    216764,West Chester University of Pennsylvania,https://www.wcupa.edu/,https://www.wcupa.edu/_admissions/sch_adm/,https://twitter.com/wcuofpa,https://www.facebook.com/WCUPA/videos/889988421192735/,https://www.instagram.com/wcuofpa/
    166638,University of Massachusetts-Boston,https://www.umb.edu/,https://www.umb.edu/academics/cnhs/cnhs_contact,https://twitter.com/umassboston,https://www.facebook.com/umassboston, https://www.instagram.com/umassboston/
    206604,Wright State University-Main Campus,https://www.wright.edu/,https://www.wright.edu/admissions,http://twitter.com/WrightState,http://www.facebook.com/WrightStateUniversity,https://www.instagram.com/wrightstateu/
    227368,The University of Texas-Pan American,https://en.wikipedia.org/wiki/University_of_Texas%E2%80%93Pan_American,https://www.collegefactual.com/colleges/the-university-of-texas-pan-american/applying/admission-applications/,https://twitter.com/CollegeFactual/,https://www.facebook.com/CollegeFactual/,No Instagram found
    182290,University of Nevada-Reno,https://www.unr.edu/,https://www.unr.edu/admissions,https://twitter.com/unevadareno,https://www.facebook.com/UniversityofNevada,http://www.instagram.com/unevadareno
    188030,New Mexico State University-Main Campus,https://www.nmsu.edu/,https://admissions.nmsu.edu/,http://twitter.com/#!/nmsu,http://www.facebook.com/newmexicostateuniversity,https://instagram.com/nmsu/
    127918,Regis University,https://www.regis.edu/,https://www.regis.edu/College-Admissions-and-Financial-Aid.aspx,https://twitter.com/RegisUniversity,https://www.facebook.com/regisuniversity,http://instagram.com/regisuniversity
    166683,Massachusetts Institute of Technology,http://www.mit.edu/,https://mitadmissions.org/apply/,https://twitter.com/mit,https://www.facebook.com/sharer/sharer.php?u=http://web.mit.edu/spotlight/reunion-6000-miles-home-saturday,https://www.instagram.com/mitpics/
    216597,Villanova University,https://www1.villanova.edu/university.html,https://www1.villanova.edu/university/undergraduate-admission.html,https://www.twitter.com/VillanovaU,https://www.facebook.com/VillanovaU,http://www.instagram.com/villanovau
    166513,University of Massachusetts-Lowell,https://www.uml.edu/,https://www.uml.edu/Admissions-Aid/,https://twitter.com/umasslowell,https://www.facebook.com/umlowell,https://instagram.com/umasslowell
    213020,Indiana University of Pennsylvania-Main Campus,https://www.iup.edu/,https://www.iup.edu/admissions/,http://www.twitter.com/iupedu,http://www.facebook.com/iupedu,http://instagram.com/iupedu
    156620,Eastern Kentucky University,https://www.eku.edu/,https://admissions.eku.edu/,https://twitter.com/eku,https://www.facebook.com/easternkentuckyuniversity,https://instagram.com/easternkentuckyu
    196264,SUNY Empire State College,https://www.esc.edu/,https://www.esc.edu/admissions/,https://twitter.com/@SUNYEmpire,No Facebook found,https://www.instagram.com/p/BpU_L09FKyj/
    239105,Marquette University,https://www.marquette.edu/,https://www.marquette.edu/admissions/,No Twitter found,No Facebook found,No Instagram found
    179159,Saint Louis University,https://www.slu.edu/,https://www.slu.edu/admission/index.php,https://twitter.com/SLU_Official,https://facebook.com/SaintLouisU/,https://www.instagram.com/slu_official/
    231174,University of Vermont,https://www.uvm.edu/,https://www.uvm.edu/admissions/undergraduate,http://twitter.com/uvmvermont,http://www.facebook.com/UniversityofVermont,http://instagram.com/universityofvermont
    176372,University of Southern Mississippi,https://home.usm.edu/,https://www.usm.edu/admissions,http://www.twitter.com/southernmiss,http://www.facebook.com/usm.edu,http://instagram.com/officialsouthernmiss
    174783,Saint Cloud State University,https://www.stcloudstate.edu/,https://www.stcloudstate.edu/admissions/,https://www.stcloudstate.edu/scsu/twitter.aspx,https://www.stcloudstate.edu/scsu/facebook.aspx,https://www.stcloudstate.edu/scsu/instagram.aspx
    194310,Pace University-New York,https://www.pace.edu/,https://www.pace.edu/admissions-and-aid,http://www.pace.edu/twitter/,http://www.pace.edu/facebook/,No Instagram found
    155061,Fort Hays State University,https://www.fhsu.edu/,https://www.fhsu.edu/admissions/,https://twitter.com/forthaysstate,http://www.facebook.com/forthaysstate,http://instagram.com/forthaysstate
    226091,Lamar University,https://www.lamar.edu/,https://www.lamar.edu/admissions/index.html,https://www.twitter.com/LamarUniversity,https://www.facebook.com/LamarUniversity,https://instagram.com/lamaruniversity
    199218,University of North Carolina Wilmington,https://uncw.edu/,https://uncw.edu/admissions/,http://twitter.com/uncwilmington,http://www.facebook.com/pages/Wilmington-NC/UNCW/8357990919,https://www.instagram.com/uncwilmington/
    180489,The University of Montana,http://www.umt.edu/,https://admissions.umt.edu/,https://twitter.com/share?url=http://www.umt.edu/featured-photos/?photo=celebration-of-service#celebration-of-service&via=umontana&hashtags=umontana&text=Celebration of Service,https://www.facebook.com/sharer/sharer.php?u=http://www.umt.edu/featured-photos/?photo=celebration-of-service#celebration-of-service,https://www.instagram.com/umontana
    178402,University of Missouri-Kansas City,https://www.umkc.edu/,https://www.umkc.edu/admissions/,https://twitter.com/umkansascity,https://www.facebook.com/pages/University-of-Missouri-Kansas-City/325868702683,No Instagram found
    110547,California State University-Dominguez Hills,https://www.csudh.edu/,https://www.csudh.edu/records-registration/,http://twitter.com/dominguezhills,https://www.facebook.com/csudh/,http://instagram.com/csudominguezhills?ref=badge
    191649,Hofstra University,https://www.hofstra.edu/,https://www.hofstra.edu/admission/,https://twitter.com/HofstraU?ref_src=twsrc%5Etfw,//www.facebook.com/hofstrau,//instagram.com/hofstrau
    224554,Texas A & M University-Commerce,https://www.tamuc.edu/,http://www.tamuc.edu/admissions/,https://twitter.com/tamuc,https://www.facebook.com/tamuc,https://instagram.com/tamuc/
    131113,Wilmington University,http://www.wilmu.edu/,http://www.wilmu.edu/admission/index.aspx,https://twitter.com/theWilmU,https://www.facebook.com/WilmingtonUniversity,https://www.instagram.com/wilmingtonuniversity
    173920,Minnesota State University-Mankato,https://mankato.mnsu.edu/,https://mankato.mnsu.edu/future-students/admissions/,https://twitter.com/MNSUMankato,https://www.facebook.com/MNStateMankato/,https://www.instagram.com/mnstatemankato/
    195030,University of Rochester,https://www.rochester.edu/,https://enrollment.rochester.edu/,//www.twitter.com/UofR,//www.facebook.com/pages/Rochester-NY/University-of-Rochester/6102569031,//www.instagram.com/urochester/
    154095,University of Northern Iowa,https://uni.edu/,https://admissions.uni.edu/,http://twitter.com/northerniowa,http://www.facebook.com/universityofnortherniowa,http://instagram.com/northern_iowa
    220075,East Tennessee State University,https://www.etsu.edu/ehome/,https://www.etsu.edu/admissions/,https://twitter.com/etsuadmissions,https://www.facebook.com/ETSUAdmissions/,No Instagram found
    217235,Johnson & Wales University-Providence,https://www.jwu.edu/campuses/providence/index.html,https://www.jwu.edu/admissions/,https://twitter.com/JWUProvidence,https://www.facebook.com/JWUProvidence,https://www.instagram.com/jwuprovidence/
    234827,Central Washington University,https://www.cwu.edu/,https://www.cwu.edu/programs/welcome-cwu-admissions,http://twitter.com/CentralWashU,http://www.facebook.com/cwu.wildcats,https://www.instagram.com/central_washington_university/
    127565,Metropolitan State University of Denver,https://msudenver.edu/,https://msudenver.edu/admissions/,http://twitter.com/msudenver,https://www.facebook.com/msudenver,https://instagram.com/msu_denver
    149231,Southern Illinois University-Edwardsville,http://www.siue.edu/,https://www.siue.edu/admissions/,https://twitter.com/siue,https://www.facebook.com/siuedwardsville,https://www.instagram.com/siuedwardsville/
    163268,University of Maryland-Baltimore County,https://www.umbc.edu/,https://undergraduate.umbc.edu/apply/freshmen.php,http://twitter.com/umbc,https://www.facebook.com/umbcpage,No Instagram found
    142285,University of Idaho,https://www.uidaho.edu/,https://www.uidaho.edu/admissions/apply/apply-now,https://twitter.com/uidaho,https://www.facebook.com/uidaho,http://instagram.com/uidaho
    181394,University of Nebraska at Omaha,https://www.unomaha.edu/,https://www.unomaha.edu/admissions/index.php,https://twitter.com/unomaha,https://www.facebook.com/unomaha/,https://www.instagram.com/unomaha/
    190567,CUNY City College,https://www.ccny.cuny.edu/,https://www.ccny.cuny.edu/admissions,https://twitter.com/citycollegeny,https://www.facebook.com/pages/The-City-College-of-New-York/170858093839,https://instagram.com/ccnycitycollege/
    180814,Bellevue University,http://www.bellevue.edu/,http://www.bellevue.edu/admissions-tuition/admission-requirements/bachelor-admissions,https://twitter.com/BellevueU,https://www.facebook.com/BellevueUniversity,https://instagram.com/bellevueuniversity/
    149772,Western Illinois University,http://www.wiu.edu/admissions/,http://www.wiu.edu/admissions/,https://twitter.com/WIUAdmissions,https://www.facebook.com/WIUAdmissions,http://instagram.com/wiuadmissions
    178420,University of Missouri-St Louis,https://www.umsl.edu/,https://www.umsl.edu/admissions/,https://www.twitter.com/umsl,https://www.facebook.com/UMSL.edu,https://www.instagram.com/umsl/
    193654,The New School,https://www.newschool.edu/,https://www.newschool.edu/admission/,https://twitter.com/thenewschool,https://www.facebook.com/thenewschool,https://www.instagram.com/TheNewSchool/
    156125,Wichita State University,https://www.wichita.edu/,https://www.wichita.edu/admissions/,https://twitter.com/wichitastate/,https://www.facebook.com/wichita.state/,https://www.instagram.com/wichita_state_u/
    186399,Rutgers University-Newark,https://www.rutgers.edu/,https://admissions.newark.rutgers.edu/,https://twitter.com/RutgersU,//www.facebook.com/RutgersU?utm_source=uwide_links&utm_medium=web&utm_campaign=uwide_links,//instagram.com/RutgersU?utm_source=uwide_links&utm_medium=web&utm_campaign=uwide_links
    183026,Southern New Hampshire University,https://www.snhu.edu/,https://www.snhu.edu/admission,No Twitter found,No Facebook found,No Instagram found
    206941,University of Central Oklahoma,https://www.uco.edu/,https://www.uco.edu/admissions-aid/apply-now/,https://twitter.com/UCOBronchos,https://www.facebook.com/uco.bronchos/,http://www.instagram.com/ucobronchos
    177968,Lindenwood University,http://www.lindenwood.edu/,http://www.lindenwood.edu/admissions/,https://twitter.com/lindenwoodu,https://www.facebook.com/LindenwoodUniversity,https://www.instagram.com/lindenwooduniversity
    122612,University of San Francisco,https://www.usfca.edu/,https://www.usfca.edu/admission/undergraduate,https://twitter.com/usfca,https://www.facebook.com/University.of.San.Francisco,http://instagram.com/usfca
    190600,CUNY John Jay College of Criminal Justice,http://www.jjay.cuny.edu/,https://www.jjay.cuny.edu/admissions,https://www.twitter.com/JohnJayResearch,https://www.facebook.com/johnjaycollege,https://instagram.com/johnjaycollege
    240727,University of Wyoming,http://www.uwyo.edu/,http://www.uwyo.edu/admissions/,https://twitter.com/uwyonews,https://www.facebook.com/uwpride,https://www.instagram.com/uofwyoming/
    102094,University of South Alabama,http://www.southalabama.edu/,http://www.southalabama.edu/departments/admissions/,https://twitter.com/UofSouthAlabama,https://www.facebook.com/theuniversityofsouthalabama,http://instagram.com/uofsouthalabama
    157447,Northern Kentucky University,https://www.nku.edu/,https://nku.edu/admissions.html,https://twitter.com/nkuedu,https://www.facebook.com/nkuedu/?fref=ts,https://www.instagram.com/nkuedu/?hl=en
    174914,University of St Thomas,https://www.stthomas.edu/,https://www.stthomas.edu/admissions/,//twitter.com/UofStThomasMN/,//www.facebook.com/UofStThomasMN/,//instagram.com/uofstthomasmn#
    212106,Duquesne University,https://www.duq.edu/,https://www.duq.edu/admissions-and-aid/undergraduate,http://twitter.com/duqedu,http://www.facebook.com/duquesneuniversityoftheholyspirit,http://www.instagram.com/duquesneuniversity
    184782,Rowan University,https://www.rowan.edu/home/,https://www.rowan.edu/home/admissions-aid,https://twitter.com/RowanUniversity,https://www.facebook.com/RowanUniversity,https://www.instagram.com/p/BpZdJIinlzT/
    160658,University of Louisiana at Lafayette,https://louisiana.edu/,https://louisiana.edu/admissions,https://twitter.com/ULLafayette,https://www.facebook.com/officialullafayette,https://www.instagram.com/p/BpZc5GmngcU/
    200332,North Dakota State University-Main Campus,https://www.ndsu.edu/,https://www.ndsu.edu/admission,https://twitter.com/ndsu/,https://www.facebook.com/ndsu.fargo,https://www.instagram.com/ndsu_official/
    201645,Case Western Reserve University,https://case.edu/,https://case.edu/admission/,https://twitter.com/cwru,https://www.facebook.com/casewesternreserve,https://instagram.com/cwru/
    202480,University of Dayton,https://udayton.edu/,https://udayton.edu/apply/undergraduate/index.php,https://twitter.com/univofdayton,https://www.facebook.com/univofdayton/,https://www.instagram.com/universityofdayton/
    200280,University of North Dakota,https://und.edu/,https://und.edu/admissions/,https://twitter.com/UofNorthDakota,https://www.facebook.com/UofNorthDakota,https://www.instagram.com/p/BpZjlXgAx_D/
    144892,Eastern Illinois University,https://www.eiu.edu/,https://www.eiu.edu/admissions.php,http://www.twitter.com/eiu,https://www.facebook.com/iameiu/posts/10154697624831857,https://www.instagram.com/p/BCEob7bP7XN/
    127741,University of Northern Colorado,https://www.unco.edu/,https://www.unco.edu/admissions/,https://twitter.com/UNC_Colorado,http://www.facebook.com/home.php#!/pages/University-of-Northern-Colorado/116847458387226,http://instagram.com/UNC_Colorado
    235097,Eastern Washington University,https://www.ewu.edu/,https://www.ewu.edu/apply/,https://twitter.com/ewueagles,https://www.facebook.com/ewueagles/,http://instagram.com/easternwashingtonuniversity
    231624,College of William and Mary,https://www.wm.edu/,https://www.wm.edu/admission/undergraduateadmission/index.php,https://twitter.com/williamandmary,https://www.facebook.com/williamandmary,https://instagram.com/william_and_mary
    109785,Azusa Pacific University,https://www.apu.edu/,https://www.apu.edu/admissions/,http://twitter.com/azusapacific,https://www.facebook.com/azusapacific,https://www.instagram.com/azusapacific/
    196130,Buffalo State SUNY,https://suny.buffalostate.edu/,https://admissions.buffalostate.edu/new-york-city,http://twitter.com/buffalostate,https://www.facebook.com/BuffaloStateCollege,http://instagram.com/buffalostate
    138354,The University of West Florida,https://uwf.edu/,https://uwf.edu/admissions/,https://twitter.com/UWF,https://www.facebook.com/WestFL,https://instagram.com/uwf
    141264,Valdosta State University,https://www.valdosta.edu/,https://www.valdosta.edu/admissions/undergraduate/,https://twitter.com/valdostastate,https://www.facebook.com/valdostastate/,https://www.instagram.com/valdostastate/
    117946,Loyola Marymount University,https://www.lmu.edu/,https://admission.lmu.edu/,http://twitter.com/loyolamarymount,https://www.facebook.com/lmula,https://instagram.com/loyolamarymount/?hl=en
    178721,Park University,https://www.park.edu/,https://www.park.edu/admissions/,https://twitter.com/parkuniversity,https://facebook.com/profile.php?id=119693232348,https://www.instagram.com/parkuniversity
    122436,University of San Diego,http://www.sandiego.edu/,https://www.sandiego.edu/admissions/,http://twitter.com/uofsandiego,http://www.facebook.com/usandiego,http://instagram.com/uofsandiego
    217156,Brown University,https://www.brown.edu/,https://www.brown.edu/admission,https://twitter.com/BrownUniversity,https://www.facebook.com/BrownUniversity,https://instagram.com/brownu/
    176965,University of Central Missouri,https://www.ucmo.edu/,https://www.ucmo.edu/future-students/admissions/undergraduate-admissions/index.php,https://www.twitter.com/ucentralmo/,https://www.facebook.com/UCentralMO/,https://www.instagram.com/ucentralmo/
    228431,Stephen F Austin State University,http://www.sfasu.edu/,http://www.sfasu.edu/admissions-and-aid,https://twitter.com/SFASU,https://www.facebook.com/sfasu/,https://www.instagram.com/sfa_jacks/
    237525,Marshall University,http://www.marshall.edu/home/index.html,https://www.marshall.edu/admissions/,//www.marshall.edu/connected/twitter-directory/,https://www.facebook.com/marshallu,//www.marshall.edu/connected/instagram-directory/
    217819,College of Charleston,http://www.cofc.edu/,http://admissions.cofc.edu/,https://twitter.com/CofC,https://www.facebook.com/collegeofcharleston,https://www.instagram.com/collegeofcharleston/
    165024,Bridgewater State University,https://www.bridgew.edu/,https://www.bridgew.edu/admissions-aid,https://twitter.com/BridgeStateU,http://www.facebook.com/pages/Bridgewater-State-University/110180449037531,https://www.instagram.com/bridgestateu/
    240189,University of Wisconsin-Whitewater,http://www.uww.edu/,http://www.uww.edu/admissions,https://twitter.com/uwwhitewater,https://www.facebook.com/uwwhitewater,https://www.instagram.com/uwwhitewater/
    121150,Pepperdine University,https://www.pepperdine.edu/,https://www.pepperdine.edu/admission/,https://www.twitter.com/pepperdine,https://www.facebook.com/pepperdine,http://instagram.com/pepperdine
    122931,Santa Clara University,https://www.scu.edu/,https://www.scu.edu/admission/,https://twitter.com/SantaClaraUniv,https://www.facebook.com/SantaClaraUniversity,https://instagram.com/santaclarauniversity/
    168005,Suffolk University,https://www.suffolk.edu/,https://www.suffolk.edu/ugadmission/index.php,https://twitter.com/Suffolk_U,https://www.facebook.com/suffolkuniversity,http://instagram.com/suffolk_U/
    186584,Seton Hall University,https://www.shu.edu/,http://www.shu.edu/undergraduate-admissions/,http://www.twitter.com/shuathletics,http://www.facebook.com/shuathletics,http://www.instagram.com/shuathletics
    106467,Arkansas Tech University,https://www.atu.edu/,https://www.atu.edu/admissions/,https://twitter.com/arkansastech,https://www.facebook.com/arkansastech,https://instagram.com/arkansastech
    128771,Central Connecticut State University,http://www.ccsu.edu/,http://www2.ccsu.edu/admission/,https://twitter.com/CCSU,https://www.facebook.com/CentralConnecticutStateUniversity,https://instagram.com/ccsu_official/
    140951,Savannah College of Art and Design,https://www.scad.edu/,https://www.scad.edu/admission,https://twitter.com/SCADdotedu,https://www.facebook.com/scad.edu,https://www.instagram.com/scaddotedu/
    228875,Texas Christian University,http://www.tcu.edu/,https://admissions.tcu.edu/,http://www.twitter.com/tcu,http://www.facebook.com/pages/Fort-Worth-TX/TCU-Texas-Christian-University/155151195064,https://www.instagram.com/texaschristianuniversity/
    145725,Illinois Institute of Technology,https://web.iit.edu/,https://admissions.iit.edu/undergraduate,https://twitter.com/IITUGadmission,https://www.facebook.com/TalonTheScarletHawk/,https://instagram.com/IllinoisTech/
    180461,Montana State University,http://www.montana.edu/,http://www.montana.edu/admissions/,https://twitter.com/montanastate,https://www.facebook.com/montanastate,https://www.instagram.com/p/BfTj5M7h9y4/
    130493,Southern Connecticut State University,https://www.southernct.edu/,https://www.southernct.edu/admissions/,https://twitter.com/scsu,https://www.facebook.com/southernct,https://instagram.com/scsugram
    219356,South Dakota State University,https://www.sdstate.edu/,https://www.sdstate.edu/admissions,http://twitter.com/sdstate,http://www.facebook.com/SouthDakotaStateUniversity,http://instagram.com/sdstatepics
    215770,Saint Joseph's University,https://www.sju.edu/,https://www.sju.edu/admission/undergraduate/apply-undergraduate-admission,http://twitter.com/saintjosephs,http://www.facebook.com/saintjosephsuniversity,No Instagram found
    190637,CUNY Lehman College,http://www.lehman.cuny.edu/,http://www.lehman.edu/admissions/,https://twitter.com/@LehmanCollege,https://www.facebook.com/pages/Lehman-College/186133461418370,https://www.instagram.com/lehmancuny/
    132471,Barry University,https://www.barry.edu/,https://www.barry.edu/future-students/undergraduate/admissions/,https://twitter.com/BarryUniversity,https://www.facebook.com/barryuniversity,http://instagram.com/barryuniversity
    174233,University of Minnesota-Duluth,http://www.d.umn.edu/,http://d.umn.edu/undergraduate-admissions,https://twitter.com/umnduluth,https://www.facebook.com/UMNDuluth,https://www.instagram.com/p/BpUXu5LFf97/
    188429,Adelphi University,https://www.adelphi.edu/,https://admissions.adelphi.edu/,http://twitter.com/AdelphiU,http://www.facebook.com/AdelphiU,http://instagram.com/adelphiu
    211361,California University of Pennsylvania,https://www.calu.edu/,https://www.calu.edu/admissions/index.aspx,https://twitter.com/CalUofPA,https://www.facebook.com/CalUofPA/,https://www.instagram.com/caluofpa/
    108232,Academy of Art University,https://www.academyart.edu/,https://www.academyart.edu/admissions/,https://twitter.com/academy_of_art/,https://www.facebook.com/AcademyofArtUniversity,https://www.instagram.com/academy_of_art/
    102553,University of Alaska Anchorage,https://www.uaa.alaska.edu/,https://www.uaa.alaska.edu/admissions/,//twitter.com/uaanchorage,//www.facebook.com/UAAnchorage,//instagram.com/uaaphotos/
    160612,Southeastern Louisiana University,https://www.southeastern.edu/,https://www.southeastern.edu/admin/admissions/index.html,https://twitter.com/oursoutheastern,https://www.facebook.com/southeastern,http://instagram.com/oursoutheastern
    240268,University of Wisconsin-Eau Claire,https://www.uwec.edu/,https://www.uwec.edu/admissions/,http://www.twitter.com/uweauclaire,http://www.facebook.com/uweauclaire/,http://www.instagram.com/uweauclaire/
    228705,Texas A & M University-Kingsville,http://www.tamuk.edu/,http://www.tamuk.edu/admission/,https://twitter.com/JavelinaNation,http://www.facebook.com/home.php?#!/javelinas?ref=ts,https://www.instagram.com/javelinanation/?hl=en
    240365,University of Wisconsin-Oshkosh,https://uwosh.edu/,https://uwosh.edu/admissions/,https://twitter.com/uwoshkosh,https://www.facebook.com/uwoshkosh,http://instagram.com/uwoshkosh
    225432,University of Houston-Downtown,https://www.uhd.edu/Pages/home.aspx,https://www.uhd.edu/admissions/Pages/admissions-index.aspx,https://twitter.com/UHDowntown,https://www.facebook.com/uhdowntown/,https://www.instagram.com/uhdofficial/
    193016,Mercy College,https://www.mercy.edu/,https://www.mercy.edu/admissions/,https://twitter.com/mercycollege,https://www.facebook.com/mercycollegeny,https://www.instagram.com/mercycollege/
    240329,University of Wisconsin-La Crosse,https://www.uwlax.edu/,https://www.uwlax.edu/admissions/,https://twitter.com/uwlacrosse,https://www.facebook.com/UWLaCrosse,https://instagram.com/uwlax/
    130226,Quinnipiac University,https://www.qu.edu/,https://www.qu.edu/admissions.html,http://twitter.com/quinnipiacu,https://www.facebook.com/QuinnipiacUniversity/,https://instagram.com/quinnipiacu
    235316,Gonzaga University,https://www.gonzaga.edu/,https://www.gonzaga.edu/,https://twitter.com/Gonzaga_Prez,https://www.facebook.com/GonzagaUniversity/,https://www.instagram.com/p/BpVJVwUn3ic/
    196121,SUNY College at Brockport,https://www.brockport.edu/,https://www.brockport.edu/admissions/,//www.twitter.com/brockport,https://brockport.edu/about/newsbureau/2258.html?utm_content=new_dorm&utm_medium=social&utm_term=Public+Relations&utm_source=facebook&utm_campaign=College+at+Brockport,//www.instagram.com/brockport
    106245,University of Arkansas at Little Rock,https://ualr.edu/,https://ualr.edu/admissions/,http://twitter.com/ualr,https://www.facebook.com/UALittleRock/,http://instagram.com/ualr
    142276,Idaho State University,https://www.isu.edu/,https://www.isu.edu/future/,https://twitter.com/IdahoStateU,https://www.facebook.com/idahostateu,https://instagram.com/idahostateu/
    236595,Seattle University,https://www.seattleu.edu/,https://www.washington.edu/admissions/,https://twitter.com/seattleu,https://www.facebook.com/seattleu,http://instagram.com/seattleu
    117140,University of La Verne,https://laverne.edu/,https://laverne.edu/admission/,https://twitter.com/ULaVerne/,https://www.facebook.com/ULaVerne/,https://www.instagram.com/p/BonOlCKAYXe/
    140447,Mercer University,https://www.mercer.edu/,https://admissions.mercer.edu/,http://twitter.com/MercerYou/,http://www.facebook.com/MercerUniversity/,https://www.instagram.com/p/BpXZwhAHERO/
    200004,Western Carolina University,https://www.wcu.edu/,https://www.wcu.edu/apply/undergraduate-admissions/contact-admission.aspx,https://twitter.com/wcu,https://www.facebook.com/WesternCarolinaUniversity/,https://www.instagram.com/western_carolina/
    216038,Slippery Rock University of Pennsylvania,https://www.sru.edu/,https://www.sru.edu/admissions,https://www.twitter.com/sruofpa,https://www.facebook.com/slipperyrockuniversity,http://instagram.com/slipperyrockuniversity/
    228529,Tarleton State University,https://www.tarleton.edu/,https://www.tarleton.edu/admissions/index.html,No Twitter found,No Facebook found,No Instagram found
    229780,Wayland Baptist University,https://www.wbu.edu/,https://www.wbu.edu/admissions/contact-us.htm,https://twitter.com/waylandbaptist,https://www.facebook.com/WaylandBaptistUniversity,https://www.instagram.com/p/Bpc2YgaHrfm/
    196176,State University of New York at New Paltz,https://www.newpaltz.edu/,https://www.newpaltz.edu/,http://www.twitter.com/newpaltz,http://www.facebook.com/newpaltz,http://instagram.com/sunynewpaltz
    187444,William Paterson University of New Jersey,https://www.wpunj.edu/,https://www.wpunj.edu/admissions/,https://twitter.com/WPUNJ_EDU,https://www.facebook.com/mywpu,http://instagram.com/wpunj
    199847,Wake Forest University,https://www.wfu.edu/,https://admissions.wfu.edu/,https://twitter.com/WakeForest,https://www.facebook.com/wfuniversity,https://www.instagram.com/wfuniversity/
    186867,Stevens Institute of Technology,https://www.stevens.edu/,https://www.stevens.edu/admissions/undergraduate-admissions,https://twitter.com/FollowStevens,https://www.facebook.com/Stevens1870,https://www.instagram.com/p/BoDNaVBFxRo
    190558,College of Staten Island CUNY,https://www.csi.cuny.edu/,https://www.csi.cuny.edu/admissions,http://twitter.com/share?text=CSI+St.+George&url=/about-csi/csi-glance/other-locations/csi-st-george,http://www.facebook.com/share.php?u=http://www.csistudenthousing.com/student-apartments/ny/staten-island/dolphin-cove&title=Luxury+Student+Housing%3A+Any+Closer+and+You%E2%80%99d+be+in+Class,https://www.instagram.com/collegeofstatenisland/
    133650,Florida Agricultural and Mechanical University,http://www.famu.edu/,http://admissions.famu.edu/,No Twitter found,https://www.facebook.com/FAMU1887,http://instagram.com/famu_1887
    163851,Salisbury University,https://www.salisbury.edu/,https://www.salisbury.edu/admissions/,https://www.twitter.com/salisburyu,https://www.facebook.com/SalisburyU/,No Instagram found
    181002,Creighton University,https://www.creighton.edu/,https://www.creighton.edu/admissions,https://twitter.com/creighton,https://www.facebook.com/creightonuniversity,https://www.instagram.com/creighton1878/
    221847,Tennessee Technological University,https://www.tntech.edu/,https://www.tntech.edu/admissions/,https://twitter.com/tennesseetech,https://www.facebook.com/tennesseetech,https://instagram.com/tntechuniversity/
    366711,California State University-San Marcos,https://www.csusm.edu/,https://www.csusm.edu/admissions/contact-us/index.html,http://twitter.com/csusm,http://facebook.com/csusm,http://instagram.com/csusm
    151324,Indiana State University,https://www.indstate.edu/,https://www.indstate.edu/admissions,https://twitter.com/indianastate,https://www.facebook.com/IndianaState,https://www.instagram.com/p/BpZmu2QHaRp/
    144281,Columbia College-Chicago,https://www.colum.edu/,https://www.colum.edu/admissions/,https://twitter.com/ColumAlum,https://www.facebook.com/columadmit,https://instagram.com/columadmit
    151306,University of Southern Indiana,https://www.usi.edu/,https://www.usi.edu/admissions/,https://twitter.com/USIedu,https://www.facebook.com/USIedu,https://www.instagram.com/usiedu/
    141334,University of West Georgia,https://www.westga.edu/,https://www.westga.edu/admissions/,https://twitter.com/uwgspecial?lang=en,https://www.facebook.com/groups/136625026403394/,https://www.instagram.com/uwestga/
    233277,Radford University,https://www.radford.edu/,https://www.radford.edu/content/radfordcore/home/admissions.html,No Twitter found,No Facebook found,No Instagram found
    147776,Northeastern Illinois University,https://www.neiu.edu/,https://admissions.neiu.edu/,http://www.neiu.edu/twitter,https://www.facebook.com/pg/NEIUlife/events/?ref=page_internal,https://www.instagram.com/neiulife/
    178411,Missouri University of Science and Technology,http://www.mst.edu/,https://futurestudents.mst.edu/admissions/,#twitter,https://www.facebook.com/MissouriSandT,https://www.instagram.com/missourisandt/?hl=en
    165015,Brandeis University,https://www.brandeis.edu/,http://www.brandeis.edu/admissions/,https://twitter.com/BrandeisU,https://www.facebook.com/brandeisuniversity,https://www.instagram.com/brandeiswomensrugby
    174020,Metropolitan State University,https://www.metrostate.edu/,https://www.metrostate.edu/about/departments/admissions,https://twitter.com/choose_metro,https://www.facebook.com/ChooseMetroState,https://www.instagram.com/metrostate/
    221740,The University of Tennessee-Chattanooga,https://www.utc.edu/,https://www.utc.edu/about/admissions.php,https://www.twitter.com/UTChattanooga,https://www.facebook.com/UTChattanooga,https://instagram.com/utchattanooga
    211158,Bloomsburg University of Pennsylvania,http://www.bloomu.edu/,https://www.bloomu.edu/admissions,No Twitter found,No Facebook found,No Instagram found
    161253,University of Maine,https://umaine.edu/,https://go.umaine.edu/,https://twitter.com/UMaine,https://www.facebook.com/UniversityofMaine,https://instagram.com/university.of.maine/
    219471,University of South Dakota,https://www.usd.edu/,https://www.usd.edu/admissions,https://twitter.com/usd,https://www.facebook.com/UniversityofSouthDakota,No Instagram found
    206695,Youngstown State University,https://ysu.edu/,https://ysu.edu/admissions,https://twitter.com/youngstownstate,https://facebook.com/youngstownstate,https://instagram.com/youngstownstate/
    185828,New Jersey Institute of Technology,https://www.njit.edu/,https://www.njit.edu/admissions,https://twitter.com/NJIT,#eb0cf68e34319-tab-facebook,No Instagram found
    169479,Davenport University,https://www.davenport.edu/,https://www.davenport.edu/admissions,http://www.twitter.com/davenportu,http://www.facebook.com/DavenportU,https://instagram.com/davenportuniversity
    186131,Princeton University,https://www.princeton.edu/,https://admission.princeton.edu/,https://www.twitter.com/Princeton,https://facebook.com/profile.php?id=18058830773,https://instagram.com/princeton_university/
    131520,Howard University,https://www2.howard.edu/,https://www2.howard.edu/admission,https://twitter.com/HowardU,https://www.facebook.com/HowardU/?fref=ts,https://www.instagram.com/howard1867/
    157401,Murray State University,https://www.murraystate.edu/,https://www.murraystate.edu/admissions/,http://twitter.com/murraystateuniv,http://www.facebook.com/murraystateuniv,http://instagram.com/murraystateuniv
    159939,University of New Orleans,http://new.uno.edu/,http://new.uno.edu/admissions,https://twitter.com/UofNO,https://www.facebook.com/UniversityOfNewOrleans,https://www.instagram.com/uofno/
    240480,University of Wisconsin-Stevens Point,https://www.uwsp.edu/Pages/default.aspx,https://www.uwsp.edu/admissions/Pages/default.aspx,https://twitter.com/UWStevensPoint,https://www.facebook.com/UWStevensPoint,http://instagram.com/uw_stevens_point
    166452,Lesley University,https://lesley.edu/,https://lesley.edu/admissions-aid,https://twitter.com/lesley_u,https://www.facebook.com/LesleyUniversity/,https://www.instagram.com/lesleyuniversity/
    145619,Benedictine University,http://www.ben.edu/,https://www.ben.edu/admissions/,No Twitter found,http://www.facebook.com/BenedictineUniversity,http://instagram.com/benu1887
    186876,The Richard Stockton College of New Jersey,https://stockton.edu/,https://stockton.edu/admissions/,https://twitter.com/Stockton_edu,https://www.facebook.com/StocktonUniversity,https://www.instagram.com/stocktonuniversity
    111948,Chapman University,https://www.chapman.edu/,https://www.chapman.edu/admission/undergraduate/index.aspx,https://twitter.com/chapmanu,https://www.facebook.com/ChapmanUniversity,https://www.instagram.com/chapmanu/
    196194,SUNY College at Oswego,https://www.oswego.edu/,https://www.oswego.edu/admissions/,http://www.twitter.com/sunyoswego,http://www.facebook.com/sunyoswego,http://www.instagram.com/sunyoswego
    167729,Salem State University,https://www.salemstate.edu/,https://www.salemstate.edu/admissions-and-aid/welcome-undergraduate-admissions,http://www.twitter.com/salemstate,https://www.facebook.com/118242334891805,https://www.instagram.com/salemstate
    182670,Dartmouth College,https://home.dartmouth.edu/,https://admissions.dartmouth.edu/,https://www.twitter.com/dartmouth,https://www.facebook.com/Dartmouth,https://www.instagram.com/dartmouthcollege/
    194091,New York Institute of Technology,https://www.nyit.edu/,https://www.nyit.edu/admissions,http://www.twitter.com/nyit/,http://www.facebook.com/mynyit,https://www.instagram.com/nyit_photos/
    106704,University of Central Arkansas,https://uca.edu/,https://uca.edu/admissions/,http://twitter.com/ucabears,http://facebook.com/ucentralarkansas,http://instagram.com/ucabears
    164739,Bentley University,https://www.bentley.edu/,https://www.bentley.edu/admission,http://www.twitter.com/bentleyu,http://www.facebook.com/bentleyuniversity,No Instagram found
    179557,Southeast Missouri State University,https://semo.edu/,https://semo.edu/admissions/,https://www.semo.edu/twitter/,https://www.semo.edu/facebook/,https://www.semo.edu/instagram/
    151102,Indiana University-Purdue University-Fort Wayne,https://www.pfw.edu/,https://www.collegesimply.com/colleges/indiana/indiana-university-purdue-university-fort-wayne/admission/,/twitter,/facebook,/instagram
    213349,Kutztown University of Pennsylvania,https://www.kutztown.edu/,https://www3.kutztown.edu/admissions/good.html,http://twitter.com/KutztownU,http://www.facebook.com/KutztownU,http://instagram.com/kutztownu
    161554,University of Southern Maine,https://usm.maine.edu/,https://usm.maine.edu/admissions,http://twitter.com/USouthernMaine,http://www.facebook.com/USouthernMaine,http://instagram.com/usouthernmaine/
    219602,Austin Peay State University,https://www.apsu.edu/,https://www.apsu.edu/admissions/,https://twitter.com/austinpeay,https://www.facebook.com/austinpeay,https://www.instagram.com/p/BpcWm8WnWhw/
    213543,Lehigh University,https://www1.lehigh.edu/,https://www1.lehigh.edu/admissions/undergrad,http://www.lehigh.edu/twitter,http://www.lehigh.edu/facebook,http://www.lehigh.edu/instagram
    Error: 154493 - Upper Iowa University
    123572,Sonoma State University,https://www.sonoma.edu/,https://admissions.sonoma.edu/,http://twitter.com/JudySakaki,https://www.facebook.com/sonomastateuniversity,https://www.instagram.com/p/BMFFQTOhH-V/
    185572,Monmouth University,https://www.monmouth.edu/,https://www.monmouth.edu/admission/,https://twitter.com/hashtag/ThisIsMonmouth?src=hash,https://www.facebook.com/monmouthuniversity,https://www.instagram.com/p/BpZvwgcAnxW
    227757,Rice University,https://www.rice.edu/,https://www.rice.edu/admission-aid.shtml,https://twitter.com/ricealumni,https://www.facebook.com/ricealumni,https://instagram.com/riceuniversity
    196149,SUNY College at Cortland,http://www2.cortland.edu/home/,http://www2.cortland.edu/home/,https://twitter.com/suny_cortland,https://www.facebook.com/sunycortland,https://instagram.com/sunycortland/
    147536,National Louis University,https://www.nl.edu/,https://www.nl.edu/admissions/,https://twitter.com/NationalLouisU,https://www.facebook.com/NationalLouis?fref=ts,https://www.instagram.com/nationallouisu/
    110495,California State University-Stanislaus,https://www.csustan.edu/,https://www.csustan.edu/admissions,https://twitter.com/stan_state,https://www.facebook.com/stanstate,http://instagram.com/stanstate
    240417,University of Wisconsin-Stout,https://www.uwstout.edu/,https://www.uwstout.edu/directory/admissions-office,https://twitter.com/UWStout,https://www.facebook.com/uwstout,https://www.instagram.com/uwstoutpics/
    152248,Purdue University-Calumet Campus,https://www.pnw.edu/,https://admissions.pnw.edu/undergraduate/contact-us/,https://twitter.com/PurdueNorthwest,https://www.facebook.com/purduenorthwest/,https://www.instagram.com/purduenorthwest/
    175272,Winona State University,https://www.winona.edu/,https://www.winona.edu/admissions/,https://twitter.com/winonastateu,https://www.facebook.com/WinonaStateU,https://www.instagram.com/WinonaStateU
    187134,The College of New Jersey,https://tcnj.pages.tcnj.edu/,https://admissions.tcnj.edu/,https://twitter.com/tcnj,https://www.facebook.com/tcnjlions,https://instagram.com/tcnj_official
    129941,University of New Haven,https://www.newhaven.edu/,https://www.newhaven.edu/admissions/undergraduate/,https://twitter.com/unewhaven,https://www.facebook.com/unewhaven/,https://www.instagram.com/unewhaven/
    206622,Xavier University,https://www.xavier.edu/,https://www.xavier.edu/admission/,https://www.twitter.com/XavierUniv,https://facebook.com/profile.php?id=5826202566,https://www.instagram.com/xavieruniversity
    191968,Ithaca College,https://www.ithaca.edu/,https://www.ithaca.edu/admission,https://twitter.com/IthacaCollege/,https://www.facebook.com/ithacacollege/,https://www.instagram.com/ithacacollege/
    216010,Shippensburg University of Pennsylvania,https://www.ship.edu/,https://www.ship.edu/admissions/,http://twitter.com/shippensburgU,http://www.facebook.com/ShippensburgUniversity,https://www.instagram.com/shippensburguniv/
    192439,LIU Brooklyn,http://www.liu.edu/Brooklyn.aspx,http://www.liu.edu/Brooklyn/Admissions,No Twitter found,https://www.facebook.com/LIUBrooklyn/,https://www.instagram.com/LIUBrooklyn/
    120883,University of the Pacific,https://www.pacific.edu/,https://www.pacific.edu/admission.html,http://twitter.com/#!/UOPacific,http://www.facebook.com/pages/University-of-the-Pacific/28972094743,No Instagram found
    224147,Texas A & M University-Corpus Christi,https://www.tamucc.edu/,http://admissions.tamucc.edu/,http://www.twitter.com/IslandCampus/?utm_source=footer&utm_campaign=tamucc.edu&utm_medium=twitter_logo,https://www.facebook.com/events/2419114411648601/,http://instagram.com/island_university/?utm_source=footer&utm_campaign=tamucc.edu&utm_medium=instagram_logo
    126580,University of Colorado Colorado Springs,https://www.uccs.edu/,https://www.uccs.edu/admissionsenrollment/,https://twitter.com/uccs,https://www.facebook.com/UCCS-243117181362/,https://www.instagram.com/uccs/
    159647,Louisiana Tech University,https://www.latech.edu/,https://www.latech.edu/admissions/,http://twitter.com/latech,http://www.facebook.com/latech,https://www.instagram.com/louisianatech/
    110486,California State University-Bakersfield,http://www.csub.edu/,http://www.csub.edu/admissions/,https://twitter.com/csubakersfield,http://www.facebook.com/csubakersfield,https://www.instagram.com/csubakersfield/
    202806,Franklin University,https://www.franklin.edu/,https://www.franklin.edu/admissions/undergraduate-students,//twitter.com/FranklinU,//www.facebook.com/FranklinUniversity,//www.instagram.com/franklinuniv/
    196246,SUNY College at Plattsburgh,https://www.plattsburgh.edu/,https://www.plattsburgh.edu/admissions/,https://twitter.com/sunyplattsburgh,https://www.facebook.com/sunyplattsburgh,https://www.instagram.com/sunyplattsburgh/
    172051,Saginaw Valley State University,https://www.svsu.edu/,https://www.svsu.edu/admissions/undergraduate/,https://twitter.com/SVSU,https://www.facebook.com/svsu.edu,http://instagram.com/svsucardinals
    171456,Northern Michigan University,https://www.nmu.edu/programs,https://www.nmu.edu/admissions/home,http://twitter.com/NorthernMichU,http://www.facebook.com/NorthernMichiganU,http://instagram.com/northernmichiganu?ref=badge
    207263,Northeastern State University,https://www.nsuok.edu/,https://offices.nsuok.edu/admissions/AdmissionsHome.aspx,http://twitter.com/NSURiverhawks,https://www.facebook.com/NSURiverHawks,http://instagram.com/NSURiverhawks
    194824,Rensselaer Polytechnic Institute,https://www.rpi.edu/,https://admissions.rpi.edu/undergraduate,No Twitter found,No Facebook found,No Instagram found
    160038,Northwestern State University of Louisiana,https://www.nsula.edu/,https://www.nsula.edu/admissions/,https://twitter.com/nsula,https://www.facebook.com/NorthwesternState,http://instagram.com/nsula
    213367,La Salle University,https://www.lasalle.edu/,https://www.lasalle.edu/admission/,https://www.twitter.com/lasalleuniv,https://www.facebook.com/lasalleuniversity,https://www.instagram.com/lasalleuniv
    167987,University of Massachusetts-Dartmouth,https://www.umassd.edu/,https://www.umassd.edu/admissions/,https://twitter.com/umassd,https://www.facebook.com/umassd,https://www.instagram.com/umassd/
    184603,Fairleigh Dickinson University-Metropolitan Campus,https://www.fdu.edu/,https://view2.fdu.edu/admissions/,https://twitter.com/FDUWhatsNew,https://www.facebook.com/fairleighdickinsonuniversity,https://instagram.com/fduwhatsnew#
    148487,Roosevelt University,https://www.roosevelt.edu/,https://www.roosevelt.edu/admission,https://twitter.com/RooseveltU,https://www.facebook.com/RooseveltUniversity,https://www.instagram.com/RooseveltU/
    225627,University of the Incarnate Word,http://www.uiw.edu/,http://www.uiw.edu/admissions/,https://twitter.com/uiw_admissions,https://www.facebook.com/uiwartsfestival/,https://www.instagram.com/p/BpaCpSXH4m3/
    199102,North Carolina A & T State University,https://www.ncat.edu/,https://www.ncat.edu/admissions/undergraduate/index.html,http://twitter.com/ncatsuaggies,http://www.facebook.com/ncatsuaggies,https://www.instagram.com/ncatsuaggies/?hl=en
    101480,Jacksonville State University,http://www.jsu.edu/,http://www.jsu.edu/admissions/,https://twitter.com/JSUNews,https://www.facebook.com/JacksonvilleStateUniversity,https://www.instagram.com/jacksonvillestateuniversity/
    215929,University of Scranton,http://www.scranton.edu/,http://www.scranton.edu/admissions/index.shtml,No Twitter found,http://www.facebook.com/scrantonadmissions,No Instagram found
    146612,Lewis University,https://www.lewisu.edu/,https://www.lewisu.edu/admissions/application.htm,https://twitter.com/intent/follow?source=followbutton&variant=1.0&screen_name=lewisuniversity,https://www.facebook.com/lewisu.edu/app_190322544333196,https://www.instagram.com/lewisuniversity/
    129525,University of Hartford,https://www.hartford.edu/,http://www.hartford.edu/admission/,https://www.twitter.com/UofHartford,https://www.facebook.com/116079461746802/posts/2069766509711411,https://www.instagram.com/p/BpKbPVgBIkM/
    Error: 157386 - Morehead State University
    174817,Saint Mary's University of Minnesota,https://www.smumn.edu/,https://www.smumn.edu/admission,http://www.twitter.com/smumn,http://www.facebook.com/smumn,http://www.instagram.com/smumn
    131283,Catholic University of America,https://www.catholic.edu/index.html,https://www.catholic.edu/admission/index.html,http://twitter.com/CatholicUniv,http://www.facebook.com/CatholicUniversity,http://instagram.com/catholicuniversity
    214041,Millersville University of Pennsylvania,https://www.millersville.edu/,https://www.millersville.edu/admissions/,https://twitter.com/millersvilleu/,https://www.facebook.com/millersvilleu/,https://www.instagram.com/millersvilleu/
    230603,Southern Utah University,https://www.suu.edu/,https://www.suu.edu/apply.html,https://www.twitter.com/suutbirds,https://www.facebook.com/SUUTbirds,https://www.instagram.com/suutbirds
    115755,Humboldt State University,https://www.humboldt.edu/,https://admissions.humboldt.edu/,https://twitter.com/humboldtstate/,https://www.facebook.com/humboldtstate,https://www.instagram.com/livefromhsu/
    238616,Concordia University-Wisconsin,https://www.cuw.edu/,https://www.cuw.edu/admissions/,https://twitter.com/CUWisconsin,https://www.facebook.com/CUWisconsin,https://www.instagram.com/cuwisconsin/
    229814,West Texas A & M University,http://www.wtamu.edu/,http://www.wtamu.edu/admissionshub/default.aspx,http://www.twitter.com/wtamu,http://www.facebook.com/wtamu,http://www.instagram.com/wtamu
    218724,Coastal Carolina University,https://www.coastal.edu/,https://www.coastal.edu/admissions/,https://twitter.com/CCUchanticleers,https://www.facebook.com/CoastalCarolinaUniversity,https://instagram.com/ccuchanticleers/?hl=en
    189705,Canisius College,https://www.canisius.edu/,https://www.canisius.edu/admissions/undergraduate-admissions,http://twitter.com/canisiuscollege,http://facebook.com/canisiuscollege,http://instagram.com/canisius_college/
    228802,The University of Texas at Tyler,https://www.uttyler.edu/,https://www.uttyler.edu/,https://twitter.com/uttyler/,https://www.facebook.com/uttyler/,https://www.instagram.com/uttyler/
    148335,Robert Morris University Illinois,https://robertmorris.edu/,https://robertmorris.edu/admissions/,https://twitter.com/RobertMorrisU,https://www.facebook.com/RMUIllinois,http://www.instagram.com/robertmorrisu
    219709,Belmont University,http://www.belmont.edu/,http://www.belmont.edu/admissions/,https://twitter.com/BelmontUniv,https://www.facebook.com/events/2268925126457280/,https://instagram.com/belmontu/
    139861,Georgia College and State University,http://www.gcsu.edu/,http://www.gcsu.edu/admissions,https://twitter.com/georgiacollege,https://www.facebook.com/GaCollege,https://instagram.com/georgiacollege
    130253,Sacred Heart University,https://www.sacredheart.edu/,https://www.sacredheart.edu/admissions/undergraduateadmissions/,No Twitter found,https://www.facebook.com/SacredHeartUniversity/,No Instagram found
    Error: 211644 - Clarion University of Pennsylvania
    199157,North Carolina Central University,http://www.nccu.edu/,http://www.nccu.edu/admissions/,https://twitter.com/NCCU,http://www.facebook.com/nccueagle,http://instagram.com/nccueagle
    100706,University of Alabama in Huntsville,https://www.uah.edu/,https://www.uah.edu/admissions,https://twitter.com/uahuntsville,https://www.facebook.com/UAHuntsville,https://www.instagram.com/uahuntsville/
    212160,Edinboro University of Pennsylvania,https://www.edinboro.edu/,http://www.edinboro.edu/admissions/,http://www.twitter.com/edinboro,http://www.facebook.com/edinboro,https://instagram.com/edinborou/
    169716,University of Detroit Mercy,https://www.udmercy.edu/,http://udmercy.edu/admission/,http://www.twitter.com/UDMDetroit/,http://www.facebook.com/udmercy,https://www.instagram.com/udmdetroit/
    153269,Drake University,https://www.drake.edu/,https://www.drake.edu/admission/,https://www.twitter.com/DrakeUniversity,https://www.facebook.com/DrakeUniversity,https://www.instagram.com/p/Bk5kKGplaGM/?taken-by=onpaintedstreet
    185129,New Jersey City University,https://www.njcu.edu/,https://www.njcu.edu/admissions/apply,https://twitter.com/InvestorsBank,https://www.facebook.com/NewJerseyCityUniversity,https://www.instagram.com/p/BgWe3AJgJaq/
    163046,Loyola University Maryland,https://www.loyola.edu/,https://www.loyola.edu/admission,http://twitter.com/loyolamaryland,https://www.facebook.com/LoyolaHoundBots/photos/a.1686897701416569/1686898288083177/?type=3&theater,https://www.instagram.com/loyoladining/
    167783,Simmons College,http://www.simmons.edu/,http://www.simmons.edu/admission-and-financial-aid/undergraduate-admission,https://twitter.com/simmonsuniv,https://www.facebook.com/SimmonsUniversity,https://www.instagram.com/simmonsuniversity/
    159717,McNeese State University,https://www.mcneese.edu/,https://www.mcneese.edu/admissions/,https://twitter.com/mcneese,https://www.facebook.com/McNeeseStateU/,https://www.instagram.com/mcneese/
    171128,Michigan Technological University,https://www.mtu.edu/,https://www.mtu.edu/admissions/,https://www.twitter.com/MichiganTech,https://www.facebook.com/MichiganTech,https://instagram.com/michigantech
    137847,The University of Tampa,http://www.ut.edu/,http://www.ut.edu/freshman/,#social-twitter,#social-facebook,#social-instagram
    240277,University of Wisconsin-Green Bay,https://www.uwgb.edu/,https://www.uwgb.edu/admissions/,https://twitter.com/uwgb,https://www.facebook.com/uwgreenbay,https://instagram.com/explore/tags/uwgb/
    221838,Tennessee State University,http://www.tnstate.edu/,http://www.tnstate.edu/admissions/apply.aspx,http://www.tnstate.edu/twitter,http://www.tnstate.edu/facebook ,http://www.tnstate.edu/instagram
    230995,Norwich University,https://www.norwich.edu/,https://www.norwich.edu/admissions,http://twitter.com/norwichnews,http://facebook.com/NorwichUniversity,https://www.instagram.com/norwichuniversity/
    201104,Ashland University,https://www.ashland.edu/,https://www.ashland.edu/admissions/,https://twitter.com/Ashland_Univ,http://www.facebook.com/121733227868116/posts/2343222605719156,https://instagram.com/ashland_university
    155681,Pittsburg State University,https://www.pittstate.edu/,https://admission.pittstate.edu/index.html,https://www.twitter.com/pittstate,https://www.facebook.com/pittstate,https://www.instagram.com/pittsburg_state
    198516,Elon University,https://www.elon.edu/,https://www.elon.edu/e/admissions/undergraduate/,http://www.twitter.com/elonuniversity,http://www.facebook.com/ElonUniversity,http://www.instagram.com/elonuniversity
    168263,Westfield State University,http://www.westfield.ma.edu/,https://www.westfield.ma.edu/admissions,https://twitter.com/westfieldstate,http://www.facebook.com/WestfieldStateUniversity,https://www.instagram.com/westfieldstate/
    186371,Rutgers University-Camden,https://www.camden.rutgers.edu/,https://admissions.camden.rutgers.edu/contact-us,https://twitter.com/Rutgers_Camden,https://www.facebook.com/RutgersCamden,http://instagram.com/rutgers_camden
    217420,Rhode Island College,http://www.ric.edu/Pages/default.aspx,http://www.ric.edu/admissions/Pages/default.aspx,https://twitter.com/RICNews/,https://www.facebook.com/rhodeislandcollege,http://www.instagram.com/rhodeislandcollege/?hl=en
    196185,SUNY Oneonta,https://suny.oneonta.edu/,https://suny.oneonta.edu/admissions,https://twitter.com/SUNY_Oneonta,https://www.facebook.com/SUNYOneonta,https://www.instagram.com/sunyoneonta/
    216931,Wilkes University,https://www.wilkes.edu/,https://www.wilkes.edu/admissions/index.aspx,https://twitter.com/wilkesU,https://www.facebook.com/WilkesUniversity,https://instagram.com/wilkesu
    141644,Hawaii Pacific University,https://www.hpu.edu/,https://www.hpu.edu/admissions/index.html,https://twitter.com/hpu,https://www.facebook.com/hawaiipacific,https://www.instagram.com/hawaiipacificuniversity/
    199999,Winston-Salem State University,https://www.wssu.edu/,https://www.wssu.edu/admissions/index.html,https://www.twitter.com/WSSURAMS,https://www.facebook.com/WSSU1892,https://www.instagram.com/p/BpXF7K2g5N6/
    178624,Northwest Missouri State University,https://www.nwmissouri.edu/,https://www.nwmissouri.edu/admissions/,https://www.twitter.com/KNWT8,https://www.facebook.com/nwmissouri,https://instagram.com/nwmostate/
    168421,Worcester Polytechnic Institute,https://www.wpi.edu/,https://www.wpi.edu/admissions/undergraduate,https://twitter.com/WPI,https://www.facebook.com/wpi.edu,https://www.instagram.com/wpi
    171146,University of Michigan-Flint,https://www.umflint.edu/,http://www.umflint.edu/admissions/admissions,https://twitter.com/UMFlint,https://www.facebook.com/umflint,http://instagram.com/umflint/
    139366,Columbus State University,https://www.columbusstate.edu/,https://admissions.columbusstate.edu/,https://twitter.com/ColumbusState,//www.facebook.com/ColumbusState,//instagram.com/ColumbusState
    212115,East Stroudsburg University of Pennsylvania,https://www.esu.edu/,https://www.esu.edu/admissions/index.cfm,https://twitter.com/#!/esuniversity,https://www.facebook.com/eaststroudsburguniversity,https://www.instagram.com/esuniversity/
    240471,University of Wisconsin-River Falls,https://www.uwrf.edu/,https://www.uwrf.edu/Admissions/,https://twitter.com/UWRiverfalls,https://www.facebook.com/UWriverfalls,https://instagram.com/uwriverfalls
    170806,Madonna University,https://www.madonna.edu/,https://www.madonna.edu/admissions/,No Twitter found,No Facebook found,No Instagram found
    195544,Saint Joseph's College-New York,https://www.sjcny.edu/,https://www.sjcny.edu/long-island/admissions,https://twitter.com/SJCNY,https://www.facebook.com/SJCNY,https://instagram.com/sjcny/
    143358,Bradley University,https://www.bradley.edu/,https://www.bradley.edu/admissions/,https://www.twitter.com/bradleyalumni,https://facebook.com/profile.php?id=75522152268,https://www.instagram.com/bradleystudentinvolvement
    196042,Farmingdale State College,https://www.farmingdale.edu/,https://www.farmingdale.edu/admissions/,https://twitter.com/farmingdalesc,https://www.facebook.com/farmingdale,https://instagram.com/farmingdalesc/
    240462,University of Wisconsin-Platteville,https://www.uwplatt.edu/,https://www.uwplatt.edu/admission,https://twitter.com/intent/tweet?url=https://www.uwplatt.edu&text=&via=https://www.uwplatt.edu/news/pioneer-spotlight-louis-nzegwu,https://www.facebook.com/sharer.php?u=https://www.uwplatt.edu/news/pioneer-spotlight-louis-nzegwu,//instagram.com/uwplatteville
    192819,Marist College,https://www.marist.edu/,https://www.marist.edu/admission/undergraduate,https://twitter.com/Marist,https://www.facebook.com/marist/,https://www.instagram.com/marist/
    174358,Minnesota State University-Moorhead,https://www.mnstate.edu/,https://www.mnstate.edu/admissions/,https://www.twitter.com/MSUMoorhead,https://www.facebook.com/msumoorhead,https://www.instagram.com/msumoorhead
    196167,SUNY College at Geneseo,https://www.geneseo.edu/,https://www.geneseo.edu/,https://www.twitter.com/SUNYGeneseo,https://www.facebook.com/events/772673766410821/,https://www.instagram.com/sunygeneseo
    133881,Florida Institute of Technology,https://www.fit.edu/,https://www.fit.edu/admissions-overview/,https://twitter.com/floridatech,https://www.facebook.com/FloridaInstituteofTechnology,https://www.instagram.com/floridatech/
    196158,SUNY at Fredonia,http://www.fredonia.edu/,http://www.fredonia.edu/admissions-aid,https://twitter.com/FredoniaU,https://www.facebook.com/fredoniau,https://www.instagram.com/fredoniau/p/BpUccgplrfj/
    175856,Jackson State University,http://www.jsums.edu/,http://www.jsums.edu/admissions/,https://twitter.com/jacksonstateU,https://www.facebook.com/JacksonState,http://instagram.com/jacksonstateu/
    173665,Hamline University,https://www.hamline.edu/,https://www.hamline.edu/general-admission/,https://twitter.com/#!/HamlineU,http://www.facebook.com/hamline,http://instagram.com/hamlineu
    186201,Ramapo College of New Jersey,https://www.ramapo.edu/,https://www.ramapo.edu/admissions/,https://twitter.com/ramapocollegenj,https://www.facebook.com/RamapoCollege,//instagram.com/ramapocollegenj#
    193973,Niagara University,https://www.niagara.edu/,https://www.niagara.edu/admissions,https://twitter.com/niagarauniv,https://www.facebook.com/niagarau,http://instagram.com/niagarauniversity
    198561,Gardner-Webb University,https://gardner-webb.edu/,https://gardner-webb.edu/admissions-and-financial-aid/undergraduate-admissions/index,https://twitter.com/gardnerwebb,https://www.facebook.com/gardnerwebb?ref=hl,https://www.instagram.com/gardnerwebb/
    229063,Texas Southern University,http://www.tsu.edu/home/,http://www.tsu.edu/admissions/,http://twitter.com/TexasSouthern,https://www.facebook.com/texassouthernuniversity,https://www.instagram.com/texassouthern/
    194541,Polytechnic Institute of New York University,https://engineering.nyu.edu/,https://www.collegesimply.com/colleges/new-york/polytechnic-institute-of-new-york-university/admission/,https://twitter.com/nyutandon,https://www.facebook.com/nyutandon,https://www.instagram.com/nyutandon
    198136,Campbell University,https://www.campbell.edu/,https://www.campbell.edu/admissions/,https://twitter.com/campbelledu,https://www.facebook.com/campbelluniversity/,https://www.instagram.com/campbelledu/
    227526,Prairie View A & M University,https://www.pvamu.edu/,https://www.pvamu.edu/admissions/undergraduate/,https://twitter.com/PVAMU?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor,https://www.facebook.com/pvamu/,https://www.instagram.com/pvamu/
    156082,Washburn University,https://www.washburn.edu/,https://washburn.edu/admissions/,https://twitter.com/washburnuniv,https://facebook.com/washburnuniversity,https://www.instagram.com/washburnuniversity/
    224226,Dallas Baptist University,https://www.dbu.edu/,https://www.dbu.edu/admissions,https://www.twitter.com/dbupatriots,https://www.facebook.com/DallasBaptistUniv,https://www.instagram.com/DBUPatriots/
    181215,University of Nebraska at Kearney,https://www.unk.edu/,https://www.unk.edu/admissions/,https://twitter.com/UNKearney,https://www.facebook.com/UNKearney,http://instagram.com/UNKearney
    210429,Western Oregon University,http://www.wou.edu/,http://www.wou.edu/admission/,https://twitter.com/wounews,https://www.facebook.com/WOUnews,https://www.instagram.com/p/BohZd-DljGA/
    147828,Olivet Nazarene University,https://www.olivet.edu/,https://www.olivet.edu/admissions,https://twitter.com/olivetnazarene,https://www.facebook.com/OlivetNazareneUniversity/,https://www.instagram.com/olivetnazarene/
    159993,University of Louisiana at Monroe,https://www.ulm.edu/,https://www.ulm.edu/admissions/,No Twitter found,http://www.facebook.com/universitylouisianamonroe,No Instagram found
    176053,Mississippi College,https://www.mc.edu/,https://www.mc.edu/admissions/,http://twitter.com/misscollege,No Facebook found,http://instagram.com/misscollege
    161457,University of New England,https://www.une.edu/,https://www.une.edu/admissions,http://www.twitter.com/unetweets,http://www.facebook.com/universityofnewengland,http://instagram.com/uniofnewengland
    177214,Drury University,https://www.drury.edu/,https://www.drury.edu/admission/,https://twitter.com/DruryUniversity,https://www.facebook.com/DruryUniversity,https://instagram.com/DruryUniversity
    128744,University of Bridgeport,https://www.bridgeport.edu/,https://www.bridgeport.edu/admissions/,https://twitter.com/UBridgeport,https://www.facebook.com/UBridgeport,http://instagram.com/ubridgeport
    102049,Samford University,https://www.samford.edu/,https://www.samford.edu/admission/,https://twitter.com/SamfordU/,https://www.facebook.com/SamfordUniversity,https://instagram.com/samfordu/
    233374,University of Richmond,https://www.richmond.edu/,https://admission.richmond.edu/,https://twitter.com/urichmond,https://www.facebook.com/urichmond,https://www.instagram.com/urichmond/
    138789,Armstrong Atlantic State University,https://www.georgiasouthern.edu/,https://www.armstrong.edu/armstrong/1/admissions,http://twitter.com/georgiasouthern,https://www.facebook.com/GeorgiaSouthern,http://instagram.com/georgiasouthernuniversity
    178615,Truman State University,http://www.truman.edu/,http://www.truman.edu/admission-cost/,https://twitter.com/TrumanState,https://www.facebook.com/trumanstateuniversity,https://instagram.com/trumanphotos/
    222831,Angelo State University,https://www.angelo.edu/,https://www.angelo.edu/content/profiles/2495-admissions/Templates/profiles-office-directory.php,https://twitter.com/AngeloState,https://www.facebook.com/angelostateuniversity,https://www.instagram.com/angelostate/
    183080,Plymouth State University,https://www.plymouth.edu/,https://www.plymouth.edu/prospective/undergraduate/undergraduate/admissions/,https://www.twitter.com/PlymouthState,https://www.facebook.com/PlymouthState,https://www.instagram.com/PlymouthState
    232681,University of Mary Washington,https://www.umw.edu/,https://www.umw.edu/admissions/,https://twitter.com/marywash/,https://www.facebook.com/UniversityofMaryWashington,https://www.instagram.com/marywash/
    194578,Pratt Institute-Main,https://www.pratt.edu/,https://www.pratt.edu/admissions/,http://twitter.com/prattinstitute,http://facebook.com/prattinstitute,http://instagram.com/prattinstitute
    129242,Fairfield University,https://www.fairfield.edu/,https://www.fairfield.edu/undergraduate/visit-and-apply/,https://www.twitter.com/fairfieldu,https://www.facebook.com/FairfieldUniversity,https://www.instagram.com/fairfieldu
    221768,The University of Tennessee-Martin,https://www.utm.edu/,http://www.utm.edu/admis.php,https://twitter.com/utmartin,https://www.facebook.com/utmartin,http://instagram.com/utmartin
    143118,Aurora University,https://aurora.edu/,https://www.aurora.edu/admission/index.html#.W9U3OeIpC00,https://twitter.com/AuroraU?ref_src=twsrc%5Etfw,http://www.facebook.com/aurorauniversity,http://instagram.com/aurorauniversity#
    102614,University of Alaska Fairbanks,https://www.uaf.edu/uaf/,https://uaf.edu/admissions/,httpx://twitter.com/uafairbanks/,https://www.facebook.com/uafairbanks/,https://instagram.com/uafairbanks/
    171492,Northwood University-Michigan,https://www.northwood.edu/,https://www.northwood.edu/admissions,http://twitter.com/northwoodu,http://facebook.com/northwoodu,https://www.instagram.com/northwoodu
    186283,Rider University,https://www.rider.edu/,https://www.rider.edu/admissions,https://twitter.com/RiderUniversity,https://www.facebook.com/RiderUniversity,https://www.instagram.com/p/BpPeTCdnHfF/
    101879,University of North Alabama,https://www.una.edu/,https://www.una.edu/admissions/,http://www.twitter.com/north_alabama,http://www.facebook.com/northalabama,https://www.instagram.com/north_alabama
    218964,Winthrop University,https://www.winthrop.edu/,https://www.winthrop.edu/admissions/,http://twitter.com/winthropu,http://www.facebook.com/Winthrop.University,http://instagram.com/winthropu
    148654,University of Illinois at Springfield,https://www.uis.edu/,https://www.uis.edu/,https://twitter.com/UISedu,https://www.facebook.com/uis.edu,https://www.instagram.com/uisedu/
    165820,Fitchburg State University,https://m.fitchburgstate.edu/default/home/index,https://www.fitchburgstate.edu/admissions/,/default/social/index?feed=twitter,/default/social/index?feed=facebook,No Instagram found
    226833,Midwestern State University,https://msutexas.edu/,https://msutexas.edu/admissions/,https://www.twitter.com/msutexas,https://www.facebook.com/msutexas,https://www.instagram.com/msutexas
    168430,Worcester State University,https://www.worcester.edu/,https://www.worcester.edu/Admissions/,https://twitter.com/worcesterstate,https://www.facebook.com/WorcesterStateUniversity,https://www.instagram.com/worcesterstate
    159966,Nicholls State University,https://www.nicholls.edu/,https://www.nicholls.edu/admission/,//twitter.com/nichollsstate/,//www.facebook.com/Nicholls-State-University/,//www.instagram.com/nichollsstate/
    155025,Emporia State University,https://www.emporia.edu/,https://www.emporia.edu/admissions/,http://twitter.com/emporiastate/,https://www.facebook.com/emporiastateuniversity,No Instagram found
    110097,Biola University,https://www.biola.edu/,https://www.biola.edu/admissions/undergrad,https://twitter.com/biolau,https://www.facebook.com/Biola,https://instagram.com/biolauniversity
    238430,Cardinal Stritch University,https://www.stritch.edu/programs,https://www.stritch.edu/Admissions/Request-Information,No Twitter found,No Facebook found,No Instagram found
    235167,The Evergreen State College,https://www.evergreen.edu/,https://www.evergreen.edu/admissions,https://twitter.com/EvergreenStCol,https://facebook.com/TheEvergreenStateCollege,https://instagram.com/EvergreenStCol
    183062,Keene State College,https://www.keene.edu/,https://www.keene.edu/admissions/,https://twitter.com/kscadmissions,https://www.facebook.com/Keene-State-College-Owl-Admissions-140293459341245/,https://www.instagram.com/kscadmissions/
    148627,Saint Xavier University,https://www.sxu.edu/,https://www.sxu.edu/admissions/,https://twitter.com/SaintXavier,https://www.facebook.com/SXUCougarDiaries,https://instagram.com/sxucougars
    107044,Harding University,https://www.harding.edu/,https://www.harding.edu/admissions,https://twitter.com/HardingU,//www.facebook.com/HardingU,//www.instagram.com/hardinguniversity
    110413,California Lutheran University,https://www.callutheran.edu/,https://www.callutheran.edu/admission/undergraduate/,http://www.twitter.com/callutheran,http://www.facebook.com/callutheran,http://www.instagram.com/callutheran
    129215,Eastern Connecticut State University,http://www.easternct.edu/,http://www.easternct.edu/admissions/,https://twitter.com/EasternCTStateU,https://www.facebook.com/EasternCTStateUniversity/,https://www.instagram.com/easternctstateuniv/
    199281,University of North Carolina at Pembroke,https://www.uncp.edu/,https://www.uncp.edu/admissions,https://twitter.com/uncpembroke,https://www.facebook.com/uncpembroke,https://instagram.com/uncpembroke/
    112075,Concordia University-Irvine,https://www.cui.edu/en-us,https://www.cui.edu/admissions,https://twitter.com//ConcordiaIrvine,https://www.facebook.com/concordiairvine,https://www.instagram.com/concordiairvine/
    126775,Colorado School of Mines,https://www.mines.edu/,https://www.mines.edu/admissions/,https://twitter.com/coschoolofmines,https://www.facebook.com/ColoradoSchoolofMines,https://www.instagram.com/coloradoschoolofmines
    173160,Bethel University,https://www.bethel.edu/,https://www.bethel.edu/admissions/,https://twitter.com/BethelU,https://www.facebook.com/betheluniversity,https://www.instagram.com/bethelumn
    163453,Morgan State University,https://www.morgan.edu/,https://www.morgan.edu/admissions.html,https://twitter.com/MorganStateU,https://www.facebook.com/morganstateu,https://instagram.com/morganstateu/
    210146,Southern Oregon University,https://sou.edu/,https://sou.edu/admissions/,https://twitter.com/@souashland/,https://www.facebook.com/SOUAshland/,https://www.instagram.com/souashland/
    209056,Lewis & Clark College,https://www.lclark.edu/,https://college.lclark.edu/offices/admissions/,http://twitter.com/lewisandclark,http://www.facebook.com/lewisandclarkcollege,http://instagram.com/lewisandclarkcollege
    165866,Framingham State University,https://www.framingham.edu/,https://www.framingham.edu/admissions-and-aid/admissions/,https://twitter.com/FraminghamU,https://www.facebook.com/FraminghamStateUniversity,No Instagram found
    162584,Frostburg State University,https://www.frostburg.edu/,https://www.frostburg.edu/admissions-and-cost/undergraduate/index.php,http://twitter.com/frostburgstate,http://www.facebook.com/FrostburgStateUniversity/,https://www.instagram.com/p/BpcXdn1Hd07/
    134945,Jacksonville University,https://www.ju.edu/,https://www.ju.edu/admissions/,https://twitter.com/JacksonvilleU,https://www.facebook.com/jacksonvilleuniversity,https://www.instagram.com/jacksonvilleu/
    139311,Clayton  State University,http://www.clayton.edu/,http://www.clayton.edu/admissions/,https://twitter.com/ClaytonState,https://www.facebook.com/ClaytonStateUniversity/,https://www.instagram.com/claytonstateuniv/
    191931,Iona College,https://www.iona.edu/home.aspx,https://www.iona.edu/admissions.aspx,https://twitter.com/ionacollege,https://www.facebook.com/IonaCollegeNY,https://instagram.com/ionacollege/
    123554,Saint Mary's College of California,https://www.stmarys-ca.edu/,https://www.stmarys-ca.edu/undergraduate-admissions,https://twitter.com/stmarysca,https://www.facebook.com/stmarysca,https://www.instagram.com/stmarysca/
    154688,Baker University,https://www.bakeru.edu/,https://www.bakeru.edu/admissions/,http://twitter.com/BakerUniversity,http://www.facebook.com/BakerUniversity,http://instagram.com/bakeruniversity
    231712,Christopher Newport University,http://cnu.edu/,http://cnu.edu/admission/,https://twitter.com/cnucaptains,https://www.facebook.com/christophernewportuniversity,https://www.instagram.com/p/BpcpZdknb1r/
    151263,University of Indianapolis,https://www.uindy.edu/,https://www.uindy.edu/admissions/,https://twitter.com/uindy,https://www.facebook.com/uindy,https://www.instagram.com/uindy
    151379,Indiana University-Southeast,https://www.ius.edu/,https://www.ius.edu/admissions/,https://twitter.com/IUSoutheast,https://www.facebook.com/iusoutheast,https://www.instagram.com/p/BpZe8q4nFnx
    165662,Emerson College,https://www.emerson.edu/,https://www.emerson.edu/undergraduate-admission/contact-us,http://twitter.com/EmersonCollege,http://www.facebook.com/EmersonCollege,http://instagram.com/emersoncollege
    215655,Robert Morris University,https://www.rmu.edu/,https://www.rmu.edu/admissions,No Twitter found,https://www.facebook.com/RMUpgh/,https://www.instagram.com/robertmorrisuniversity/
    212133,Eastern University,https://www.eastern.edu/,https://www.eastern.edu/admissions,https://twitter.com/easternU,https://www.facebook.com/EasternUniversity,https://www.instagram.com/p/BpZ6GDMB2kw/
    195720,Saint John Fisher College,https://www.sjfc.edu/,https://www.sjfc.edu/admissions-aid/,https://twitter.com/FisherNews,https://www.facebook.com/StJohnFisherCollege/,https://www.instagram.com/stjohnfishercollege
    160621,Southern University and A & M College,http://www.subr.edu/,http://www.subr.edu/subhome/43,https://twitter.com/SouthernU_BR,https://www.facebook.com/southernuniversitybatonrouge/,https://www.instagram.com/southernu_br/
    173045,Augsburg College,http://www.augsburg.edu/,http://www.augsburg.edu/admissions/,https://twitter.com/augsburgu,https://www.facebook.com/AugsburgUniversity/,https://www.instagram.com/p/Bo9Pb9BFh1G/
    189228,Berkeley College-New York,https://berkeleycollege.edu/locations_bc/nyc_midtown.htm,https://berkeleycollege.edu/admissions.htm,https://twitter.com/berkeleycollege,https://www.facebook.com/BerkeleyCollegePage,https://www.instagram.com/berkeleycollege/
    216852,Widener University-Main Campus,http://www.widener.edu/,http://www.widener.edu/admissions/,https://twitter.com/wideneruniv,https://www.facebook.com/wideneruniversity,https://www.instagram.com/wideneruniversity/
    219976,Lipscomb University,https://www.lipscomb.edu/,https://www.lipscomb.edu/admissions,https://twitter.com/lipscomb,https://www.facebook.com/lipscombuniversity,https://instagram.com/lipscombuniversity
    130776,Western Connecticut State University,http://www.wcsu.edu/,http://www.wcsu.edu/admissions/ugrad/,http://www.twitter.com/westconn,http://www.facebook.com/westconn,http://www.instagram.com/westconn
    165334,Clark University,https://www.clarku.edu/,https://www.clarku.edu/admissions/undergraduate-admissions/,https://twitter.com/clarkuniversity,https://www.facebook.com/ClarkUniversityWorcester,https://www.instagram.com/clarkuniversity/
    228149,St Mary's University,https://www.stmarytx.edu/,https://www.stmarytx.edu/admission/,https://www.twitter.com/StMarysU,https://www.facebook.com/31373936394/posts/10155957464411395,https://www.instagram.com/p/BpaWVHIn48D/
    172334,Spring Arbor University,https://www.arbor.edu/,https://www.arbor.edu/admissions/,https://twitter.com/springarboru,https://www.facebook.com/springarboru/?fref=ts,https://www.instagram.com/springarboru/
    207458,Oklahoma City University,https://www.okcu.edu/,https://www.okcu.edu/admissions/home,https://twitter.com/okcu,https://www.facebook.com/oklahomacityuniversity,https://www.instagram.com/oklahomacityuniversity/
    130697,Wesleyan University,https://www.wesleyan.edu/,https://www.wesleyan.edu/admission/,https://twitter.com/intent/tweet?original_referer=https%3A%2F%2Fabout.twitter.com%2Fresources%2Fbuttons&text=&tw_p=tweetbutton&url=https%3A%2F%2Fwww.wesleyan.edu%2F,https://www.facebook.com/sharer/sharer.php?u=https%3A%2F%2Fwww.wesleyan.edu%2F&title=,https://www.instagram.com/wesleyan_u/
    173124,Bemidji State University,https://www.bemidjistate.edu/,https://www.bemidjistate.edu/admissions/,No Twitter found,No Facebook found,No Instagram found
    Error: 219718 - Bethel University
    221971,Union University,https://www.uu.edu/,https://www.uu.edu/admissions/,http://twitter.com/UnionUniversity/,http://facebook.com/UnionUniversity/,http://instagram.com/UnionUniversity/
    201195,Baldwin Wallace University,https://www.bw.edu/,https://www.bw.edu/admission/,https://twitter.com/BaldwinWallace,http://www.facebook.com/baldwinwallaceuniversity,https://www.instagram.com/explore/tags/YJ4L/
    207971,University of Tulsa,https://utulsa.edu/,https://admission.utulsa.edu/,No Twitter found,No Facebook found,No Instagram found
    218742,University of South Carolina-Upstate,https://www.uscupstate.edu/,https://www.uscupstate.edu/admissions-and-financial-aid/,https://twitter.com/USCUpstate,https://www.facebook.com/uscupstate/,https://www.instagram.com/usc_upstate/?hl=en
    213613,Lock Haven University,https://www.lockhaven.edu/,https://www.lockhaven.edu/admissions/,https://twitter.com/LockHavenUniv,https://www.facebook.com/LockHavenUniv/,https://www.instagram.com/lockhavenuniv/
    196200,SUNY College at Potsdam,https://www.potsdam.edu/,https://www.potsdam.edu/admissions,No Twitter found,http://www.facebook.com/SUNYPotsdam,https://www.instagram.com/sunypotsdam/
    197045,Utica College,https://www.utica.edu/,https://www.utica.edu/enrollment/admissions/,http://www.twitter.com/uticacollege,http://www.facebook.com/uticacollege,http://instagram.com/uticacollege/
    198543,Fayetteville State University,https://www.uncfsu.edu/,https://www.uncfsu.edu/fsu-admissions,https://twitter.com/uncfsu,https://www.facebook.com/Admissions.FSU?fref=ts,No Instagram found
    211088,Arcadia University,https://www.arcadia.edu/,https://www.arcadia.edu/admissions,//twitter.com/NBEA,https://www.facebook.com/arcadia.university,/admissions/admitted-students/arcadiabound-instagram-contest
    152600,Valparaiso University,https://www.valpo.edu/,https://www.valpo.edu/admission/,http://twitter.com/valpou,http://www.nwitimes.com/news/local/porter/valparaiso-university-invents-competition-for-innovation/article_ec23e647-07fb-5d11-a800-aab06df6c3e9.html?utm_content=bufferaa1d9&utm_medium=social&utm_source=facebook.com&utm_campaign=LEEDCC,http://instagram.com/valparaiso_university#
    236577,Seattle Pacific University,http://spu.edu/,http://spu.edu/,http://spu.edu/twitter,http://spu.edu/facebook,http://spu.edu/instagram
    154235,Saint Ambrose University,https://www.sau.edu/,https://www.sau.edu/admissions,https://twitter.com/stambrose,https://www.facebook.com/stambroseuniversity/,No Instagram found
    211291,Bucknell University,https://www.bucknell.edu/,https://www.bucknell.edu/admissions,http://twitter.com/BucknellU,http://facebook.com/BucknellU,https://www.instagram.com/BucknellU/
    197151,School of Visual Arts,http://www.sva.edu/,http://www.sva.edu/admissions,https://twitter.com/sva_news,http://www.facebook.com/SchoolOfVisualArts,http://www.instagram.com/SVANYC
    222178,Abilene Christian University,http://www.acu.edu/,http://www.acu.edu/admissions-aid/undergraduate/admissions.html,http://www.twitter.com/acuedu,http://www.facebook.com/abilenechristian,http://www.instagram.com/acuedu/
    217518,Roger Williams University,https://www.rwu.edu/,https://www.rwu.edu/admission,https://twitter.com/myrwu,https://www.facebook.com/myrwu,https://www.instagram.com/myrwu
    234155,Virginia State University,http://www.vsu.edu/,http://vsu.edu/admissions/index.php,https://twitter.com/#!/VSUTrojans,http://www.facebook.com/VirginiaStateUniversity,No Instagram found
    214713,Pennsylvania State University-Penn State Harrisburg,https://harrisburg.psu.edu/,https://harrisburg.psu.edu/office-of-admissions,https://twitter.com/PSUHarrisburg,https://www.facebook.com/pennstateharrisburg?v=wall&filter=1,https://instagram.com/pennstateharrisburg
    209825,University of Portland,https://www.up.edu/,https://www.up.edu/admissions/,https://twitter.com/UPortland,https://www.facebook.com/universityofportland/,https://www.instagram.com/uportland/
    178341,Missouri Southern State University,https://www.mssu.edu/,https://www.mssu.edu/advancement/admissions/,https://twitter.com/mosolions,https://www.facebook.com/mssulions/,https://www.instagram.com/mosolions/
    150163,Butler University,https://www.butler.edu/,https://www.butler.edu/admission,https://twitter.com/butleru,https://www.facebook.com/butleruniversity,https://www.instagram.com/p/BpZtdsLhH5D/
    214591,Pennsylvania State University-Penn State Erie-Behrend College,https://behrend.psu.edu/,https://behrend.psu.edu/admission,https://twitter.com/psbehrend,https://www.facebook.com/pennstatebehrend,http://instagram.com/psbehrend
    130183,Post University,https://post.edu/,https://post.edu/admissions,https://www.twitter.com/Postuniversity,https://www.facebook.com/Postuniversity,https://www.instagram.com/postuniversity
    232706,Marymount University,https://www.marymount.edu/,https://www.marymount.edu/Admissions/Welcome,https://twitter.com/marymountu,https://www.facebook.com/marymount.university,http://instagram.com/marymountu
    164580,Babson College,http://www.babson.edu/Pages/default.aspx,http://www.babson.edu/admission/Pages/default.aspx,http://www.babson.edu/social-media/twitter/Pages/twitter.aspx,http://www.babson.edu/social-media/Pages/facebook.aspx,http://www.babson.edu/social-media/Pages/instagram.aspx
    137546,Stetson University,https://www.stetson.edu/home/,https://www.stetson.edu/portal/admissions/,https://twitter.com/stetsonu,https://www.facebook.com/stetsonU,https://instagram.com/stetsonu
    211352,Cabrini College,https://www.cabrini.edu/,https://www.cabrini.edu/about/departments/admissions-and-enrollment,http://twitter.com/CabriniUniv ,http://facebook.com/Cabrini,http://instagram.com/cabriniuniversity
    144005,Chicago State University,http://www.csu.edu/,http://www.csu.edu/admissions/,No Twitter found,No Facebook found,No Instagram found
    167835,Smith College,https://www.smith.edu/,https://www.smith.edu/admission-aid,https://twitter.com/smithcollege,https://www.facebook.com/smithcollege,https://www.instagram.com/smithcollege/
    151342,Indiana University-South Bend,https://www.iusb.edu/,https://admissions.iusb.edu/,https://www.twitter.com/IUSouthBend,https://www.facebook.com/IUSouthBend,https://www.instagram.com/explore/tags/iusb2018/
    193292,Molloy College,https://www.molloy.edu/,https://www.molloy.edu/admissions,https://twitter.com/MolloyCollege,http://www.facebook.com/GoMolloy,https://www.instagram.com/molloycollege/
    153001,Buena Vista University,https://www.bvu.edu/,https://www.bvu.edu/admissions,https://twitter.com/buenavistauniv,https://www.facebook.com/buenavistauniversity,https://www.instagram.com/buenavistauniversity
    141097,Southern Polytechnic State University,http://engineering.kennesaw.edu/,http://www.kennesaw.edu/admissions.php,https://twitter.com/KSUEngineering,https://www.facebook.com/KSUEngineering/,https://www.instagram.com/KSUEngineering/
    240107,Viterbo University,http://www.viterbo.edu/,http://www.viterbo.edu/admission,https://twitter.com/viterbo_univ,https://www.facebook.com/ViterboUniversity,https://www.instagram.com/viterbouniversity/
    208822,George Fox University,https://www.georgefox.edu/,https://www.georgefox.edu/admission/index.html,https://twitter.com/georgefox,https://www.facebook.com/georgefoxuniversity,https://www.instagram.com/georgefoxuniversity/
    133553,Embry-Riddle Aeronautical University-Daytona Beach,https://daytonabeach.erau.edu/,https://daytonabeach.erau.edu/admissions/,https://twitter.com/ERAU_Daytona,https://www.facebook.com/eraudb,https://www.instagram.com/embryriddledaytona/
    Error: 232566 - Longwood University
    190770,Dowling College,https://www.dowling.edu/,https://www.hofstra.edu/admission/dowling.html,//www.twitter.com/hofstrau,//www.facebook.com/hofstrau,//instagram.com/hofstrau
    164748,Berklee College of Music,https://www.berklee.edu/,https://www.berklee.edu/admissions,https://www.twitter.com/BerkleeCollege,https://www.facebook.com/BerkleeCollege,https://www.instagram.com/berkleecollege/
    180179,Montana State University-Billings,http://www.msubillings.edu/,http://www.msubillings.edu/reg/admission.htm,https://twitter.com/msubillings,https://www.facebook.com/MSUBillings,https://instagram.com/msubillings
    162007,Bowie State University,https://www.bowiestate.edu/,https://www.bowiestate.edu/admissions-financial-aid/,http://www.twitter.com/bowiestate,http://www.facebook.com/bowiestate,No Instagram found
    149781,Wheaton College,https://www.wheaton.edu/,https://www.wheaton.edu/admissions-and-aid/undergraduate-admissions/,https://twitter.com/WheatonCollege,http://www.facebook.com/wheatoncollegeil,https://www.instagram.com/wheatoncollegeil
    173328,Concordia University-Saint Paul,https://www.csp.edu/,https://www.csp.edu/admissions/,http://www.twitter.com/CSPBearsWBB,https://www.facebook.com/ConcordiaStPaul/,No Instagram found
    128106,Colorado State University-Pueblo,https://www.csupueblo.edu/,https://www.csupueblo.edu/admissions/,https://twitter.com/csupueblo,https://www.facebook.com/ColoradoStateUniversityPueblo,https://www.instagram.com/csupueblo/
    212601,Gannon University,http://www.gannon.edu/,http://www.gannon.edu/Admissions/,http://www.gannon.edu/twitter,http://www.gannon.edu/facebook,https://www.instagram.com/gannonu
    230807,Westminster College,https://www.westminstercollege.edu/,https://www.westminstercollege.edu/about/resources/admissions,https://twitter.com/westminsterslc,https://www.facebook.com/westminsterslc,https://www.instagram.com/p/Bpa9LGWH-J-/
    202763,The University of Findlay,https://www.findlay.edu/,https://www.findlay.edu/admissions/,https://twitter.com/ufindlay,https://www.facebook.com/universityfindlay,https://www.instagram.com/ufindlay/
    159009,Grambling State University,http://www.gram.edu/,http://www.gram.edu/admissions/,No Twitter found,https://www.facebook.com/Grambling1901/,No Instagram found
    207865,Southwestern Oklahoma State University,https://www.swosu.edu/,https://www.swosu.edu/admissions/index.aspx,//twitter.com/swosu,//www.facebook.com/pages/Southwestern-Oklahoma-State-University/35024399817?ref=s,//instagram.com/swosu
    193584,Nazareth College,https://www2.naz.edu/,https://www2.naz.edu/admissions,http://twitter.com/nazarethcollege,http://www.facebook.com/NazarethCollege,http://instagram.com/nazarethcollege
    178059,Maryville University of Saint Louis,https://www.maryville.edu/,https://www.maryville.edu/admissions/,http://www.twitter.com/maryvilleu,http://www.facebook.com/MaryvilleUniversity,https://instagram.com/maryvilleu/
    198695,High Point University,http://www.highpoint.edu/,http://www.highpoint.edu/admissions/,https://twitter.com/HighPointU,https://www.facebook.com/HighPointU,http://instagram.com/highpointu
    156286,Bellarmine University,https://www.bellarmine.edu/,https://www.bellarmine.edu/admissions/,https://twitter.com/bellarmineu,https://facebook.com/bellarmineu,https://www.instagram.com/bellarmineu/
    151290,Indiana Institute of Technology,https://www.indianatech.edu/,https://admissions.indianatech.edu/,https://twitter.com/indianatech,https://www.facebook.com/indianatech,https://instagram.com/indianatech
    172264,Siena Heights University,http://sienaheights.edu/,http://sienaheights.edu/Admissions,https://twitter.com/sienaheights?ref_src=twsrc%5Etfw,https://www.facebook.com/Siena.Heights/,https://www.instagram.com/realsaints/
    209612,Pacific University,https://www.pacificu.edu/,https://www.pacificu.edu/admissions,https://twitter.com/share?url=https%3A%2F%2Fwww.instagram.com%2Fp%2FBpaYbEqHujC%2F&text=,http://www.facebook.com/sharer.php?u=https%3A%2F%2Fwww.instagram.com%2Fp%2FBpaYbEqHujC%2F&t=,https://scontent.cdninstagram.com/vp/862aa793b977696f568a7c7e82bf3668/5C50606B/t51.2885-15/sh0.08/e35/s640x640/43915182_464086937416722_7325035482064854073_n.jpg
    193645,The College of New Rochelle,https://www.cnr.edu/,https://www.cnr.edu/admissions,No Twitter found,https://www.facebook.com/TheCollegeofNewRochelle,http://instagram.com/thecollegeofnewrochelle#
    144962,Elmhurst College,https://www.elmhurst.edu/,https://www.elmhurst.edu/admission/,https://twitter.com/ConnieMixon,https://www.facebook.com/86293358032/posts/10156847304023033,https://www.instagram.com/p/Bpc1cnDHVpI/
    199069,Mount Olive College,https://umo.edu/,https://umo.edu/admissions/,https://twitter.com/OfficialUMO,https://www.facebook.com/universityofmountolive,https://www.instagram.com/officialumo/
    215099,Philadelphia University,https://www.upenn.edu/,https://www.upenn.edu/admissions,http://twitter.com/Penn,https://www.facebook.com/UnivPennsylvania,http://instagram.com/uofpenn
    Error: 232265 - Hampton University
    174844,St Olaf College,https://wp.stolaf.edu/,https://wp.stolaf.edu/admissions/,https://twitter.com/hashtag/spooky?src=hash&ref_src=twsrc%5Etfw,https://www.facebook.com/stolafcollege,https://www.instagram.com/p/Bop3sNNHqAz/
    169080,Calvin College,https://calvin.edu/,https://calvin.edu/admissions/,https://twitter.com/calvincollege,https://www.facebook.com/calvincollege,https://www.instagram.com/p/Bpabi-ygaTB/
    217165,Bryant University,https://www.bryant.edu/,https://admission.bryant.edu/,https://twitter.com/BryantHonors,https://facebook.com/bryantuniversity,https://instagram.com/bryantuniversity
    168254,Western New England University,http://wne.edu/,https://www1.wne.edu/admissions/,https://twitter.com/wneuniversity/,https://www.facebook.com/WesternNewEnglandUniversity/,https://www.instagram.com/wneuniversity/
    196219,SUNY at Purchase College,https://www.purchase.edu/,https://www.purchase.edu/admissions/,https://twitter.com/SUNY_Purchase,https://www.facebook.com/SUNYPurchaseCollege,https://www.instagram.com/purchasecollege/
    148584,University of St Francis,https://www.stfrancis.edu/,https://www.stfrancis.edu/admissions/,https://twitter.com/UofStFrancis,https://www.facebook.com/pages/USF-Student-Activities-Board/166555307993,https://www.instagram.com/uofstfrancis
    192323,Le Moyne College,https://www.lemoyne.edu/,https://www.lemoyne.edu/Admission/First-Year-Admission,http://www.twitter.com/lemoyne,http://www.facebook.com/lemoyne,https://instagram.com/lemoyne_college/
    204501,Oberlin College,https://www.oberlin.edu/,https://www.oberlin.edu/admissions-and-aid/arts-and-sciences,http://twitter.com/ObieAdmissions,https://www.facebook.com/OberlinAdmissions/,https://www.instagram.com/oberlincollege
    203368,John Carroll University,http://sites.jcu.edu/,http://sites.jcu.edu/admission/john-carroll-university-admission/,No Twitter found,No Facebook found,No Instagram found
    201548,Capital University,https://www.capital.edu/,https://www.capital.edu/admission/,https://twitter.com/Capital_U,https://www.facebook.com/CapitalU,http://instagram.com/capitalu
    213011,Immaculata University,https://www.immaculata.edu/,https://www.immaculata.edu/admissions/,https://twitter.com/immaculatau,https://www.facebook.com/Immaculata.University,https://www.instagram.com/immaculatau/
    220613,Lee University,http://www.leeuniversity.edu/,http://www.leeuniversity.edu/admissions/,http://twitter.com/leeu,http://www.facebook.com/LeeUniversity,http://instagram.com/leeuniversity
    224004,Concordia University-Texas,https://www.concordia.edu/,https://www.concordia.edu/admissions/,https://twitter.com/concordiatx,https://www.facebook.com/concordiatx,https://www.instagram.com/concordiatx
    155089,Friends University,https://www.friends.edu/,https://www.friends.edu/admissions/,https://twitter.com/friendsu,https://www.facebook.com/FriendsUniversity,https://instagram.com/friendsu/
    236230,Pacific Lutheran University,https://www.plu.edu/,https://www.plu.edu/admission/,https://twitter.com/PLUNEWS,https://www.facebook.com/Pacific.Lutheran.University,https://instagram.com/pacificlutheran
    147013,McKendree University,https://www.mckendree.edu/,https://www.mckendree.edu/admission/,No Twitter found,/facebook/index-1.php,#instagram-collage
    170675,Lawrence Technological University,https://www.ltu.edu/,https://www.ltu.edu/futurestudents/,https://twitter.com/LawrenceTechU,https://www.facebook.com/lawrencetechu,https://www.instagram.com/lawrencetechu/
    215442,Point Park University,https://www.pointpark.edu/,https://www.pointpark.edu/Admissions/index,https://twitter.com/pointparku,https://www.facebook.com/PointParkU,https://www.instagram.com/PointParkU/
    200217,University of Mary,https://www.umary.edu/,https://www.umary.edu/admissions/,https://twitter.com/umary,https://www.facebook.com/universityofmary,https://instagram.com/universityofmary
    168227,Wentworth Institute of Technology,https://wit.edu/,https://wit.edu/undergraduate-admissions,https://twitter.com/wentworthinst,https://www.facebook.com/WentworthInst,https://www.instagram.com/wentworthinstitute/
    190044,Clarkson University,https://www.clarkson.edu/,https://www.clarkson.edu/admissions,https://twitter.com/ClarksonUniv,https://www.facebook.com/ClarksonUniversity,https://www.instagram.com/clarksonuniv/
    136950,Rollins College,https://www.rollins.edu/,https://www.rollins.edu/admission-aid/,http://twitter.com/#!/rollinscollege,http://www.facebook.com/Rollins.College,http://instagram.com/rollinscollege
    179964,William Woods University,https://www.williamwoods.edu/index.html,https://www.williamwoods.edu/admissions/index.html,http://twitter.com/WilliamWoodsU,http://www.facebook.com/WilliamWoodsUniversity,https://www.instagram.com/WilliamWoodsU/
    192925,Medaille College,https://www.medaille.edu/,https://www.medaille.edu/admissions/how-apply/undergraduate-admissions,https://twitter.com/MedailleCollege,https://www.facebook.com/medaillecollege,http://instagram.com/medaillecollege
    217749,Bob Jones University,https://www.bju.edu/,https://www.bju.edu/admission/,No Twitter found,No Facebook found,No Instagram found
    210401,Willamette University,http://willamette.edu/,https://willamette.edu/cla/admission/index.html,https://twitter.com/willamette_u,https://www.facebook.com/Willamette,https://www.instagram.com/p/BpZgQ0LFIPx/
    195474,Siena College,https://www.siena.edu/,https://www.siena.edu/offices/admissions/,https://twitter.com/SienaCollege,https://www.facebook.com/sienacollege/,https://www.instagram.com/sienacollege/
    200253,Minot State University,https://www.minotstateu.edu/,https://www.minotstateu.edu/enroll/,//twitter.com/Minotstateu,//www.facebook.com/pages/Minot-ND/Minot-State-University/163882279568,//www.instagram.com/minotstate
    206914,Cameron University,https://www.cameron.edu/,https://www.cameron.edu/admissions,http://www.twitter.com/CUAggies,http://www.facebook.com/CameronUniversity,No Instagram found
    217864,Citadel Military College of South Carolina,http://www.citadel.edu/root/,http://www.citadel.edu/root/admissions,No Twitter found,http://www.facebook.com/thecitadel,http://instagram.com/TheCitadel1842/
    147660,North Central College,https://www.northcentralcollege.edu/,https://www.northcentralcollege.edu/apply,https://twitter.com/northcentralcol,https://www.facebook.com/NorthCentralCollege/,https://www.instagram.com/northcentralcollege/
    121309,Point Loma Nazarene University,https://www.pointloma.edu/,https://www.pointloma.edu/undergraduate/admissions,https://twitter.com/plnu,https://www.facebook.com/PointLomaNazareneUniversity/,https://www.instagram.com/p/BpU6mKdHcGD/
    192703,Manhattan College,https://manhattan.edu/,https://manhattan.edu/admissions/index.php,https://twitter.com/ManhattanEdu/,https://www.facebook.com/ManhattanCollegeEdu,https://www.instagram.com/manhattanedu/
    178387,Missouri Western State University,https://www.missouriwestern.edu/admissions/,https://www.missouriwestern.edu/admissions/,https://twitter.com/missouriwestern,https://www.facebook.com/MissouriWestern,https://www.instagram.com/missouriwesternstateuniversity/
    213826,Marywood University,http://www.marywood.edu/,http://www.marywood.edu/admissions/index.html,http://twitter.com/marywooduadm,https://www.facebook.com/marywooduadmissions,http://www.instagram.com/marywooduniversity
    236328,University of Puget Sound,https://www.pugetsound.edu/,https://www.pugetsound.edu/admission/,http://www.twitter.com/univpugetsound,http://www.facebook.com/univpugetsound,http://instagram.com/univpugetsound
    164173,Stevenson University,http://www.stevenson.edu/,http://www.stevenson.edu/admissions-aid/,https://twitter.com/stevensonu,http://www.facebook.com/stevensonuniversity,http://instagram.com/stevensonuniversity#
    143048,School of the Art Institute of Chicago,http://www.saic.edu/,http://www.saic.edu/admissions/ug/,https://twitter.com/saic_news#act,http://www.facebook.com/saic.events#act,http://instagram.com/saicpics##act
    187648,Eastern New Mexico University-Main Campus,https://www.enmu.edu/,https://www.enmu.edu/admission,https://twitter.com/enmu,https://www.facebook.com/goenmu,https://www.instagram.com/enmu
    237367,Fairmont State University,https://www.fairmontstate.edu/,https://www.fairmontstate.edu/admit/,https://twitter.com/fairmontstate,https://www.facebook.com/FairmontState,https://instagram.com/fairmontstate?ref=badge
    212984,Holy Family University,https://www.holyfamily.edu/,https://www.holyfamily.edu/choosing-holy-family-u/admissions/contact-admissions,https://www.twitter.com/holyfamilyu,https://www.facebook.com/HolyFamilyUniversity,https://www.instagram.com/holyfamilyu/
    221892,Trevecca Nazarene University,https://www.trevecca.edu/,https://www.trevecca.edu/admissions,https://twitter.com/Trevecca,http://www.facebook.com/treveccanazarene,No Instagram found
    218238,Limestone College,https://www.limestone.edu/,https://www.limestone.edu/day/admissions,https://twitter.com/at_LimestoneCo,https://www.facebook.com/LimestoneCollege,No Instagram found
    Error: 100654 - Alabama A & M University
    166124,College of the Holy Cross,https://www.holycross.edu/,https://www.holycross.edu/admissions-aid,https://twitter.com/holy_cross,https://www.facebook.com/collegeoftheholycross,http://instagram.com/collegeoftheholycross
    220516,King University,http://www.king.edu/,https://www.king.edu/admissions/,https://twitter.com/KingUnivBristol,https://www.facebook.com/kutornado,https://instagram.com/kinguniversity/
    190099,Colgate University,http://www.colgate.edu/,http://www.colgate.edu/admission-financial-aid,https://twitter.com/colgateuniv,http://www.facebook.com/sharer.php?u=http%3a%2f%2fwww.colgate.edu%2fimages%2fdefault-source%2fdefault-library%2fvisit-large.jpg%3fsfvrsn%3d0&t=Visit%20Campus,No Instagram found
    229267,Trinity University,https://new.trinity.edu/,https://new.trinity.edu/admissions-aid,http://twitter.com/Trinity_U,http://www.facebook.com/pages/Trinity-University/20015082725,http://instagram.com/trinityu
    100830,Auburn University at Montgomery,http://www.aum.edu/,http://www.aum.edu/admissions,https://twitter.com/aumontgomery,https://www.facebook.com/auburnmontgomery,https://www.instagram.com/auburnmontgomery/
    100724,Alabama State University,http://www.alasu.edu/index.aspx,http://www.alasu.edu/admissions/undergrad-admissions/index.aspx,https://twitter.com/ASUHornetNation,http://www.facebook.com/pages/Alabama-State/149537805116515?sk=wall,http://instagram.com/alabamastateuniversity
    176035,Mississippi University for Women,https://www.muw.edu/,https://www.muw.edu/admissions,http://www.twitter.com/muwedu,http://www.facebook.com/muwedu,http://instagram.com/muwedu
    210739,DeSales University,https://www.desales.edu/,https://www.desales.edu/error?aspxerrorpath=/docs/default-source/campus-maps/desales-campus-map.pdf#gsc.tab=0,//twitter.com/desales,//www.facebook.com/DeSalesUniversity,//www.instagram.com/desalesuniversity
    215743,Saint Francis University,https://www.francis.edu/,https://www.francis.edu/Admissions/,https://twitter.com/SaintFrancisPA,https://www.facebook.com/SaintFrancisUniversity/,http://instagram.com/SaintFrancisPA
    179326,Southwest Baptist University,https://www.sbuniv.edu/,https://www.sbuniv.edu/admissions/,https://twitter.com/SBUniv,https://www.facebook.com/SBUniv,https://instagram.com/sbuniv/
    212832,Gwynedd Mercy University,https://www.gmercyu.edu/,https://www.gmercyu.edu/admissions-aid,http://twitter.com/GMercyUFH,https://www.facebook.com/GMercyU,http://instagram.com/gmercyu
    119173,Mount St Mary's College,https://www.msmu.edu/,https://www.msmu.edu/admission/,https://twitter.com/msmu_la,https://www.facebook.com/MountSaintMarysU,https://instagram.com/msmu_la/
    207847,Southeastern Oklahoma State University,https://www.se.edu/,http://www.se.edu/future-students/admissions-application/,No Twitter found,//www.facebook.com/SoutheasternOklahomaStateUniversity,No Instagram found
    175616,Delta State University,http://www.deltastate.edu/,http://www.deltastate.edu/admissions/,https://twitter.com/deltastate,https://www.facebook.com/deltastateuniversity,https://instagram.com/deltastateuniversity/
    211583,Chestnut Hill College,https://www.chc.edu/,https://www.chc.edu/undergraduate-admissions,http://www.twitter.com/chestnuthill,http://www.facebook.com/chestnuthillcollege,https://instagram.com/chestnut_hill_college/
    179043,Rockhurst University,https://ww2.rockhurst.edu/,https://ww2.rockhurst.edu/admissions,https://twitter.com/RockhurstU,https://www.facebook.com/RockhurstU?ref=br_tf,http://instagram.com/rockhurstuniversity
    107071,Henderson State University,http://www.hsu.edu/,http://www.hsu.edu/ProspectiveStudents/Admissions/index.html,https://twitter.com/hendersonstateu,https://www.facebook.com/HendersonStateU,http://instagram.com/HendersonStateU
    163578,Notre Dame of Maryland University,https://www.ndm.edu/,https://www.ndm.edu/admissions-aid,https://twitter.com/NotreDameofMD,http://www.facebook.com/NotreDameOfMaryland,https://instagram.com/notredameofmd
    168740,Andrews University,https://www.andrews.edu/,https://www.andrews.edu/admissions/,https://twitter.com/andrewsuniv,https://www.facebook.com/andrewsuniversity,https://instagram.com/andrews_university/
    168342,Williams College,https://www.williams.edu/,https://admission.williams.edu/,https://twitter.com/williamscollege,https://www.facebook.com/williamscollege,https://instagram.com/williamscollege
    170301,Hope College,https://hope.edu/,https://hope.edu/admissions/,https://twitter.com/search?q=GoHope,https://www.facebook.com/hopecollege,https://instagram.com/hopecollege/
    199111,University of North Carolina at Asheville,https://www.unca.edu/,https://www.unca.edu/,https://twitter.com/UncAvl,https://www.facebook.com/UNCAsheville,https://www.instagram.com/unc_asheville/
    101189,Faulkner University,https://www.faulkner.edu/,https://www.faulkner.edu/undergrad/admissions/,https://twitter.com/FaulknerEdu,https://www.facebook.com/FaulknerUniversity,No Instagram found
    190716,D'Youville College,http://www.dyc.edu/,http://www.dyc.edu/admissions/,/twitter,/facebook,/instagram
    237792,Shepherd University,http://www.shepherd.edu/,http://www.shepherd.edu/admissions,https://twitter.com/ShepherdU,https://www.facebook.com/ShepherdUniversity,http://instagram.com/shepherdu
    214069,Misericordia University,https://www.misericordia.edu/,https://www.misericordia.edu/page.cfm?p=783,No Twitter found,https://www.facebook.com/MisericordiaUniversity/videos/1574816025998479/,No Instagram found
    167996,Stonehill College,https://www.stonehill.edu/,https://www.stonehill.edu/admission/,https://twitter.com/stonehill_info,https://www.facebook.com/stonehillcollege,No Instagram found
    218070,Furman University,https://www.furman.edu/,https://www.furman.edu/,https://twitter.com/FurmanU,http://www.facebook.com/FurmanUniversity,http://instagram.com/furmanuniversity/
    206862,Southern Nazarene University,http://snu.edu/,http://snu.edu/admissions,http://twitter.com/FollowSNU,https://www.facebook.com/FollowSNU,No Instagram found
    168218,Wellesley College,https://www.wellesley.edu/,https://www.wellesley.edu/admission,http://www.twitter.com/wellesley,http://www.facebook.com/WellesleyCollege,http://www.instagram.com/wellesleycollege
    175421,Belhaven University,http://www.belhaven.edu/,http://www.belhaven.edu/admissions.htm,https://twitter.com/belhavenu,https://www.facebook.com/belhavenuniversity,https://instagram.com/belhavenu/
    186432,Saint Peter's University,https://www.saintpeters.edu/,https://www.saintpeters.edu/admission/,http://twitter.com/saintpetersuniv,http://www.facebook.com/saintpetersuniversity,https://www.instagram.com/saintpetersuniversity/
    238980,Lakeland College,https://lakeland.edu/,https://lakeland.edu/Academics/admissions,https://twitter.com/LakelandWI,https://www.facebook.com/LakelandUniversityWI,No Instagram found
    240374,University of Wisconsin-Parkside,https://www.uwp.edu/,https://www.uwp.edu/apply/admissions/,https://twitter.com/uwparkside,https://www.facebook.com/universityofwisconsinparkside,https://www.instagram.com/uw_parkside_admissions/
    183974,Centenary College,http://www.centenaryuniversity.edu/,http://www.centenaryuniversity.edu/freshmen/,https://twitter.com/Centenary_NJ,https://www.facebook.com/centenaryuniversity,https://www.instagram.com/centenaryuniversity
    Error: 204635 - Ohio Northern University
    214175,Muhlenberg College,https://www.muhlenberg.edu/,https://www.muhlenberg.edu/main/admissions/,http://twitter.com/muhlenberg,http://www.facebook.com/MuhlenbergCollege,https://www.instagram.com/p/BpZzITfnmHZ/
    126669,Colorado Christian University,https://www.ccu.edu/,https://www.ccu.edu/admissions/,https://twitter.com/my_ccu,https://www.facebook.com/myCCU,https://instagram.com/myccu
    166939,Mount Holyoke College,https://www.mtholyoke.edu/,https://www.mtholyoke.edu/admission,https://twitter.com/mtholyoke,https://www.facebook.com/mountholyokecollege,https://instagram.com/mtholyoke/
    212674,Gettysburg College,http://www.gettysburg.edu/,http://www.gettysburg.edu/admissions/,https://twitter.com/gettysburg,https://www.facebook.com/Gburg.College,http://instagram.com/gettysburgcollege
    237066,Whitworth University,https://www.whitworth.edu/cms/,https://www.whitworth.edu/cms/administration/admissions/,https://twitter.com/whitworth,https://www.facebook.com/whitworthuniversity,https://www.instagram.com/whitworthuniversity/
    139199,Brenau University,https://www.brenau.edu/,https://www.brenau.edu/admissions/,http://twitter.com/BrenauU,https://www.facebook.com/brenauuniversity,http://instagram.com/brenauuniversity?ref=badge
    217536,Salve Regina University,https://salve.edu/,https://salve.edu/admission,https://twitter.com/SalveRegina,http://www.facebook.com/SalveReginaUniversity,http://instagram.com/salveregina/
    147679,North Park University,https://www.northpark.edu/,https://www.northpark.edu/admissions-aid/undergraduate-admissions/,http://www.twitter.com/npu,http://www.facebook.com/npuchicago,http://www.instagram.com/npuchicago
    238458,Carroll University,https://www.carrollu.edu/,https://www.carrollu.edu/admissions,https://twitter.com/carrollu,https://www.facebook.com/carroll.university,https://www.instagram.com/carrolluniversity
    Error: 181783 - Wayne State College
    150534,University of Evansville,https://www.evansville.edu/,https://www.evansville.edu/admission/,https://twitter.com/UEvansville,https://www.facebook.com/UniversityofEvansville,http://instagram.com/uevansville
    203775,Malone University,https://www.malone.edu/,https://www.malone.edu/admissions-aid/,https://twitter.com/MaloneU,https://www.facebook.com/MaloneUniversity/,https://www.instagram.com/malone_u/
    173300,Concordia College at Moorhead,https://www.concordiacollege.edu/,https://www.concordiacollege.edu/admission/meet-your-admissions-representative/,https://twitter.com/concordia_mn,https://www.facebook.com/concordiacollege,http://instagram.com/concordia_mn#
    195526,Skidmore College,https://www.skidmore.edu/,https://www.skidmore.edu/admissions/,https://twitter.com/SkidmoreCollege,https://www.facebook.com/SkidmoreCollege,http://instagram.com/skidmorecollege
    204194,Mount Vernon Nazarene University,https://www.mvnu.edu/,https://www.mvnu.edu/undergraduate,https://twitter.com/MVNUNews,https://www.facebook.com/thisismvnu/,https://www.instagram.com/mvnu1968/
    226231,LeTourneau University,https://www.letu.edu/,https://www.letu.edu/admissions/,https://twitter.com/letourneauuniv,https://www.facebook.com/myletu/,https://www.instagram.com/letourneauuniversity
    150400,DePauw University,https://www.depauw.edu/,https://www.depauw.edu/admission-aid/,http://twitter.com/DePauwU,http://www.facebook.com/DePauwUniversity,https://www.instagram.com/depauwu/ 
    214272,Neumann University,https://www.neumann.edu/,https://www.neumann.edu/admissions/default.asp,https://twitter.com/NeumannUniv,https://www.facebook.com/neumannuniversity,https://www.instagram.com/neumannuniv/
    218733,South Carolina State University,https://www.scsu.edu/,http://www.scsu.edu/admissions.aspx,http://twitter.com/share,https://www.facebook.com/SCState1896,http://instagram.com/scstate1896#
    238476,Carthage College,https://www.carthage.edu/,https://www.carthage.edu/admissions/,https://www.carthage.edu/twitter/,https://www.carthage.edu/facebook/,https://www.carthage.edu/instagram/
    164562,Assumption College,https://www.assumption.edu/,https://www.assumption.edu/admissions,https://twitter.com/AssumptionNews,https://www.facebook.com/assumptioncollege,http://instagram.com/achoundbound/
    213385,Lafayette College,https://www.lafayette.edu/,https://admissions.lafayette.edu/,//twitter.com/LafCol,//www.facebook.com/LafayetteCollege.edu,//instagram.com/lafayettecollege/
    Error: 234207 - Washington and Lee University
    194161,Nyack College,http://www.nyack.edu/,http://www.nyack.edu/content/FCSExplore,http://twitter.com/NyackCollege,http://facebook.com/NyackCollege,http://instagram.com/NyackCollege
    Error: 213783 - Mansfield University of Pennsylvania
    138947,Clark Atlanta University,http://www.cau.edu/,http://www.cau.edu/admissions/admissions-requirements.html,http://twitter.com/cau,https://www.facebook.com/ClarkAtlantaUniversity,http://instagram.com/cau1988
    207582,Oral Roberts University,https://www.oru.edu/,https://www.oru.edu/admissions/,https://twitter.com/OralRobertsU,https://www.facebook.com/OralRobertsUniversity/,https://instagram.com/OralRobertsU
    213987,Mercyhurst University,http://www.mercyhurst.edu/,https://www.mercyhurst.edu/admissions-aid/undergraduate-admissions,https://twitter.com/mercyhurstu,https://www.facebook.com/HurstU,http://instagram.com/mercyhurstu
    175078,Southwest Minnesota State University,https://www.smsu.edu/,http://www.smsu.edu/admission/index.html,https://www.twitter.com/smsutoday,https://www.facebook.com/SMSUToday,https://www.instagram.com/smsuadmission/
    150066,Anderson University,https://www.anderson.edu/,https://www.anderson.edu/admissions,https://twitter.com/AndersonU,https://www.facebook.com/andersonuniversity,https://www.instagram.com/andersonuniversity/
    166850,Merrimack College,https://www.merrimack.edu/,https://www.merrimack.edu/admission/,https://twitter.com/merrimack,https://www.facebook.com/merrimackcollege/,https://www.instagram.com/merrimackcollege/
    174491,University of Northwestern-St Paul,https://unwsp.edu/,https://unwsp.edu/become-a-student/contact-admissions,https://twitter.com/NorthwesternMN,https://www.facebook.com/universityofnorthwestern/,https://www.instagram.com/northwesternmn/
    213996,Messiah College,https://www.messiah.edu/,https://www.messiah.edu/admissions,/twitter,/facebook,https://www.instagram.com/p/BpZ8hv1no0G/
    227331,Our Lady of the Lake University,http://www.ollusa.edu/s/1190/hybrid/18/start-hybrid-ollu.aspx,http://www.ollusa.edu/s/1190/hybrid/18/wide-hybrid-ollu.aspx?sid=1190&gid=1&pgid=7492,https://twitter.com/intent/follow?screen_name=OLLUnivSATX,https://www.facebook.com/OurLadyoftheLakeUniversity,https://www.instagram.com/ollu_saints/
    218061,Francis Marion University,https://www.fmarion.edu/,https://www.fmarion.edu/admissions/,https://twitter.com/FrancisMarionU,https://www.facebook.com/francismarionu/,https://www.instagram.com/francismarionu/
    143084,Augustana College,https://www.augustana.edu/,https://www.augustana.edu/admissions,https://twitter.com/Augustana_IL,https://www.facebook.com/AugustanaCollege,https://instagram.com/augustana_illinois/
    161217,University of Maine at Augusta,https://www.uma.edu/,https://www.uma.edu/,https://twitter.com/umaugusta,https://www.facebook.com/UMAugusta,No Instagram found
    173902,Macalester College,https://www.macalester.edu/,https://www.macalester.edu/admissions/,https://www.twitter.com/macalester,https://www.facebook.com/pages/Saint-Paul-MN/Macalester-College/23345490827,https://www.instagram.com/macalestercollege
    193353,Mount Saint Mary College,https://www.msmc.edu/,https://www.msmc.edu/Admissions,/twitter,/Alumniaffairs/msmc_alumni_facebook_group,/instagram
    195216,St Lawrence University,https://www.stlawu.edu/,https://www.stlawu.edu/admissions,https://twitter.com/stlawrenceu,http://www.facebook.com/StLawrenceU,https://www.instagram.com/stlawrenceu/
    202523,Denison University,https://denison.edu/,https://denison.edu/campus/admission,https://twitter.com/denisonu,https://www.facebook.com/denisonuniversity,https://www.instagram.com/denisonu
    209506,Oregon Institute of Technology,https://www.oit.edu/,https://www.oit.edu/admissions,https://twitter.com/OregonTech,https://www.facebook.com/OregonTech,https://www.instagram.com/oregontech/
    184694,Fairleigh Dickinson University-College at Florham,https://www.fdu.edu/,https://view2.fdu.edu/admissions/,https://twitter.com/FDUWhatsNew,https://www.facebook.com/fairleighdickinsonuniversity,https://instagram.com/fduwhatsnew#
    155520,MidAmerica Nazarene University,https://www.mnu.edu/,https://www.mnu.edu/undergraduate/admissions,http://twitter.com/followMNU,https://www.facebook.com/MNUPioneers/,https://www.instagram.com/mnupioneers/
    184773,Georgian Court University,https://georgian.edu/,https://georgian.edu/admissions/,https://twitter.com/Georgiancourt,https://www.facebook.com/georgiancourtu/,https://www.instagram.com/georgiancourt/
    212577,Franklin and Marshall College,https://www.fandm.edu/,https://www.fandm.edu/admission,http://twitter.com/FandMCollege,/alumni-connections/alumni-facebook-pages,No Instagram found
    217493,Rhode Island School of Design,https://www.risd.edu/,https://www.risd.edu/admissions/,http://twitter.com/risd,http://facebook.com/risd1877,https://www.instagram.com/risd1877/
    130934,Delaware State University,https://www.desu.edu/,https://www.desu.edu/admissions,https://twitter.com/DelStateUniv,https://www.facebook.com/DESUedu,http://instagram.com/delstateuniv
    189097,Barnard College,https://barnard.edu/,https://admissions.barnard.edu/,https://twitter.com/BarnardCollege,https://www.facebook.com/barnardcollege,http://instagram.com/barnardcollege
    Error: 153834 - Luther College
    204936,Otterbein University,http://www.otterbein.edu/,http://www.otterbein.edu/public/FutureStudents/Undergraduate/AdmissionCounselors.aspx,http://twitter.com/otterbein,http://www.facebook.com/pages/Otterbein-University/154106131291768,http://instagram.com/OtterbeinU
    210775,Alvernia University,https://www.alvernia.edu/,https://www.alvernia.edu/admissions-aid,https://twitter.com/intent/follow?source=followbutton&variant=1.0&screen_name=AlverniaUniv,https://www.facebook.com/AlverniaUniversity,https://instagram.com/alverniauniversity
    239080,Marian University,https://www.marianuniversity.edu/,https://www.marianuniversity.edu/admission-financial-aid/freshman-students/admissions/,https://twitter.com/marian_wi,https://www.facebook.com/marianuniversitywi,https://www.instagram.com/marianuniversitywi/
    127185,Fort Lewis College,https://www.fortlewis.edu/,https://www.fortlewis.edu/Home/Admission/AdmissiontoFortLewisCollege.aspx,https://twitter.com/FLCDurango,http://www.facebook.com/FortLewis,https://www.instagram.com/p/BpaJHA2HfJ_/
    Error: 216694 - Waynesburg University
    204617,Ohio Dominican University,https://www.ohiodominican.edu/,https://www.ohiodominican.edu/admissions,http://twitter.com/OhioDominican,http://www.facebook.com/OhioDominican,http://instagram.com/ohiodominican
    198613,Guilford College,https://www.guilford.edu/,https://www.guilford.edu/admissions,https://twitter.com/GuilfordCollege,https://www.facebook.com/guilfordcollege/,https://instagram.com/guilfordcollege/
    161086,Colby College,https://www.colby.edu/,https://www.colby.edu/admission/,http://www.twitter.com/colbycollege,http://www.facebook.com/colbycollege,https://www.instagram.com/thiscolbylife/
    201654,Cedarville University,https://www.cedarville.edu/,https://www.cedarville.edu/Admissions.aspx,https://twitter.com/cedarville,https://www.facebook.com/cedarville/,https://www.instagram.com/cedarville/
    151786,Marian University,https://www.marian.edu/,https://marian.edu/admissions,http://twitter.com/marianuniv,https://www.facebook.com/marianuniversity,http://instagram.com/marianuniversity
    130590,Trinity College,https://www.trincoll.edu/,https://www.trincoll.edu/admissions/,https://twitter.com/trinitycollege?lang=en,https://www.facebook.com/TrinityCollege/,https://www.instagram.com/trinitycollege/
    232609,Lynchburg College,https://www.lynchburg.edu/,https://www.lynchburg.edu/undergraduate-admission/,https://twitter.com/lynchburg,https://www.facebook.com/university.of.lynchburg/,https://www.instagram.com/university.of.lynchburg/
    177418,Fontbonne University,https://www.fontbonne.edu/,https://www.fontbonne.edu/admission/,http://www.twitter.com/FontbonneU,http://www.facebook.com/Fontbonne,http://instagram.com/fontbonneu
    192192,Keuka College,https://www.keuka.edu/,https://www.keuka.edu/admissions/freshman,http://www.twitter.com/keukacollege,http://www.facebook.com/keukacollege,http://www.instagram.com/keukacollege
    226471,University of Mary Hardin-Baylor,https://go.umhb.edu/,https://go.umhb.edu/admissions/home,https://twitter.com/umhb,https://facebook.com/umhb,https://go.umhb.edu/news/2018/instagram-feed-leads-alumna-to-syrian-refugee-ministry
    211431,Carlow University,https://www.carlow.edu/,https://www.carlow.edu/Admissions.aspx,http://twitter.com/CarlowU,https://www.facebook.com/CarlowUniversity,https://instagram.com/carlowuniversity/
    196112,SUNY Institute of Technology at Utica-Rome,https://sunypoly.edu/,https://sunypoly.edu/,https://twitter.com/SUNYPolyInst,https://www.facebook.com/sunypolytechnic,https://www.instagram.com/p/BpcUC1ngSeO/
    198950,Meredith College,https://www.meredith.edu/,https://www.meredith.edu/admissions,https://twitter.com/meredithcollege,https://www.facebook.com/MeredithCollege,http://instagram.com/meredithcollege/
    212009,Dickinson College,http://www.dickinson.edu/,http://www.dickinson.edu/homepage/287/admissions,http://twitter.com/DickinsonCol,https://www.facebook.com/Dickinson-College-6628797891/,http://instagram.com/dickinsoncollege
    173647,Gustavus Adolphus College,https://gustavus.edu/,https://gustavus.edu/admission/,https://twitter.com/gustavus,https://www.facebook.com/gustavusadolphuscollege,https://instagram.com/gustavusadolphuscollege
    170639,Lake Superior State University,https://www.lssu.edu/,https://www.lssu.edu/admissions/,https://twitter.com/lifeatlssu,http://www.facebook.com/lakestate,https://www.instagram.com/lakestateu/
    194958,Roberts Wesleyan College,https://www.roberts.edu/,https://www.roberts.edu/undergraduate/admissions.aspx,http://twitter.com/RobertsWesleyan,http://www.facebook.com/RobertsWesleyanCollege,http://instagram.com/RobertsWesleyan
    217688,Charleston Southern University,https://www.charlestonsouthern.edu/,https://www.charlestonsouthern.edu/admissions/,http://www.twitter.com/csuniv,http://www.facebook.com/charlestonsouthernuniversity,http://instagram.com/charlestonsouthern
    107141,John Brown University,https://www.jbu.edu/,https://www.jbu.edu/admissions/,https://twitter.com/johnbrownuniv,https://www.facebook.com/JohnBrownUniversity/,https://www.instagram.com/johnbrownuniversity/
    150145,Bethel College-Indiana,https://www.bethelcollege.edu/,https://www.bethelcollege.edu/admission-aid,https://www.twitter.com/BethelCollegeIN,https://facebook.com/profile.php?id=22839919973,https://www.instagram.com/bethelcollege
    120254,Occidental College,https://www.oxy.edu/,https://www.oxy.edu/admission-aid,https://twitter.com/occidental,https://www.facebook.com/occidental,https://www.instagram.com/occidentalcollege/
    197197,Wagner College,http://wagner.edu/,http://wagner.edu/admissions/,https://twitter.com/wagnercollege,https://facebook.com/wagnercollege,https://instagram.com/wagnercollege
    182795,Franklin Pierce University,https://www.franklinpierce.edu/,https://www.franklinpierce.edu/admissions/,https://twitter.com/FPUniversity,https://www.facebook.com/Franklin-Pierce-University-55894844572/,https://www.instagram.com/fpuadmissions/
    191630,Hobart William Smith Colleges,https://www2.hws.edu/,https://www.hws.edu/admissions/,http://www.twitter.com/hwscolleges,http://www.facebook.com/hwscolleges,http://instagram.com/hwscolleges
    204200,College of Mount St Joseph,https://www.msj.edu/,https://www.msj.edu/admission/overview5,https://twitter.com/MountStJosephU/,https://www.facebook.com/MountStJosephU/,https://www.instagram.com/MountStJosephU/
    Error: 188641 - Alfred University
    154350,Simpson College,https://simpson.edu/,https://simpson.edu/admission-aid,https://www.twitter.com/simpsoncollege,https://www.facebook.com/SimpsonCollege/,No Instagram found
    163295,Maryland Institute College of Art,https://www.mica.edu/,https://www.mica.edu/applying-to-mica/,http://www.twitter.com/mica,http://www.facebook.com/mica.edu,https://www.instagram.com/marylandinstitutecollegeofart/
    133492,Eckerd College,https://www.eckerd.edu/,https://www.eckerd.edu/admissions/,https://www.twitter.com/eckerdcollege,https://www.facebook.com/eckerdcollege,https://www.instagram.com/eckerdlife
    239318,Milwaukee School of Engineering,https://www.msoe.edu/,https://www.msoe.edu/admissions-aid/,https://twitter.com/MSOE,http://www.facebook.com/milwaukeeschoolofengineering,https://www.instagram.com/msoephoto/
    110404,California Institute of Technology,http://www.caltech.edu/,http://www.caltech.edu/content/admissions,http://twitter.com/caltech,http://www.facebook.com/californiainstituteoftechnology,https://instagram.com/caltechedu
    134079,Florida Southern College,https://www.flsouthern.edu/,https://www.flsouthern.edu/undergraduate/admissions.aspx,https://twitter.com/flsouthern/,https://www.facebook.com/FloridaSouthern,https://instagram.com/floridasouthern
    128498,Albertus Magnus College,https://www.albertus.edu/,https://www.albertus.edu/admission-aid/,//twitter.com/AlbertusSocial,//www.facebook.com/AlbertusMagnusCT,//instagram.com/albertussocial
    221953,Tusculum College,https://home.tusculum.edu/,https://home.tusculum.edu/tusculum-university-admission/,http://www.twitter.com/tusculum_univ,http://www.facebook.com/tusculum.univ,https://instagram.com/tusculum.univ/
    126678,Colorado College,https://www.coloradocollege.edu/,https://www.coloradocollege.edu/admission/,https://twitter.com/coloradocollege,https://www.facebook.com/coloradocollege,https://instagram.com/coloradocollege/
    191515,Hamilton College,https://www.hamilton.edu/,https://www.hamilton.edu/admission,//twitter.com/hamiltoncollege,//www.facebook.com/HamiltonCollege,//www.instagram.com/hamiltoncollege/
    137564,Southeastern University,https://www.seu.edu/,https://www.seu.edu/admission/,https://twitter.com/seuniversity,https://www.facebook.com/seuniversity,https://www.instagram.com/seuniversity/
    212656,Geneva College,https://www.geneva.edu/,https://www.geneva.edu/admissions/,http://twitter.com/GenevaCollege,news/2018/10/nr-your-fafsa-toolbox-facebook-live.php,https://instagram.com/genevacollege/
    177339,Evangel University,https://www.evangel.edu/,https://www.evangel.edu/admissions/,https://twitter.com/evangeluniv,https://www.facebook.com/evangeluniversity,https://instagram.com/evangel_university/
    225399,Houston Baptist University,https://www.hbu.edu/,https://www.hbu.edu/admissions/,http://twitter.com/HoustonBaptistU,http://facebook.com/HoustonBaptistUniversity,http://instagram.com/houstonbaptistuniversity
    215105,The University of the Arts,https://www.uarts.edu/,https://www.uarts.edu/admissions,https://twitter.com/uarts,https://www.facebook.com/uarts,https://www.instagram.com/p/BkGcY6kFxk8/?hl=en&taken-by=beyonce
    190761,Dominican College of Blauvelt,https://www.dc.edu/,https://www.dc.edu/admission-aspx/,https://twitter.com/DominicanOburg,https://www.facebook.com/pages/Dominican-College/83517950904,https://instagram.com/dominicancollege
    154013,Mount Mercy University,https://www.mtmercy.edu/,https://www.mtmercy.edu/admissions,http://twitter.com/MountMercy,http://www.facebook.com/mountmercyuniversity,No Instagram found
    207324,Oklahoma Christian University,http://oc.edu/,http://oc.edu/admissions/index.html,http://www.twitter.com/okchristian/,http://www.facebook.com/oklahomachristian/,https://instagram.com/ok_christian/
    216278,Susquehanna University,https://www.susqu.edu/,https://www.susqu.edu/admission-and-aid,https://twitter.com/susquehannau,https://www.facebook.com/SusquehannaU,https://instagram.com/susquehannau/
    153375,Grand View University,https://www.grandview.edu/,https://www.grandview.edu/admissions,https://twitter.com/grandviewuniv,https://www.facebook.com/Grand-View-University-315068091675/,https://www.instagram.com/explore/locations/4007904/grand-view-university/
    181446,Nebraska Wesleyan University,https://www.nebrwesleyan.edu/,https://www.nebrwesleyan.edu/,http://www.twitter.com/NEWesleyan,http://www.facebook.com/NebraskaWesleyan,https://instagram.com/newesleyan
    238193,Alverno College,https://www.alverno.edu/,https://www.alverno.edu/admissions/apply.php,https://twitter.com/alvernocollege,https://facebook.com/alvernocollege,https://www.instagram.com/alvernocollege/
    184348,Drew University,https://www.drew.edu/1/,http://www.drew.edu/admissions-aid/undergraduate-admissions/,https://twitter.com/drewuniversity,https://www.facebook.com/DrewUniversity/,https://www.instagram.com/drewuniversity/
    145646,Illinois Wesleyan University,https://www.iwu.edu/,https://www.iwu.edu/admissions/,https://twitter.com/IL_Wesleyan,https://www.facebook.com/illinoiswesleyan,No Instagram found
    128902,Connecticut College,https://www.conncoll.edu/,https://www.conncoll.edu/admission/,http://www.twitter.com/ConnCollege,https://www.facebook.com/ConnecticutCollege,http://instagram.com/ConnCollege
    173258,Carleton College,https://www.carleton.edu/,https://www.carleton.edu/admissions/,//twitter.com/CarletonCollege,//www.facebook.com/CarletonCollege,http://instagram.com/carletoncollege
    101709,University of Montevallo,https://www.montevallo.edu/,https://www.montevallo.edu/admissions-aid/undergraduate-admissions/,https://twitter.com/Montevallo,https://www.facebook.com/UMontevallo/,https://www.instagram.com/p/BpaMxwCnYlD/
    147244,Millikin University,https://millikin.edu/,https://millikin.edu/admission-aid,http://www.twitter.com/MillikinU,http://www.facebook.com/millikinuniversity,http://www.instagram.com/millikinu
    204185,University of Mount Union,https://www.mountunion.edu/,https://www.mountunion.edu/admission,https://twitter.com/mountunion,https://www.facebook.com/UniversityofMountUnion/,https://www.instagram.com/mountunion
    210571,Albright College,https://www.albright.edu/,https://www.albright.edu/admission-aid/,https://www.twitter.com/albrightcollege/,https://www.facebook.com/AlbrightCollege/,https://instagram.com/AlbrightCollege/
    139144,Berry College,https://www.berry.edu/,https://www.berry.edu/admission/,http://twitter.com/berrycollege,http://www.facebook.com/berrycollege,http://instagram.com/berrycollege
    210669,Allegheny College,https://allegheny.edu/,https://sites.allegheny.edu/admissions/,https://twitter.com/alleghenycol?lang=en,https://www.facebook.com/alleghenycollege/,https://www.instagram.com/alleghenycollege/
    160904,Xavier University of Louisiana,http://xula.edu/,http://www.xula.edu/applyanddepositundergraduate,No Twitter found,https://www.facebook.com/XULA1925,http://www.instagram.com/xula1925
    152530,Taylor University,https://www.taylor.edu/,https://www.taylor.edu/admission,https://twitter.com/tayloru/,https://facebook.com/tayloruniversity/,https://instagram.com/tayloruniv/
    219000,Augustana College,http://www.augie.edu/,https://www.augie.edu/admission,https://twitter.com/augustanasd,https://www.facebook.com/pg/AugustanaSD/photos/?tab=album&album_id=10156041369236219,https://instagram.com/augustanasd
    152318,Rose-Hulman Institute of Technology,https://www.rose-hulman.edu/,https://www.rose-hulman.edu/admissions-and-aid/,https://twitter.com/rosehulman,https://www.facebook.com/Rose-Hulman-Institute-of-Technology-10046401301/?ref=ts,https://www.instagram.com/rosehulman/
    237932,West Liberty University,https://westliberty.edu/,https://westliberty.edu/admissions/,https://twitter.com/westlibertyu,https://www.facebook.com/WestLibertyUniversity/,https://instagram.com/discoverwestlib
    204909,Ohio Wesleyan University,https://www.owu.edu/,https://www.owu.edu/admission/,https://twitter.com/OhioWesleyan,https://www.facebook.com/OhioWesleyanUniversity,https://instagram.com/ohiowesleyan/
    225247,Hardin-Simmons University,https://www.hsutx.edu/,https://www.hsutx.edu/admissions/,https://twitter.com/HSUTX,https://www.facebook.com/hardin.simmons.university/posts/,https://www.instagram.com/hsutx/
    140553,Morehouse College,http://www.morehouse.edu/,http://www.morehouse.edu/admissions/,https://twitter.com/Morehouse,https://www.facebook.com/Morehouse1867/,No Instagram found
    211468,Cedar Crest College,https://www.cedarcrest.edu/,https://www.cedarcrest.edu/traditional/index.shtm,https://www.twitter.com/cedarcrestcolle,https://www.facebook.com/127294303966359/posts/2335661199796314,https://www.instagram.com/evergreensoccerclub
    110370,California College of the Arts,https://www.cca.edu/,https://www.cca.edu/admissions,http://www.twitter.com/CACollegeofArts,http://www.facebook.com/CaliforniaCollegeoftheArts,http://instagram.com/cacollegeofarts
    212197,Elizabethtown College,https://www.etown.edu/,https://www.etown.edu/admissions/,http://twitter.com/etowncollege,http://www.facebook.com/etowncollege,http://instagram.com/etowncollege
    216524,Ursinus College,https://www.ursinus.edu/,https://www.ursinus.edu/admission/,http://twitter.com/UrsinusAdmit,No Facebook found,No Instagram found
    198385,Davidson College,https://www.davidson.edu/,https://www.davidson.edu/admission-and-financial-aid,https://www.twitter.com/DavidsonCollege,https://facebook.com/davidsoncollege.nc,https://instagram.com/davidsoncollege
    218229,Lander University,https://www.lander.edu/,https://www.lander.edu/admissions,http://twitter.com/follow_lander,http://www.facebook.com/followlander,https://instagram.com/landeruniversity/
    214157,Moravian College,https://www.moravian.edu/,https://www.moravian.edu/admissions,https://twitter.com/MoravianCollege,https://www.facebook.com/99087857199_10156735740487200,https://www.instagram.com/p/BoWvda5jiC3/
    162283,Coppin State University,https://www.coppin.edu/,https://www.coppin.edu/admissions,http://twitter.com/coppinstateuniv,https://www.facebook.com/coppinstateuniversity/videos/526987977763884/,https://www.instagram.com/coppinstateuniversity/
    215284,University of Pittsburgh-Johnstown,http://www.upj.pitt.edu/,http://www.upj.pitt.edu/en/admissions/admissions/,http://twitter.com/PittJohnstown,https://www.facebook.com/PittJohnstown,http://instagram.com/pitt_johnstown
    237330,Concord University,https://www.concord.edu/,https://www.concord.edu/admissions/node/1,https://twitter.com/CampusBeautiful,https://www.facebook.com/concorduniversity,https://www.instagram.com/concorduniversity/
    Error: 128391 - Western State Colorado University
    211981,Delaware Valley College,https://www.delval.edu/,https://www.delval.edu/admission,http://twitter.com/DelVal,http://www.facebook.com/delval,http://www.instagram.com/delawarevalleyuniversity
    203535,Kenyon College,https://www.kenyon.edu/,https://www.kenyon.edu/admissions-aid/,https://twitter.com/KenyonCollege,https://www.facebook.com/kenyoncollege,https://www.instagram.com/p/Bo9ooXGHmhn/
    140960,Savannah State University,https://www.savannahstate.edu/,https://www.savannahstate.edu/undergraduate/,https://twitter.com/savannahstate,https://www.facebook.com/savannahstate,https://www.instagram.com/savannahstate/
    174792,Saint Johns University,https://www.csbsju.edu/about/saint-johns-university,https://www.csbsju.edu/admission/contactus,https://twitter.com/CSBSJU,https://www.facebook.com/csbsju,https://instagram.com/csbsju
    206589,The College of Wooster,https://www.wooster.edu/,https://www.wooster.edu/admissions/,https://twitter.com/woosteredu,https://www.facebook.com/CollegeofWooster,https://instagram.com/wooinsider/
    218973,Wofford College,https://www.wofford.edu/,https://www.wofford.edu/admission/,http://twitter.com/woffordcollege,http://www.facebook.com/woffordcollege,http://instagram.com/woffordcollege
    Error: 102377 - Tuskegee University
    154527,Wartburg College,http://www.wartburg.edu/,http://www.wartburg.edu/admissions/,https://twitter.com/WartburgCollege,http://facebook.com/WartburgCollege,http://instagram.com/wartburgcollege
    166674,Massachusetts College of Art and Design,https://massart.edu/,https://massart.edu/admissions,https://twitter.com/MassArt,https://www.facebook.com/MassArtBoston/,https://www.instagram.com/massartboston/
    219806,Carson-Newman University,https://www.cn.edu/,https://www.cn.edu/admissions,http://twitter.com/cneagles,https://www.facebook.com/media/set/?set=a.10157059181044467&type=3&__xts__[0]=68.ARCWsqnyMCmy5JyJX8uSLRNViQU6IJeFjTpDgWJGRvPQB3OZJgzK1fOjJZtu2mWojBffy0H8I_PZ9HgFqPKHwuk44qVNKCy3dTVJlnvk9xHaB70VeC1ACTeoSAunAvI7ZbcN6D3K4mpmsSIsPg8NgnCzTjoUPGRqESRSL3KO6egVZY,http://instagram.com/carsonnewman
    120184,Notre Dame de Namur University,https://www.ndnu.edu/,http://www.ndnu.edu/admissions/,https://twitter.com/NDNU,https://www.facebook.com/NDNUBelmont,https://www.instagram.com/ndnu_argos/
    Error: 161226 - University of Maine at Farmington
    183239,Saint Anselm College,https://www.anselm.edu/,https://www.anselm.edu/admission-aid,http://www.twitter.com/saintanselm,http://www.facebook.com/saintanselm,https://www.instagram.com/saintanselm/
    121345,Pomona College,https://www.pomona.edu/,https://www.pomona.edu/admissions,No Twitter found,https://www.facebook.com/PomonaAdmissions,No Instagram found
    167288,Massachusetts College of Liberal Arts,http://www.mcla.edu/,https://dynamicforms.ngwebsolutions.com/Error.aspx?aspxerrorpath=/Submit/Form/Start/bc0b32dd-66f3-488e-a8c1-c4de293226b9,https://twitter.com/MCLA_EDU,https://www.facebook.com/pages/MCLA/82654875561,http://instagram.com/mcla_edu
    106412,University of Arkansas at Pine Bluff,http://www.uapb.edu/,https://www.uapb.edu/admissions.aspx,http://www.twitter.com/uapbinfo,http://www.facebook.com/uapinebluff,http://www.instagram.com/uapb
    Error: 191676 - Houghton College
    146481,Lake Forest College,https://www.lakeforest.edu/,https://www.lakeforest.edu/admissions/,https://twitter.com/lfcollege,https://www.facebook.com/lakeforestcollege,https://instagram.com/lakeforestcollege/
    152390,Saint Mary's College,https://www.saintmarys.edu/,https://www.saintmarys.edu/admission-aid/first-year-students/connect-with-saint-marys/admission,https://twitter.com/saintmarys,https://www.facebook.com/saintmaryscollege,https://instagram.com/saintmaryscollege/
    196291,SUNY Maritime College,https://www.sunymaritime.edu/,https://www.sunymaritime.edu/admissions,https://twitter.com/MaritimeCollege?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor,https://www.facebook.com/sunymaritimecollege/,No Instagram found
    237899,West Virginia State University,http://www.wvstateu.edu/,http://www.wvstateu.edu/Admissions.aspx,https://twitter.com/WVStateU,https://www.facebook.com/wvstateu,https://www.instagram.com/wvstateu/
    Error: 153384 - Grinnell College
    206525,Wittenberg University,https://www.wittenberg.edu/,https://www.wittenberg.edu/admission,https://twitter.com/wittenberg,https://www.facebook.com/wittenberguniversity,https://www.instagram.com/wittenberguniversity/
    192864,Marymount Manhattan College,https://www.mmm.edu/,https://www.mmm.edu/admissions/,http://twitter.com/NYCMarymount,http://www.facebook.com/MarymountManhattan,http://instagram.com/nycmarymount
    203845,Marietta College,https://www.marietta.edu/,https://www.marietta.edu/contact-us,https://twitter.com/MariettaCollege,https://www.facebook.com/MariettaCollege,https://www.instagram.com/mariettacollege
    216667,Washington & Jefferson College,https://www.washjeff.edu/,https://www.washjeff.edu/future-students,http://www.twitter.com/wjcollege,http://www.facebook.com/washjeff,No Instagram found
    237057,Whitman College,https://www.whitman.edu/,https://www.whitman.edu/admission-and-aid,https://twitter.com/whitmancollege,https://facebook.com/whitmancollege,https://instagram.com/whitmancollege
    221351,Rhodes College,https://www.rhodes.edu/,https://www.rhodes.edu/admission,https://twitter.com/RhodesCollege,https://www.facebook.com/rhodescollege,https://instagram.com/rhodescollege/
    Error: 153108 - Central College
    109651,Art Center College of Design,http://www.artcenter.edu/,http://www.artcenter.edu/admissions/overview.html,https://twitter.com/artcenteredu,https://www.facebook.com/artcenteredu/,https://www.instagram.com/artcenteredu/
    199209,North Carolina Wesleyan College,https://ncwc.edu/,https://ncwc.edu/admissions/,https://twitter.com/ncwesleyan,https://www.facebook.com/NorthCarolinaWesleyanCollege,https://www.instagram.com/ncwesleyan
    146427,Knox College,https://www.knox.edu/,https://www.knox.edu/admission,No Twitter found,https://www.facebook.com/KnoxCollege,http://instagram.com/knoxcollege1837
    228343,Southwestern University,https://www.southwestern.edu/,https://www.southwestern.edu/admission/,http://twitter.com/southwesternu,http://www.facebook.com/SouthwesternUniversity,http://instagram.com/southwesternu
    Error: 234085 - Virginia Military Institute
    125727,Westmont College,https://www.westmont.edu/,https://www.westmont.edu/admissions-aid,https://twitter.com/westmontnews,https://www.facebook.com/Westmont,https://www.instagram.com/westmontcollege/
    120865,Pacific Union College,https://www.puc.edu/,https://www.puc.edu/admissions,http://twitter.com/pucnow,https://www.facebook.com/pacificunioncollege,https://www.instagram.com/pucnow
    209922,Reed College,https://www.reed.edu/,https://www.reed.edu/apply/,https://twitter.com/Reed_College_,https://www.facebook.com/reedcollege,https://instagram.com/reedcollege
    197984,Belmont Abbey College,https://belmontabbeycollege.edu/,https://belmontabbeycollege.edu/admissions/,https://twitter.com/BelmontAbbey,https://www.facebook.com/belmontabbey/,https://www.instagram.com/belmontabbey/?hl=en
    100937,Birmingham Southern College,https://www.bsc.edu/,https://www.bsc.edu/admission/,https://twitter.com/FromTheHilltop,https://www.facebook.com/43199567384/posts/10155533542147385,https://www.instagram.com/p/BmwOlGPnwS3/
    101435,Huntingdon College,https://www.huntingdon.edu/,https://www.collegesimply.com/colleges/alabama/huntingdon-college/admission/,https://twitter.com/HuntingdonColl,https://www.facebook.com/HuntingdonCollege,http://instagram.com/huntingdoncollege
    115409,Harvey Mudd College,https://www.hmc.edu/,https://www.hmc.edu/admission/,https://twitter.com/harveymudd,https://www.facebook.com/harveymuddcollege,https://www.instagram.com/harvey_mudd
    
