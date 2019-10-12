---
layout: post
title:  Self joins en SQL, o cómo hacer ingeniería de datos ingeniosa
categories: ["data engineering"]
tags: [sql, self-join, mysql]
published: true
language: es
comments: true
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
Definamos primero el requerimiento. Debo crear una visualización (una tabla pivote, en este caso) que muestre un valor, un conteo, una frecuencia absoluta, para varios periodos de tiempo en diferentes columnas. El periodo base debe ser un parámetro; es decir, no es necesariamente el mes en curso (como obtenerlo de la fecha del sistema). La tabla final esperada se debiera ver así:
{% highlight shell %}
+----------------+---------------------+-------------------+-----------------+---------------+-------------------+
| etiqueta       | Periodo base        | Mes anterior      | 3 meses atrás   | 1 año atrás   | Otros períodos    |
|----------------+---------------------+-------------------+-----------------+---------------+-------------------|
| Etiqueta 1     | 111222333           | 111222111         | 111222000       | 111222999     | ...               |
| Etiqueta 2     | 333444555           | 333222111         | 333222222       | 333444222     | ...               |
| Etiqueta 3     | 11223344            | 11223322          | 11223333        | 11223355      | ...               |
| ...            | ...                 | ...               | ...             | ...           | ...               |
| Etiqueta n     | 1234567             | 123321            | 123212          | 987654        | ...               |
+-------------+---------------------+-------------------+-----------------+---------------+-------------------+
{% endhighlight %}

La estructura de datos entregada es la siguiente:
{% highlight shell %}
root
  |-- etiqueta_de_fila
  |-- conteo
  |-- fecha
  |-- otros_campos_irrelevantes
{% endhighlight %}

# Configurando el entorno de desarrollo
Como entiendo que este blog no sólo es para contar cómo hice algo, sino también tiene que ser de ayuda para ustedes, voy a entregar *snippets* de código que permitan reproducir el problema y la data, además de poder implementar las soluciones en una instalación de MySQL sobre WSL.

## Instalando WSL en Windows 10
Tengo en mi máquina Windows 10 con el **Subsistema de Linux para Windows (WSL)** activado. Así, puedo usar Bash dentro de windows. Qué tan cool es eso?

