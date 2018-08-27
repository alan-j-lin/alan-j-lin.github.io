---
layout: post
title: A Feature I believe Yelp is Missing
---

If you have ever been out for drinks in the financial district of San Francisco, you may have stumbled across [Mikkeller bar]. This dutch beer hall is a great place for conversation with friends after work with an atmosphere that is just lively enough that it good for conversation. With an expansive draft list and selection of sours, it is the type of place that I want to always drink at.

Now, if I looked up Mikkeller on Yelp I would not only get the bar itself but bars of a similar profile that are in the area. These additional options are very useful as I might not always be in the mood to go back to Mikkeler, but would want to try a different place that has a similar vibe.

![](/public/Project_Fletcher/mikkeller_sf.png)

However this type of search does not translate when Yelp is trying to base its search on a different location. For instance, I tried to search for Mikkeller bar in Chicago and got results that did not look similar.

![](/public/Project_Fletcher/mikkeller_chicago.png)

Why is it that Yelp that does not have a feature where the user could input a specific bar or list of bars that they enjoy, and have Yelp expand its search to find bars that are similar to the input at a location of the user's choosing? While you could get by searching for specific keywords like beer hall or gastropub, I felt that this search method could be more specific because part of the reason why people enjoy bars beyond their alcohol selection are more subjective things like ambience. Therefore I decided that for my 4th [Metis] project, I would try to implement this feature using unsupervised learning and NLP techniques.

[Mikkeller bar]: https://www.yelp.com/biz/mikkeller-bar-san-francisco
[Metis]: https://www.thisismetis.com/

## Data

### Data Sources:

For this project, I ended up getting my data from the [Yelp challenge dataset]. A publicly available set of data for students to do research on, this was an invaluable resource as Yelp's TOS is not very accepting of webscraping. The dataset itself was huge as there was over 6 million reviews for 800,000+ different businesses spread out over 10 different metropolitan areas in the US and Canada. The fact that there was so much geographic diversity was a bonus for my project as I wanted it to be location agnostic. The data from Yelp was stored in .json format so I ended up querying and filtering the data using a local MongoDB instance. In retrospect as the .json files were massive (the review .json was 4.72 GB), analyzing data for this project would have been much more efficient if I had transferred everything to a large memory AWS instance. The 8 GB of RAM that my Macbook Air has proved to be not quite adequate enough to run analysis on the data without micromanaging computer resources.

[Yelp challenge dataset]: https://www.yelp.com/dataset/challenge

### Data Processing:

After loading my data into mongo, my first step for data processing was to filter the list of businesses for only bars. My first approach (using [pymongo]) was to perform a regex to match for any bar that contained the word "Bars" in its category type. I also only filtered for bars that had at least 50 reviews as I assumed that would be enough text data to get some inferences:

```
# Create subset of business that contain bars in question
db.business.aggregate([
     {"$match": {'categories': {'$regex': 'Bars'}}},
     {"$match": {"review_count": {"$gte": 50}}},
     {"$out": "bars"}]);

```

This approach did ending up decreasing the number of businesses to search through from 180,000+ to 5000+. However, one quirk I quickly realized was that using this regex approach caused me to not only capture bar types like wine bars or sports bars, but also sushi bars. In order to get around this, I ended up being more restrictive and wrote a two functions to return a business that only contained a specific category (in this case "bars"):

```
def check_for_cat(categories, cat_type):
    """
    Function that returns whether or not given business is a given type, returns a boolean
    """
    # splits input category list from json file
    categories_list = categories.split(',')
    # loop through each category to see if it is equal to cat_type
    for category in categories_list:
        # lowercase the category and strip any leading/trailing spaces
        mod_cat = category.strip().lower()
        if mod_cat == cat_type:
            return True

    return False

def filter_yelplist(locations, cat_type):
    """
    Function that returns list of yelp locations filtered for presence of a business category
    """
    return [loc for loc in locations if check_for_cat(loc['categories'], cat_type) == True]

```

This additional filtering gave me about 4000+ bars to work with.

Next, I had to join this list of bars with the reviews for each bar. Normally in a SQL database, this would be accomplish by a simple join command as both the review data and the business data had a business id column. However MongoDB has a NoSQL database structure, so I was not able to accomplish the same thing (I did experiment with the $lookup command but it didn't do exactly what I wanted). I ended up writing the following query to get the relevant data where I would would loop the reviews database and do a match based on whether or not the review's business id was in the list of business ids from the bars subset table.

```
# Create subset of reviews to only find reviews that contain the bar ids
locations = list(db.bars.find({}, {'_id': 0, 'business_id': 1, 'categories': 1}))
filtered_bars = filter_yelplist(locations, 'bars')
allbar_ids = [i['business_id'] for i in filtered_bars]

db.reviews.aggregate([
   {'$match': {'business_id': {'$in': allbar_ids}}},
   {"$out": "bar_reviews"}]);

```

After all of the data processing I ended up with a subsetted set of data that contained 4000+ bars that each had at least 50 reviews. In total there were 800,000+ reviews that I could analyze.

[pymongo]: https://api.mongodb.com/python/current/

## Feature Engineering

### Topic Modeling

As I felt that the ambience of bar was a very important feature to capture, I thought that analyze the reviews for each bar by using NLP techniques would be a good approach to accomplish that. After experimenting with the different token vectorizers and dimensionality reduction techniques, I ended up with a combination of CountVectorizer and Latent Dirichlet Allocation (LDA) giving me the most interpretability. Before passing the individual words into the CountVectorizer I did lemmatize each word and the CountVectorizer did count both unigrams and bigrams. (Talk about removing stop words) I believe that CountVectorizer performed better than TFIDF because important words to describe a bar (i.e. beer, music, drinks) would show up consistently and CountVectorizer ended up promoting those words while TFIDF penalized them. By using LDA I was able to perform topic modeling and generated 9 topics that I felt were indicative of different types of bars in my dataset:

![](/public/Project_Fletcher/topic_table.png)

2 of the bar archetypes ended up being the most descriptive (Outdoor and Brunch) as most of the bars ended up being described by those two topics. This could be demonstrated by how those two topics mapped to the TSNE plots:

![](/public/Project_Fletcher/TSNE_outdoor.png)

![](/public/Project_Fletcher/TSNE_brunch.png)

The rest of the 7 topics ended up being more specific as they generally described a few bars in the dataset. For example the cigar lounge bar ended up being mapped to my TSNE plot as follows:

![](/public/Project_Fletcher/TSNE_cigar.png)

### Categorical Features

* Yelp dataset contained a lot of categorical features like whether a bar was good for dancing or if they had a television available, however I decided not to utilize any of those features as I wanted to determine those characteristics through the topic modeling. The only categorical feature I ended up taking from the dataset was the price range of the business.

* While the topic modeling would hopefully give me insight on the ambience of a bar, I wanted my model to focus on other aspects of a successful bar as well namely the types of alcohol. To accomplish this, I decided to try to amplify the presence of certain keywords as they were detected in the reviews. For example, for each bar I detected the number of times the token "whiskey" was mentioned. Dividing that count by the number of total reviews, I could then utilize that proportion to signal how much of a whiskey bar a given bar was as I believe that the more times whiskey was mentioned, the larger focus that bar had on whiskey. I ended up doing this presence type categorical feature on the keywords: whiskey, vodka, beer, wine, tequila, rum, shot, gin, brandy, soju, cider, and sake.

## Recommendation Engine

### Giving recommendations

* The way that a recommendation was given was by comparing the cosine similarity for the given bar to all the other bars in the dataset.

* Recommendation engine was deployed on a Flask app that can be found [here].
