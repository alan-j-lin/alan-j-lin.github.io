---
layout: post
title: Is That New Lunch Spot Overpriced?
---

How many times have you gone to a new restaurant where every dish seems overpriced? Consider when your regular lunch spot is offering a new special, is it even worth trying? Are restaurants trying to upcharge you on things like decor or reputation? What if you had a model that could you help predict the likelihood of that?

![](/public/Project_Luther/sushiburrito.jpg)
<center>
    <font size="2">
    <figcaption> <b>$15 for a sushi burrito???</b> </figcaption>
    </font>
</center>

I decided to design this model for my 2nd Metis project, where I would utilize linear regression to predict the price of a lunch dish based on the information that one could gather from the restaurant menu.

## Data

To train this model, there were three types of data that were obtained:

#### Restaurant Data

To put constraints on types of restaurants for the dataset, only restaurants in San Francisco, CA were picked. Also these restaurants would focus on 6 types of cuisines that were a diverse representative of lunch choices in SF:

1. American
2. Mexican
3. Mediterranean
4. African
5. Pakistani
6. Creole & Cajun

As there was no freely available database of restaurant menu data, I had to resort other tactics. Using [BeautifulSoup] (BS4) I was able to obtain the menu information for over 1000 restaurants in SF from a restaurant menu aggregator. These 1000 restaurants in turn gave me over 16,000 data points for dishes at different price points.

![](/public/Project_Luther/AllDishPriceHist.png)

[BeautifulSoup]: https://www.crummy.com/software/BeautifulSoup/

#### Ingredient Data

In order establish meaning from the restaurant menu text, my model would need a reference list of important words. Thus using BS4, I obtained a base ingredient list from some recipe related websites that contained 600+ unique food-related nouns.

#### Demographic Data

I was curious to see if there was any correlation between dish price and the neighborhood that each restaurant resided in. Luckily there were addresses and geolocation information for each restaurant in the dataset. As a proxy for neighborhood I used a restaurant's zipcode instead, as there was demographic data for each zipcode in SF from the [US Census]. I specifically used the 2016 American Community Survey (ACS) and the 2016 Economic Census.

[US Census]: https://www.census.gov/

![](/public/Project_Luther/zipcodemap.jpg)
<center>
    <font size="2">
    <figcaption> <b> There are over 20 zipcodes within just San Francisco </b> </figcaption>
    </font>
</center>


## Exploratory Data Analysis (EDA)

With the data in hand there were three aspects that I focused on during the EDA process:

#### Data Cleaning

In examining the raw data there was some necessary cleaning:

1. **Missing Info:** For certain entries, the dish price was not listed. This was generally the case when dishes were all grouped and set at the same price (i.e. dim sum). For these dishes I ended up dropping them from the dataset.

2. **Outliers**: Some dishes were much more expensive than expected based off their ingredients and stuck out as outliers. Digging in deeper, I discovered that many of these dishes were actually labeled as sharing platters or group meals. Because I was focusing on a meal for one, I ended up dropping these entries from the dataset.

#### Data Cut

As I wanted to restrict my model to predict the prices of dishes at lunch, I ended up removing any dishes that cost more than $20 or less than $7. However, even with this subset there were still 10,000+ data points.

![](/public/Project_Luther/SSDishPriceHist.png)

#### Feature Generation

With the restaurant text gathered, there was flexibility in feature generation using NLP (Natural Language Processing) such as:

1. By identifying nouns in the dish text and cross-referencing with the base ingredient list, the model was able to identify food-related nouns. These nouns were then fed into NLTK's [WordNet] corpus reader to identify base words (needed to account for tenses). These nouns were our "ingredients" and each was assigned as a categorical variable.

2. The frequency that each ingredient appeared in high-cost and low-cost dishes from the training set was used to generate a list of high-cost and low-cost ingredients. The frequency that these price-categorized ingredients appeared in each dish were also used as features in the model.

[WordNet]: http://www.nltk.org/howto/wordnet.html

## Model Selection

As price was what my model's target variable, the parameter I decided to optimize for was **[RMSE]** (root mean square error). RMSE was the appropriate metric because the RMSE value would be translated as the standard error for any predicted price. As an example, if my model had a test RMSE of $2.00 and the predicted price was $4.00 that would mean my price prediction would be $4.00 ± $2.00

In trying to improve my RMSE there were a couple of modifications that I did to my base linear regression model:

[RMSE]: https://en.wikipedia.org/wiki/Root-mean-square_deviation

#### Data Transformations

As seen previously, the pricing data was very skewed. By applying a [Box-Cox transformation] I was able to transform the data to be more Gaussian in nature.

![](/public/Project_Luther/BCSSDishPriceHist.png)

[Box-Cox transformation]: https://en.wikipedia.org/wiki/Power_transform

#### Regularization

With about 600 starting features in my model, regularization was sorely needed in order to help reduce the number of features. By using Lasso regularization, I was able to subtract 130 features from my model. Most likely, if I was more aggressive in reducing the number of features during the cross-validation process even more features would have been dropped.

#### Important Features

With regards to the most predictive features, there a mixture of expected and unexpected results. The most predictive features were:

| Feature             | Positive Weight | Negative Weight |
| :-----------------: | :------------:| :--------------:|
|Expensive Ingredients|          X      |                 |
|Cheap Ingredients    |                 | X               |
|Dish Text Length     |          X      |                 |
|Restaurant Type      |                 | X               |

While restaurant types (Mexican and Jerk) and specific ingredients were expectedly predictive, a real surprise was **dish text length**. Behind expensive ingredients (like crab, lobster, and duck) dish text length was actually the 4th strongest positive predictor where it demonstrated a log-squared relation. That actually made sense because more expensive restaurants would generally be more verbose and embellish dish descriptions.

One more note was that demographic features did not end up being particularly predictive, so any information regarding restaurant location ended up being dropped from the model.

#### Model Performance

My final model ended up having a train RMSE of **$2.44** and a test RMSE of **$2.63**.

![](/public/Project_Luther/Train_Residual_Plot.png)

![](/public/Project_Luther/Test_Residual_Plot.png)

As both values are similar and the residual plots are similar in shape, I believe my model was generalizing well and not overfitting.

So bringing this back to the original question, if you went to the restaurant *Best Mexican* and ordered a burrito that contained:

* Steak
* Salsa
* Cheese
* Tortilla
* Beans  


My model would predict a price of $7.80. If *Best Mexican* was charging anything more than **$10.43** then they would be ripping you off.

## Future Improvements

If I had more time and could redo this model, a couple of improvements I think would further minimize the RMSE would be to:

1. Capture more descriptive text features
  * Cooking descriptors (fried vs. baked), multi-word ingredients (i.e. goat cheese), and brand names aren't currently captured
2. Create temporal features
  * Different groups of dishes take different amounts of time (i.e. burritos vs pies) and this labor cost is currently not captured in the price prediction.
3. Select a better model
  * If I were to choose a different model I would utilize a Random Forest regressor due to the high number of categorical features in my model
  * Test RMSE using Random Forest is **$2.41**

The code and data for this model are available at my [Github repo].

[Github repo]: https://github.com/alan-j-lin/lunch_price_prediction
