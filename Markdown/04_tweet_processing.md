
# How to Process Tweets for Text Analysis

### Objective: Process Tweets for text analysis including topic modelling and wordcloud visualization. Processing includes separating words within hashtags, removing entity-specific words, stop words, and lemmatization.

In the previous step we obtained all tweets for our sample of colleges and universities. Now, however, we must process the messy tweet text for analysis. There are many different approaches to cleaning tweet data for text analysis and topic modelling. For example, some authors combine Tweets with the same hashtag into a single document. Upon inspection, however,  I decided that would not be helpful in the context of college and university's social media use. This is because many universities use hashtags that are just that college's name (too generic and frequent) e.g. #somecollege or specific university hashtags that are not used frequently (e.g. #somecollegehomecoming). 

Additionally, some colleges use only a hashtag when mentioning the topic I am most interested in for this work -- #diversity, while others do not use a hashtag, e.g. "We support #womenoncampus" vs "We support women on campus." Because of this, I use the `wordsegment` package to attempt to split multi-word hashtags. Note however that this is not a perfect process when words overlap. 

Finally, because this analysis is used in words used across colleges, not just at a particular college, I remove words that  are only used by one college in the data. This helps with the problem mentioned above where colleges hashtag their own names. Their own names will occur frequently in the data, but not frequently enough to be removed by standard text processing techniques, such as removing words that occur in over a certain percentage of documents.

## Set Up

### Load packages


```python
import pandas as pd
```


```python
import numpy as np
```


```python
from collections import defaultdict
```

The `preprocessor` package will remove emojis, mentions, and URLs from tweets.


```python
# Note: There is a bug installing this package in windows that was fixed by a kind user on 
# Github. Windows users should download as follows
# pip install git+git://github.com/iamRusty/preprocessor
import preprocessor as pre
```


```python
import re, glob, datetime
```

The `wordsegment` package splits multiword hashtags into individual words.


```python
from wordsegment import load, segment
```


```python
# Help with memory issues
import gc
```

The `unicodedata` package helps to convert unicode to ascii.


```python
import unicodedata2
```

The `nltk` and `gensim` packages are used for stemming and removing stop words.


```python
# Tools for LDA
import gensim
from gensim.utils import simple_preprocess
from gensim.parsing.preprocessing import STOPWORDS
from nltk.stem import WordNetLemmatizer, SnowballStemmer
from nltk.stem.porter import *
import nltk
```

    C:\Users\laure\Anaconda3\lib\site-packages\gensim\utils.py:1197: UserWarning: detected Windows; aliasing chunkize to chunkize_serial
      warnings.warn("detected Windows; aliasing chunkize to chunkize_serial")
    


```python
nltk.download('wordnet')
```

    [nltk_data] Downloading package wordnet to
    [nltk_data]     C:\Users\laure\AppData\Roaming\nltk_data...
    [nltk_data]   Package wordnet is already up-to-date!
    




    True




```python
nltk.download('stopwords')
```

    [nltk_data] Downloading package stopwords to
    [nltk_data]     C:\Users\laure\AppData\Roaming\nltk_data...
    [nltk_data]   Package stopwords is already up-to-date!
    




    True



## Functions to Clean the Tweet Text


```python
# Test data 
twitter_files = glob.glob('../data/twitter_data/*.json')
test_file = twitter_files[1]
test_df = pd.read_json(test_file, encoding='utf-8', lines=True)
```


```python
# Test tweet
test_tweet = test_df.tweet[99]
print(test_tweet)
```

    #UAHDiscoveryDay2017 Group Silver: Your first session is the Welcome! Join us in the Charger Union Theater for a quick welcome and rundown of your day on campus.
    


```python
# Load segmentation dictionary
load()
# Set options for Twitter processing - remove url, mention, and emojis
pre.set_options(pre.OPT.URL, pre.OPT.EMOJI, pre.OPT.MENTION, pre.OPT.SMILEY)
```

The following function will clean the hashtags, removing hashes and numbers, as well as segmenting out individual words.


```python
# Hash fix - will be called from within tweet cleaning function
def hash_fix(h):
    h1 = re.sub(r'[0-9]+', '', h)
    h2 = re.sub(r'#', '', h1)
    h3 = segment(str(h2))
    h4 = ' '.join(map(str, h3)) 
    return h4
```


```python
hash_fix('#UAHDiscoveryDay2017')
```




    'uah discovery day'



Create a dictionary that contains each hash tag and the text it should be replaced with.


```python
# Inputs: dataframe with the tweets and the column with the hashtags
def hash_dict(df,hash_col):
    # Create a datafame of all hashtags in a column and their counts
    # Note: hashtags are in lists inside a cell e.g. [#hash1, #hash2] 
    tag_counts = df[hash_col].apply(pd.Series).stack().value_counts().to_frame()
    tag_counts = tag_counts.reset_index()
    tag_counts.columns = ['hash','freq']
    # Remove numbers and segment multiple words using hash fix
    tag_counts = tag_counts.assign(clean_tag = tag_counts.hash.apply(lambda x: hash_fix(x)))
    # Create a dictionary of the hashtags and their clean strings
    tag_counts.set_index('hash', inplace=True)
    tag_dict = tag_counts['clean_tag'].to_dict()
    return tag_dict
```


