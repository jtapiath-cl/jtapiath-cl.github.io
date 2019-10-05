---
layout: post
title: SQL self joins, like ARIMA in my mind
categories: [programming]
tags: [sql, self-join]
published: true
---

I have a problem, where I was handed an analytical table and I have to display, in a data viz tool, several different periods of the same data in the same table. Something like this:
{% highlight shell %}
| row_label   | count_now   | count_last_month | count_last_quarter | ... |
|-------------+-------------+------------------+--------------------+-----|
| Value row 1 | 2576821     | 2576521          | 33348930           | ... | 
| Value row 2 | 346786      | 256891           | 1654928            | ... | 
| Value row 3 | 566         | 1000             | 95                 | ... |
{% endhighlight %}

But my table layout is as follows:
{% highlight shell %}
root
  |-- row_label
  |-- count
  |-- date
  |-- other_unrelated_fields
{% endhighlight %}

So, my frustration began to build up when I could not do that in Tableau. And I needed to vent.
<!--more-->
# Problem statement
The requirement at hand is to create a visualization (in this case, a cross table) that displays a value (in this case, a count such as an absolute frequency) over several periods in different columns. The base period must be a parameter; i.e., is not necessarily the current month but something for the user to select. The output table is expected to look like this:
{% highlight shell %}
| label       | Selected period     | Previous month    | 3 months back   | 1 year back   | Any other periods |
|-------------+---------------------+-------------------+-----------------+---------------+-------------------|
| Label 1     | 111222333           | 111222111         | 111222000       | 111222999     | ...               |
| Label 2     | 333444555           | 333222111         | 333222222       | 333444222     | ...               |
| Label 3     | 11223344            | 11223322          | 11223333        | 11223355      | ...               |
| ...         | ...                 | ...               | ...             | ...           | ...               |
| Label n     | 1234567             | 123321            | 123212          | 987654        | ...               |
{% endhighlight %}

The provided data structure is as follows:
{% highlight shell %}
root
  |-- row_label
  |-- count
  |-- date
  |-- other_unrelated_fields
{% endhighlight %}

