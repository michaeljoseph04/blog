---
layout: post
title:  "Modeling sales Prices"
date:   2019-04-05 17:00:00 -0700
categories: Project
---

*An introduction to building and refining multiple linear regression models*

![histF](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/histF.jpeg)

## 1.1 Overview

Beginning to take data science methods and apply them to various urban policy questions can be useful, if only for exploratory analysis. While policy questions require more calibrated economic modeling, often we want some effective models to make adjustments to assumptions or to inform decisions. For this, the multiple linear regression is a useful tool, and the data science method of splitting data into training and test datasets works well to begin to build a model and learn more about the dataset. For instance, what factors go into sales sale prices? In this quick introduction, which draws from Brett Lantz's *Machine Learning with R*, I am going to do some basic multiple linear regression modeling to predict housing selling prices in R.

This will include:
- Importing the dataset
- Looking for relationships between variables
- Fitting a model
- Evaluating a model

Be sure to go to the project [repository](https://github.com/michaeljoseph04/housing-prices) to find the code.

## 1.2 Importing and Wrangling Data

The data I will be using is the famous Ames housing dataset, often used to teach data science. It is available from [Kaggle](https://www.kaggle.com/)

First, let's import the libraries we will need:
```
  library(Hmisc)
  library(psych)
  library(car)
  library(tidyverse)
```
`psych` has useful tools for looking for relationships between variables. `car` is a regression modeling library.

Next, I'll import the data, which I've saved to disk:
```
  sales <- read.csv("AmesHousing.csv", stringsAsFactors = FALSE)
```
If we look at the data, it has several variables.
```
Order       PID MS.SubClass MS.Zoning Lot.Frontage Lot.Area Street Alley Lot.Shape
1     1 526301100          20        RL          141    31770   Pave  <NA>       IR1
2     2 526350040          20        RH           80    11622   Pave  <NA>       Reg
3     3 526351010          20        RL           81    14267   Pave  <NA>       IR1
4     4 526353030          20        RL           93    11160   Pave  <NA>       Reg
5     5 527105010          60        RL           74    13830   Pave  <NA>       IR1
6     6 527105030          60        RL           78     9978   Pave  <NA>       IR1
```
79 variables in fact, ranging from how many chimneys to the pool size to the condition of the sales.

## 1.3 Exploring relationships

What we are interested in is the sale price, *SalePrice*. We can plot its distribution:
```
  ggplot(data=sales, aes(x=SalePrice)) +
    geom_histogram()
```
![hist1](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/hist1.jpeg)

We can also see that there are relationships between the sale price and other variables. For instance, we can plot the relation of sale price to lot size:
```
ggplot(data=sales, aes(x=SalePrice, y=Lot.Area)) +
  geom_point()+
  geom_smooth(method="lm", se=FALSE)+
  labs(title="Sale Price vs. Lot Area")
```
![sale-price-lot](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/sale-price-lot.jpeg)

## Fitting the Model

Next, we can create a test and training data set:
```
  set.seed(2019)
  split <- sample(seq_len(nrow(sales)), size = floor(0.75 * nrow(sales)))
  train <- sales[split, ]
  test <- sales[-split, ]
```
For the model, I'll select several other variables to make the regression model:
- Lot area
- Living area
- Garage area
- Basement square feet
- Year built
- Wood deck square feet

```
  # Make the initial model.
  train <- train %>% select(SalePrice,
                            Lot.Area,
                            Gr.Liv.Area,
                            Garage.Area,
                            Total.Bsmt.SF,
                            Year.Built,
                            Wood.Deck.SF)
```
At this point, we should do several things: namely, check for missing variables in each of the areas, and decide what to do with these elements in the data. Next, we should look at the initial correlations. We can do a pair panel with the `psych` package:
```
  pairs.panels(train, col = "green")
```
![pairpanel](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/pairpanel.jpeg)

We see the correlations with each variable. Interestingly, some of these are not as high as one would expect. Finally, we can fit a linear model with `lm()`:
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
There are three, but overall the model seems to work for 72% of cases.

# Evaluating the model

At this point, knowing both how our model is working and how well it is not working, we could modify our model, in particular in four ways:
- Adding a non-linear term
- Creating a binary indicator, rather than a continious variable, for something which might matter over a certain threshold (such as our wood deck size)
- Specifying interaction effects between two highly correlated variables (the total living area and the total basement square feet, for instance)
- Or eliminating variables which have p-values over 0.05

When we have completed this, we then select the variables we want from our test set and make the prediction of the model on that:
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

  e <- sum((test$SalePrice - prediction) ^ 2)
  t <- sum((test$SalePrice - mean(test$SalePrice)) ^ 2)
  1 - e/t

This gives us:
  ```
  [1] 0.7720008
  ```

Further refinements to the model may effectively make this value closer to 1, which is what we would want most. In further posts I will indicate more how to do that, but I hope I've given you the tools to begin creating and evaluating models for the purposes of data exploration and for eventual analysis.