```python
tag_dict = hash_dict(test_df,'hashtags')
tag_dict
```




    {'#chargernation': 'charger nation',
     '#uahuntsville': 'ua huntsville',
     '#uahopenhouse': 'uah open house',
     '#gochargers': 'go chargers',
     '#foundmyhome': 'found my home',
     '#chargeon': 'charge on',
     '#uah20': 'uah',
     '#uahorientation': 'uah orientation',
     '#awesome': 'awesome',
     '#uah17': 'uah',
     '#uah': 'uah',
     '#uahadmittedstudentday': 'uah admitted student day',
     '#uah18': 'uah',
     '#uahchargerpreview': 'uah charger preview',
     '#uahdiscoverydays': 'uah discovery days',
     '#uahroadtrip': 'uah road trip',
     '#uahtour': 'uah tour',
     '#uahdiscoveryday2017': 'uah discovery day',
     '#thisismyuah': 'this is my uah',
     '#chargerpride': 'charger pride',
     '#uah16': 'uah',
     '#uahnso': 'uah n so',
     '#uah19': 'uah',
     '#uah2022': 'uah',
     '#uah17facebook': 'uah facebook',
     '#uahbasketballbash': 'uah basketball bash',
     '#protip': 'pro tip',
     '#ff': 'ff',
     '#uahadmissions': 'uah admissions',
     '#foundmyhome16': 'found my home',
     '#universityofawesomeinhuntsville': 'university of awesome in huntsville',
     '#uahol': 'ua hol',
     '#myuah': 'my uah',
     '#uah21': 'uah',
     '#nomnoms': 'nom noms',
     '#puckapalooza': 'pucka palooza',
     '#uahvisit': 'uah visit',
     '#uah2016': 'uah',
     '#parentsessions': 'parent sessions',
     '#swag': 'swag',
     '#yourock': 'you rock',
     '#college': 'college',
     '#futurechargers': 'future chargers',
     '#followfriday': 'follow friday',
     '#uah22': 'uah',
     '#uahsummerevents': 'uah summer events',
     '#excited': 'excited',
     '#tbt': 'tbt',
     '#chargerpreview2018': 'charger preview',
     '#teamrockets': 'team rockets',
     '#ussrc': 'us src',
     '#destinationchargernation': 'destination charger nation',
     '#hunwx': 'hun wx',
     '#swagbag': 'swag bag',
     '#collegebound': 'collegebound',
     '#spooky': 'spooky',
     '#chargerblue': 'charger blue',
     '#win': 'win',
     '#pumped': 'pumped',
     '#blueschools': 'blue schools',
     '#huntsvillebound': 'huntsville bound',
     '#orientation': 'orientation',
     '#uahunstville': 'uah unst ville',
     '#gsc': 'gsc',
     '#battleofthebuffalo': 'battle of the buffalo',
     '#alwx': 'al wx',
     '#beautiful': 'beautiful',
     '#huntsville': 'huntsville',
     '#lunch': 'lunch',
     '#nom': 'nom',
     '#chargersat1': 'chargers at',
     '#uahchargers': 'uah chargers',
     '#parentsession': 'parent session',
     '#cwl': 'cwl',
     '#brightandearly': 'bright and early',
     '#welcome': 'welcome',
     '#runbabyrun': 'run baby run',
     '#high5friday': 'high friday',
     '#teambluebabies': 'team blue babies',
     '#o2santa': 'o santa',
     '#goblueandwhite': 'go blue and white',
     '#perfectweather': 'perfect weather',
     '#nature': 'nature',
     '#asduah': 'as du ah',
     '#alabama': 'alabama',
     '#sopretty': 'so pretty',
     '#winit': 'win it',
     '#uahsaidyes': 'uah said yes',
     '#ireallyshouldputachairontheroof': 'i really should put a chair on the roof',
     '#takeuahhomefortheholidays': 'take uah home for the holidays',
     '#winitweekend': 'win it weekend',
     '#teamusa': 'team usa',
     '#fcw': 'fcw',
     '#uahtastytuesday': 'uah tasty tuesday',
     '#rocketcity': 'rocket city',
     '#pumpyouup': 'pump you up',
     '#homecomingweek2011': 'homecoming week',
     '#proud': 'proud',
     '#gobigblue': 'go big blue',
     '#classof2013': 'class of',
     '#atx': 'atx',
     '#education': 'education',
     '#kyfbla': 'ky fbla',
     '#repost': 'repost',
     '#wilsonhall': 'wilson hall',
     '#nso': 'n so',
     '#iloveuah': 'i love uah',
     '#sunset': 'sunset',
     '#cbahotdogroast': 'cba hot dog roast',
     '#chargerbound': 'charger bound'}




```python
test_df.tweet.replace(tag_dict, regex=True)
```




    0       College can be scary, but getting here doesn’t...
    1       We’re so excited to see everyone in the mornin...
    2       Online registration is open for our 3 Discover...
    3       If you’re interested in UAH and want to learn ...
    4       Yes! We will announce those dates a little clo...
    5       We will be hosting 3 overnight visits this sem...
    6       Interested in becoming a UAH Orientation Leade...
    7       Another record-breaking year! https://www.uah....
    8       Interested in UAH, but live too far away to vi...
    9       We are looking forward to having tons of stude...
    10      tbt Back when Charlie was just a high school s...
    11      Charger Preview is 2 weeks from Saturday! Don'...
    12      It's STILL not too late! https://twitter.com/U...
    13      We just love our Rocket City! Huntsville was r...
    14      Tomorrow is New Student Orientation Round ✋ Ge...
    15      Charger Preview is July 21! This event is for ...
    16      The University of Alabama in Huntsville will b...
    17      Tomorrow is our first Orientaion session of th...
    18      Our undergraduate students get to do some cool...
    19      For the third year in a row, UAH is ranked #1 ...
    20      Make sure you're following us on Snapchat (uah...
    21      UAH takes first place in the College/Universit...
    22      Register for Orientation TODAY! Sessions are f...
    23      It's finally Admitted Student Day! We are exci...
    24      Wonderful! Can't wait to see the pictures you ...
    25      The weather for Admitted Student Day tomorrow ...
    26      UAH has two teams in the competition this year...
    27      We can't wait for Admitted Student Day this Sa...
    28      Sign up for Orientation! New Student Orientati...
    29      Admitted Student Day is quickly approaching! A...
                                  ...                        
    1949    Don't forget to check out Diving for Dollars s...
    1950    Having a blast at #UAHuntsville WOW week? Send...
    1951    Move-in day is here!! Welcome to campus new Ch...
    1952    Move-in day is here!! Welcome to campus new Ch...
    1953    Kicking off the last Orientation! Welcome home...
    1954    The scholarship application will be closed for...
    1955    Scholarship applications for the 2011/2012 aca...
    1956    #UAHuntsville is up 18% in 2011 Freshmen admis...
    1957                           Move-in day is in 17 days!
    1958    RT @uahuntsville: R&B singer @jasonderulo will...
    1959    RT @uahuntsville: #UAHuntsville is now on Goog...
    1960    RT @uahuntsville: RT @SophieStrong: Im #Excite...
    1961    RT @waaytv: UAHuntsville Researchers to Improv...
    1962    RT @tomspacecamp: RT @SpaceCampSara: You can e...
    1963    Congrats to Rakeem Fuller for winning the $500...
    1964    RT @name_is_mucho: @UAHuntsville i got orienta...
    1965    RT @cweis1992: @UAHuntsville i am and can not ...
    1966    RT @haylibager: Orientation tomorrow @UAHuntsv...
    1967    RT @uahuntsville: Looking forward to having yo...
    1968    Good morning new Chargers! We are ready for or...
    1969    Orientation starts up again tomorrow! Welcome ...
    1970    Orientation 4 is underway! Welcome to the family!
    1971    Make sure to look for your button match while ...
    1972    Yes, we actually hold a record for the longest...
    1973    We haven't yet perfected time travel in scient...
    1974    You heard right, there is no charge or tuition...
    1975    New students at Orientation, we really think y...
    1976    Excited to see you all out for Orientation thi...
    1977    Don't forget to check in on Foursquare if you ...
    1978    Orientation Number 2 is underway! Remember to ...
    Name: tweet, Length: 1979, dtype: object




