---
layout: post
title: Self joins en SQL, o cómo hacer ingeniería de datos ingeniosa
categories: ["data engineering"]
tags: [sql, self-join, mysql]
published: true
language: es
---

Hace unos días, se me presentó un problema analítico, donde tenía que mostrar, en una herramienta de visualización de datos (digamos Tableau o Qlikview), varios periodos distintos de la misma data, en la misma tabla. Algo parecido a esto:
{% highlight shell %}
+--------------+-------------+------------------+----------------------+-----+
| encabezado   | cnt_actual  | cnt_mes_pasado   | cnt_trimestre_pasado | ... |
|--------------+-------------+------------------+----------------------+-----|
| Valor fila 1 | 2576821     | 2576521          | 33348930             | ... | 
| Valor fila 2 | 346786      | 256891           | 1654928              | ... | 
| Valor fila 3 | 566         | 1000             | 95                   | ... |
+-------------+-------------+------------------+-----------------------+-----+
{% endhighlight %}

La estructura de la tabla original es esta:
{% highlight shell %}
root
  |-- encabezado
  |-- conteo
  |-- fecha
  |-- otros_campos_no_relevantes
{% endhighlight %}

Mi frustración comenzó a crecer cuando la situación se volvió completa, y necesitaba hacer algo al respecto.
<!--more-->
# El problema
Definamos primero el requerimiento. Debo crear una visualización (una tabla pivote, en este caso) que muestre un valor, un conteo, una frecuencia absoluta, a través de varios periodos de tiempo en diferentes columnas. El periodo base debe ser un parámetro; es decir, no es necesariamente el mes en curso (como obtenerlo de )
The requirement at hand is to create a visualization (in this case, a cross table) that displays a value (in this case, a count such as an absolute frequency) over several periods in different columns. The base period must be a parameter; i.e., is not necessarily the current month but something for the user to select. The output table is expected to look like this:
{% highlight shell %}
+-------------+---------------------+-------------------+-----------------+---------------+-------------------+
| label       | Selected period     | Previous month    | 3 months back   | 1 year back   | Any other periods |
|-------------+---------------------+-------------------+-----------------+---------------+-------------------|
| Label 1     | 111222333           | 111222111         | 111222000       | 111222999     | ...               |
| Label 2     | 333444555           | 333222111         | 333222222       | 333444222     | ...               |
| Label 3     | 11223344            | 11223322          | 11223333        | 11223355      | ...               |
| ...         | ...                 | ...               | ...             | ...           | ...               |
| Label n     | 1234567             | 123321            | 123212          | 987654        | ...               |
+-------------+---------------------+-------------------+-----------------+---------------+-------------------+
{% endhighlight %}

The provided data structure is as follows:
{% highlight shell %}
root
  |-- row_label
  |-- count
  |-- date
  |-- other_unrelated_fields
{% endhighlight %}

# Setting up the working environment
I understand this is also a blog that's supposed to be helpful, so I will provide now some code snippets that create reproducible data, as well as implementing said data to a MySQL installation on WSL.

## Installing WSL on Windows 10
I have Windows 10 with **Windows Subsystem for Linux** activated. This means I can use Bash inside Windows. How cool is that?

Of course, if you're a dev, this is no news for you. But if, somehow, you live underneath a rock at Bikini's Bottom and you don't know you can run an almost-full Linux distro on Windows 10, please take a look at [this article](https://cepa.io/2018/02/10/linuxizing-your-windows-pc-part1/), and [this one for installing MySQL and others in WSL](https://medium.com/@fiqriismail/how-to-setup-apache-mysql-and-php-in-linux-subsystem-for-windows-10-e03e67afe6ee). I am assuming you already know you can install Linux on Windows, so all examples will be run against said environment.

