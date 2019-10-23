---
layout: post
title:  Análisis de texto de los discursos del presidente Piñera en relación al estado de Emergencia en Chile, octubre de 2019
categories: ["data science"]
tags: [text-mining, tm]
published: true
language: es
comments: true
---
Hace casi una semana, el presidente de Chile, Sebastián Piñera, decretó estado de emergencia (un estado de excepción, es decir, el presidente declara que Chile se encuentra bajo condiciones extremas; puedes ver más en [Wikipedia en español](https://es.wikipedia.org/wiki/Estado_de_excepci%C3%B3n) o en [Wikipedia en inglés](https://en.wikipedia.org/wiki/State_of_emergency)). Ayer en la noche, 22 de octubre de 2019, tras tres días de toque de queda, presentó en su discurso el primer paquete de medidas sociales del gobierno. Este discurso ha estado circulando por redes, en formato Word. Y para tener una mejor digestión de lo que se dijo, les presento en este artículo un análisis de texto de dicho discurso, para tener más claro lo que está proponiendo.

El análisis lo realizaré con R, usando el paquete `tm`, lo tradicional para la ejecución de minería de texto. Todo, como en el artículo anterior, está basado en ambientes `conda`. El documento Word que está circulando lo pueden ver [aquí, en GitHub](https://github.com/jtapiath-cl/ssibd/blob/master/tm_speech_20191023/Mensaje%20Pdte%20-%2022%20de%20octubre%20de%202019.docx).
<!--more-->
# Hacer minería de texto con documentos Word
Si has hecho minería de texto antes, sabrás que el formato aceptado de entrada para generar un corpus es un archivo de texto `txt`. Por eso, tener un documento de Word 2010+ es un problema.

Entra `docx2txt`. Descubrí esta pequeña herramienta en Debian que hace *exactamente* lo que su nombre dice: transforma un documento Word, `docx`, a un archivo de texto plano, `txt`. Su modo de uso es tremendamente sencillo: `docx2txt infile.docx outfile.txt`; también existe la posibilidad de enviar el documento hacia `stdout`. Entonces, convertimos el documento para verificar la salida:

{% highlight shell %}
$ docx2txt input_speech.docx - | head -n 20
                                                          22 de octubre de 2019
                             DECLARACIÓN PÚBLICA
               PRESIDENTE DE LA REPÚBLICA SEBASTIÁN PIÑERA E.

Chilenas y chilenos:  Muy buenas noches,

En los últimos días hemos conocido graves hechos de violencia, delincuencia, vandalismo y destrucción.

Pero también hemos escuchado, fuerte y clara, la voz de la gente expresando pacíficamente sus problemas, sus dolores, sus carencias, sus sueños y sus esperanzas de una vida mejor.

Frente a los graves hechos de violencia, delincuencia, vandalismo y destrucción el Gobierno ha reaccionado utilizando todos los instrumentos que contempla la Constitución y la Ley, para cumplir con nuestro deber de resguardar el Orden Público y la Seguridad Ciudadana y proteger las libertades y derechos de todos los chilenos a movilizarse, estudiar, trabajar, abastecerse y poder vivir sus vidas con libertad, normalidad y seguridad.

A pesar de todos los problemas, la situación está mejorando.

Gracias a la valiosa y sacrificada labor de nuestras Fuerzas Armadas y de Orden y la colaboración de muchos ciudadanos, el orden público y la seguridad ciudadana están mejorando.

Adicionalmente, estamos haciendo nuestros mejores esfuerzos para avanzar hacia una normalización de la vida cotidiana de las personas.  Estamos trabajando duro para mejorar el transporte público, el acceso a alimentos, salud, farmacias, bencina, escuelas y otros servicios de vital importancia para los ciudadanos.  De hecho, con el sacrificado esfuerzo de los trabajadores del Metro, mañana reiniciarán operaciones las Líneas 3 y 6 del Metro, las que se unirán a la Línea 1 ya funcionando.

Sé que algunos piden terminar con los Estados de Emergencia y el Toque de Queda.  Todos lo queremos.  Pero como Presidente es mi deber levantar los Estados de Emergencia cuando tenga seguridades que el Orden Público, la Seguridad Ciudadana y los bienes, tanto públicos como privados, estén debidamente resguardados.
{% endhighlight %}

Pareciera ser que el documento se transforma adecuadamente. Vamos a guardarlo en un archivo plano, `input_speech.txt`, y lo importaremos a R.

{% highlight shell %}
$ ll
total 48
drwxrwxrwx 1 jtapia jtapia  4096 Oct 23 09:05  ./
drwxrwxrwx 1 jtapia jtapia  4096 Oct 23 08:23  ../
-rwxrwxrwx 1 jtapia jtapia 24080 Oct 23 07:24 'Mensaje Pdte - 22 de octubre de 2019.docx'*
-rwxrwxrwx 1 jtapia jtapia 24080 Oct 23 08:48  input_speech.docx*
$ docx2txt input_speech.docx input_speech.txt
$ ll
total 60
drwxrwxrwx 1 jtapia jtapia  4096 Oct 23 09:23  ./
drwxrwxrwx 1 jtapia jtapia  4096 Oct 23 08:23  ../
-rwxrwxrwx 1 jtapia jtapia 24080 Oct 23 07:24 'Mensaje Pdte - 22 de octubre de 2019.docx'*
-rwxrwxrwx 1 jtapia jtapia 24080 Oct 23 08:48  input_speech.docx*
-rwxrwxrwx 1 jtapia jtapia  9996 Oct 23 09:23  input_speech.txt*
{% endhighlight %}

# Definiendo el ambiente de R
Como la vez anterior con los SELF JOINS, definiré un ambiente de `conda` nuevo, para tener instalaciones frescas de los paquetes de R necesarios. Para ello, usé el siguiente comando en WSL:
{% highlight shell %}
$ conda create -n tm_speech_20191023 r-essentials r-base

## Package Plan ##

  environment location: /home/jtapia/anaconda3/envs/tm_speech_20191023

  added / updated specs:
    - r-base
    - r-essentials
{% endhighlight %}

Con ello, tengo el entorno de R definido. Ahora, hay que instalar el paquete `tm` (de *Text Mining*) para `conda`:

{% highlight shell %}
$ conda search -f r-tm
WARNING: The conda.compat module is deprecated and will be removed in a future release.
Loading channels: done
# Name                       Version           Build  Channel
r-tm                           0.6_2        r3.2.1_0  pkgs/r
r-tm                           0.6_2       r3.2.1_0a  pkgs/r
r-tm                           0.6_2        r3.2.2_0  pkgs/r
r-tm                           0.6_2       r3.2.2_0a  pkgs/r
r-tm                           0.6_2        r3.3.1_0  pkgs/r
r-tm                           0.6_2        r3.3.2_0  pkgs/r
r-tm                           0.7_1        r3.4.1_0  pkgs/r
r-tm                           0.7_1  r342h8736426_0  pkgs/r
r-tm                           0.7_3 mro343h599a50d_0  pkgs/r
r-tm                           0.7_3 mro350hc05d2f9_0  pkgs/r
r-tm                           0.7_3  r343h599a50d_0  pkgs/r
r-tm                           0.7_3  r350hebe7666_0  pkgs/r
r-tm                           0.7_5 mro351hebc1506_0  pkgs/r
r-tm                           0.7_5  r351h29659fb_0  pkgs/r
r-tm                           0.7_6   r36h29659fb_0  pkgs/r

$ conda install -c r r-tm
## Package Plan ##

  environment location: /home/jtapia/anaconda3/envs/tm_speech_20191023

  added / updated specs:
    - r-tm


The following packages will be downloaded:

    package                    |            build
    ---------------------------|-----------------
    r-nlp-0.2_0                |    r36h6115d3f_0         432 KB  r
    r-slam-0.1_45              |    r36h96ca727_0         205 KB  r
    r-tm-0.7_6                 |    r36h29659fb_0         849 KB  r
    ------------------------------------------------------------
                                           Total:         1.5 MB

The following NEW packages will be INSTALLED:

  r-nlp              r/noarch::r-nlp-0.2_0-r36h6115d3f_0
  r-slam             r/linux-64::r-slam-0.1_45-r36h96ca727_0
  r-tm               r/linux-64::r-tm-0.7_6-r36h29659fb_0


Proceed ([y]/n)? y


Downloading and Extracting Packages
r-tm-0.7_6           | 849 KB    | ################################################################################################################## | 100%
r-slam-0.1_45        | 205 KB    | ################################################################################################################## | 100%
r-nlp-0.2_0          | 432 KB    | ################################################################################################################## | 100%
Preparing transaction: done
Verifying transaction: done
Executing transaction: done
{% endhighlight %}

Instalamos también `wordcloud`:
{% highlight shell %}
(tm_speech_20191023) $ conda search -f r-wordcloud
WARNING: The conda.compat module is deprecated and will be removed in a future release.
Loading channels: done
# Name                       Version           Build  Channel
r-wordcloud                      2.6   r36h29659fb_0  pkgs/r
(tm_speech_20191023) $ conda install -c r r-wordcloud
## Package Plan ##

  environment location: /home/jtapia/anaconda3/envs/tm_speech_20191023

  added / updated specs:
    - r-wordcloud


The following packages will be downloaded:

    package                    |            build
    ---------------------------|-----------------
    r-wordcloud-2.6            |    r36h29659fb_0         124 KB  r
    ------------------------------------------------------------
                                           Total:         124 KB

The following NEW packages will be INSTALLED:

  r-wordcloud        r/linux-64::r-wordcloud-2.6-r36h29659fb_0


Proceed ([y]/n)? y


Downloading and Extracting Packages
r-wordcloud-2.6      | 124 KB    | ################################################################################################################## | 100%
Preparing transaction: done
Verifying transaction: done
Executing transaction: done
{% endhighlight %}

Verificamos el acceso a R y la existencia de `tm` y `wordcloud`:
{% highlight shell %}
$ conda activate tm_speech_20191023
(tm_speech_20191023) $ which R
/home/jtapia/anaconda3/envs/tm_speech_20191023/bin/R
{% endhighlight %}

{% highlight R %}
R version 3.6.1 (2019-07-05) -- "Action of the Toes"
Copyright (C) 2019 The R Foundation for Statistical Computing
Platform: x86_64-conda_cos6-linux-gnu (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

> require("tm")
Loading required package: tm
Loading required package: NLP
> require(wordcloud)
Loading required package: wordcloud
Loading required package: RColorBrewer
{% endhighlight %}

## Iniciando RStudio con el ambiente `conda`
Lo último que falta por hacer es poder levantar RStudio usando el ambiente `conda` que definimos, `tm_speech_20191023`, como base para los paquetes. Para ello, un usuario en GitHub ([Gregor Sturm](https://github.com/grst)), creó [una excelente solución para Linux, usando dos scripts `sh`](https://github.com/grst/rstudio-server-conda). Para implementarla, sólo hace falta clonar el repo y lanzar el script:

{% highlight shell %}
$ sudo systemctl disable rstudio-server
rstudio-server.service is not a native service, redirecting to systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable rstudio-server
$ sudo service rstudio-server stop
$ git clone https://github.com/grst/rstudio-server-conda.git
Cloning into 'rstudio-server-conda'...
remote: Enumerating objects: 48, done.
remote: Counting objects: 100% (48/48), done.
remote: Compressing objects: 100% (39/39), done.
remote: Total 48 (delta 27), reused 18 (delta 9), pack-reused 0
Unpacking objects: 100% (48/48), done.
$ ll
total 60
drwxrwxrwx 1 jtapia jtapia  4096 Oct 23 09:30  ./
drwxrwxrwx 1 jtapia jtapia  4096 Oct 23 08:23  ../
-rwxrwxrwx 1 jtapia jtapia 24080 Oct 23 07:24 'Mensaje Pdte - 22 de octubre de 2019.docx'*
-rwxrwxrwx 1 jtapia jtapia 24080 Oct 23 08:48  input_speech.docx*
-rwxrwxrwx 1 jtapia jtapia  9996 Oct 23 09:23  input_speech.txt*
drwxrwxrwx 1 jtapia jtapia  4096 Oct 23 09:31  rstudio-server-conda/
$ conda activate tm_speech_20191023
(tm_speech_20191023) $ rstudio-server-conda/start_rstudio_server.sh 8787
## Current env is >>
... verbose output ...
{% endhighlight %}

Verificamos, desde RStudio, que R está ubicado en el ambiente definido:
{% highlight R %}

R version 3.6.1 (2019-07-05) -- "Action of the Toes"
Copyright (C) 2019 The R Foundation for Statistical Computing
Platform: x86_64-conda_cos6-linux-gnu (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

> R.home()
[1] "/home/jtapia/anaconda3/envs/tm_speech_20191023/lib/R"

{% endhighlight %}

Finalmente, tenemos todo lo que necesitamos en su lugar. Ahora, vamos al análisis.

# Análisis de texto con `tm`
## Importando el documento y creando un `corpus`
Vamos a lo primero: tenemos que leer el archivo y limpiarlo.
{% highlight R %}
> setwd("/path/to/folder/tm_speech_20191023")
> getwd()
[1] "/path/to/folder/tm_speech_20191023"
> list.files()
[1] "input_speech.docx"                        
[2] "input_speech.txt"                         
[3] "Mensaje Pdte - 22 de octubre de 2019.docx"
[4] "rstudio-server-conda"                     
> speech_txt <- readChar(con = paste0(getwd(), "/input_speech.txt"),
                          nchars = file.info(paste0(getwd(), "/input_speech.txt"))$size)
> cleanText <- function(x)
+ {
+     x <- chartr('áéíóú', 'aeiou', x)
+     x <- gsub("\r?\n|\r|\t", " ", x)
+     x <- gsub("[ |\t]{2,}", " ", x)
+     x <- gsub("^ ", "", x)
+     x <- gsub(" $", "", x)
+     x <- gsub("<", "", x)
+     x <- gsub(">", "", x)
+     x <- gsub("[[:punct:]]", " ", x)
+     x <- gsub("[[:digit:]]", " ", x)
+     x <- gsub("(?<=[\\s])\\s*|^\\s+|\\s+$", " ", x, perl=TRUE)
+ }
> speech_clean <- cleanText(speech_txt)
{% endhighlight %%}

La función `cleanText` permite sacar las tildes, puntuaciones, y, más importante, dejar todo monoespaciado. Esto es fundamental para poder tener un corpus limpio de texto, que es lo que generamos a continuación:
{% highlight R %}
> library(tm)
Loading required package: NLP
> speech_corpus <- VCorpus(VectorSource(speech_clean))
> speech_corpus <- tm_map(speech_corpus,
                          stripWhitespace)
> speech_corpus <- tm_map(speech_corpus,
                          content_transformer(tolower))
> speech_corpus <- tm_map(speech_corpus,
                          removeWords, stopwords("spanish"))
> speech_corpus
<<VCorpus>>
Metadata:  corpus specific: 0, document level (indexed): 0
Content:  documents: 1
{% endhighlight %}

Hay que notar que se aplicaron tres transformaciones al corpus original: 
* `stripWhitespace` remueve los espacios en blanco
* `tolower` (a través de la función auxiliar `content_transformer`) lleva todo a minúscula
* `removeWords` - `stopwords` elimina [palabras vacías](https://es.wikipedia.org/wiki/Palabra_vac%C3%ADa) (artículos, pronombres, interjecciones, preposiciones, etc.)

Finalmente, creamos una matriz término documento y la convertimos a una matriz tradicional:
{% highlight R %}
> speech_tdm <- TermDocumentMatrix(speech_corpus)
> speech_tdm
<<TermDocumentMatrix (terms: 506, documents: 1)>>
Non-/sparse entries: 506/0
Sparsity           : 0%
Maximal term length: 15
Weighting          : term frequency (tf)
> speech_matrix <- as.matrix(speech_tdm)
> speech_matrix
                 Docs
Terms              1
  abastecerse      1
  ... verbose output ...
{% endhighlight %}

## Generar nube de palabras
Para generar una nube de palabras, los términos tienen que estar en un `data.frame`. Para eso, realizamos el siguiente proceso (que incluye una revisión de la estructura creada):
{% highlight R %}
> speech_vector <- sort(rowSums(speech_matrix),
                        decreasing = TRUE)
> speech_df <- data.frame(word = names(speech_vector),
                          freq = speech_vector)
> head(speech_df)
               word freq
mayores     mayores   10
chilenos   chilenos    8
social       social    8
agenda       agenda    7
mas             mas    7
seguridad seguridad    6
{% endhighlight %}

Lo único que queda es generar la nube de palabras:
{% highlight R %}
wordcloud(words = speech_df$word,
          freq = speech_df$freq,
          min.freq = 1,
          max.words = 100,
          random.order = FALSE,
          colors = brewer.pal(8, "Dark2"),
          scale = c(2, .2),
          rot.per = 0.5)
{% endhighlight %}

Y nuestro resultado es:
![Nube de palabras discurso 2019/10/22](/images/wordcloud.png)

## Otros análisis
Vemos muchas palabras (elegimos un tope máximo de 100, con el argumento `max.words`), así que tomemos el top 10 de palabras y hagamos un histograma:
{% highlight R %}
barplot(height = speech_df[1:10,]$freq,
        las = 2,
        names.arg = speech_df[1:10,]$word,
        col = "lightblue",
        main = "Palabras más comunes, top 10",
        ylab = "Palabras")
{% endhighlight %}

![Histograma del top 10 de palabras](/images/barplot.png)

# Interpretación de los resultados
Con un poco de conocimiento de lo que está aconteciendo, me parece extraño que, dadas las reacciones de la clase política, haya en las 10 palabras más comunes sólo menciones a hechos actuales y meta comentarios (*"vamos a impulsar una agenda social"*, *"vamos a crear proyectos"*). Me parece extraño, también, que el décimo término sea la palabra **mil**.

Otros análisis le corresponden a analistas políticos. Por lo pronto, pueden encontrar el script en [el repo de GitHub](https://github.com/jtapiath-cl/ssibd/blob/master/tm_speech_20191023/tm_speech.R).