```python
# Function to clean tweet dataframe
def clean_tweets(df, drop_cols):
    # Create dictionary of "cleaned" hashtags
    tag_dict = hash_dict(df,'hashtags')
    # Create column with clean tweets
    df = df.assign(clean_tw = df.tweet.apply(lambda x: pre.clean(str(x))))
    # Replace hashtags in clean tweets using dictionary
    df = df.assign(clean_tw = df.clean_tw.replace(tag_dict, regex=True))
    # Drop unused cols
    df = df.drop(drop_cols, axis=1)
    return df
```


```python
# Example: Clean DF
# Columns to drop
drop_cols = ['created_at', 'gif_thumb', 'is_quote_status', 'is_reply_to', 
    'location', 'mentions', 'name', 'place', 'quote_id', 'quote_url',
    'replies', 'retweet', 'tags', 'time', 'timezone']
new_df = clean_tweets(test_df, drop_cols)
```


```python
new_df.clean_tw.head(5)
```




    0    College can be scary, but getting here doesn’t...
    1    We’re so excited to see everyone in the mornin...
    2    Online registration is open for our 3 Discover...
    3    If you’re interested in UAH and want to learn ...
    4    Yes! We will announce those dates a little clo...
    Name: clean_tw, dtype: object



## Concatenate Twitter Data from JSON Files into Single Dataframe


```python
# Get list of JSON files
twitter_files = glob.glob('../data/twitter_data/*.json')
```


```python
print(len(twitter_files)/100)
```

    11.29
    


```python
# Note: Having memory issues so I'm breaking the file list into chunks 
# If you have files you need to run individually, set no_tricks = False
# and put your "tricky" file into line 9
def appended_files(file_list, chunk_num, drop_cols, no_tricks):
    appended_data = []
    for file in file_list:
        # Print file and time for debugging purposes
        print(str(re.findall(r"[0-9].*", str(file))[0]) + ' - ' + str(datetime.datetime.now()))
        tw_df = pd.read_json(file, encoding='utf-8', lines=True)
        # Code is hanging on this large file - skip cleaning it
        if (no_tricks == True and str(re.findall(r"[0-9].*", str(file))[0]) == '217156_main.json'):
            continue
        # Clean dataframe using function from previous step
        tw_df = clean_tweets(tw_df, drop_cols)
        # Extract ID number
        id_num = ''.join([i for i in str(file) if i.isdigit()])
        # Add ID number to DF
        tw_df.loc[:,'ipeds_id']=id_num
        # Add indicator for main page vs admissions page
        if 'main' in str(file):
            tw_df.loc[:,'main_page']=1
        elif 'adm' in str(file):
            tw_df.loc[:,'main_page']=0
        else:
            pass
        # Reset index
        tw_df.reset_index(drop=True, inplace=True)
        # Add to list to append
        if (len(file_list) > 1):
            appended_data.append(tw_df)
        # Run garbage collection
        gc.collect()
    # Concatenate files in list
    if (len(file_list) > 1):
        tw_df = pd.concat(appended_data, ignore_index=True)
    # Save concatenated files to pickle
    tw_df.to_pickle(path=r'../data/twitter_data/pickle/concat_tw_' + str(chunk_num) + '.pkl')
    # Delete object
    del tw_df
```


```python
# Create a function to create "chunks" of the list of files
# https://stackoverflow.com/questions/434287/what-is-the-most-pythonic-way-to-iterate-over-a-list-in-chunks
def chunker(seq, size):
    return (seq[pos:pos + size] for pos in range(0, len(seq), size))
```


```python
# Columns to drop
drop_cols = ['created_at', 'gif_thumb', 'is_quote_status', 'is_reply_to', 
    'location', 'mentions', 'name', 'place', 'quote_id', 'quote_url',
    'replies', 'retweet', 'tags', 'time', 'timezone']
```


```python
gc.collect()
```




    15474




