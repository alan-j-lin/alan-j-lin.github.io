---
layout: post
title: Using LDA to build a missing Yelp feature
---

If you have ever been out for drinks in the financial district of San Francisco, you may have stumbled across [Mikkeller bar]. This Dutch beer hall is a great place to go with friends after work with an atmosphere that is just lively enough for great conversation. With an expansive draft list and selection of sours, it is the type of place that I want to always drink at.

Now, if I looked up Mikkeller on Yelp I would not only get the bar itself but bars of a similar profile that are in the area. These additional options are very useful as I might not always want to go to Mikkeler, but would want to try a different place that has a similar vibe.

![](/public/Project_Fletcher/mikkeller_sf.png)

However this type of search does not translate when Yelp is trying to base its search on a different location. For instance, if I tried to search for Mikkeller bar in Chicago:

![](/public/Project_Fletcher/mikkeller_chicago.png)

While you could possibly get similar results by searching for specific keywords like beer hall or gastropub, I felt that such a more specific bar search method could be implemented. Why is it that Yelp does not have a feature where the user could provide inputs on bars that they enjoy, and then expand on that search at a location of the user's choosing? This search method would  also be able to capture less tangible aspects like ambience. Therefore I decided that for my 4th [Metis] project, I would build this feature using unsupervised learning and NLP techniques.

[Mikkeller bar]: https://www.yelp.com/biz/mikkeller-bar-san-francisco
[Metis]: https://www.thisismetis.com/

## Data

### Data Sources:

For this project, I ended up getting my data from the [Yelp challenge dataset]. A publicly available set of data for students to do research on, this was an invaluable resource as Yelp's TOS is very anti-webscraping. The dataset itself was huge as there was over 6 million reviews for 800,000+ different businesses, spread out over 10 different metropolitan areas in the US and Canada. So much geographic diversity was a bonus for my project as I wanted it to be location agnostic. The data from Yelp was stored in .json format so I queried and filtered the data using a local MongoDB instance. In retrospect as the .json files were massive (the review text JSON was 4.72 GB), analyzing data for this project would have been more efficient if I had transferred everything to a large memory AWS instance. The 8 GB of RAM that my Macbook Air was not quite adequate enough to run analysis on the data without micromanaging computer resources.

[Yelp challenge dataset]: https://www.yelp.com/dataset/challenge

### Data Filtering:

After loading my data into MongoDB, my first step for data processing was to filter the list of businesses for bars only. My first approach (using [pymongo]) was to perform a regex, matching any business that had "Bars" in its category type. "Bars" also had to have at least 50 reviews as so that there would be enough text data to draw inferences:

```
# Create subset of business that contain bars in question
db.business.aggregate([
     {"$match": {'categories': {'$regex': 'Bars'}}},
     {"$match": {"review_count": {"$gte": 50}}},
     {"$out": "bars"}]);

```

This approach did ending up decreasing the number of businesses to search through from 180,000+ to 5000+. However, one quirk I quickly realized was that the regex approach not only captured bar types like wine bars or sports bars, but also sushi bars. I ended up being more restrictive and wrote two functions to return a business that only contained a specific category (in this case "bars"):

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

Next, I had to join the list of bars with the reviews for each bar. Normally in a SQL database, this would be accomplish by a simple join command. However MongoDB has a NoSQL database structure, so I was not able to do that (I did experiment with the $lookup command). I ended up writing the following query to get the relevant data:

```
# Create subset of reviews to only find reviews that contain the bar ids
locations = list(db.bars.find({}, {'_id': 0, 'business_id': 1, 'categories': 1}))
filtered_bars = filter_yelplist(locations, 'bars')
allbar_ids = [i['business_id'] for i in filtered_bars]

db.reviews.aggregate([
   {'$match': {'business_id': {'$in': allbar_ids}}},
   {"$out": "bar_reviews"}]);

```

After all of the data processing I ended up with a subset of data that contained 4000+ bars that each had at least 50 reviews. In total there were 800,000+ reviews that I could analyze.

[pymongo]: https://api.mongodb.com/python/current/

### Text Processing

After I had the data filtered, I still had to process the review text itself before passing it to a tokenizer. There were three things that I did:

1. Remove numbers and punctuation by using regex
2. Convert all case types to lowercase
3. Lemmatize all words

