---
layout: post
title:  "Modeling housing sale prices"
date:   2019-04-05 17:00:00 -0700
categories: Project
---

*An introduction to building and refining multiple linear regression models*

![test1](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/test1.jpeg)

## 1.1 Overview

Beginning to take data science methods and apply them to various urban policy questions can be useful, if only for exploratory analysis. While policy questions require more domain specific (that is, economic) modeling, often we want some effective models to begin to inform how our domain knowledge will fit with the work of generalizing from datasets--that is, the most useful reductions of the total number of possible models to the more domain-specific.

For this still-exploratory work, multiple linear regression is a useful tool, as is the data science method of splitting data into training and test datasets. For instance, what factors go into house sale prices? In this quick introduction, which draws from the general outlook of Brett Lantz's *Machine Learning with R*, and several great walkthroughs emerging from a Kaggle competition using the dataset, I am going to show how to do some basic multiple linear regression modeling to predict housing selling prices in R.

This will include:
- Importing the dataset
- Looking for relationships between variables
- Fitting a model
- Evaluating a model

Be sure to go to the project [repository](https://github.com/michaeljoseph04/housing-prices) to find the code. The analysis will work with one linear model, and then suggest ways that it can be integrated into further analysis

## 1.2 Importing and Wrangling Data

The data I will be using is the famous Ames, Ohio housing dataset. This data is made of 80 variables about housing sales in Ames, Ohio from 2006-2010, taken from the county assessor's office (more can be found out from [this paper on the origins of the dataset](http://jse.amstat.org/v19n3/decock.pdf)). The variables range from living area, to number of rooms, all the way to how big the bathrooms and garages are, to the quality and condition of the house. It even includes how many chimneys are available. It is similar to other data sets used by large companies like Zillow. It is available from [Kaggle](https://www.kaggle.com/).

First, let's import the libraries we will need:
```
  library(psych)
  library(car)
  library(Hmisc)
  library(tidyverse)
```
`psych` has useful tools for looking for relationships between variables, particularly the pair panel plot. The [Companion to Applied Regression](https://cran.r-project.org/web/packages/car/index.html) package, `car`, has all the tools I need for advanced regression. I like to load [Harrel Miscellaneous package](https://cran.r-project.org/web/packages/Hmisc/index.html) as well, which has many handy tools for data science.

Next, I'll import the data, which I've saved to disk:
```
  sales <- read.csv("AmesHousing.csv", stringsAsFactors = FALSE)
```
Let's look at the data with `head()` to get a glimpse:
```
Order       PID MS.SubClass MS.Zoning Lot.Frontage Lot.Area Street Alley Lot.Shape
1     1 526301100          20        RL          141    31770   Pave  <NA>       IR1
2     2 526350040          20        RH           80    11622   Pave  <NA>       Reg
3     3 526351010          20        RL           81    14267   Pave  <NA>       IR1
4     4 526353030          20        RL           93    11160   Pave  <NA>       Reg
5     5 527105010          60        RL           74    13830   Pave  <NA>       IR1
6     6 527105030          60        RL           78     9978   Pave  <NA>       IR1
...
```
Now we will need to explore the data some more.

## 1.3 Exploring relationships

A quick call to `str()` shows us the structure of the dataset:
```
'data.frame':	2930 obs. of  82 variables:
 $ Order          : int  1 2 3 4 5 6 7 8 9 10 ...
 $ PID            : int  526301100 526350040 526351010 526353030 527105010 527105030 527127150 527145080 527146030 527162130 ...
 $ MS.SubClass    : int  20 20 20 20 60 60 120 120 120 60 ...
 $ MS.Zoning      : chr  "RL" "RH" "RL" "RL" ...
 $ Lot.Frontage   : int  141 80 81 93 74 78 41 43 39 60 ...
 $ Lot.Area       : int  31770 11622 14267 11160 13830 9978 4920 5005 5389 7500 ...
 $ Street         : chr  "Pave" "Pave" "Pave" "Pave" ...
 ...
```
There are 82 variables in the data set: 79 of them are variables which describe the house itself (the others serve to identify it). What we are interested in, our dependent variable, is the sale price, *SalePrice*. We can plot its distribution:
```
ggplot(data=sales, aes(x=SalePrice)) +
  geom_histogram(fill="lightblue", color="darkblue")+
  theme_classic()
```
![hist1](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/hist1.jpeg)

The default bins for the histogram are of course 30 cases. We can see that the data has a bit of a skew. This might requrie a log transformation for our regression, but we will come back to that in a moment.

We can also see that there are a range of relationships between the sale price and other variables. For instance, we can plot the relation of sale price to lot size by plotting with `geom_point()`. We can also see what a simple linear regression (unlike a multiple linear regression) would look like by adding a `geom_smooth()` layer, and passing the method `lm`, or *linear model*, to it (I specify `se=FALSE` to decline showing the standard error of the regression):
```
ggplot(data=sales, aes(x=Lot.Area, y=SalePrice)) +
  geom_point()+
  geom_smooth(method="lm", se=FALSE)+
  labs(title="Sale Price vs. Lot Area")
```
![lot-area](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/lot-area.jpeg)

Let's look at this more closely. It is hard to tell whether there is any relationship at all between the variables, in fact. If we would remove the regression line, could I tell whether having a larger lot actually influences the sale price? I'm not so sure. However, let's compare this to the *Garage.Area*:

![garage-area](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/garage-area.jpeg)

Here there appears to be a relationship, except we have a number of housing sales with no garage area. This may indicate that we have to weight the variable appropriately. Let's compare both of these to another variable, *Gr.Liv.Area*, that of the living area in square feet inside the house:

![living-area](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/living-area.jpeg)

There is a clear relationship between the variables. There are three outliers, it appears: all houses with extremely large living areas which sold for low prices, and two variables which have high living areas which sold for high prices. Aggregating the data might prove tighter relationships, but it is hard to deny that living area appears to matter to sale price. One way to deny this, however, would be to consider the possibility that the sheer amount of houses built recently (which tend to have a larger living area than houses in the past) might also be accounting in some way for the high sale price:

![year-built](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/year-built.jpeg)

Indeed, houses which are recently built seem to sell well. We can get more specific to understand our dataset better, and indeed questions begin to multiply: houses built between 1940-1980 seem to account for a large amount of the total sales. Also, there are less sales of houses between 1980-1990, and a perhaps-not-insignificant amount of houses sold which are older than 1940, and yet, even given this, the relationship between year built and sale price appears to be positive: the more recent the year, the more a house sells for. One thing to note: though the years progress as a continuous variable, sales cluster around each year almost as if it were discrete. This is something to pay attention to as we progress: ideally, it would be good to have an exact date at which the house was built, but we are operating without this. For similar reasons which applies to the dependent variable, it may be useful to understand that we are not necessarily dealing with sales themselves which are the same across time:
```
ggplot(data=sales, aes(x=Mo.Sold))+
  geom_bar(color="darkblue", fill="lightblue")+
  scale_x_continuous(breaks=seq(1:12))+
  labs(title=paste("Sales per Month", min(sales$Yr.Sold), "to", max(sales$Yr.Sold)))+
  theme_classic()
```
![year-built](/images/sales-per-month.jpeg)

This seems to have little overall effect on the sale price itself, when we plot it:

![year-built](/images/sale-prices-month.jpeg)

But it is worth noting that a different kind of analysis may take this into account as well.

Overall, this should indicate the range of relationships with sale price possible with several variables in the dataset, and indicate how some variables might be interacting with each other. In the end, it is important to remember that I am not trying to understand which factor *caused* housing sales prices to increase, but to understand which are *most influential to the sales price.*

One more important thing to note: this initial exploration of the relationships simply considered the continuous variables, and I saw how different each of the possible relationships actually could be. However, there are, in fact, 51 categorical variables and 28 continuous variables in the dataset, and so it may be important for us to figure out a way to weight the categorical variables and factor them into our analysis. We will return to this question.

## Fitting the Model

Next, we can create a test and training data set. We set seed to sample from so that the sample is reproducible, then use R's `sample()` to produce a random number of row values to sample from. To do this, we specify the whole sequence of rows to sample from with `seq_len()`, and pass it the number of rows of the sales dataset (`nrow(sales)`). Then we have two choices: either split the size in half, or, to have a bigger sample, make the size 1/2 of the number of total rows, (the number will be an whole number, but in other cases we might want to round this down by wrapping it with `floor()`). I'll do the latter. I then use the split to sample from the rows specified, and then the rest of the rows:
```
  set.seed(2019)
  split <- sample(seq_len(nrow(sales)), size = 0.5 * nrow(sales))
  train <- sales[split, ]
  test <- sales[-split, ]
```
For the model, I'll select several variables to make the regression model. I'll try and select some continuous variables which seem important to begin with. A better model would try and integrate as many variables as available. For now, let's select:

- Lot area
- Living area
- Garage area
- Basement square feet
- Year built
- Wood deck square feet

I'll do this with `dplyr`'s `select()`, to select them along with the sale price:
```
  train <- train %>% select(SalePrice,
                            Lot.Area,
                            Gr.Liv.Area,
                            Garage.Area,
                            Total.Bsmt.SF,
                            Year.Built,
                            Wood.Deck.SF)
```
At this point, we should do several things: namely, check for missing variables, and decide what to do with these missing elements in the data. After investigating them, I'm happy with imputing these variables to a 0 value, and not assuming that they are missing variables--except in one case, that of the wood deck square footage, which ideally should be a binary variable. For now I will impute that also to 0. Next, we should seriously consider the distribution of the dataset: a linear regression assumes that the residuals have a normal distribution, and so the data might need to be transformed to a log scale: for now, I'm okay with the skew, though I will return to this later when I evaluate the model.

After that, we should look at the initial correlations from the variables we selected. We can do a pair panel with the `psych` package:
```
  pairs.panels(train, col = "green")
```
![pairpanel](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/pairpanel.jpeg)

We see the correlations with each variable. Interestingly, some of these are not as high as one would expect (lot area), others are higher (wood decks?). Finally, we can fit a linear model with `lm()`:

```
  fit <-  lm(SalePrice ~ Lot.Area +
               Gr.Liv.Area +
               Garage.Area +
               Total.Bsmt.SF +
               Year.Built +
               Wood.Deck.SF,
             data = train)
```
If we look at the summary of the fit:
```
  summary(fit)
```
This shows that the coefficient of determination, R-squared, adjusted for the amount of variables, is 0.723, and shows that each of the variables has a p-value nearer to 0 than even .0001:
```
Coefficients:
                Estimate Std. Error t value Pr(>|t|)    
(Intercept)   -1.428e+06  6.842e+04 -20.872  < 2e-16 ***
Lot.Area       3.870e-01  1.125e-01   3.438 0.000596 ***
Gr.Liv.Area    6.602e+01  2.161e+00  30.558  < 2e-16 ***
Garage.Area    6.590e+01  5.497e+00  11.990  < 2e-16 ***
Total.Bsmt.SF  3.860e+01  2.502e+00  15.426  < 2e-16 ***
Year.Built     7.260e+02  3.531e+01  20.560  < 2e-16 ***
Wood.Deck.SF   3.642e+01  7.503e+00   4.855 1.29e-06 ***
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1
```
We may also want to consider the outliers which are visible in the Q-Q plot:
```
  plot(fit)
```
There are three, but the adjusted R-squared tells us overall the model seems to work for 72% of cases. That's not bad for a first try.

# Evaluating the model

Still, it's not the best, and at this point, knowing both how our model is working and how well it is not working, we could modify our model, in particular in four major ways:

- Adding a non-linear term
- Creating a binary indicator, rather than a continuous variable, for something which might matter over a certain threshold (such as our wood deck size)--and eliminating NA-values on that basis
- Specifying interaction effects between two highly correlated variables (the total living area and the total basement square feet, for instance)
- Or eliminating variables which have p-values over 0.05 or even smaller.

Also we may want to transform the data to log scale, and/or integrate more of the categorical variables, which we need to understand better. When we have completed this, we then select the variables we want from our test set and make the prediction of the model on that:
```
  test <- test %>% select(SalePrice,
                            Lot.Area,
                            Gr.Liv.Area,
                            Garage.Area,
                            Total.Bsmt.SF,
                            Year.Built,
                            Wood.Deck.SF)

  prediction <- predict(fit, newdata = test)
```
We can then compare the prediction data set with our test dataset:
```
  a <- data.frame(test$SalePrice) %>% mutate(type = "test") %>% rename(price = test.SalePrice)
  b <- data.frame(prediction) %>% mutate(type = "model") %>%
    rename(price = prediction) %>%
    bind_rows(a)

  ggplot(data = b, aes(x=price, fill=type))+
    geom_histogram(alpha=.9)+
    theme_classic()
```
![modeltest](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/modeltest.jpeg)

Finally, what we ideally want is the r-squared value to evaluate the goodness of the model's fit. So we can calculate this by comparing the prediction and the test dataset:
```
  e <- sum((test$SalePrice - prediction) ^ 2)
  t <- sum((test$SalePrice - mean(test$SalePrice)) ^ 2)
  1 - e/t
```
This gives us:
  ```
  [1] 0.7720008
  ```
Further refinements to the model may effectively make our r-squared value closer to 1, which is what we would want most.
A quick plot of the prediction and the test against each other shows how far the two differ:

![test1](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/test1.jpeg)

In further posts I will indicate more how to do that, but I hope I've given you the tools to begin creating and evaluating models for the purposes of data exploration and for eventual analysis.