```python
# Clean and append files in chunks
# Note: Computer hanging up - do this in chunks
# If process hangs and you need to restart at a chunk, change twitter files to 
# twitter_files[start_file_number:] and i to the number of the chunk
# Example: i=5 twitter_files[250:] will start at the 5th chunk for chunks of 50
i = 1
for group in chunker(twitter_files, 50):
    chunk_str = "%02d" % (i,)
    appended_files(group, chunk_str, drop_cols, no_tricks=True)
    # Run garbage collection
    gc.collect()
    i += 1
```

    216852_main.json - 2018-12-15 09:40:05.825548
    216931_adm.json - 2018-12-15 09:40:28.776106
    216931_main.json - 2018-12-15 09:40:30.202339
    217156_adm.json - 2018-12-15 09:40:38.210865
    217156_main.json - 2018-12-15 09:40:39.469350
    217165_adm.json - 2018-12-15 09:40:40.678950
    217165_main.json - 2018-12-15 09:40:52.651845
    217235_main.json - 2018-12-15 09:41:07.459032
    217420_adm.json - 2018-12-15 09:41:38.809539
    217420_main.json - 2018-12-15 09:41:41.815739
    217484_main.json - 2018-12-15 09:41:48.265548
    217493_main.json - 2018-12-15 09:41:48.668729
    217518_adm.json - 2018-12-15 09:41:54.018408
    217518_main.json - 2018-12-15 09:41:54.489165
    217536_main.json - 2018-12-15 09:42:10.210251
    217688_main.json - 2018-12-15 09:42:16.796390
    217749_main.json - 2018-12-15 09:42:30.173260
    217819_adm.json - 2018-12-15 09:42:33.692169
    217819_main.json - 2018-12-15 09:42:52.600567
    217864_main.json - 2018-12-15 09:43:31.124285
    217882_adm.json - 2018-12-15 09:43:37.932327
    217882_main.json - 2018-12-15 09:43:39.734309
    218061_main.json - 2018-12-15 09:43:50.672358
    218070_main.json - 2018-12-15 09:43:56.320167
    218229_main.json - 2018-12-15 09:44:21.422759
    218238_main.json - 2018-12-15 09:44:23.831659
    218663_adm.json - 2018-12-15 09:44:24.743224
    218663_main.json - 2018-12-15 09:44:29.319711
    218724_adm.json - 2018-12-15 09:45:06.466742
    218724_main.json - 2018-12-15 09:45:19.635414
    218733_main.json - 2018-12-15 09:45:32.519776
    218742_main.json - 2018-12-15 09:45:40.859098
    218964_main.json - 2018-12-15 09:46:10.300855
    218973_adm.json - 2018-12-15 09:46:17.633433
    218973_main.json - 2018-12-15 09:46:21.075775
    219000_main.json - 2018-12-15 09:46:22.884960
    219356_main.json - 2018-12-15 09:46:41.858735
    219471_adm.json - 2018-12-15 09:46:43.140073
    219471_main.json - 2018-12-15 09:46:43.949916
    219602_adm.json - 2018-12-15 09:46:46.399623
    219602_main.json - 2018-12-15 09:46:47.731211
    219709_main.json - 2018-12-15 09:46:53.823935
    219718_main.json - 2018-12-15 09:46:55.613410
    219806_main.json - 2018-12-15 09:46:58.828086
    219976_adm.json - 2018-12-15 09:47:01.089869
    219976_main.json - 2018-12-15 09:47:04.018030
    220075_adm.json - 2018-12-15 09:47:10.294415
    220075_main.json - 2018-12-15 09:47:11.553270
    220516_main.json - 2018-12-15 09:47:16.557988
    220613_main.json - 2018-12-15 09:47:20.564185
    220862_adm.json - 2018-12-15 09:47:30.263817
    220862_main.json - 2018-12-15 09:47:32.700185
    220978_adm.json - 2018-12-15 09:47:38.098459
    220978_main.json - 2018-12-15 09:47:39.230326
    221351_main.json - 2018-12-15 09:48:13.144293
    221740_main.json - 2018-12-15 09:48:19.625061
    221759_adm.json - 2018-12-15 09:48:41.536906
    221759_main.json - 2018-12-15 09:48:45.342488
    221768_adm.json - 2018-12-15 09:49:01.970390
    221768_main.json - 2018-12-15 09:49:05.706181
    221838_adm.json - 2018-12-15 09:49:07.107002
    221838_main.json - 2018-12-15 09:49:09.157963
    221847_adm.json - 2018-12-15 09:49:17.953691
    221847_main.json - 2018-12-15 09:49:21.085349
    221892_main.json - 2018-12-15 09:49:35.286218
    221953_main.json - 2018-12-15 09:49:46.631095
    221971_main.json - 2018-12-15 09:49:53.441247
    221999_main.json - 2018-12-15 09:49:57.513660
    222178_adm.json - 2018-12-15 09:50:52.088997
    222178_main.json - 2018-12-15 09:50:53.285300
    222831_adm.json - 2018-12-15 09:50:57.116234
    222831_main.json - 2018-12-15 09:50:58.183151
    223232_adm.json - 2018-12-15 09:51:01.418895
    223232_main.json - 2018-12-15 09:51:09.853514
    224004_adm.json - 2018-12-15 09:51:52.161166
    224004_main.json - 2018-12-15 09:51:55.415822
    224147_adm.json - 2018-12-15 09:52:09.558103
    224147_main.json - 2018-12-15 09:52:11.657230
    224226_adm.json - 2018-12-15 09:52:32.857693
    224226_main.json - 2018-12-15 09:52:34.273650
    224554_main.json - 2018-12-15 09:52:43.503674
    225247_main.json - 2018-12-15 09:52:50.032140
    225399_main.json - 2018-12-15 09:52:56.771453
    225432_adm.json - 2018-12-15 09:53:03.197551
    225432_main.json - 2018-12-15 09:53:03.981996
    225627_adm.json - 2018-12-15 09:53:12.191919
    225627_main.json - 2018-12-15 09:53:14.612136
    226091_main.json - 2018-12-15 09:53:27.223761
    226231_main.json - 2018-12-15 09:53:33.381082
    226471_main.json - 2018-12-15 09:53:38.001829
    226833_adm.json - 2018-12-15 09:53:41.013412
    226833_main.json - 2018-12-15 09:53:45.205185
    227331_main.json - 2018-12-15 09:54:00.387464
    227368_main.json - 2018-12-15 09:54:03.518559
    227526_adm.json - 2018-12-15 09:54:12.517065
    227526_main.json - 2018-12-15 09:54:14.796946
    227757_main.json - 2018-12-15 09:54:22.982959
    227881_main.json - 2018-12-15 09:55:20.961040
    228149_adm.json - 2018-12-15 09:55:43.484404
    228149_main.json - 2018-12-15 09:55:45.954028
    228246_main.json - 2018-12-15 09:56:31.383441
    228343_main.json - 2018-12-15 09:56:58.264767
    228431_main.json - 2018-12-15 09:57:24.509846
    228459_adm.json - 2018-12-15 09:57:29.051135
    228459_main.json - 2018-12-15 09:57:30.988475
    228529_main.json - 2018-12-15 09:59:11.307468
    228705_adm.json - 2018-12-15 09:59:16.448504
    228705_main.json - 2018-12-15 09:59:19.852077
    228723_adm.json - 2018-12-15 09:59:30.304444
    228723_main.json - 2018-12-15 09:59:36.821327
    228778_adm.json - 2018-12-15 10:00:26.844340
    228778_main.json - 2018-12-15 10:00:29.671582
    228787_main.json - 2018-12-15 10:01:10.273169
    228802_adm.json - 2018-12-15 10:01:36.549744
    228802_main.json - 2018-12-15 10:01:44.611273
    228875_adm.json - 2018-12-15 10:01:49.842294
    228875_main.json - 2018-12-15 10:01:51.309125
    229063_adm.json - 2018-12-15 10:02:03.103598
    229063_main.json - 2018-12-15 10:02:03.685216
    229115_adm.json - 2018-12-15 10:02:05.985510
    229115_main.json - 2018-12-15 10:02:13.788750
    229179_main.json - 2018-12-15 10:03:08.847627
    229267_adm.json - 2018-12-15 10:03:12.161921
    229267_main.json - 2018-12-15 10:03:23.928212
    229780_main.json - 2018-12-15 10:03:45.723907
    229814_main.json - 2018-12-15 10:03:47.731581
    230038_adm.json - 2018-12-15 10:03:59.269025
    230038_main.json - 2018-12-15 10:03:59.887932
    230603_main.json - 2018-12-15 10:04:07.191657
    230728_main.json - 2018-12-15 10:04:16.713584
    230737_main.json - 2018-12-15 10:04:25.890197
    230764_adm.json - 2018-12-15 10:04:28.952842
    230764_main.json - 2018-12-15 10:04:30.145091
    230782_main.json - 2018-12-15 10:05:49.270622
    230807_main.json - 2018-12-15 10:05:54.431944
    230995_main.json - 2018-12-15 10:06:11.783807
    231174_main.json - 2018-12-15 10:06:19.241286
    231624_adm.json - 2018-12-15 10:06:33.534370
    231624_main.json - 2018-12-15 10:06:35.593883
    231712_adm.json - 2018-12-15 10:06:54.796048
    231712_main.json - 2018-12-15 10:06:56.774414
    232423_main.json - 2018-12-15 10:06:59.819222
    232557_adm.json - 2018-12-15 10:07:21.564342
    232557_main.json - 2018-12-15 10:07:30.499124
    232566_adm.json - 2018-12-15 10:07:48.340644
    232566_main.json - 2018-12-15 10:07:56.968301
    232609_adm.json - 2018-12-15 10:08:19.601780
    232609_main.json - 2018-12-15 10:08:20.996833
    232681_main.json - 2018-12-15 10:08:39.745853
    232706_adm.json - 2018-12-15 10:08:45.316772
    232706_main.json - 2018-12-15 10:08:51.703668
    232982_main.json - 2018-12-15 10:08:55.286009
    233277_adm.json - 2018-12-15 10:09:00.126319
    233277_main.json - 2018-12-15 10:09:01.126804
    233374_adm.json - 2018-12-15 10:09:06.858417
    233374_main.json - 2018-12-15 10:09:09.269080
    233921_adm.json - 2018-12-15 10:09:26.192664
    233921_main.json - 2018-12-15 10:09:28.882497
    234030_adm.json - 2018-12-15 10:09:48.818476
    234030_main.json - 2018-12-15 10:09:50.941061
    234076_adm.json - 2018-12-15 10:10:31.291687
    234076_main.json - 2018-12-15 10:10:31.714912
    234155_adm.json - 2018-12-15 10:11:09.647369
    234155_main.json - 2018-12-15 10:11:10.181678
    234207_adm.json - 2018-12-15 10:11:20.440426
    234207_main.json - 2018-12-15 10:11:22.349236
    234827_main.json - 2018-12-15 10:11:35.709595
    235097_main.json - 2018-12-15 10:11:41.472088
    235167_main.json - 2018-12-15 10:11:45.552659
    235316_adm.json - 2018-12-15 10:11:48.158019
    235316_main.json - 2018-12-15 10:11:49.516514
    236230_adm.json - 2018-12-15 10:12:01.498055
    236230_main.json - 2018-12-15 10:12:03.746305
    236328_main.json - 2018-12-15 10:12:13.659014
    236577_main.json - 2018-12-15 10:12:50.127751
    236595_adm.json - 2018-12-15 10:13:06.267144
    236595_main.json - 2018-12-15 10:13:07.227192
    236948_adm.json - 2018-12-15 10:13:16.941654
    236948_main.json - 2018-12-15 10:13:17.909074
    237011_main.json - 2018-12-15 10:13:53.188303
    237057_main.json - 2018-12-15 10:14:38.634305
    237066_main.json - 2018-12-15 10:14:43.997505
    237330_main.json - 2018-12-15 10:14:48.558478
    237367_main.json - 2018-12-15 10:14:50.770827
    237525_adm.json - 2018-12-15 10:15:01.573148
    237525_main.json - 2018-12-15 10:15:03.818168
    237792_main.json - 2018-12-15 10:15:12.842286
    237899_main.json - 2018-12-15 10:15:16.160757
    237932_main.json - 2018-12-15 10:15:19.255441
    238193_main.json - 2018-12-15 10:15:24.732266
    238430_adm.json - 2018-12-15 10:15:27.750687
    238430_main.json - 2018-12-15 10:15:28.744925
    238458_main.json - 2018-12-15 10:15:33.303112
    238476_main.json - 2018-12-15 10:15:46.258073
    238616_main.json - 2018-12-15 10:15:51.865389
    238980_main.json - 2018-12-15 10:16:02.273826
    239080_main.json - 2018-12-15 10:16:07.817503
    239105_adm.json - 2018-12-15 10:16:15.112221
    239105_main.json - 2018-12-15 10:16:22.676174
    239318_main.json - 2018-12-15 10:17:51.789181
    240107_main.json - 2018-12-15 10:17:59.693522
    240189_adm.json - 2018-12-15 10:18:18.476516
    240189_main.json - 2018-12-15 10:18:24.699190
    240268_adm.json - 2018-12-15 10:18:29.324248
    240268_main.json - 2018-12-15 10:18:33.068975
    240277_adm.json - 2018-12-15 10:18:50.139031
    240277_main.json - 2018-12-15 10:18:52.282026
    240329_adm.json - 2018-12-15 10:19:18.694392
    240329_main.json - 2018-12-15 10:19:19.252992
    240365_adm.json - 2018-12-15 10:19:21.954444
    240365_main.json - 2018-12-15 10:19:29.001280
    240374_adm.json - 2018-12-15 10:19:48.420129
    240374_main.json - 2018-12-15 10:19:51.099373
    240417_adm.json - 2018-12-15 10:19:54.878363
    240417_main.json - 2018-12-15 10:19:57.579096
    240444_adm.json - 2018-12-15 10:20:01.222918
    240444_main.json - 2018-12-15 10:20:04.818923
    240462_main.json - 2018-12-15 10:23:38.100349
    240471_main.json - 2018-12-15 10:23:49.683924
    240480_main.json - 2018-12-15 10:23:58.358429
    240727_main.json - 2018-12-15 10:24:03.396915
    243744_adm.json - 2018-12-15 10:24:07.490239
    243744_main.json - 2018-12-15 10:24:08.359617
    243780_adm.json - 2018-12-15 10:24:45.493916
    243780_main.json - 2018-12-15 10:24:48.281308
    366711_main.json - 2018-12-15 10:26:19.436469
    450933_adm.json - 2018-12-15 10:26:26.106177
    450933_main.json - 2018-12-15 10:26:32.805968
    


