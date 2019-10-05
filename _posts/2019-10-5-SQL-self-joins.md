---
layout: post
title: SQL self joins, like ARIMA in my mind
categories: [programming]
tags: [sql, self-join]
---

I have a problem, where I was handed an analytical table and I have to display, in a data viz tool, several different periods of the same data in the same table. Something like this:
```
value       | count_now   | count_last_month | count_last_quarter | ...
------------+-------------+------------------+--------------------+-----
Value row 1 | 2576821     | 2576521          | 33348930           | ...
Value row 2 | 346786      | 256891           | 1654928            | ...
Value row 3 | 566         | 1000             | 95                 | ...
```
But my table layout is as follows:
```
root
  |-- value
  |-- count
  |-- date
  |-- other_unrelated_fields
```
So, my frustation began to build up when I could not do that in Tableau.