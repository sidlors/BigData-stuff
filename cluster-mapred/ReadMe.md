## Proyecto Map/Reduce Hadoop

En este proyecto se pretende mostar como es que el big data puede generar valor agregado 
a una solucion informatica sacando partido de informacion *"Chatarra"* como lo son logs, e-mails, 
informacion de formularios de una pagina web simple, etc...

Para este proyecto se plantea como escenario que una pequeña empresa tiene acceso a salidas txt
con informacion de entidades bancarias, su identificador (id) y una direccion

Ejemplo:

<pre>
001f000000Z6Pn0AAF	LEGAL & GENERAL INVESTMENT MANAGEMENT (HOLDINGS) LTD	One Coleman St	London		EC2R 5AA	England		
0013000000IcuORAAZ	Merrill Lynch Canada	181 Bay St Suite 400	Toronto	ON	M5J 2V8	Canada		
001300000063ZKKAA2	SEI Investments	1 Freedom Valley Dr	Oaks	PA	19456-9989	United States		
001300000063ZJyAAM	Pershing LLC	1 Pershing Plz	Jersey City	NJ	07399-0002	United States
</pre>

Como podemos darnos cuenta, la informacion esta no estructurada, no hay un claro separador que nos ayude
a interpretar correctamente desde el puto de vista computacional los campos que corresponden a cada 
atributo de nuestro modelo de datos.

El enfoque que ocuparemos para esta tarea es basicamente utilizar dos procesos Big Data Hadoop
para lograr nuestro cometido, a saber:

  * Un proceso que se encarge de mapear cada *registro* y lo reduzca en terminos de ids para determina si una misma entidad bacarea viene duplicada y hacer un calculo de palabras para le siguiente proceso:
  * Un segundo proceso que a la salida del anterior realize un *calculo de distancia entre frases*
  
  El primero de los procesos se puede localizar en package com.eduonix.hadoop.partone
  El segundo de los procesos se puede localizar en el package com.eduonix.hadoop.partone.etl

En ambos procesos se codifican a manera de un ETL, donde viene la parte de la extraccion que basicamente es leer los datos de entrada (ComercialBanks.csv) en algun filesystem que puede ser local (C: \.\etc\\...) mediante la ejecion tipica de culaquier porgrama java SE o En un cluster como un Job Hadoop con el comando: *$hadoop jar <nombreDelJar>.jar*

La parte de la Transformacion es la logica correspondiente a efectuar una vez cargado en un dataSet serlizado y mapeado usando la libreria Apache Crunch que define el manejo de datos en terminos de *pipelines*

La salida o Carga es escribir los resultados del proceso mapReduce al HDFS, una vez que le primer proceso termino el segundo antes mencionado se encarga del calculo de palabras, que en este caso ayudaran a determinar si los elementos duplicados en realidad si tiene direcciones practicamente iguales, veremos que en la mayoria de los casos las direcciones difieren por los numeros telefonicos, demostrando asi que la calidad de los datos no necesariamente tiene que ser bueno o reular para que un sistema con procesos hadoop mapReduce nos funcione perfectamente