```python
# For some reason one .json file will not process. Tried stripping non-ascii characters.
# It will have to be left out for now. 
# Run one large, pesky file separately - 217156_main.json
#appended_files(['../data/twitter_data/217156_fixed_main.json'], '217156', drop_cols, no_tricks=False)
```

    217156_fixed_main.json - 2018-12-17 19:33:11.123541
    


```python
# Concatenate only the cleaned tweets from the cleaned pkl files
# Saves memory space
gc.collect()
pkl_files = glob.glob(r'../data/twitter_data/pickle/concat_tw_*.pkl')
append_tweets = []
for file in pkl_files:
    print(str(file))
    tw_df = pd.read_pickle(file)
    tw_df = tw_df[['ipeds_id', 'id', 'clean_tw']]
    append_tweets.append(tw_df)
    del tw_df
    gc.collect()
tw_final = pd.concat(append_tweets, ignore_index=True)
tw_final.to_pickle(path=r'../data/clean/clean_tweets_full.pkl')
```

    ../data/twitter_data/pickle\concat_tw_01.pkl
    ../data/twitter_data/pickle\concat_tw_02.pkl
    ../data/twitter_data/pickle\concat_tw_03.pkl
    ../data/twitter_data/pickle\concat_tw_04.pkl
    ../data/twitter_data/pickle\concat_tw_05.pkl
    ../data/twitter_data/pickle\concat_tw_06.pkl
    ../data/twitter_data/pickle\concat_tw_07.pkl
    ../data/twitter_data/pickle\concat_tw_08.pkl
    ../data/twitter_data/pickle\concat_tw_09.pkl
    ../data/twitter_data/pickle\concat_tw_10.pkl
    ../data/twitter_data/pickle\concat_tw_11.pkl
    ../data/twitter_data/pickle\concat_tw_12.pkl
    ../data/twitter_data/pickle\concat_tw_13.pkl
    