## Anaconda environments
I also am running Anaconda on the WSL. The installation guide was taken from [this website](https://gist.github.com/kauffmanes/5e74916617f9993bc3479f401dfec7da), including the symlink part, and the usage guide (and how to create new environments to isolate the packages and development requirements) was taken straight from [the `conda` documentation](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html). How to create a Conda R environment comes [from this `conda` page](https://docs.anaconda.com/anaconda/user-guide/tasks/using-r-language/).

## Creating data files
To be able to test the multiple solutions at hand here, we need to have some data files. The following R script will generate a single CSV file with some test data:

{% highlight r %}
dates <- seq(as.Date("2016/01/01"),
             by = "month",
             length.out = 40)
row_label <- c("A: format 1",
               "B: format 2",
               "C: format 3",
               "D: format 4")
library(data.table)
init_table <- CJ(dates, row_label)
count_curr <- sample(33000:55000,
                     size = nrow(init_table),
                     replace = TRUE)
sample_data <- data.frame(date = init_table$dates,
                          row_label = init_table$row_label,
                          count = count_curr)
str(sample_data)
head(sample_data)
write.csv(x = sample_data,
          file = "sample_data.csv",
          row.names = FALSE,
          col.names = TRUE,
          quote = FALSE)
{% endhighlight %}

## Importing files to MySQL tables
The newly created CSV file is stored, in my case, in a local directory in my WSL:
{% highlight shell %}
jtapia@DESKTOP-2019LPH:~$ head /path/to/file/sample_data.csv
date,row_label,count
2016-01-01,A: format 1,36123
2016-01-01,B: format 2,37129
2016-01-01,C: format 3,40047
2016-01-01,D: format 4,51673
2016-02-01,A: format 1,42566
2016-02-01,B: format 2,37832
2016-02-01,C: format 3,54721
2016-02-01,D: format 4,50022
2016-03-01,A: format 1,43116
{% endhighlight %}

(Notice I used `quote = FALSE` on the R `write.csv` call, because I don't like the quoted files when loading data into a table.)

Now I log in to MySQL and:
* Create a project database, `ssibd`
* Create a new table, `self_join_20191005`
* Load the CSV data into said table
* Check the contents of the data

{% highlight shell %}
mysql> CREATE DATABASE ssibd;
Query OK, 1 row affected (0.00 sec)

mysql> USE ssibd;
Database changed

mysql> CREATE TABLE self_join_20191005 (
  date DATE NOT NULL,
  row_label VARCHAR(12) NOT NULL,
  count INT NOT NULL
  );
Query OK, 0 rows affected (0.37 sec)

mysql> LOAD DATA 
        LOCAL 
        INFILE '/path/to/file/sample_data.csv' 
        INTO TABLE self_join_20191005 
        FIELDS TERMINATED BY ',' 
        LINES TERMINATED BY '\n' 
        IGNORE 1 ROWS 
        (date, row_label, count);
Query OK, 160 rows affected (0.09 sec)
Records: 160  Deleted: 0  Skipped: 0  Warnings: 0

mysql> SELECT * FROM self_join_20191005 LIMIT 10;
+------------+-------------+-------+
| date       | row_label   | count |
+------------+-------------+-------+
| 2016-01-01 | A: format 1 | 36123 |
| 2016-01-01 | B: format 2 | 37129 |
| 2016-01-01 | C: format 3 | 40047 |
| 2016-01-01 | D: format 4 | 51673 |
| 2016-02-01 | A: format 1 | 42566 |
| 2016-02-01 | B: format 2 | 37832 |
| 2016-02-01 | C: format 3 | 54721 |
| 2016-02-01 | D: format 4 | 50022 |
| 2016-03-01 | A: format 1 | 43116 |
| 2016-03-01 | B: format 2 | 46211 |
+------------+-------------+-------+
10 rows in set (0.00 sec)

mysql> SELECT COUNT(*) FROM self_join_20191005;
+----------+
| COUNT(*) |
+----------+
|      160 |
+----------+
1 row in set (0.13 sec)
{% endhighlight %}

# First approach to solving the problem
My first approach was to brute-force the entire thing when creating the Tableau sheet. And I pretty soon discovered that creating a lagged new calculated value is a deal breaker. I mean, I got to some resources [such as this one](https://community.tableau.com/thread/242741), or [this blog post](http://onenumber.biz/blog-1/2017/10/9/comparing-year-over-year-in-tableau) or [this website](https://blog.zuar.com/tableau-trick-quarter-to-date-over-prior-quarter-to-date-hierarchy/). **BUT**, all these resources point to table calculations and looking at the data from today. I need to be able to choose any period and get updated values for the month previous to the selection, 3 months back and last year. So these resources, while cool and helpful, were not on point with my needs.

After a while staring at the screen and thinking about my problem, I noticed this looks very similar, in my mind, to an ARIMA calculation. If you're not familiar with the concept, ARIMA stands for AutoRegressive Integrated Moving Average (and you can gloss over the [Wikipedia page on ARIMA](https://en.wikipedia.org/wiki/Autoregressive_integrated_moving_average)) and its an econometric and statistics modelling technique. A generalization of the Autoregressive Moving Average, or ARMA, model, what it essentially does is to auto regress the time series. In plain english, I am joining the current data with the same dataset but lagged a few periods. Which, in turn, looks something like this:
{% highlight shell %}
+--------+---------+-----------+-----+-----------+
| period | value_n | value_n-1 | ... | value_n-m |
+--------+---------+-----------+-----+-----------+
| n      | 200     | 199       | ... | 45        |
| n-1    | 199     | 180       | ... | NULL      |
| n-2    | 180     | ...       | ... | NULL      |
| ...    | ...     | ...       | ... | NULL      | 
| n-m    | 45      | NULL      | ... | NULL      |
+--------+---------+-----------+-----+-----------+
{% endhighlight %}

Why NULLs? Because in the current time series those values do not exists. So, at some point in the series, I won't be able to get said values for periods previous to the selected one (`n`).

In all honesty, my thought can be translated to an auto regressive model; the integrated moving average part is not necessary here. All we need is to self join the data with itself but lagged a few periods. 

Once the possible solution was on my mind, I had a new problem at hand: *how to implement this over my current environment?*

# How to code an answer in a SQL dialect
I had to take my data to an appropriate format. And, currently, this is what I had:
{% highlight sql %}
SELECT
  row_label,
  count,
  date
FROM self_join_20191005
{% endhighlight %}

I first create a new table definition, with a twist:
{% highlight sql %}
SELECT
  row_label,
  count AS count_freq,
  CAST(date AS DATE) AS date_dt,
  STR_TO_DATE(
    CONCAT(
      YEAR(CAST(date AS DATE)), "-",
      MONTH(CAST(date AS DATE)), "-",
      1
    ),
    "%Y-%m-%d"
  ) AS date_join_dt
FROM self_join_20191005
{% endhighlight %}

Please notice I created the `date_join_dt` field since I'm working with end of month dates. Thus, I need to control for the possibility of some dates being different than the target dates when subtracting months. This is done creating a string variable with the `CONCAT` function, fetching the `YEAR` and `MONTH` of the base date, and finishing the date string with the integer 1. This yields a string in the form of YYYY-MM-1, which, converted to date with `STR_TO_DATE(date, format)`, will deliver the first day of the month always.

This new table looks like this:
{% highlight shell %}
+-------------+------------+------------+--------------+
| row_label   | count_freq | date_dt    | date_join_dt |
+-------------+------------+------------+--------------+
| A: format 1 |      36123 | 2016-01-01 | 2016-01-01   |
| B: format 2 |      37129 | 2016-01-01 | 2016-01-01   |
| C: format 3 |      40047 | 2016-01-01 | 2016-01-01   |
| D: format 4 |      51673 | 2016-01-01 | 2016-01-01   |
| A: format 1 |      42566 | 2016-02-01 | 2016-02-01   |
| B: format 2 |      37832 | 2016-02-01 | 2016-02-01   |
| C: format 3 |      54721 | 2016-02-01 | 2016-02-01   |
| D: format 4 |      50022 | 2016-02-01 | 2016-02-01   |
| A: format 1 |      43116 | 2016-03-01 | 2016-03-01   |
| B: format 2 |      46211 | 2016-03-01 | 2016-03-01   |
+-------------+------------+------------+--------------+
{% endhighlight %}

And this is helpful because the joining columns will be `date_join_dt` for all tables, since this guarantees we won't run into any issue with month subtractions and controlling days.

# Solving the problem with SQL
Now that the table can be created with the desired layout, I can perform a self `JOIN` operation, to deliver the expected result. This can be done in pure SQL as follows:
{% highlight sql %}
SELECT
  A.row_label,
  A.date AS date_c,
  A.count AS cnt,
  B.count AS cnt_1m,
  C.count AS cnt_1q,
  D.count AS cnt_1y
FROM self_join_20191005 A
LEFT JOIN self_join_20191005 B
  ON A.row_label = B.row_label 
  AND STR_TO_DATE(
    CONCAT(
      YEAR(DATE_ADD(CAST(A.date AS DATE), INTERVAL -1 month)), "-",
      MONTH(DATE_ADD(CAST(A.date AS DATE), INTERVAL -1 month)), "-",
      1
    ),
    "%Y-%m-%d"
  ) = STR_TO_DATE(
    CONCAT(
      YEAR(CAST(B.date AS DATE)), "-",
      MONTH(CAST(B.date AS DATE)), "-",
      1
    ),
    "%Y-%m-%d"
  )
LEFT JOIN self_join_20191005 C
  ON A.row_label = C.row_label 
  AND STR_TO_DATE(
    CONCAT(
      YEAR(DATE_ADD(CAST(A.date AS DATE), INTERVAL -3 month)), "-",
      MONTH(DATE_ADD(CAST(A.date AS DATE), INTERVAL -3 month)), "-",
      1
    ),
    "%Y-%m-%d"
  ) = STR_TO_DATE(
    CONCAT(
      YEAR(CAST(C.date AS DATE)), "-",
      MONTH(CAST(C.date AS DATE)), "-",
      1
    ),
    "%Y-%m-%d"
  )
LEFT JOIN self_join_20191005 D
  ON A.row_label = D.row_label 
  AND STR_TO_DATE(
    CONCAT(
      YEAR(DATE_ADD(CAST(A.date AS DATE), INTERVAL -1 year)), "-",
      MONTH(DATE_ADD(CAST(A.date AS DATE), INTERVAL -1 year)), "-",
      1
    ),
    "%Y-%m-%d"
  ) = STR_TO_DATE(
    CONCAT(
      YEAR(CAST(D.date AS DATE)), "-",
      MONTH(CAST(D.date AS DATE)), "-",
      1
    ),
    "%Y-%m-%d"
  )
;
{% endhighlight %}

And the output of this query is as follows (I ordered by `date` and `row_label` for the sake of visualization. I'm also displaying the first 15 records of the table, for brevity's sake):
{% highlight shell %}
+-------------+------------+-------+--------+--------+--------+
| row_label   | date_c     | cnt   | cnt_1m | cnt_1q | cnt_1y |
+-------------+------------+-------+--------+--------+--------+
| A: format 1 | 2016-01-01 | 36123 |   NULL |   NULL |   NULL |
| B: format 2 | 2016-01-01 | 37129 |   NULL |   NULL |   NULL |
| C: format 3 | 2016-01-01 | 40047 |   NULL |   NULL |   NULL |
| D: format 4 | 2016-01-01 | 51673 |   NULL |   NULL |   NULL |
| A: format 1 | 2016-02-01 | 42566 |  36123 |   NULL |   NULL |
| B: format 2 | 2016-02-01 | 37832 |  37129 |   NULL |   NULL |
| C: format 3 | 2016-02-01 | 54721 |  40047 |   NULL |   NULL |
| D: format 4 | 2016-02-01 | 50022 |  51673 |   NULL |   NULL |
| A: format 1 | 2016-03-01 | 43116 |  42566 |   NULL |   NULL |
| B: format 2 | 2016-03-01 | 46211 |  37832 |   NULL |   NULL |
+-------------+------------+-------+--------+--------+--------+
{% endhighlight %}

A fairly inelegant but very effective solution to this issue. Now we have the data as we want it to be!

But there's a catch: we cannot implement something like this in Tableau, since Custom SQL JOIN syntax is strange and I don't want to start to debug a new problem. So, can we design a solution that allows us to use a single table and just `LEFT JOIN` it to death?

# Multiple table approach
To deploy our solution to Tableau (or PySpark for that matter), we will have 4 different tables:
* Current period table: the base table (dubbed `base_table` for this example) {`row_label`, `date`, `count`, `date_join_1m`, `date_join_1q`, `date_join_1y`}
* Last month table (`prevm_table`): the base table, but only with the modified date {`row_label`, `date`, `count`, `date_join`}
* Last quarter table (`prevq_table`): {`row_label`, `date`, `count`, `date_join`}
* Last year table (`prevy_table`): {`row_label`, `date`, `count`, `date_join`}

I'm keeping the base date on each table for testing purposes. This way, later down the road, I can test if all the JOINs were successful.

The table definitions are as follows:
{% highlight sql %}
CREATE TABLE base_table AS
SELECT
  row_label,
  count AS count_freq,
  CAST(date AS DATE) AS date,
  STR_TO_DATE(
    CONCAT(
      YEAR(DATE_ADD(CAST(date AS DATE), INTERVAL -1 month)), "-",
      MONTH(DATE_ADD(CAST(date AS DATE), INTERVAL -1 month)), "-",
      1
    ),
    "%Y-%m-%d"
  ) AS date_join_1m,
  STR_TO_DATE(
    CONCAT(
      YEAR(DATE_ADD(CAST(date AS DATE), INTERVAL -3 month)), "-",
      MONTH(DATE_ADD(CAST(date AS DATE), INTERVAL -3 month)), "-",
      1
    ),
    "%Y-%m-%d"
  ) AS date_join_1q,
  STR_TO_DATE(
    CONCAT(
      YEAR(DATE_ADD(CAST(date AS DATE), INTERVAL -1 year)), "-",
      MONTH(DATE_ADD(CAST(date AS DATE), INTERVAL -1 year)), "-",
      1
    ),
    "%Y-%m-%d"
  ) AS date_join_1y
FROM self_join_20191005
{% endhighlight %}

{% highlight sql %}
CREATE TABLE prevm_table AS
SELECT
  row_label,
  count,
  CAST(date AS DATE) AS date,
  STR_TO_DATE(
    CONCAT(
      YEAR(CAST(date AS DATE)), "-",
      MONTH(CAST(date AS DATE)), "-",
      1
    ),
    "%Y-%m-%d"
  ) AS date_join
FROM self_join_20191005
{% endhighlight %}

{% highlight sql %}
CREATE TABLE prevq_table AS
SELECT
  row_label,
  count,
  CAST(date AS DATE) AS date,
  STR_TO_DATE(
    CONCAT(
      YEAR(CAST(date AS DATE)), "-",
      MONTH(CAST(date AS DATE)), "-",
      1
    ),
    "%Y-%m-%d"
  ) AS date_join
FROM self_join_20191005
{% endhighlight %}

{% highlight sql %}
CREATE TABLE prevy_table AS
SELECT
  row_label,
  count,
  CAST(date AS DATE) AS date,
  STR_TO_DATE(
    CONCAT(
      YEAR(CAST(date AS DATE)), "-",
      MONTH(CAST(date AS DATE)), "-",
      1
    ),
    "%Y-%m-%d"
  ) AS date_join
FROM self_join_20191005
{% endhighlight %}

All the extra tables are the same. Makes sense, since we're doing a `SELF JOIN`, and all the extra structure should be equivalent. Note all source tables are the same `ssibd`.`self_join_20191005` table. This aids later on in the deployment of the solution in a software like Tableau.

And now, for the fun part, we have to JOIN these tables together. As I mentioned, we'll keep all the dates in place, to check the validity of our query:
{% highlight sql %}
CREATE TABLE final_result AS
SELECT
  A.row_label,
  A.date AS date_c,
  A.count AS cnt,
  A.date_join_1m AS date_1m_A,
  B.date_join AS date_1m,
  B.count AS cnt_1m,
  A.date_join_1q AS date_1q_A,
  C.date_join AS date_1q,
  C.count AS cnt_1q,
  A.date_join_1y AS date_1y_A,
  D.date_join AS date_1y,
  D.count AS cnt_1y
FROM base_table A
LEFT JOIN prevm_table B
  ON A.row_label = B.row_label 
  AND A.date_join_1m = B.date_join
LEFT JOIN prevq_table C
  ON A.row_label = C.row_label 
  AND A.date_join_1q = C.date_join
LEFT JOIN prevy_table D
  ON A.row_label = D.row_label 
  AND A.date_join_1y = D.date_join
{% endhighlight %}

The final table structure should be as follows:
{% highlight shell %}
root: final_result
  |-- row_label
  |-- date_c
  |-- cnt
  |-- date_1m_A
  |-- date_1m
  |-- cnt_1m
  |-- date_1q_A
  |-- date_1q
  |-- cnt_1q
  |-- date_1y_A
  |-- date_1y
  |-- cnt_1y
{% endhighlight %}

## Implementing the solution and testing the results
Finally we have all the data in place to deploy the newly created scripts we have and see if we get the proper results.

I created a new SQL script, `create_tables.sql`, which will have all the table definitions we mentioned earlier in this article. We run said script using the following shell command:

{% highlight shell %}
mysql --user="<user>" -password="<pass>" --database="ssibd" < /path/to/file/filename.sql
{% endhighlight %}

> Keep in mind using the `--password` parameter with your password written on the command is being done **only because this is a development environment**. You should never use such practices in a production environment.

Let's check the final output:
{% highlight shell %}
jtapia@DESKTOP-2019LPH:~$ mysql -u USER -p -e "SELECT * FROM ssibd.final_result ORDER BY date_c ASC, row_label LIMIT 20;"
Enter password:
+-------------+------------+-------+------------+------------+--------+------------+------------+--------+------------+---------+--------+
| row_label   | date_c     | cnt   | date_1m_A  | date_1m    | cnt_1m | date_1q_A  | date_1q    | cnt_1q | date_1y_A  | date_1y | cnt_1y |
+-------------+------------+-------+------------+------------+--------+------------+------------+--------+------------+---------+--------+
| A: format 1 | 2016-01-01 | 36123 | 2015-12-01 | NULL       |   NULL | 2015-10-01 | NULL       |   NULL | 2015-01-01 | NULL    |   NULL |
| B: format 2 | 2016-01-01 | 37129 | 2015-12-01 | NULL       |   NULL | 2015-10-01 | NULL       |   NULL | 2015-01-01 | NULL    |   NULL |
| C: format 3 | 2016-01-01 | 40047 | 2015-12-01 | NULL       |   NULL | 2015-10-01 | NULL       |   NULL | 2015-01-01 | NULL    |   NULL |
| D: format 4 | 2016-01-01 | 51673 | 2015-12-01 | NULL       |   NULL | 2015-10-01 | NULL       |   NULL | 2015-01-01 | NULL    |   NULL |
| A: format 1 | 2016-02-01 | 42566 | 2016-01-01 | 2016-01-01 |  36123 | 2015-11-01 | NULL       |   NULL | 2015-02-01 | NULL    |   NULL |
| B: format 2 | 2016-02-01 | 37832 | 2016-01-01 | 2016-01-01 |  37129 | 2015-11-01 | NULL       |   NULL | 2015-02-01 | NULL    |   NULL |
| C: format 3 | 2016-02-01 | 54721 | 2016-01-01 | 2016-01-01 |  40047 | 2015-11-01 | NULL       |   NULL | 2015-02-01 | NULL    |   NULL |
| D: format 4 | 2016-02-01 | 50022 | 2016-01-01 | 2016-01-01 |  51673 | 2015-11-01 | NULL       |   NULL | 2015-02-01 | NULL    |   NULL |
| A: format 1 | 2016-03-01 | 43116 | 2016-02-01 | 2016-02-01 |  42566 | 2015-12-01 | NULL       |   NULL | 2015-03-01 | NULL    |   NULL |
| B: format 2 | 2016-03-01 | 46211 | 2016-02-01 | 2016-02-01 |  37832 | 2015-12-01 | NULL       |   NULL | 2015-03-01 | NULL    |   NULL |
| C: format 3 | 2016-03-01 | 45483 | 2016-02-01 | 2016-02-01 |  54721 | 2015-12-01 | NULL       |   NULL | 2015-03-01 | NULL    |   NULL |
| D: format 4 | 2016-03-01 | 35423 | 2016-02-01 | 2016-02-01 |  50022 | 2015-12-01 | NULL       |   NULL | 2015-03-01 | NULL    |   NULL |
| A: format 1 | 2016-04-01 | 50027 | 2016-03-01 | 2016-03-01 |  43116 | 2016-01-01 | 2016-01-01 |  36123 | 2015-04-01 | NULL    |   NULL |
| B: format 2 | 2016-04-01 | 46290 | 2016-03-01 | 2016-03-01 |  46211 | 2016-01-01 | 2016-01-01 |  37129 | 2015-04-01 | NULL    |   NULL |
| C: format 3 | 2016-04-01 | 41263 | 2016-03-01 | 2016-03-01 |  45483 | 2016-01-01 | 2016-01-01 |  40047 | 2015-04-01 | NULL    |   NULL |
| D: format 4 | 2016-04-01 | 39287 | 2016-03-01 | 2016-03-01 |  35423 | 2016-01-01 | 2016-01-01 |  51673 | 2015-04-01 | NULL    |   NULL |
| A: format 1 | 2016-05-01 | 43899 | 2016-04-01 | 2016-04-01 |  50027 | 2016-02-01 | 2016-02-01 |  42566 | 2015-05-01 | NULL    |   NULL |
| B: format 2 | 2016-05-01 | 39966 | 2016-04-01 | 2016-04-01 |  46290 | 2016-02-01 | 2016-02-01 |  37832 | 2015-05-01 | NULL    |   NULL |
| C: format 3 | 2016-05-01 | 35504 | 2016-04-01 | 2016-04-01 |  41263 | 2016-02-01 | 2016-02-01 |  54721 | 2015-05-01 | NULL    |   NULL |
| D: format 4 | 2016-05-01 | 38437 | 2016-04-01 | 2016-04-01 |  39287 | 2016-02-01 | 2016-02-01 |  50022 | 2015-05-01 | NULL    |   NULL |
+-------------+------------+-------+------------+------------+--------+------------+------------+--------+------------+---------+--------+
{% endhighlight %}

Which is **exactly** what we were after. Congratulations to ourselves on getting the proper data!

# Conclusion
This problem, deceivingly simple, proved some data engineering challenge. The deployment of the solution in a data visualization platform, such as Tableau or Spotfire, will also come with its challenges. Those are to be discussed in a different post.

Also, I'd like to mention that I fully expected Tableau to have lag functions **not based on the table layout but the data layout**, and it was quite a disappointment to be proven wrong. If you happen to know how to produce such a result, you can [email me](mailto:javier@ctjconsult.com) and let me know how it works for you.

You can find the source files in [my GitHub page](https://github.com/jtapiath-cl/ssibd), if you want to give this a spin by yourself.