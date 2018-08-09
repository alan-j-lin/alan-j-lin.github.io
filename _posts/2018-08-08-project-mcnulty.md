---
layout: post
title: Modeling Chance: Stealing Bases
---
One of the most exciting things that can occur in sports is when a baserunner attempts to steal a base. There are two outcomes, if the base runner times it properly they are able to easily steal a base and help advance the team's scoring chances:  
![](base steal gif)  
or they get caught and it's a wasted opportunity:   
![](base fail gif)  
Sometimes even something amazing can happen, look at this play by Jason Werth:  
![](jason werth gif)  

At heart of base stealing though is that it is a calculated gamble, it is highly dependent on the game situation whether or not stealing a base would be a good idea. It relies on decades of experience playing high level baseball and requires sound judgement from both the base runner and their manager. However, what if you could transform this subjective judgment to something that is more objective? Thus, I thought that with enough data I could build a model that could aid in this decision:

## Data:

### Acquiring Data:

To build this model I was able to find a data set on [Kaggle] from Sportsradar. Hosted on
Google's Bigquery platform, this MLB data set contained information on every play from the 2016
season which I ended up uploading this data to a PostgreSQL server as the initial dataset contained 760,000+ rows and 145 features.

One quirk in exploring this data set was first finding those base steal events.Initially I tried to find the base steal events
by just trying to find the entries whose description contained 'steal', but this query only resulted in
only about 500 entries which was too low for a single MLB season. I discovered that multiple relevant entries were
missing descriptions. Instead I queried the steal events using the following SQL query:

```
    SELECT *
    FROM baseball
    WHERE rob1_outcomeid LIKE '%CS%'
    OR rob1_outcomeid LIKE '%SB%'
    OR rob2_outcomeid LIKE '%CS%'
    OR rob2_outcomeid LIKE '%SB%'
    OR rob3_outcomeid LIKE '%CS%'
    OR rob3_outcomeid LIKE '%SB%'

```

As each rob (runner on base) had an outcome id, by looking for the presence of either 'SB' (stolen base)
or 'CS' (caught stealing) I was able to grab the steal events.

With 145 features, there was a lot of information that was contained in this dataset. While a lot of these features
would deem to be non-predictive, there were a couple features that I thought would be great to have but would allow the model too much information. To be specific, for each event there were features describing the pitch type, pitch speed, and pitch location.
From a baserunner's point of view this would be information that would be great to have as one could determine approximately
how much time the runner would have to reach the next base. However in the real world this information would not be available
to the runner or manager, instead they would have to guess based on past behavior the expected pitch that the pitcher would deliver. To mimic this, instead of using those three features I joined my dataset with player statistics from [Fangraphs]. Specifically I utilized the pitcher [pitch type distribution statistics] as a proxy for the pitch type and velocity, while the hitter [plate discipline statistics] was a proxy for pitch location.

[Kaggle]: kaggle.com/baseball
[Fangraphs]: fangraphs.com
[pitch type distribution statistics]: fangraphs.com
[plate discipline statistics]: fangraphs.com

### Data Cleaning:

While the Sportsradar data was incredibly detailed, there was some cleaning that had to be done to find the relevant entries. While this data set contained 760,000+ entries I had to subset the data to only including base steals events and excluding any pickoff events or games in the post season. My reasoning was that in order to reasonably categorize pickoffs I would need access to player movement data (which I did not) and that behavior in the post season would be different than that in the regular
season. These data cuts left me without about 3000+ events relevant to base steals.  This ended up giving me a distribution of events as follows:

![](data distribution)

I also ended up not including a large number features as information like gameID, attendance, venueName would not be predictive.

Besides subsetting there was entries where I was missing the player and hitter statistics from Fangraphs. This occurred when the listed pitcher and hitter from the Sportsradar entry was not included in the Fangraphs statistics. This was something that I expected as because of the year difference (2016 vs 2015), there would be players, like rookies, who would be playing in 2016 that did not play in 2015. To account for these missing values, I ended up imputing those columns with median values of each feature as calculated from the training data.

### Feature Generation:  

With the large number of categorical variables like pitcher and hitter handedness, I ended up having to dummifying a lot of features to make them interpretable to my models. In doing so, I made sure to not fall for the [dummy variable trap] (DVT). For instance, to determine the handedness of a hitter I created one column 'is_hitter_R' instead of two as that would fall into the DVT for this data set (there was only one hitter that was ambidextrous).

The other type of feature engineering that I utilized involved box-cox transformations. Many of the player statistic features were not normally distributed and tightly grouped, so to rectify that I applied a box-cox transformation. Box-cox transformations have the following form:  


Each feature was transformed with their corresponding optimized lambda, as an example:  


[dummy variable trap]: https://en.wikipedia.org/wiki/Dummy_variable_(statistics)


## Modeling:

### Metric:
As this was a classification problem there were a couple of different metrics that I could try to optimize my model for. Thinking about in terms of baseball manager the first option would to optimize in order to maximize the number base steals (minimizing false negatives), however I felt that such a strategy would be too risky as that would constantly put base runners in danger of being caught stealing. The converse would be that I could try to optimize for the caught stealing events (minimizing false positives), however this strategy could be deemed to be too passive. So instead, I decided to optimize for a metric which would balance for both considerations which ended up being an [F1] score:  

![](f1 formula)

### Model Selection:
In trying to optimize for F1 there were multiple types of classification models that I ended experimenting with. In the end, the two models that performed the best were a Logistic Regression (LR) model and a Gradient Boosted Trees model:

![](table)

As you can see in the table above, both models had very similar F1 scores on the training set. Thus, I ended up choosing the logistic regression model as my model of choice for two reasons. The first was that a LR model would be easier to interpret in terms of feature importance. The second was that in terms of run time a LR model would also be faster, which is important if you are trying to deploy an app that would be used in real time.

[F1]: https://en.wikipedia.org/wiki/F1_score

### Feature Importance:

From the logistic regression model, there were a couple of features that stuck out as shown in the table below:

![](table)

The most dominant and important feature by far was whether or not the base runner was on 1st base. In fact, out of all of my features it appeared to be twice as predictive as the next feature.

### Model Performance:

My LR model ended up having a F1 score of 0.924, which was very similar to training F1 score. This meant that my model was generalizing well to new data and was not overfitting. This was also reinforced by the learning curve for my model:  

![](learning curve)

Retraining my model on the full data set, I was able to then generate the following confusion matrix:  

![](confusion matrix)

With a final F1 score of 0.9354, the model performance ended up right between the training and test F1 scores. In addition, one way to interpret this model would be that for the 922 CS (caught stealing) events that it was able to classify, a ballpark figure of number of runs it could have possibly saved would be approximately 123. I came to this number by translating the game situation where each of the steals were attempted and relating it to an expected runs [table].

[table]: expectedruns

## Future Improvements:

(Talk about future improvements and flask app)