## Process Text and Create Diversity Tweet Dataset


```python
# Load the pickled data
tw_data = pd.read_pickle(path=r'../data/clean/clean_tweets_full.pkl')
```

Unfortunately there are still non-ascii characters - let's remove them now.


```python
def convert_unicode(text):
    return unicodedata2.normalize('NFKD', text).encode('ascii', 'ignore')

tw_data = tw_data.assign(clean_tw = tw_data.clean_tw.apply(convert_unicode))
```

Preprocessing code comes from link below. Though, note the author forgot to define stemmer. 

https://towardsdatascience.com/topic-modeling-and-latent-dirichlet-allocation-in-python-9bf156893c24


```python
# Stem and remove stop words
stemmer = SnowballStemmer("english", ignore_stopwords=True)

def lemmatize_stemming(text):
    return stemmer.stem(WordNetLemmatizer().lemmatize(text, pos='v'))

def preprocess(text):
    result = []
    for token in gensim.utils.simple_preprocess(text):
        if token not in gensim.parsing.preprocessing.STOPWORDS and len(token) > 3:
            result.append(lemmatize_stemming(token))
    return result
```


```python
tw_data = tw_data.assign(words = tw_data['clean_tw'].map(preprocess))
```


```python
gc.collect()
```




    44




```python
# Save a copy in case you need to reload
tw_data.to_pickle(path=r'../data/clean/pre_processed.pkl')
```


```python
tw_data = pd.read_pickle(path=r'../data/clean/pre_processed.pkl')
```


```python
# Create a copy of the data with only the processed data, id, and school id
tw_words = tw_data[['ipeds_id', 'id', 'words']].copy()
del tw_data
gc.collect()
```




    7




```python
np.random.seed(12345)
# Create a column with random number - will be used to subset the data
tw_words = tw_words.assign(rand_int = np.random.randint(0, 99, tw_words.shape[0]))
```

## Remove words used by only one college


```python
# Remove words used by only one school - this will help to filter out school names and mascots
# This is tricky again because of memory issues
# Stack the words in chunks - remove words that are used by multiple schools
# Final result will be list of words used only by one school

# Must have list of words in tweet in column called 'words'
def single_sch_words(df):

    # Stack the data - create row for each word in list
    df = df.set_index(['id'])
    df.sort_index(inplace=True)
    s = df.apply(lambda x: pd.Series(x['words']),axis=1).stack().reset_index(level=1, drop=True)
    s.name = 'word'
    tw_data_stacked = df.drop('words', axis=1).join(s)
    del s
    gc.collect()
    
    # Keep only unique instances of each word and school
    tw_data_stacked = tw_data_stacked.drop_duplicates(subset=['word', 'ipeds_id'], keep=False)
    
    # Create count of number of colleges that use each word
    num_colleges = tw_data_stacked.groupby('word').agg({'ipeds_id': lambda x: x.nunique()})
    num_colleges.name = 'word'
    num_colleges.reset_index()
    num_colleges.rename(index=str, columns={'ipeds_id':'school_count'}, inplace=True)
    tw_data_stacked = tw_data_stacked.join(num_colleges,on='word')
    del num_colleges
    gc.collect()
    
    # Keep words used by only one college
    tw_data_stacked.reset_index()
    tw_data_stacked = tw_data_stacked.loc[tw_data_stacked.school_count == 1]
    tw_data_stacked.drop('school_count', axis=1, inplace=True)
    gc.collect()
    
    return tw_data_stacked
```


```python
# Datasets of single use words for random chunks of the data
# Example using 1% of data
single_words1 = single_sch_words(tw_words.loc[tw_words.rand_int < 1])
single_words1.head(5)
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
      <th>ipeds_id</th>
      <th>rand_int</th>
      <th>word</th>
    </tr>
    <tr>
      <th>id</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>93773412</th>
      <td>183239</td>
      <td>0</td>
      <td>orvieto</td>
    </tr>
    <tr>
      <th>789579480</th>
      <td>210669</td>
      <td>0</td>
      <td>ascent</td>
    </tr>
    <tr>
      <th>836384863</th>
      <td>209056</td>
      <td>0</td>
      <td>mcdow</td>
    </tr>
    <tr>
      <th>908199047</th>
      <td>164580</td>
      <td>0</td>
      <td>nextworth</td>
    </tr>
    <tr>
      <th>922205934</th>
      <td>228875</td>
      <td>0</td>
      <td>buccan</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Get single words from random samples of 3% of data at a time then append
append_single = []
```