# First approach
My first approach was to brute-force the entire thing when creating the Tableau sheet. And I pretty soon discovered that creating a lagged new calculated value is a deal breaker. I mean, I got to some resources [such as this one](https://community.tableau.com/thread/242741), or [this blog post](http://onenumber.biz/blog-1/2017/10/9/comparing-year-over-year-in-tableau) or [this website](https://blog.zuar.com/tableau-trick-quarter-to-date-over-prior-quarter-to-date-hierarchy/). **BUT**, all these resources point to table calculations and looking at the data from today. I need to be able to choose any period and get updated values for the month previous to the selection, 3 months back and last year. So these resources, while cool and helpful, were not on point with my needs.

After a while staring at the screen and thinking about my problem, I noticed this looks very similar, in my mind, to an ARIMA calculation. If you're not familiar with the concept, ARIMA stands for AutoRegressive Integrated Moving Average (and you can gloss over the [Wikipedia page on ARIMA](https://en.wikipedia.org/wiki/Autoregressive_integrated_moving_average)) and its an econometric and statistics modelling technique. A generalization of the Autoregressive Moving Average, or ARMA, model, what it essentially does is to auto regress the time series. In plain english, I am joining the current data with the same dataset but lagged a few periods. Which, in turn, looks something like this:
{% highlight shell %}
| period | value_n | value_n-1 | ... | value_n-m |
|--------+---------+-----------+-----+-----------|
| n      | 200     | 199       | ... | 45        |
| n-1    | 199     | 180       | ... | NULL      |
| n-2    | 180     | ...       | ... | NULL      |
| ...    | ...     | ...       | ... | NULL      | 
| n-m    | 45      | NULL      | ... | NULL      |
{% endhighlight %}

Why NULLs? Because in the current time series those values do not exists. So, at some point in the series, I won't be able to get said values for periods previous to the selected one (`n`).

Once the possible solution was on my mind, I had a new problem at hand. How to implement this over my current environment?

# How to code an answer in a SQL dialect
I had to take my data to an appropriate format. And, currently, this is what I had:
{% highlight sql %}
SELECT
  row_label,
  count,
  date
FROM source_table
{% endhighlight %}

I first create a new table definition, with a few extra fields:
{% highlight sql %}
SELECT
  row_label,
  count AS count_curr,
  CAST(date AS DATE) AS date_dt,
  ADD_MONTHS(CAST(date AS DATE), -1) AS date_prev_dt,
  TO_DATE(CONCAT(YEAR(ADD_MONTHS(CAST(date AS DATE), -1)), MONTH(ADD_MONTHS(CAST(date AS DATE), -1)), 1)) AS date_join_dt
  CASE row_label
    WHEN 1 THEN format_1
    WHEN 2 THEN format_2
    ELSE format_otherwise
  END AS row_label_fmt
FROM source_table
{% endhighlight %}

The extra fields are:
* `date_prev_dt`: I'm testing the `ADD_MONTHS` Oracle SQL function, so I created this field to check if I can subtract a full month from a date and check the validity of the result.
* `date_join_dt`:since I'm working with end of month dates, I need to control for the possibility of some dates being different than the target dates when subtracting months. This is done creating a string variable with the `CONCAT` function, fetching the `YEAR` and `MONTH` of the base date, and finishing the date string with the integer 1. This yields a string in the form of YYYY-MM-1, which, converted to date, will deliver the first day of the month always.
* `row_label_fmt`: this is a required manipulation, to deliver the format the client expected.

This new table looks like this:
{% highlight shell %}
| row_label   | count_curr  | date_dt    | date_prev_dt | date_join_dt | row_label_fmt      |
|-------------+-------------+------------+--------------+--------------+--------------------|
| Value row 1 | 2576821     | 2017/01/29 | 2016/12/29   | 2016/12/01   | format_1           |
| Value row 2 | 346786      | 2017/01/29 | 2016/12/29   | 2016/12/01   | format_2           |
| Value row 3 | 566         | 2017/01/29 | 2016/12/29   | 2016/12/01   | format_otherwise   |
| ...         | ...         | ...        | ...          | ...          | ...                | 
{% endhighlight %}

And this is helpful because the joining columns will be `date_join_dt` for all tables, since this guarantees we won't run into any issue with month subtractions and controlling days.

Since the test is successful, we will have 4 different tables:
* Current period table: the base table (dubbed `base_table` for this example) {`row_label`, `date`, `count_curr`, `date_join_1m`, `date_join_1q`, `date_join_1y`}
* Last month table: the base table, but only with the modified date {`row_label`, `date`, `count_1m`, `date_join_1m`}
* Last quarter table: {`row_label`, `date`, `count_1q`, `date_join_1q`}
* Last year table: {`row_label`, `date`, `count_1y`, `date_join_1t`}

I'm keeping the base date on each table for testing purposes. This way, later down the road, I can test if all the JOINs were successful.

The table definitions are as follows:
{% highlight sql %}
CREATE TABLE prev_month_table AS
SELECT
  row_label,
  date,
  count_curr AS count_1m,
  TO_DATE(CONCAT(YEAR(ADD_MONTHS(CAST(date AS DATE), -1)), MONTH(ADD_MONTHS(CAST(date AS DATE), -1)), 1)) AS date_join_1m
FROM base_table
{% endhighlight %}

{% highlight sql %}
CREATE TABLE prev_quarter_table AS
SELECT
  row_label,
  date,
  count_curr AS count_1q,
  TO_DATE(CONCAT(YEAR(ADD_MONTHS(CAST(date AS DATE), -3)), MONTH(ADD_MONTHS(CAST(date AS DATE), -3)), 1)) AS date_join_1q
FROM base_table
{% endhighlight %}

{% highlight sql %}
CREATE TABLE prev_year_table AS
SELECT
  row_label,
  date,
  count_curr AS count_1q,
  TO_DATE(CONCAT(YEAR(ADD_MONTHS(CAST(date AS DATE), -12)), MONTH(ADD_MONTHS(CAST(date AS DATE), -12)), 1)) AS date_join_1q
FROM base_table
{% endhighlight %}

And now, for the fun part, we have to JOIN these tables together. As I mentioned, we'll keep all the dates in place, to check the validity of our query:
{% highlight sql %}
SELECT
  A.row_label,
  A.row_label_fmt,
  A.date AS selected_dt,
  A.count_curr AS count_seldt,
  B.date AS date_1m,
  B.count_1m AS count_prevm,
  C.date AS date_1q,
  C.count_1q AS count_prevq,
  D.date AS date_1y,
  D.count_1y AS count_prevy
FROM base_table A
LEFT JOIN prev_month_table B
  ON A.row_label = B.row_label AND A.date_join_1m = B.date_join_1m
LEFT JOIN prev_quarter_table C
  ON A.row_label = C.row_label AND A.date_join_1q = C.date_join_1q
LEFT JOIN prev_year_table D
  ON A.row_label = D.row_label AND A.date_join_1y = D.date_join_1y
{% endhighlight %}

The final table structure should be as follows:
{% highlight shell %}
root
  |-- row_label
  |-- row_label_fmt
  |-- selected_dt
  |-- count_seldt
  |-- date_1m
  |-- count_prevm
  |-- date_1q
  |-- count_prevq
  |-- date_1y
  |-- count_prevy
{% endhighlight %}

# Implementing the solution in a reproducible way
I understand this is also a blog that's supposed to be helpful, so I will provide now some code snippets that create reproducible data, as well as implementing this solution over a MySQL installation on WSL.