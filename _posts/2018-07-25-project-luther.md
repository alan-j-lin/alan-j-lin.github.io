---
layout: post
title: Predicting Lunch Price (Metis Project 2)
---

How many times have you gotten a restaurant recommendation from a friend or review, where the price of dishes when you sit down is significantly more expensive than what you had expected? Is it the case that the restaurant is ripping you off based on their reputation, or that the components of the dish justify that price? What if you had a model that could you help predict the likelihood of that?

Such a model was what 2nd Metis project was based around, where I built a linear regression model to predict the price of a single dish at lunch based on the information that one could gather from the restaurant menu.

## Data

To come up with a model to predict price, I decided that there were three types of data that required:

#### Restaurant Data

To put constraints on types of restaurants the model would examine, I decided to focus on restaurants in San Francisco, CA. In turn, I decided to only look at 6 different cuisines that I felt were representative of different lunch choices in SF:

1. American
2. Mexican
3. Mediterranean
4. African
5. Pakistani
6. Creole & Cajun

As there was no freely available database of restaurant menu data, I had to resort to scraping. Using [BeautifulSoup] I was able to retrieve the menu information for over 1000 restaurants in SF by scraping a restaurant menu aggregator. These 1000 restaurants in turn gave me over 16,000 data points for dishes at different price points.

![](alldatahist)

[BeautifulSoup]: www.BeautifulSoup.com

#### Ingredient Data

In order to make sense of the restaurant text data, my model would need a reference list of important words. Thus, BeautifulSoup was used to scrap the [BBC ingredient] and [Foodwise] websites to build a base ingredient list which would be referenced by the model. From these two websites I was able to get a set of 600+ unique food-related nouns.

[BBC ingredient]: www.bbc.co.uk/ingredient
[Foodwise]: www.foodwise.com

#### Demographic Data

Contained in the restaurant data, there was also addresses and geolocation information. Thus, I was curious to see if there was any correlation between dish price and the neighborhood that each restaurant resided in. As a proxy for neighborhood, there was demographic data for each zipcode in SF from [US Census], specifically utilizing the 2016 American Community Survey (ACS) and the 2016 Economic Census.

[US Census]: www.uscensus.gov


## Exploratory Data Analysis

With the data in hand there were three aspects that I focused on during the EDA process:

#### Data Cleaning

In examining the raw scraped restaurant data there was some data cleaning necessary in order to make the analysis fruitful:

1. Based on the menu layout at times there were dishes were prices were not listed. This was generally the case when dishes were all of the same price based on menu section (i.e. dim sum). For these dishes I ended up dropping their entries from the dataset.

2. Some dishes were much more expensive than expected and stuck out as outliers when looking at all of the prices as a whole. Digging in deeper, I discovered that many of these dishes were actually labeled as sharing platters or group meals. Because of this discrepancy in price, I ended up dropping these entries from the dataset.

#### Data Cut

As I wanted to restrict my model to predict the prices of dishes at lunch, I ended up removing any dishes that cost more than $20 or less than $7. However, even with this subset there were still more than 10000 data points.

#### Feature Generation

With so much text gathered from the restaurant menus, there was a lot of room to do feature generation based on NLP. For this model, I ended up creating a few features:

1. By identifying nouns in the dish text and cross-referencing it with the base ingredient list, the model was able to identify food-related nouns. These nouns were then fed into NLTK's [WordNet] corpus reader to identify base words, this was done to account for tenses. These nouns were then our "ingredients".

2. The frequency that each ingredient appeared in high-cost and low-cost dishes was used to generate a list of high-cost and low-cost ingredients based on our training set.

3. Each ingredient and cuisine type was also used as a categorical dummy variable in the model.

[WordNet]: http://www.nltk.org/howto/wordnet.html

## Model Selection

As price was what my model was trying to predict, the parameter I decided to optimize for was **root mean square error** (RMSE). RMSE was an appropriate measure because the RMSE value would be translated as the standard error for any predicted price from my model.

In trying to improve my RMSE there were a couple of modifications that I did to my base linear regression model.

#### Data Transformations

As can be seen below, the pricing data was very skewed.

![](hist)

By applying a [Box-Cox transformation] we were able to transform the data to be more Gaussian in nature.

![](bchist)

[Box-Cox transformation]: https://en.wikipedia.org/wiki/Power_transform

#### Regularization

With about 600 starting features in my model, regularization was sorely needed in order to help reduce the amount of features present. By using Lasso regularization, I was able to reduce the amount of total features present in my model by 130. Most likely, if I was more aggressive in reducing the number of features during the cross-validation process these amount would have reduced further.

#### Important Features

With regards to important features identified by the model, there a mixture of expected and unexpected results. The strongest predictive features were:

| Feature             | Positive Weight | Negative Weight |
| :-----------------: | :------------:| :--------------:|
|Expensive Ingredients|          X      |                 |
|Cheap Ingredients    |                 | X               |
|Dish Text Length     |          X      |                 |
|Restaurant Type      |                 | X               |

While restaurant types (Mexican and Jerk) and specific ingredients were expectedly predictive, a real surprise was **dish text length**. Behind ingredients like crab, lobster, and duck dish text length was actually the 4th strongest positive predictor where it has a log-squared relation. I interpreted that made sense because more expensive restaurants would generally be more verbose and embellish dish descriptions.

#### Model Performance

My final model ended up having a train RMSE of **$2.44** and a test RMSE of **$2.63**. As both values are similar and the residual plots are similar in shape (as shown below), I believe my model was generalizing well and not overfitting. However there could have been improvements made to decrease RMSE.

![](trainimage)

![](testimage)

## Future Improvements

With regards to improving my model if I had further time in the future, a couple of ideas would be to:

1. Capture more descriptive text features
  * Descriptions on the cooking process, multi-word ingredients (i.e. goat cheese), and brand names aren't currently captured
2. Temporal Features
  * Different dishes take differing amounts of time, this labor cost is currently not captured in the price prediction
3. Model Selection
  * If I were to choose a different model I would utilize a Random Forest regressor due to the high number of categorical features in my model
  * Test RMSE using Random Forest is **$2.41**

All jupyter notebooks and data contained at the following [repo].

  [repo]: https://github.com/alan-j-lin/lunch_price_prediction