```python
for i in range(0,102,3):
    j = i + 2
    print('Start at: ' + str(i) + ' End at: ' + str(j) + ' - ' + str(datetime.datetime.now())) 
    df = single_sch_words(tw_words.loc[(tw_words.rand_int >= i) & (tw_words.rand_int <= j)])
    append_single.append(df)
    gc.collect()
```

    Start at: 0 End at: 2 - 2018-12-18 13:30:07.326287
    Start at: 3 End at: 5 - 2018-12-18 13:31:59.735666
    Start at: 6 End at: 8 - 2018-12-18 13:33:41.026483
    Start at: 9 End at: 11 - 2018-12-18 13:35:33.474505
    Start at: 12 End at: 14 - 2018-12-18 13:37:22.630545
    Start at: 15 End at: 17 - 2018-12-18 13:39:05.125096
    Start at: 18 End at: 20 - 2018-12-18 13:40:41.995758
    Start at: 21 End at: 23 - 2018-12-18 13:42:21.380203
    Start at: 24 End at: 26 - 2018-12-18 13:43:59.934431
    Start at: 27 End at: 29 - 2018-12-18 13:45:41.805603
    Start at: 30 End at: 32 - 2018-12-18 13:47:18.330342
    Start at: 33 End at: 35 - 2018-12-18 13:48:58.463605
    Start at: 36 End at: 38 - 2018-12-18 13:50:38.119495
    Start at: 39 End at: 41 - 2018-12-18 13:52:28.329703
    Start at: 42 End at: 44 - 2018-12-18 13:54:18.916873
    Start at: 45 End at: 47 - 2018-12-18 13:56:03.736739
    Start at: 48 End at: 50 - 2018-12-18 13:57:48.435560
    Start at: 51 End at: 53 - 2018-12-18 13:59:37.243402
    Start at: 54 End at: 56 - 2018-12-18 14:01:30.655424
    Start at: 57 End at: 59 - 2018-12-18 14:03:15.413516
    Start at: 60 End at: 62 - 2018-12-18 14:05:07.227998
    Start at: 63 End at: 65 - 2018-12-18 14:06:52.656103
    Start at: 66 End at: 68 - 2018-12-18 14:08:40.924504
    Start at: 69 End at: 71 - 2018-12-18 14:10:27.186727
    Start at: 72 End at: 74 - 2018-12-18 14:12:03.697649
    Start at: 75 End at: 77 - 2018-12-18 14:13:35.829157
    Start at: 78 End at: 80 - 2018-12-18 14:15:11.689331
    Start at: 81 End at: 83 - 2018-12-18 14:17:02.079004
    Start at: 84 End at: 86 - 2018-12-18 14:18:34.542096
    Start at: 87 End at: 89 - 2018-12-18 14:20:26.194343
    Start at: 90 End at: 92 - 2018-12-18 14:22:11.956262
    Start at: 93 End at: 95 - 2018-12-18 14:23:54.153736
    Start at: 96 End at: 98 - 2018-12-18 14:25:38.045927
    Start at: 99 End at: 101 - 2018-12-18 14:27:21.715944
    


```python
# Combine the three sets of single words, see if any are repeats
single_words_all = pd.concat(append_single, ignore_index=True)
# Save in case of crash
single_words_all.to_pickle(path=r'../data/clean/single_words.pkl')
```


```python
single_words_all = pd.read_pickle(path=r'../data/clean/single_words.pkl')
```


```python
# On further inspection, many of the single words look like mentions that weren't removed with the Twitter preprocessor.
single_words_all.head(10)
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
      <th>ipeds_id</th>
      <th>rand_int</th>
      <th>word</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>152080</td>
      <td>1</td>
      <td>markup</td>
    </tr>
    <tr>
      <th>1</th>
      <td>164580</td>
      <td>2</td>
      <td>zachrarki</td>
    </tr>
    <tr>
      <th>2</th>
      <td>209056</td>
      <td>0</td>
      <td>mcdow</td>
    </tr>
    <tr>
      <th>3</th>
      <td>210669</td>
      <td>2</td>
      <td>shemelya</td>
    </tr>
    <tr>
      <th>4</th>
      <td>218742</td>
      <td>1</td>
      <td>dsc_</td>
    </tr>
    <tr>
      <th>5</th>
      <td>173300</td>
      <td>2</td>
      <td>gooseberri</td>
    </tr>
    <tr>
      <th>6</th>
      <td>209056</td>
      <td>2</td>
      <td>rouari</td>
    </tr>
    <tr>
      <th>7</th>
      <td>218742</td>
      <td>2</td>
      <td>liisa</td>
    </tr>
    <tr>
      <th>8</th>
      <td>218742</td>
      <td>2</td>
      <td>salosaari</td>
    </tr>
    <tr>
      <th>9</th>
      <td>218742</td>
      <td>2</td>
      <td>jasinski</td>
    </tr>
  </tbody>
</table>
</div>




```python
# On the combined dataset of random samples, count the number of schools that use each word
num_colleges = single_words_all.groupby('word').agg({'ipeds_id': lambda x: x.nunique()})
num_colleges.name = 'word'
num_colleges.reset_index()
num_colleges.rename(index=str, columns={'ipeds_id':'school_count'}, inplace=True)
```


```python
single_words_all = num_colleges
single_words_all = single_words_all.loc[single_words_all.school_count == 1]
```


```python
single_words_all.head(10)
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
      <th>school_count</th>
    </tr>
    <tr>
      <th>word</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>a__schedul</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a__www</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a_clever_alia</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a_dzi</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a_feiii</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a_garza_duh</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a_n_g</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a_scot_</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a_suarez</th>
      <td>1</td>
    </tr>
    <tr>
      <th>a_wizzzl</th>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>




```python
single_words_all = single_words_all.reset_index()
single_words_all.columns
```




    Index(['word', 'school_count'], dtype='object')




```python
single_word_list = single_words_all.word.values.tolist()
```


```python
single_word_list[0:10]
```




    ['a__schedul',
     'a__www',
     'a_clever_alia',
     'a_dzi',
     'a_feiii',
     'a_garza_duh',
     'a_n_g',
     'a_scot_',
     'a_suarez',
     'a_wizzzl']



Remove words used by only one college from the dataset.


```python
tw_words.drop(['rand_int'], inplace=True, axis=1)
```


```python
def remove_single(old_list, not_list):
    return [x for x in old_list if x not in not_list]
```


