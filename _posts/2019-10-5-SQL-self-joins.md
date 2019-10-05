---
layout: post
title: SQL self joins, like ARIMA in my mind
categories: [programming]
tags: [sql, self-join]
published: true
excerpt-separator: <!--more-->
---

I have a problem, where I was handed an analytical table and I have to display, in a data viz tool, several different periods of the same data in the same table. Something like this:
`
| row_label   | count_now   | count_last_month | count_last_quarter | ... |
|-------------|-------------|------------------|--------------------|-----|
| Value row 1 | 2576821     | 2576521          | 33348930           | ... | 
| Value row 2 | 346786      | 256891           | 1654928            | ... | 
| Value row 3 | 566         | 1000             | 95                 | ... |
`
But my table layout is as follows:
`
root
  |-- row_label
  |-- count
  |-- date
  |-- other_unrelated_fields
`
So, my frustation began to build up when I could not do that in Tableau. And I needed to vent.
<!--more-->
# Problem Statement and first approach
My first approach was to brute-force the entire thing when creating the Tableau Sheets. And I pretty soon discovered that creating a lagged new calculated value is a dealbreaker. I mean, I got to some resources [such as this one](https://community.tableau.com/thread/242741), or [this blog post](http://onenumber.biz/blog-1/2017/10/9/comparing-year-over-year-in-tableau) or [this website](https://blog.zuar.com/tableau-trick-quarter-to-date-over-prior-quarter-to-date-hierarchy/). **BUT**, all these resources point to table calculations and looking at the data from today. I need to be able to choose any period and get updated values for the month previous to the selection, 3 months back and last year. So these resources, while cool and helpful, were not on point with my needs.

After a while staring at the screen and thinking about my problem, I noticed this looks very similar, in my mind, to an ARIMA calculation. If you're not familiar with the concept, ARIMA stands for AutoRegressive Integrated Moving Average (and you can gloss over the [Wikipedia page on ARIMA](https://en.wikipedia.org/wiki/Autoregressive_integrated_moving_average)) and its an econometric and statistics modelling technique. A generalization of the Autoregressive Moving Average, or ARMA, model, what it essentially does is to autoregress the time series. In plain english, I am joining the current data with the same dataset but lagged a few periods. Which, in turn, looks something like this:

| period | value_n | value_n-1 | ... | value_n-m |
|--------|---------|-----------|-----|-----------|
| n      | 200     | 199       | ... | 45        |
| n-1    | 199     | 180       | ... | NULL      |
| n-2    | 180     | ...       | ... | NULL      |
| ...    | ...     | ...       | ... | NULL      | 
| n-m    | 45      | NULL      | ... | NULL      |

Why NULLs? Because in the current time series those values do not exists. So, at some point in the series, I won't be able to get said values for periods previous to the selected one (`n`).

Once the possible solution was on my mind, I had a new problem at hand. How to implement this over my current environment?

# How to code an answer in a SQL dialect