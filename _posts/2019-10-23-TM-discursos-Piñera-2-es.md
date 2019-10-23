---
layout: post
title:  Análisis comparativo de los discursos del presidente Piñera en relación al estado de Emergencia en Chile, octubre de 2019
categories: ["data science"]
tags: [text-mining, tm]
published: true
language: es
comments: true
---
Realicé anteriormente un análisis del último discurso del presidente Piñera, realizado el 22 de octubre, donde se plantean medidas políticas para limitar la desigualdad en Chile. El estudio de desigualdad es para otro momento, pero me parece profundamente tendencioso e injusto no mostrar los discursos en contexto, porque ha sido una situación en desarrollo, y no una instancia específica.

Tomé de la [página oficial de Prensa de la Presidencia](https://prensa.presidencia.cl) los discursos de los días [sábado 19](https://prensa.presidencia.cl/comunicado.aspx?id=103668), [domingo 20](https://prensa.presidencia.cl/comunicado.aspx?id=103689), [lunes 21](https://prensa.presidencia.cl/comunicado.aspx?id=103727) y [martes 22](https://prensa.presidencia.cl/comunicado.aspx?id=123766) de octubre, para analizar todos los discursos por separado y en contexto. El código va a ir en la sección final, completo, para dar paso sólo a los resultados y el análisis.
<!--more-->
# Definiendo el ambiente de R
La definición del ambiente es la misma que la anterior; esto implica que usaremos `tm` y `wordcloud`, en un entorno `conda` fresco instalado en WSL.

# Análisis de texto con `tm`
## Análisis individuales: nubes de palabras e histogramas
Generamos una nube de palabras y un histograma del top 10 de palabras para cada discurso:

#### Discurso presidencial, sábado 19 de octubre
![Primer discurso, sábado 19 de octubre](/images/wordcloud-oct19.png)
![Top 10 palabras del primer discurso](/images/barplot-oct19.png)

Vemos un discurso cargado hacia lo apelativo: "chilenos", "compatriotas", "familias". Vemos una leve mención a la seguridad: "problemas", "seguridad", "violencia" y "delincuencia".

#### Discurso presidencial, domingo 20 de octubre
![Segundo discurso, domingo 20 de octubre](/images/wordcloud-oct20.png)
![Top 10 de palabras del segundo discurso](/images/barplot-oct20.png)

El segundo discurso está orientado hacia el deseo: "paz", "quiero", "libertad", "mañana". También es profundamente apelativo: "compatriotas" y "chilenos", suman 17 instancias en el discurso. Vemos, de manera interesante, la irrupción de la palabra "región": el sábado 20 de octubre se sumaron regiones al estado de emergencia, lo que se ve reflejado en el texto entregado a la nación.

Es importante notar que este discurso es el que incluye la muy bullada frase ["estamos en guerra"](https://www.elmundo.es/internacional/2019/10/21/5dad3cee21efa0a82d8b45db.html); sin embargo, en términos discursivos, no se aprecia un movimiento hacia lo violento sino todo lo contrario.

#### Discurso presidencial, lunes 21 de octubre
![Tercer discurso, lunes 21 de octubre](/images/wordcloud-oct21.png)
![Top 10 de palabras del tercer discurso](/images/barplot-oct21.png)

Un discurso *pegamento*: 12 veces se repitió la palabra "también". Se ve un giro discursivo, reconociendo la *violencia* que *chilenas* y *chilenos* han sentido, que genera *daño*, donde los *compatriotas* sufren la *delincuencia*.

Llama la atención las cinco repeticiones de la palabra "presidente". Cabe destacar que en este discurso, el presidente Piñera indica que van a reunirse las cabezas de los otros dos poderes del estado, lo que explica dicha aparición.

#### Discurso presidencial, martes 22 de octubre
![Cuarto discurso, martes 22 de octubre. Se entregan propuestas de medidas](/images/wordcloud-oct22.png)
![Top 10 de palabras del cuarto discurso](/images/barplot-oct22.png)

Como ya fue mencionado en un [artículo anterior]({% post_url 2019-10-23-TM-discursos-Piñera-es %}), el foco del discurso es la meta discusión ("agenda", "social", "congreso", "creación"). También hay menciones fuertes hacia la *seguridad* y la *delincuencia*. Tiene sentido, también, el foco en la meta discusión, dado que fue un discurso enfocado en lo propositivo más que en lo factual.

## Análisis conjuntos: nube de comparación
El paquete de R `tm` tiene una capacidad interesante, la ***nube de comparación***. Esta es una nube de palabras que pone lado a lado todas las palabras más importantes de cada *corpus* de palabras.

Este es el resultado del análisis:
![Nube de comparación de discursos](/images/comparison-cloud.png)

Podemos ver que lo que más comparten los discursos es la idea de la familia, de la violencia (especialmente los últimos dos discursos) y el "también". Un análisis más profundo corresponde a personas con mayor formación discursiva y política que yo.

## Análisis conjuntos: nube de similitudes
Para finalizar, el ejercicio más interesante desde lo discursivo cuantitativo es revisar las similitudes entre los documentos; en este caso, ver lo que tienen los discursos en común.

![Nube de similaridades](/images/commonality-cloud.png)

Este resultado nos muestra, finalmente, lo que mi propia impresión me indicaba: el discurso del presidente, a través de los días, ha estado permeado por los conceptos de *delincuencia*, *violencia*, apelando a los *chilenos*. Una parte que me ha sorprendido, en la elaboración de este artículo, es la inclusión de las ideas fuerza de *orden* y la constante apelación a la nación. Finalmente, en esta nube de similitudes, podemos ver menciones ocultas en los discursos individuales: *medicamentos*, *metro* (el [detonante de las protestas](https://www.bbc.com/mundo/noticias-america-latina-50112080)), *emergencia*. Vemos menciones muy menores (hay que recordar que el tamaño de la palabra indica el peso de la palabra) a *pensiones*, *transporte* y *ley*.

Un disclaimer importante: el análisis acá realizado es absolutamente subjetivo respecto de los resultados obtenidos a partir de métodos cuantitativos de minería de texto. Todas las opiniones vertidas aquí son de mi exclusiva responsabilidad, y no buscan en ningún momento tratar de ser representativas. Mi visión de los hechos ocurridos hasta el momento en Chile es personal; si he ofendido a alguien, o consideran que son ofensivas, por favor contáctenme a [mi correo](mailto:javier@ctjconsult.com) para modificar el artículo y bajar lo que pueda resultar ofensivo.

# Anexo técnico: código fuente
El código usado está a continuación:

{% gist 6d910f4241902773d814474f9495f1cd %}