```python
# This takes a very very very long time to run
# Did not have time to finish running, sadly
tw_words = tw_words.assign(words = tw_words.words.apply(lambda x: remove_single(x, single_word_list)))
```


    ---------------------------------------------------------------------------

    KeyboardInterrupt                         Traceback (most recent call last)

    <ipython-input-29-a4786b178c97> in <module>
    ----> 1 tw_words = tw_words.assign(words = tw_words.words.apply(lambda x: remove_single(x, single_word_list)))
    

    ~\Anaconda3\lib\site-packages\pandas\core\series.py in apply(self, func, convert_dtype, args, **kwds)
       3192             else:
       3193                 values = self.astype(object).values
    -> 3194                 mapped = lib.map_infer(values, f, convert=convert_dtype)
       3195 
       3196         if len(mapped) and isinstance(mapped[0], Series):
    

    pandas/_libs/src\inference.pyx in pandas._libs.lib.map_infer()
    

    <ipython-input-29-a4786b178c97> in <lambda>(x)
    ----> 1 tw_words = tw_words.assign(words = tw_words.words.apply(lambda x: remove_single(x, single_word_list)))
    

    <ipython-input-27-eb913ead9e07> in remove_single(old_list, not_list)
          1 def remove_single(old_list, not_list):
    ----> 2     return [x for x in old_list if x not in not_list]
    

    <ipython-input-27-eb913ead9e07> in <listcomp>(.0)
          1 def remove_single(old_list, not_list):
    ----> 2     return [x for x in old_list if x not in not_list]
    

    KeyboardInterrupt: 



```python
tw_words.to_pickle(path=r'../data/clean/clean_tweets_final.pkl')
```

## Diversity Analysis Dataset


```python
tw_data = pd.read_pickle(path=r'../data/clean/pre_processed.pkl')
tw_words = tw_data[['ipeds_id', 'id', 'words']].copy()
del tw_data
gc.collect()
```




    7




```python
tw_words.drop(['ipeds_id'], axis=1, inplace=True)
tw_words.head(5)
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
      <th>id</th>
      <th>words</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1053360850965315584</td>
      <td>[uabhomecom, favorit, time, year]</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1052568776414298117</td>
      <td>[admit, incom, freshman, class, congrat, check...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1021825776553996288</td>
      <td>[growth, record, enrol, build, transit, commut...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1016333689486233600</td>
      <td>[hope, have, great, summer, admit, freshmen, t...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1006548327687745536</td>
      <td>[recent, admit, freshman, congrat, help, check...</td>
    </tr>
  </tbody>
</table>
</div>




```python
gc.collect()
pkl_files = glob.glob(r'../data/twitter_data/pickle/concat_tw_*.pkl')
append_tweets = []
for file in pkl_files:
    print(str(file))
    tw_df = pd.read_pickle(file)
    tw_df = tw_df[['ipeds_id', 'id', 'date','tweet','likes_count','photos']]
    append_tweets.append(tw_df)
    del tw_df
    gc.collect()
diversity_df = pd.concat(append_tweets, ignore_index=True)
```

    ../data/twitter_data/pickle\concat_tw_01.pkl
    ../data/twitter_data/pickle\concat_tw_02.pkl
    ../data/twitter_data/pickle\concat_tw_03.pkl
    ../data/twitter_data/pickle\concat_tw_04.pkl
    ../data/twitter_data/pickle\concat_tw_05.pkl
    ../data/twitter_data/pickle\concat_tw_06.pkl
    ../data/twitter_data/pickle\concat_tw_07.pkl
    ../data/twitter_data/pickle\concat_tw_08.pkl
    ../data/twitter_data/pickle\concat_tw_09.pkl
    ../data/twitter_data/pickle\concat_tw_10.pkl
    ../data/twitter_data/pickle\concat_tw_11.pkl
    ../data/twitter_data/pickle\concat_tw_12.pkl
    ../data/twitter_data/pickle\concat_tw_13.pkl
    


```python
diversity_df = pd.merge(diversity_df, tw_words, on=['id'])
```


```python
del tw_words
gc.collect()
```




    14




```python
def list_contains(cell_list, word_list):
    if [i for i in cell_list if i in word_list]:
        return True
    else:
        return False
```


```python
print(list_contains(['1','2','3'], ['3']))
print(list_contains(['1','2','3'], ['4']))
```

    True
    False
    


```python
# Identify diversity-related tweets
div_words = ['divers', 'multicultur']
diversity_df = diversity_df.assign(diversity_flag = diversity_df.words.apply(lambda x: list_contains(x,div_words)))
```


```python
# Identify race-related tweets
race_words = ['black', 'african', 'asian', 'hispan', 'latino', 'latina']
diversity_df = diversity_df.assign(race_flag = diversity_df.words.apply(lambda x: list_contains(x,race_words)))
```


```python
# Identify gender-related tweets
gender_words = ['woman', 'women','gender']
diversity_df = diversity_df.assign(gender_flag = diversity_df.words.apply(lambda x: list_contains(x,gender_words)))
```


```python
diversity_df[diversity_df.gender_flag==True].tweet.head(5)
```




    425    The 2013 Fall Leadership Conference is underwa...
    528    Happy Memorial Day, friends! Thank you to all ...
    874    RT @uabnews: They did it! The UAB Women's Bask...
    878    Women's Basketball Invitational Championship G...
    933    2 new women's sports at UAB!!!  http://fb.me/x...
    Name: tweet, dtype: object




```python
gc.collect()
```




    0




```python
diversity_df.columns
```




    Index(['ipeds_id', 'id', 'date', 'tweet', 'likes_count', 'photos', 'words',
           'diversity_flag', 'race_flag', 'gender_flag'],
          dtype='object')




```python
len(diversity_df.index)
```




    5907245




```python
# Save data in chunks
diversity_df[0:1000000].to_pickle(path=r'../data/clean/diversity_full_01.pkl')
```


```python
gc.collect()
diversity_df[1000001:2000000].to_pickle(path=r'../data/clean/diversity_full_02.pkl')
gc.collect()
diversity_df[2000001:3000000].to_pickle(path=r'../data/clean/diversity_full_03.pkl')
gc.collect()
diversity_df[3000001:4000000].to_pickle(path=r'../data/clean/diversity_full_04.pkl')
gc.collect()
diversity_df[4000001:5000000].to_pickle(path=r'../data/clean/diversity_full_05.pkl')
gc.collect()
diversity_df[5000001:].to_pickle(path=r'../data/clean/diversity_full_06.pkl')
```