Por supuesto, si eres un desarrollador, esto no es ninguna noticia ni novedad. Pero, si por alguna razón vives bajo una roca en Fondo de Bikini, y no sabes que se puede tener una distribución casi completa de Linux en Windows 10, puedes mirar [este artículo](https://cepa.io/2018/02/10/linuxizing-your-windows-pc-part1/) para comenzar con WSL, mientras que [este artículo te mostrará cómo instalar MySQL y otras herramientas en WSL](https://medium.com/@fiqriismail/how-to-setup-apache-mysql-and-php-in-linux-subsystem-for-windows-10-e03e67afe6ee). Para fines de la solución vista en este post, asumo que ya sabes instalar WSL, que lo activaste, y que lo estás usando, así que todos los ejemplos serán ejecutados en dicho ambiente.

## Ambientes de Anaconda
También instalé y estoy usando Anaconda en WSL. La guía de instalación la tomé de [este sitio](https://gist.github.com/kauffmanes/5e74916617f9993bc3479f401dfec7da), incluyendo la sección sobre los symlinks, y la guía de uso (que muestra cómo crear ambientes nuevos y aislar el desarrollo en términos de paquetes y dependencias) la tomé directo de [la documentación de `conda`](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html). Finalmente, la creación de un ambiente de Conda con R viene [de esta página de `conda`](https://docs.anaconda.com/anaconda/user-guide/tasks/using-r-language/).

## Creando archivos de datos
Para poder testear las soluciones aquí propuestas, necesitamos tener archivos con datos que cumplan las especificaciones del problema. El siguiente script de R generará un archivo CSV con datos de prueba:
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

## Importando los archivos a tablas MySQL
Este recién creado archivo CSV se guarda, en mi caso, dentro de WSL:
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

(Usé `quote = FALSE` en la llamada a `write.csv` de R porque no me gusta que hayan comillas en los archivos CSV, y es más fácil importar data así.)

Ahora, ingreso a MySQL y:
* Crearé una base de datos para el proyecto, llamada `ssibd`
* Crearé una tabla nueva, `self_join_20191005`
* Cargaré la data en CSV en dicha tabla
* Revisaré los contenidos de la tabla
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

# Primeros pasos hacia la resolución del problema
Mi primer intento de solución fue fuerza bruta pura: intentar crear la tabla pivote requerida en una hoja de Tableau, sin estructurar previamente los datos. Muy luego descubrí que crear un campo calculado con lag en tiempo era imposible en esa herramienta. O sea, encontré recursos muy buenos, [como éste](https://community.tableau.com/thread/242741), o [este artículo](http://onenumber.biz/blog-1/2017/10/9/comparing-year-over-year-in-tableau) o [este sitio](https://blog.zuar.com/tableau-trick-quarter-to-date-over-prior-quarter-to-date-hierarchy/). **PERO**, estos tres sitios de ayuda apuntan a cálculos de tabla, y a mirar la data desde la fecha de sistema. Yo necesito poder elegir cualquier periodo de datos, y tener valores actualizados para los meses previos a la selección (1 mes, 3 meses y 12 meses/un año). Entonces, lo que encontré navegando por internet, aún cuando es cool y permite mucha ayuda, no resolvió mis necesidades.

Pasé mucho rato mirando la pantalla, al infinito, y a ambas juntas, antes de notar que este problema es sumamente similar (o al menos así lo veo) a la aplicación de un modelo econométrico llamado ARIMA. Si no te suena el concepto, ARIMA es el acrónimo para un modelo AutoRegresivo Integrado y de Medias Móviles (en inglés, por supuesto; puedes encontrar más información en la [página de Wikipedia de ARIMA](https://es.wikipedia.org/wiki/Modelo_autorregresivo_integrado_de_media_m%C3%B3vil), aunque creo que es mejor fuente [Wikipedia en inglés](https://en.wikipedia.org/wiki/Autoregressive_integrated_moving_average)). ARIMA es una generalización del modelo ARMA (autoregresivo no integrado de medias móviles), pero lo importante de esto no es el modelo exacto sino el mecanismo de la autoregresión: voy a hacer, en esencia, un JOIN de una tabla consigo misma, sólo que en periodos anteriores. Finalmente, la tabla debiera verse más menos así:
{% highlight shell %}
+---------+---------+-----------+-----+-----------+
| periodo | valor_n | valor_n-1 | ... | valor_n-m |
+---------+---------+-----------+-----+-----------+
| n       | 200     | 199       | ... | 45        |
| n-1     | 199     | 180       | ... | NULL      |
| n-2     | 180     | ...       | ... | NULL      |
| ...     | ...     | ...       | ... | NULL      | 
| n-m     | 45      | NULL      | ... | NULL      |
+---------+---------+-----------+-----+-----------+
{% endhighlight %}

"¿Por qué nulos?" veo que preguntan, y la explicación para esos valores es que no existen en la data original. En algún punto de la serie autoregresiva no voy a poder obtener los valores previos al mes seleccionado (`n`). Y claro, antes del primer mes calculado no existen valores previos. Ergo, `NULL`.

En toda honestidad, mi proceso de solución es sólo la autoregresión; la parte de las medias móviles integradas no es necesaria aquí. Todo lo que necesitamos es el `SELF JOIN` con un retraso en los periodos de la serie de tiempo.

Una vez que la solución posible estaba en mi cabeza, me enfrenté a una nueva complicación: *¿cómo implementar esto en el ambiente actual en el que me encuentro?*

# Cómo escribir una respuesta en MySQL
Primero tuve que poner la data en un formato apropiado. Actualmente, esto es lo que tengo:
{% highlight sql %}
SELECT
  row_label,
  count,
  date
FROM self_join_20191005
{% endhighlight %}

Creo una nueva definición de mis datos originales:
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

Fíjense que creé el campo `date_join_df`, dado que en mis datos originales trabajo con fechas diferentes, hacia fin de mes. Entonces, necesito controlar la posibilidad de que algunas fechas sean diferentes al agregar o quitar meses. Esto lo hago creando una variable de texto (con la función `CONCAT`) que junte el año (usando `YEAR`) y el mes (usando `MONTH`) de la fecha base, y terminando con un valor literal '1'. Esto entrega un valor de YYYY-MM-1, que, convertido a fecha con `STR_TO_DATE(fecha, formato)`, entregará siempre el primer día del mes.

Por otro lado, las definiciones de tabla, a partir de este punto, serán con nombres de columna en inglés, para permitir la reutilización de los scripts.

La nueva tabla se ve así:
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

Y esto es de mucha ayuda, y prueba nuestro concepto, porque vamos a usar una columna del tipo `date_join_dt` para todas las tablas. Esto garantiza que no vamos a encontrarnos con periodos que no podamos unir.

# Resolviendo el problema con SQL
Ahora que la tabla puede ser creada con el diseño adecuado, podemos hacer un `JOIN` de la tabla consigo misma, y obtener el resultado esperado. En SQL puro (dialecto de MySQL), esto se hace así:
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

El resultado de esta query se ve a continuación. Ordené por `date` y `row_label` para ver mejor los resultados. Además, sólo 15 filas se muestran, para no extender tanto el post:
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

La query anterior es una forma efectiva y no muy elegante de resolver este problema. Probamos, con éxito, que la data se puede visualizar de la manera que requerimos.

El siguiente quebradero es que implementar esta solución directamente es complejo, dado que la sintaxis de los SQL ad-hoc de Tableau son más extraños de lo normal, y no quiero empezar a hacer debug de una implementación atípica. Así las cosas, podemos diseñar una solución con una tabla física y 3 lógicas, y hacer `LEFT JOIN` hasta la muerte. Es la idea, al menos.

# Solución con múltiples tablas
Para implementar nuestra solución a Tableau (o PySpark, o alguna otra plataforma en verdad), tendremos cuatro tablas diferentes:
* Tabla de periodo base: tabla original (llamada `base_table` en este ejemplo) {`row_label`, `date`, `count`, `date_join_1m`, `date_join_1q`, `date_join_1y`}
* Tabla del mes anterior (`prevm_table`): la tabla base, con menos campos, que representa al mes anterior {`row_label`, `date`, `count`, `date_join`}
* Tabla del trimestre anterior (`prevq_table` por 'quarter'): {`row_label`, `date`, `count`, `date_join`}
* Tabla del año anterior (`prevy_table`): {`row_label`, `date`, `count`, `date_join`}

Estoy dejando la fecha original en las tablas para validar luego los resultados. Con esta estructura, luego se puede validar que los JOINs hayan sido exitosos.

Las definiciones de tabla son:
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

Las 3 tablas lógicas adicionales son iguales. Tiene sentido, dado que el `SELF JOIN` requiere que la tabla se una con ella misma. Por lo mismo, la tabla fuente siempre será `ssibd`.`self_join_20191005`. Esto ayudará a la implementación de la solución en Tableau.

Y ahora, para la parte divertida, haremos el JOIN final. Como mencioné, dejaremos todas las fechas en el resultado, para validar los resultados luego:
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

La tabla final debiera tener la siguiente estructura:
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

## Implementando la solución y probando los resultados
Ya tenemos todos los datos requeridos para implementar nuestros scrips recién creados, y ver si obtenemos los resultados esperados.

Creé un nuevo script SQL, `create_tables.sql`, en el que se encuentran todas las definiciones de tablas anteriores. Corremos dicho script con el siguiente comando en bash:
{% highlight shell %}
mysql --user="<user>" -password="<pass>" --database="ssibd" < /path/to/file/filename.sql
{% endhighlight %}

> El uso del parámetro `--password` con tu contraseña escrita en el comando es permisible **sólo porque estamos en un ambiente de desarrollo propio y controlado**. Nunca debieras hacer algo parecido en un ambiente de producción.

Revisemos los resultados finales:
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

Esto es **exactamente** lo que buscamos. Felicidades a nosotros por obtener los datos de la manera correcta!

# Conclusión
Este problema, engañoso en su simpleza, significó un desafío en su resolución. La implementación de la solución en una plataforma de visualización de datos, como Tableau o Spotfire, vendrá con sus propias complicaciones. Ellas serán discutidas en un nuevo artículo.

Quería mencionar que esperaba, al iniciar este desarrollo, que Tableau tuviera funciones de lag **basadas en la estructura de la data y no de la tabla usada**, y fue una decepción descubrir que me equivqué. Si conoces de una forma de producir el resultado esperado sólo con Tableau, puedes [escribirme](mailto:javier@ctjconsult.com) y contarme cómo lo resolviste.

Finalmente, todo el código está disponible en [mi perfil de GitHub](https://github.com/jtapiath-cl/ssibd), si prefieres clonar el repo y probar esta solución.