[Lemmatization] was important because I didn't want the model to include different forms of the same word. Lemmatizing accomplishes that as it returns the word's lemma or "dictionary form", so plurals and participial forms of a given word are replaced with the base form. For example, the lemma of "walking" would be "walk" and the lemma of "better" would be "good".

[Lemmatization]: https://en.wikipedia.org/wiki/Lemmatisation

## Feature Engineering

As I felt that the ambience of a bar was important, I thought that analyzing the reviews for each bar by using NLP techniques would allow me to capture that. After experimenting with the different token vectorizers and dimensionality reduction techniques, I ended up utilizing a combination of count vectorization and Latent Dirichlet Allocation (LDA) to give me the most interpretability.

### Vectorization

In performing vectorization I needed to generate a relevant list of tokens from the review text. While NLTK provides a base set of English stop words to be excluded, there were additional tokens that had to be excluded in the topic generation. The types of tokens that ended up in the exclusion list fell into two categories: location-related and food-related. Location-related tokens were excluded in order to make the model location agnostic, while food-related tokens were excluded so that the topics were focused on the drinks and the ambience. In terms of the type of vectorizer, the CountVectorizer model ended up performing better than TFIDF. I believe that the CountVectorizer performed better than TFIDF because important words to describe a bar (i.e. beer, music, drinks) would show up repeatedly and the CountVectorizer ended up promoted repetition while TFIDF penalized it.

### Topic Modeling

By using LDA I was able to perform topic modeling and generated 9 topics that I felt were indicative of different types of bars in my dataset:

![](/public/Project_Fletcher/topic_table.png)

2 of the bar archetypes ended up being the most descriptive (Outdoor and Brunch) as most of the bars ended up being described by those two topics. This could be demonstrated by how those two topics mapped to the TSNE plots:

![](/public/Project_Fletcher/TSNE_outdoor.png)

![](/public/Project_Fletcher/TSNE_brunch.png)

The rest of the 7 topics ended up being more specific as they generally described a few bars in the dataset. For example the arcade and video game bars ended up being mapped to my TSNE plot as follows:

![](/public/Project_Fletcher/TSNE_arcade.png)

### Categorical Features

In addition to the NLP generated features, there were some valuable categorical features added to the model. The Yelp dataset actually contained many categorical features: i.e. whether or not a place was good for dancing or if they had a television available. However I decided not to utilize any of those features as I wanted to determine proxies for those characteristics through topic modeling. Thus, the only categorical feature I ended up taking directly from the dataset was the price range of the business.

While the topic modeling would hopefully give me insight on the ambience of a bar, I wanted my model to focus on the type of alcohol the bar primarily served too. To accomplish this, I decided to try to amplify the presence of certain keywords in the reviews. For example, for each bar I detected the number of times the token "whiskey" was mentioned. Dividing that count by the number of total reviews, I have a proportion that signaled how much of a whiskey bar a given bar was. I ended up creating these presence type categorical features on the keywords: whiskey, vodka, beer, wine, tequila, rum, shot, gin, brandy, soju, cider, and sake.

## Recommendation Engine

### Giving recommendations

With the features established, I created a pipeline to generate features for each of the bars in my dataset. A recommendation could then be given by comparing the [cosine similarity] of an input bar against all the other bars in the dataset. If there were multiple bars given as input, the cosine similarity of the averaged vector of the input bars was what was used. This similarity comparison formed the basis of my recommendation engine. Based off the dataset these were some of the possible recommendations:

![](/public/Project_Fletcher/recommendation1.png)

![](/public/Project_Fletcher/recommendation2.png)

This recommendation engine was then deployed on Heroku through a Flask app and an interactive Tableau dashboard as shown:

![](/public/Project_Fletcher/app.gif)

If you would like to try out some recommendations yourself you can access the app [here].

[cosine similarity]: www.google.com/cosine_similarity
[here]: www.app.com

## Future Work

If given more time to extend this project, some improvements that I would have liked to work on:

* **More Relevant Data**: The current dataset did not include bar locations from major metro areas like San Francisco or New York, which I believe would help to create more relevant recommendations

* **Linking Interactive Graphics**: If this feature was actually to be deployed through Yelp or a similar service, I feel that interactive graphics would be a big selling point. Thus, I would have liked to link the Tableau dashboard and the Flask interface so that there was a greater degree of control of the end user.

If you would like to learn more about this project you can find the code and data at my [Github repo].

[Github repo]: https://github.com/alan-j-lin/new_yelp_feature
