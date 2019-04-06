---
layout: post
title:  "Modeling housing sale prices"
date:   2019-04-05 17:00:00 -0700
categories: Project
---

*An introduction to building and refining multiple linear regression models*

![histF](https://raw.githubusercontent.com/michaeljoseph04/blog/gh-pages/images/histF.jpeg)

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

The data I will be using is the famous Ames, Ohio housing dataset. This data is made of 79 variables about housing sales in Ames, Ohio. The variables range from square feet, to number of rooms, all the way to how big the bathrooms and garages are--even how many chimneys are available. It is similar to other data sets used by large companies like Zillow. It is available from [Kaggle](https://www.kaggle.com/)

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
If we look at the data, we can see the factors:
```
Order       PID MS.SubClass MS.Zoning Lot.Frontage Lot.Area Street Alley Lot.Shape
1     1 526301100          20        RL          141    31770   Pave  <NA>       IR1
2     2 526350040          20        RH           80    11622   Pave  <NA>       Reg
3     3 526351010          20        RL           81    14267   Pave  <NA>       IR1
4     4 526353030          20        RL           93    11160   Pave  <NA>       Reg
5     5 527105010          60        RL           74    13830   Pave  <NA>       IR1
6     6 527105030          60        RL           78     9978   Pave  <NA>       IR1
```

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

Next, we can create a test and training data set. We set seed to sample from so that the sample is reproducible, then use R's `sample()` to produce a random number of row values to sample from. To do this, we specify the whole sequence of rows to sample from with `seq_len()`, and pass it the number of rows of the sales dataset (`nrow(sales)`). Then we have two choices: either split the size in half, or, to have a bigger sample, make the size 3/4 of the number of total rows, rounded down (with `floor()`). I'll do the latter. I then use the split to sample from the rows specified, and then the rest of the rows:
```
  set.seed(2019)
  split <- sample(seq_len(nrow(sales)), size = floor(0.75 * nrow(sales)))
  train <- sales[split, ]
  test <- sales[-split, ]
```
For the model, I'll select several variables to make the regression model. There are 51 categorical variables and 28 continuous variables. I'll try and select some continuous variables which seem important to begin with. A better model would try and integrate as many variables as available. For now, let's select:

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
At this point, we should do several things: namely, check for missing variables, and decide what to do with these missing elements in the data. I'm happy with how things look, for now, since not all the cases can be eliminated from the beginning. Next, we should seriously consider the distribution of the dataset: a linear regression assumes that the residuals have a normal distribution, and so the data might need to be transformed to a log scale: in this case, I'm okay with the skew, though I will return to this later.

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
- Creating a binary indicator, rather than a continious variable, for something which might matter over a certain threshold (such as our wood deck size)
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

  e <- sum((test$SalePrice - prediction) ^ 2)
  t <- sum((test$SalePrice - mean(test$SalePrice)) ^ 2)
  1 - e/t

This gives us:
  ```
  [1] 0.7720008
  ```

Further refinements to the model may effectively make this value closer to 1, which is what we would want most. In further posts I will indicate more how to do that, but I hope I've given you the tools to begin creating and evaluating models for the purposes of data exploration and for eventual analysis.
