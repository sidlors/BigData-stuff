## Introducción a Crunch

### Motivación

Comencemos con una pregunta básica: 
* ¿Por qué debería usar cualquier herramienta de alto nivel para escribir datos de pipelines, en contraste a desarrollar con las propias API's MapReduce, Spark o Tez directamente? 
* ¿El añadir otra capa de abstracción sólo aumentar el número de piezas en movimiento que nos interesa?

Como con cualquier decisión como esta la respuesta es "depende". Durante mucho tiempo, el principal beneficio de utilizar una herramienta de alto nivel fue poder aprovechar el trabajo realizado por otros desarrolladores para soportar patrones comunes de MapReduce, como combinaciones y agregaciones, sin tener que aprenderlos y reescribirlos uno mismo.

Si fueras a necesitar aprovechar estos patrones a menudo en tu trabajo, 
valia la pena la inversión para aprender cómo usar la herramienta y lidiar con las inevitables lagunas de las abstracciones de la herramienta.

Con Hadoop 2.0, estamos empezando a ver la aparición de nuevos motores para ejecutar data  pipelines encima de los datos almacenados en HDFS. Además de MapReduce, hay nuevos proyectos como *Apache Spark* y *Apache Tez*. 

Los desarrolladores ahora tienen más opciones para implementar  y ejecutar sus pipelines, y puede ser difícil saber de antemano qué motor es mejor para su problema, especialmente porque los pipelines tienden a evolucionar con el tiempo para procesar más fuentes de datos y mayores volúmenes de datos. Esta opción significa que hay una nueva razón para utilizar una herramienta de alto nivel para expresar su pipeline de datos: a medida que las herramientas agregan soporte para nuevos framewors puede probar el rendimiento de su pipeline en el nuevo marco sin tener que reescribir su lógica Contra nuevas API.

Hay muchas herramientas de alto nivel disponibles para la creación de pipelines de datos en la parte superior de Apache Hadoop, y cada uno tiene pros y contras en función del desarrollador y el caso de uso. **Apache Hive** y **Apache Pig** definen lenguajes específicos de dominio (DSLs) estinados a facilitar el trabajo de los analistas de datos con los datos almacenados en Hadoop, mientras que **Cascading** y **Apache Crunch** desarrollan ***bibliotecas Java dirigidas a desarrolladores que están construyendo pipelines y aplicaciones con un enfoque en el rendimiento y la testabilidad.***

### Entonces, ¿qué herramienta es adecuada para mi problema? 

Si la mayor parte de su trabajo de pipelines implica datos y operaciones relacionales,  *Hive* *Pig* o *Cascading* proporciona muchas funciones de alto nivel y herramientas que harán su vida más fácil. 

Si su problema implica trabajar con datos no relacionales (registros complejos, tablas de HBase, vectores, datos geoespaciales, etc.)  o requiere que escriba mucha lógica personalizada a través de funciones definidas por el usuario (UDF), entonces Crunch es probablemente el derecho elección.

Aunque todos estos proyectos tienen sus propias filosofías de desarrollo y comunidades, todos ellos se implementan usando aproximadamente el mismo conjunto de patrones.

La siguiente tabla ilustra la relación entre estos patrones a través de  los diversos proyectos de pipelines de datos que se ejecutan en la parte superior de Apache Hadoop.

| ﻿Concept                       | Apache Hadoop MapReduce       | Apache Crunch    | Apache Pig             | Apache Spark                      | Cascading                    | Apache Hive                       | Apache Tez |
|-------------------------------|-------------------------------|------------------|------------------------|-----------------------------------|------------------------------|-----------------------------------|------------|
| Input Data                    | InputFormat                   | Source           | LoadFunc               | InputFormat                       | Tap (Source)                 | SerDe                             | Tez Input  |
| Output Data                   | OutputFormat                  | Target           | StoreFunc              | OutputFormat                      | Tap (Sink)                   | SerDe                             | Tez Output |
| Data Container Abstraction    | N/A                           | PCollection      | Relation               | RDD                               | Pipe                         | Table                             | Vertex     |
| Data Format and Serialization | Writables                     | POJOs and PTypes | Pig Tuples and Schemas | POJOs and Java/Kryo Serialization | Cascading Tuples and Schemes | List<Object> and ObjectInspectors | Events     |
| Data Transformation           | Mapper, Reducer, and Combiner | DoFn             | PigLatin and UDFs      | Functions (Java API)              | Operations                   | HiveQL and UDFs                   | Processor  |

### Modelo de Datos y Operadores

El Java API de Crunch esta centrado en 3 interfaces que representan los datasets distribuidos: **PCollection, PTable, PGroupedTable**

***PCollection<T>*** Representa una colección distribuida e inmutable de elementos del tipo T(Java Generics). Por ejemplo, representamos un archivo de texto como una API de PCollection

El Objeto PCollection proporciona un método, **parallelDo**, que aplica un **DoFn** a cada elemento en PCollection de forma paralela y devuelve un nuevo PCollection como su resultado.

La clase PTable <K, V> es una subinterfaz de PCollection <Pair <K, V >> que representa un multimap distribuido, desordenado de su tipo de clave K a su tipo de valor V. Además de la operación parallelDo, PTable proporciona un metodo *GroupByKey* operación que agrega todos los valores en el PTable que tienen la misma clave en un único registro. Es la operación groupByKey que dispara la fase *sort* del MapReduce. 

Los desarrolladores pueden ejercer un control detallado sobre el número de reductores y las estrategias de partición, agrupación y clasificación utilizadas durante el *shuffle* mediante una instanciacion de la clase *GroupingOptions* a la función *groupByKey*

El resultado de una operación groupByKey es un objeto *PGroupedTable <K, V>* un mapa distribuido y ordenado de claves de tipo K a un Iterable que puede iterarse exactamente una vez.

Además del procesamiento de *parallelDo* a través de DoFns, *PGroupedTable* proporciona una operación *combineValues* que permite a un conmutativo y asociativo *Aggregator* para ser aplicado a los valores de la instancia *PGroupedTable* en el mapa y reducir los lados del shuffle. 

*Se proporcionan varias implementaciones comunes de Aggregator <V> en la clase Aggregators.*

Finalmente *PCollection, PTable y PGroupedTable* soportan una operación de unión, que toma una serie de PCollections distintas que tienen el mismo tipo de datos y las trata como una única PCollection virtual.

Todas las otras operaciones de transformación de datos soportadas por las API de Crunch (aggregations, joins, sorts, secondary sorts, and cogrouping) son implementados en términos de estos cuatro primitivos. 

Los patrones en si mismos son definidos en el paquete org.apache.crunch.lib y sus hijos y algunos de los patrones más comunes tienen funciones convenientes definidas en las interfaces *PCollection* y *PTable*.

Cada pipeline de datos *Crunch* está coordinado por una instancia de la interfaz Pipeline que define métodos para leer datos en un pipeline a través de instancias Source y escribir datos desde un pipline a instancias Target. Actualmente hay tres implementaciones de la interfaz de Pipeline que están disponibles para que los desarrolladores utilicen:

1. **MRPipeline**: Ejecuta el pipeline como una serie de trabajos de MapReduce.
2. **MemPipeline**: Ejecuta el pipeline en memoria en el cliente.
3. **SparkPipeline**: Ejecuta el pipeline convirtiéndola en una serie de pipelines Spark.

### Proceso de datos con DoFns

DoFns representa la  logica computacional de los pipelines Crunch. Estos estan diseñaos para  escrivir , probar y deployar facilmente con el contexto de un job MapReduce. Mucha de la codificacion usando el API Crunch APIs se escribira en el metodos DoFns por lo que es  tener un buen entedimiento  de como usarlo efectivament sera critico para elaborando pipelines elegantes y eficientes.

Si nuestro *DoFn* necesita trabajar con una clase que no implementa Serializable y no se puede modificar (por ejemplo, porque se define en una biblioteca de terceros) se debera utilizar la palabra clave *transient* en esa variable de miembro para que la serialización de la DoFn no falle si ese objeto pasa a ser definido. Se puede crear una instancia del objeto durante el tiempo de ejecución utilizando el método *initialize*.

Un lugar donde los serializables DoFns pueden tropezar con nuevos desarrolladores de Crunch es cuando especifican el DoFns dentro del métodos de clases externas no serializables. 

```java
public class MyPipeline extends Configured implements Tool {
  public static void main(String[] args) throws Exception {
    ToolRunner.run(new Configuration(), new MyPipeline(), args);
  }

  public int run(String[] args) throws Exception {
    Pipeline pipeline = new MRPipeline(MyPipeline.class, getConf());
    PCollection<String> lines = pipeline.readTextFile(args[0]);

    // This will throw a NotSerializableException!
    PCollection<String> words = lines.parallelDo(new DoFn<String, String>() {
      @Override
      public void process(String line, Emitter<String> emitter) {
        for (String word : line.split("\\s+")) {
          emitter.emit(word);
        }
      }
    }, Writables.strings());

    words.count().write(To.textFile(args[1]));

    pipeline.done();
  }
}
```

Aunque el pipelines compilan bien y se ejecutarán localmente con MemPipeline de Crunch, las versiones de MRPipeline o SparkPipeline fallarán con la excepción *NotSerializableException* de Java.


La otra forma de resolver este problema cuando se usen clases internas es definir el *new DoFn()...*  dentro de un métodos estáticos en la clase.

```java
  public static PCollection<String> splitLines(PCollection<String> lines) {
    // This will work fine because the DoFn is defined inside of a static method.
    return lines.parallelDo(new DoFn<String, String>() {
      @Override
      public void process(String line, Emitter<String> emitter) {
        for (String word : line.split("\\s+")) {
          emitter.emit(word);
        }
      }
    }, Writables.strings());
  }
```

*Conclusion:*

> *El uso de métodos estáticos para definir su lógica de negocio en términos de un clase anonima DoFns  puede hacer que su código sea más fácil de probar utilizando implementaciones PCollection en memoria en sus pruebas unitarias*

### Pasos del procesamiento en tiempo de ejecución

Después de que Crunch carga en runtime los DoFns serializados dentro de sus  tareas mapa y reducers, los DoFns se ejecutan sobre los datos de entrada a través de la siguiente secuencia:

1. En primer lugar, el DoFn tiene acceso a la implementación *TaskInputOutputContext* para la tarea actual. Esto permite que DoFn acceda a la configuración necesaria y la información de tiempo de ejecución necesaria antes o durante el procesamiento. Recuerde, DoFns puede ejecutarse en la fase mapa o reducer del pipeline; Nuestra lógica no debe asumir, en general, que sabe en que fase de la ejecutará esta un DoFn en particular.

3. A continuación, se llama al método de inicialización de DoFn. El método *initialize* es similar al método de configuración utilizado en las clases Mapper y Reducer; Se llama antes de que comience el procesamiento para poder realizar cualquier inicialización o configuración necesaria del DoFn. Por ejemplo, si estábamos hacier uso de una biblioteca no serializable de terceros, crearíamos una instancia de ella aquí.

5. En este punto comienza el proceso de datos. La tarea de mapa o reducer comenzará a pasar los registros al método de proceso de DoFn e irá capturando la salida del proceso en un *Emitter<T>* que puede puede incluso pasar los datos a otro DoFn para procesar o serializarlo como salida del pipeline de procesamiento actual.

7. Finalmente, después de que todos los registros se han procesado, el método *void cleanup(Emitter<T> emitter) * se llama en cada DoFn. El método de limpieza tiene un doble propósito: se puede utilizar para emitir cualquier información de estado que el DoFn quiere pasar a la siguiente etapa (por ejemplo *cleanup* podría ser utilizada para emitir la suma de una lista de números que se pasó a al método de procesamiento originalmente DoFn), así como para liberar cualquier recurso o realizar cualquier otra tarea de limpieza que sea apropiada una vez que la tarea haya terminado de ejecutarse.

### Acceso en Runtime al API MapReduce

DoFns da acceso directo al objeto *TaskInputPutoputContext* para usarlo con una tarea Map/Reduce via el metodo getContext; hay un buen numero de metodos utiles trabajando con *TaskInputPutoputContext*


* getConfiguration() para acceder al objeto Configuration que contiene aquellos detalles relacionados conn el sistema y parametros especificos de usuario dados para el Job MapReduce
* progress() Para indicar que un trabajo será de ejecucion lenta o computacionalmente intenso y que se debera mantener vivo evitando asi que el framework no lo mate.
* setStatus(String status) y getStatus() para configurar y leer informacion del estatus de una tarea
* getTaskAttemptID() para leer informacion del TaskAttemptID.


DoFns también tiene un número de métodos auxiliares para trabajar con los Hadoop Counters, todos llamados 'increment'. 

Los contadores son una forma increíblemente útil de mantener un registro del estado de las tuberías de datos de larga duración y detectar cualquier condición excepcional que se produzca durante el procesamiento, y están soportados tanto en los contextos de *MapReduce-based* y *in-memory Crunch pipeline*. 

Podemos recuperar el valor de los Counters en el código de cliente al final de un pipeline MapReduce por medio de los objetos *StageResult* devueltos por Crunch al final de una ejecución.


* increment(String groupName, String counterName) incrementa el valor del contador dado en 1.
* increment(String groupName, String counterName, long value) incrementa el valor del contador dado por el asignado en *value*.
* increment(Enum<?> counterName) incrementa el valor del contador dado by 1.
* increment(Enum<?> counterName, long value) incrementa el valor del contador dado por el asignado en *value*.

### Patrones Comunes DoFn

Las API de Crunch contienen una serie de subclases útiles de DoFn que manejan escenarios de procesamiento de datos comunes y son más fáciles de escribir y probar. 

El paquete *org.apache.crunch* contiene tres de las especializaciones más importantes Cada una de estas implementaciones DoFn tiene métodos asociados en las interfaces PCollection, PTable y PGroupedTable para soportar pasos comunes de procesamiento de datos.

La extensión más simple es la clase FilterFn, que define un único método abstracto boolean accept(T input). 

El filtroFn se puede aplicar a un PCollection <T> llamando al metodo filter(FilterFn<T> fn) devolverá un nuevo PCollection <T> que sólo contiene los elementos de la PCollection de entrada para que el método accept devuelto true. 

Tenga en cuenta que la función de filtro no incluye un argumento PType en su firma, porque no hay ningún cambio en el tipo de datos de la PCollection cuando se aplica FilterFn. 

Es posible componer nuevas instancias FilterFn combinando varios FilterFns juntos utilizando los métodos and, or, y no factory definidos en la clase auxiliar FilterFns.

La segunda extensión es la clase MapFn, que define un único método abstracto  T map(S input). Para tareas de transformación simples en las que cada registro de entrada tendrá exactamente una salida, es fácil probar un MapFn verificando que una entrada dada devuelve una salida dada.

MapFns también es utilizan en métodos especializados en las interfaces PCollection y PTable. PCollection <V> define el método *PTable<K,V> by(MapFn<V, K> mapFn, PType<K> keyType)* que se puede utilizar para crear un PTable desde un PCollection escribiendo una función que extrae la clave Tipo K) del valor (de tipo V) contenido en el PCollection. 

La función by sólo requiere que se da el PType de la clave y construye un PTableType <K, V> del tipo de clave dado y el tipo de valor existente de PCollection.

PTable <K, V>, a su vez, tiene métodos *PTable<K1, V> mapKeys(MapFn<K, K1> mapFn)* y *PTable<K, V2> mapValues(MapFn<V, V2>)*  que manejan el caso común de convertir en uno sólo uno de los valores pareja en una instancia PTable de un tipo a otro, dejando el otro tipo igual.

La extensión final de nivel superior de DoFn es la clase *CombineFn*, que se utiliza junto con el método *combineValues* definido en la interfaz PGroupedTable.

*CombineFns* se utilizan para representar las operaciones asociativas que se pueden aplicar con el concepto Combiner del MapReduce con el fin de reducir la cantidad de datos que se envían a través de la red durante un shuffle.

La extensión CombineFn es diferente de las clases FilterFn y MapFn en que no define un método abstracto para manejar datos más allá del método de proceso predeterminado que cualquier otro DoFn utilizaría; Más bien, extender las señales de clase CombineFn al planificador de Crunch que la lógica contenida en esta clase satisface las condiciones requeridas para su uso con el combinador MapReduce.

Crunch soporta muchos tipos de estos patrones asociativos, como sumas, conteos y uniones de conjuntos, a través de la interfaz *Aggregator*, que se define junto a la clase CombineFn en el paquete org.apache.crunch de nivel superior.

Hay una serie de implementaciones de la interfaz Aggregator definidas via static factory methods de la clase Aggregators.

### Serializacion de Datos con PTypes

Cada PCollection <T> tiene un PType<T> asociado  que encapsula la información sobre cómo serializar y deserializar el contenido de ese PCollection.

Los PType<T> son necesarios debido al aseguramiento de tipos; En el tiempo de ejecución, cuando el planificador de Crunch está mapeando de PCollections a una serie de trabajos de MapReduce, el tipo de un PCollection (es decir, el T en PCollection <T>) ya no está disponible para nosotros y debe ser proporcionado por la instancia asociada PType. 

Cuando estamos creando un nuevo PCollection para que sea usado por *parallelDo* contra un PCollection existente, **el tipo de retorno de su DoFn debe coincidir con el PType dado.**

```java
  public void runPipeline() {
    PCollection<String> lines = ...;

    // Valid
    lines.parallelDo(new DoFn<String, Integer>() { ... }, Writables.ints());

    // Compile error because types mismatch!
    lines.parallelDo(new DoFn<String, Integer>() { ... }, Writables.longs());
  }
```


Crunch soporta dos familias de tipos (formatos) diferentes, cada una de las cuales implementa la interfaz PTypeFamily: una para la interfaz Writable de Hadoop y otra basada en Apache Avro. 

También hay clases que contienen metodos static factory para cada PTypeFamily y asi facilitar la importación y el uso, uno para Writables y otro para Avros.

Las dos familias de tipos diferentes existen por razones históricas: Los Writables han sido durante mucho tiempo la forma estándar para representar datos serializables en Hadoop, pero el esquema de serialización Avro es muy compacto, rápido y permite esquemas de registros complejos evolucionar con el tiempo. 

Está bien (e incluso se recomienda) mezclar y combinar PCollections que utilizen diferentes PTypes  en la misma pipeline Crunch (por ejemplo, se puede leer en Datos Writables hacer un shuffle usando Avro y luego escribir los datos de salida en formato Writables), **pero Cada PTip de PCollection debe pertenecer a una familia de un solo tipo**; Por ejemplo, no se puede tener un PTable cuya clave se serializó como Writable y cuyo valor se serialice como Avro.

### Core PTypes

Ambas familias de tipos admiten un conjunto  de tipos primitivos común (Strings, longs, ints, floats, dobles, booleanos y bytes), así como PTypes más complejos que pueden construirse a partir de otros PTypes:


* Tuplas de otros PTypes (pairs, trips, quads, and tuples para arbitrarios N),
* Collections of other PTypes (collections para crear una coleccion Collection<T> y mapas para regresar un Map<String, T>),
* Y tableOf para construir un PTableType<K, V>, el PType usado para distinguir un PTable<K, V> de un PCollection<Pair<K, V>>.


El tipo *tableOf* es especialmente importante para estar familiarizado, ya que determina si el tipo de retorno de una llamada *parallelDo* en un *PCollection* será un *PTable* en lugar de un *PCollection*, y sólo la interfaz *PTable* tiene el método *groupByKey* que se puede utilizar para iniciar un shuffle en el cluster.

```java  
public static class IndicatorFn<T> extends MapFn<T, Pair<T, Boolean>> {
    public Pair<T, Boolean> map(T input) { ... }
  }

  public void runPipeline() {
    PCollection<String> lines = ...;

    // Return a new PCollection<Pair<String, Boolean>> by using a PType<Pair<String, Boolean>>
    PCollection<Pair<String, Boolean>> pcol = lines.parallelDo(new IndicatorFn<String>(),
        Avros.pairs(Avros.strings(), Avros.booleans()));

    // Return a new PTable<String, Boolean> by using a PTableType<String, Boolean>
    PTable<String, Boolean> ptab = lines.parallelDo(new IndicatorFn<String>(),
        Avros.tableOf(Avros.strings(), Avros.booleans()));
  }
  ```
  
  Si se encuentra en una situación en la que tiene un PCollection <Pair <K, V> y necesita una PTable <K, V>, la clase de biblioteca PTables tiene métodos que harán la conversión por ti.

Echemos un vistazo a algunos ejemplos más creados usando los tipos primitivos y de colección. Para la mayoría de nuestras pipelines, utilizará exclusivamente una familia de tipos y, por lo tanto, podrá reducir algunas de las tablas en sus clases mediante la importación de todos los métodos de las clases Writables o Avros en su clase:

```java

// Import all of the PType factory methods from Avros
import static org.apache.crunch.types.avro.Avros.*;

import org.apache.crunch.Pair;
import org.apache.crunch.Tuple3;
import org.apache.crunch.TupleN;

import java.nio.ByteBuffer;
import java.util.Collection;
import java.util.Map;

public class MyPipeline {

  // Common primitive types
  PType<Integer> intType = ints();
  PType<Long> longType = longs();
  PType<Double> doubleType = doubles();
  // Bytes are represented by java.nio.ByteBuffer
  PType<ByteBuffer> bytesType = bytes();

  // A PTableType: using tableOf will return a PTable instead of a
  // PCollection from a parallelDo call.
  PTableType<String, Boolean> tableType = tableOf(strings(), booleans());

  // Pair types: 
  PType<Pair<String, Boolean>> pairType = pairs(strings(), booleans()); 
  PType<Pair<String, Pair<Long, Long>> nestedPairType = pairs(strings(), pairs(longs(), longs()));

  // A triple
  PType<Tuple3<Long, Float, Float>> tripType = trips(longs(), floats(), floats());
  // An arbitrary length tuple-- note that we lose the generic type information
  PType<TupleN> tupleType = tupleN(ints(), ints(), floats(), strings(), strings(), ints());

  // A Collection type
  PType<Collection<Long>> longsType = collections(longs());
  // A Map Type-- note that the keys are always strings, we only specify the value.
  PType<Map<String, Boolean>> mapType = maps(booleans());

  // A Pair of collections
  PType<Pair<Collection<String>, Collection<Long>>> pairColType = pairs(
      collections(strings()),
      collections(longs()));
}
```


Ambas familias de tipos también tienen un método llamado  PType<T> records(Class<T> clazz)  que puede usarse para crear PTypes que soporten el formato de registro común para cada familia de tipos. 

Para la clase WritableTypeFamily, el método records soporta PTypes para implementaciones de la interfaz Writable y para AvroTypeFamily, el método records soporta PTypes para implementaciones de la interfaz IndexedRecord de Avro, que incluye registros genéricos y específicos de Avro:

```java
PType<FooWritable> fwType1 = Writables.records(FooWritable.class);
// The more obvious "writables" method also works.
PType<FooWritable> fwType = Writables.writables(FooWritable.class);

// For a generated Avro class, this works:
PType<Person> personType1 = Avros.records(Person.class);
// As does this:
PType<Person> personType2 = Avros.containers(Person.class); 
// If you only have a schema, you can create a generic type, like this:
org.apache.avro.Schema schema = ...;
PType<Record> avroGenericType = Avros.generics(schema);
```

La clase Avros también tiene un método *reflects*  para crear PTypes para POJOs utilizando el mecanismo de serialización basado en reflexión de Avro. Hay un par de restricciones sobre la estructura del POJO. Primero, debe tener un constructor predeterminado, no-arg. En segundo lugar, todos sus campos deben ser tipos primitivos Avro o tipos de colección que tengan equivalentes Avro, como ArrayList y HAshMap <String, T>. También puede tener matrices de tipos primitivos Avro.

```java
// Declare an inline data type and use it for Crunch serialization
public static class UrlData {
  // The fields don't have to be public, just doing this for the example.
  double curPageRank;
  String[] outboundUrls;

  // Remember: you must have a no-arg constructor. 
  public UrlData() { this(0.0, new String[0]); }

  // The regular constructor
  public UrlData(double pageRank, String[] outboundUrls) {
    this.curPageRank = pageRank;
    this.outboundUrls = outboundUrls;
  }
}

PType<UrlData> urlDataType = Avros.reflects(UrlData.class);
PTableType<String, UrlData> pageRankType = Avros.tableOf(Avros.strings(), urlDataType);
```

La *reflection*  de Avro es una gran manera de definir tipos intermedios para sus tuberías del Crunch; No sólo su lógica es clara y fácil de probar, sino que el hecho de que los datos se escriben como registros de Avro significa que puede utilizar herramientas como Hive y Pig para consultar los resultados intermedios para ayudar en la depuración de fallas de pipeline.

### Leer y escribir datos

Las principales herramientas para trabajar con datos de pipelines en Hadoop incluyen algún tipo asbtracto para trabajar con las clases InputFormat <K, V> y OutputFormat <K, V> definidas en MapReduce APIs. Por ejemplo, Hive incluye *SerDes* y Pig requiere *LoadFuncs* y *StoreFuncs*. Tomemos un momento para explicar qué funcionalidad proporcionan estas abstracciones para los desarrolladores.

La mayoría de los desarrolladores comienzan a usar una de las herramientas de la pipelines para Hadoop porque tienen un problema que requiere unir datos de varios archivos de entrada juntos. Aunque Hadoop proporciona soporte para la lectura de varias instancias de InputFormat en diferentes rutas a través de la clase *MultipleInputs*, MultipleInputs no resuelve algunos de los problemas comunes que los desarrolladores encuentran al leer múltiples entradas.

La mayoría de InputFormats y OutputFormats tienen métodos estáticos que se utilizan para proporcionar información de configuración adicional a una tarea de pipeline. Por ejemplo, FileInputFormat contiene un método static void setMaxInputSplitSize(Job job, long size)  que se puede utilizar para controlar la estrategia un job MapReduce determinado. Otros InputFormats, como MultiInputFormat de Elephant Bird, contienen parámetros de configuración que especifican qué tipo de buffer de protocolo o Thrift registrar el formato debe esperar leer de las rutas de acceso dadas.

Todos estos métodos funcionan estableciendo pares llave-valor en el objeto Configuración asociado con una ejecución de un pipeline de datos. Este es un enfoque simple y estándar para incluir información de configuración, pero nos encontramos con un problema cuando queremos leer de múltiples rutas que requieren diferentes valores para asociarse con la misma clave, como cuando estamos uniendo buffers de protocolo, Thrift records O registros Avro que tienen esquemas diferentes. 

Para poder leer adecuadamente el contenido de cada ruta de entrada, debemos ser capaces de especificar que ciertos pares llave-valor en el objeto Configuración sólo deben aplicarse a ciertas rutas.

Para manejar los diferentes ajustes de configuración necesarios para cada una de nuestras diferentes entradas entre sí, Crunch utiliza ***Source*** para envolver (wraper) un InputFormat y sus pares llave-valor asociados de una manera que puede aislarse de cualquier otra fuente que se utilice en el mismo job incluso si esas Fuentes tienen el mismo InputFormat. 

Y de la misma forma para la salida la interfaz ***Target***  se puede utilizar de la misma manera para envolver un  OutputFormat-Hadoop y sus pares llaves-valor asociados de una manera que puede aislarse de cualquier otra salida de una etapa de canalización.

## Una nota sobre fuentes, objetivos e APIs de Hadoop

Crunch, como Hive and Pig, se desarrolla contra la API org.apache.hadoop.mapreduce, no con la API org.apache.hadoop.mapred más antigua.

Esto significa que Crunch Sources y Targets esperan subclases de las nuevas clases InputFormat y OutputFormat. 

Estas nuevas clases no son compatibles  1:1 con las clases InputFormat y OutputFormat asociadas con las API org.apache.hadoop.mapred. Por lo tanto, tengamos en cuenta esta diferencia al considerar el uso de InputFormats y OutputFormats existentes con las fuentes y objetivos de Crunch.

##  Patrones de Procesamiento en Crunch

Aqui se describe los diversos patrones de procesamiento de datos implementados en las API de biblioteca de Crunch, que se encuentran en el paquete org.apache.crunch.lib.

### groupByKey

La mayoría de los patrones de procesamiento de datos descritos se basan en el método *groupByKey* de *PTable*, que controla cómo se shufflean y agregan los al  motor de ejecución subyacente. El método groupByKey tiene tres sabores en la interfaz PTable:

1. groupByKey (): Una operación aleatoria simple, donde el número de particiones de los datos será determinado por el planificador de Crunch basado en el tamaño estimado de los datos de entrada,
2. groupByKey (int numPartitions): Una operación aleatoria donde el número de particiones es proporcionado explícitamente por el desarrollador basado en algún conocimiento de los datos y la operación realizada.
3. groupByKey (Opciones GroupingOptions): Operaciones complejas aleatorias que requieren particiones y comparadores personalizados.

La clase ***GroupingOptions*** permite a los desarrolladores ejercer un control preciso sobre cómo los datos son particionados, ordenados y agrupados por el motor de ejecución subyacente. 

Crunch se desarrolló originalmente en la parte superior de MapReduce, por lo que las API GroupingOptions esperan instancias de las clases Partitioner y RawComparator de Hadoop para soportar particiones y clases. 

Dicho esto, Crunch tiene adaptadores en su lugar para que estas mismas clases también se pueden utilizar con otros motores de ejecución, como Apache Spark, sin una reescritura.

La clase GroupingOptions es inmutable; Para crear uno nuevo usa la implementación *GroupingOptions.Builder*.

```java
GroupingOptions opts = GroupingOptions.builder()
      .groupingComparatorClass(MyGroupingComparator.class)
      .sortComparatorClass(MySortingComparator.class)
      .partitionerClass(MyPartitioner.class)
      .numReducers(N)
      .conf("key", "value")
      .conf("other key", "other value")
      .build();
  PTable<String, Long> kv = ...;
  PGroupedTable<String, Long> kv.groupByKey(opts);
```

GroupingOptions al igual que Sources and Targets puede tener opciones de configuración adicionales especificadas que sólo se aplicarán al trabajo que realmente ejecuta esa fase de datos pipeline.

### combineValues


Llamar a uno de los métodos groupByKey en PTable devuelve una instancia de la interfaz PGroupedTable. 

PGroupedTable proporciona una combineValues que se puede utilizar para señalar al planificador que queremos realizar agregaciones asociativas en nuestros datos antes y después del shuffle.

Hay dos maneras de utilizar combineValues: puede crear una extensión de la clase base abstracta *CombineFn* o puedes utilizar una instancia de la interfaz de *Aggregator*. De los dos, un agregador es probablemente la manera que usted desea ir; Crunch proporciona una serie de *Aggregators* y son un poco más fáciles de escribir y componer juntos. Vamos a ver algunas agregaciones de ejemplo:

```
  PTable<String, Double> data = ...;

  // Sum the values of the doubles for each key.
  PTable<String, Double> sums = data.groupByKey().combineValues(Aggregators.SUM_DOUBLES());
  // Find the ten largest values for each key.
  PTable<String, Double> maxes = data.groupByKey().combineValues(Aggregators.MAX_DOUBLES(10));

  PTable<String, String> text = ...;
  // Get a random sample of 100 unique elements for each key.
  PTable<String, String> samp = text.groupByKey().combineValues(Aggregators.SAMPLE_UNIQUE_ELEMENTS(100));
```
También podemos usar *Aggregators* juntos para crear agregaciones más complejas, como calcular el promedio de un conjunto de valores:


```
  PTable<String, Double> data = ...;

  // Create an auxillary long that is used to count the number of times each key
  // appears in the data set.
  PTable<String, Pair<Double, Long>> c = data.mapValues(
    new MapFn<Double, Pair<Double, Long>>() { Pair<Double, Long> map(Double d) { return Pair.of(d, 1L); } },
    pairs(doubles(), longs()));

  // Aggregate the data, using a pair of aggregators: one to sum the doubles, and the other
  // to sum the auxillary longs that are the counts.
  PTable<String, Pair<Double, Long>> agg = c.groupByKey().combineValues(
      Aggregators.pairAggregator(Aggregators.SUM_DOUBLES(), Aggregators.SUM_LONGS()));

  // Compute the average by dividing the sum of the doubles by the sum of the longs.
  PTable<String, Double> avg = agg.mapValues(new MapFn<Pair<Double, Long>, Double>() {
    Double map(Pair<Double, Long> p) { return p.first() / p.second(); }
  }, doubles());
```

### Simple Aggregations

Muchos de los patrones de agregación más comunes en Crunch se proporcionan como métodos en la interfaz de PCollection, incluyendo *count*, *max*, *min*, Y *length*. 


Sin embargo, las implementaciones de estos métodos se encuentran en la clase de biblioteca Aggregate. Los métodos de la clase Aggregate exponen algunas opciones adicionales que puede utilizar para realizar agregaciones, como controlar el nivel de paralelismo para las operaciones:

```java
  PCollection<String> data = ...;
  PTable<String, Long> cnt1 = data.count();
  PTable<String, Long> cnt2 = Aggregate.count(data, 10); // use 10 reducers
```

PTable tiene métodos de agregación adicionales, superior e inferior, que se pueden utilizar para obtener la mayoría (o menos) que ocurren frecuentemente pares clave-valor en el PTable basado en el valor, que debe implementar *Comparable*. Para contar todos los elementos de un conjunto y luego obtener los 20 elementos más frecuentes, se ejecuta:

```java
  PCollection<String> data = ...;
  PTable<String, Long> top = data.count().top(20);
```

### Joining Data

Los Joins en Crunch se basan en claves de igual valor en diferentes PTables. los joins también han evolucionado mucho en Crunch durante la vida del proyecto. La API Join proporciona métodos sencillos para realizar equijoins, left joins, right joins, and full joins, pero las combinaciones modernas de Crunch se realizan generalmente mediante una implementación explícita de la interfaz de JoinStrategy, que tiene soporte para el mismo conjunto de joins que puede usar En herramientas como Apache Hive y Apache Pig.

Todos los algoritmos discutidos a continuación implementan la interfaz JoinStrategy que define un único método de unión:

```java
  PTable<K, V1> one = ...;
  PTable<K, V2> two = ...;
  JoinStrategy<K, V1, V2> strategy = ...;
  PTable<K, Pair<V1, V2>> joined = strategy.join(one, two, JoinType);
````

El enumerado **JoinType** determina qué tipo de combinación se aplica: inner, outer, left, right, or full. En general, la menor de las dos entradas debe ser el argumento más a la izquierda del método join.

Tenga en cuenta que los valores de los elementos que puede unir no deberían ser nulos. Los algoritmos de combinación en Crunch utilizan null como un marcador de posición para representar que no hay valores para una clave dada en un PCollection, por lo que unir PTables que contienen valores nulos puede tener resultados sorprendentes. **Utilizar un valor ficticio no nulo en su PCollections es una buena idea en general.**


### Reduce-side Joins

Las combinaciones de lado reducido son manejadas por *DefaultJoinStrategy*. Reduce-side Joins son el tipo más simple y más robusto de uniones en Hadoop; Las llaves de las dos entradas se mezclan en los reductores, donde los valores de la menor de las dos colecciones se recogen y luego se transmiten por los valores de la mayor de las dos colecciones. Puede controlar el número de reductores que se utiliza para realizar la combinación pasando un argumento entero al constructor *DefaultJoinStrategy*.

### Map-side Joins

Map-side Joins son manejadas por *MapsideJoinStrategy*. Map-side joins requieren que la más pequeña de las dos tablas de entrada se cargue en la memoria en las tareas del clúster, por lo que es necesario que al menos una de las tablas sea relativamente pequeña para que pueda encajar cómodamente en la memoria dentro de cada tarea .

Durante mucho tiempo, *MapsideJoinStrategy* se diferenció del resto de las implementaciones de JoinStrategy en que el argumento de la izquierda se pretendía ser más grande que el de la derecha, ya que el lado derecho PTable se cargó en la memoria. Desde Crunch 0.10.0 / 0.8.3, hemos *depreciado* el antiguo constructor de MapsideJoinStrategy que tenía los tamaños invertidos y recomendamos que utilice el método de fábrica MapsideJoinStrategy.create(), que devuelve una implementación de la *MapsideJoinStrategy* en la que el lado izquierdo PTable Se carga en la memoria en lugar del lado derecho PTable.

### Sharded Joins

Muchas combinaciones distribuidas tienen datos sesgados que pueden provocar fallas en las combinaciones regulares de lado reducido debido a problemas de falta de memoria en las particiones que contienen las claves con mayor cardinalidad. Para manejar estos problemas de sesgo, Crunch tiene la *ShardedJoinStrategy* que permite a los desarrolladores cortar cada clave a varios reducers lo que evita que unos pocos reducers se sobrecargen con los valores de las llaves sesgadas a cambio de enviar más datos a través del cable.

Para problemas con issues de desviación significativos, ShardedJoinStrategy puede mejorar significativamente el rendimiento.

### Bloom Filter Joins

Por último, pero no por ello menos importante, el BloomFilterJoinStrategy crea un filtro de bloom en la tabla lateral izquierda que se utiliza para filtrar el contenido de la tabla lateral derecha para eliminar entradas de la tabla lateral (más grande) que no tienen esperanza de Siendo unidos a valores en la tabla lateral izquierda. Esto es útil en situaciones en las que la tabla del lado izquierdo es demasiado grande para caber en la memoria en las tareas del trabajo, pero sigue siendo significativamente menor que la mesa lateral derecha y sabemos que la gran mayoría de las teclas En la tabla del lado derecho no coincidirá con las teclas en el lado izquierdo de la tabla.

## Crunch para HBase

Crunch es una excelente plataforma para la creación de pipelines que implican el procesamiento de datos de las tablas de HBase. Debido a los esquemas flexibles de Crunch para PCollections y PTables, puede escribir pipelines que operan directamente en clases de API de HBase como Put, KeyValue y Result.

Asegúrese de que la versión de Crunch que está utilizando sea compatible con la versión de HBase que está ejecutando. Las versiones 0.8.x Crunch y anteriores están desarrolladas contra HBase 0.94.x, mientras que la versión 0.10.0 y posterior se desarrollan contra HBase 0.96. 

Hubo un pequeño número de cambios incompatibles hacia atrás entre HBase 0.94 y 0.96 que se reflejan en las API de Crunch para trabajar con HBase. El más importante de ellos es que en HBase 0.96, las clases Put, KeyValue y Result de HBase ya no implementan la interfaz Writable. Para admitir el trabajo con estos tipos en Crunch 0.10.0, hemos añadido la clase HBaseTypes que tiene métodos Factories para crear PTypes que serializan las clases de cliente de HBase a bytes para que puedan seguir utilizándose como parte de las pipelines de MapReduce.

Crunch soporta trabajar con datos de HBase de dos maneras. Las clases *HBaseSourceTarget* y *HBaseTarget* soportan la lectura y escritura de datos en las tablas de HBase directamente. Las clases HFileSource y HFileTarget soportan la lectura y escritura de datos en hfiles, que son el formato de archivo subyacente para HBase. HFileSource y HFileTarget se pueden utilizar para leer y escribir datos en hfiles directamente, lo que es mucho más rápido que pasar por las API de HBase y puede utilizarse para realizar una carga masiva eficiente de datos en tablas de HBase. Consulte los métodos de utilidad en la clase HFileUtils para obtener más detalles sobre cómo trabajar con PCollections contra hfiles.

### Manejo de la Ejecucion del Pipeline

Crunch utiliza un modelo de ejecución lazy. No se ejecutan trabajos ni se crean salidas hasta que el usuario invoca explícitamente uno de los métodos en la interfaz de Pipeline que controla la planificación y la ejecución del trabajo.

El método más sencillo de estos métodos es el método PipelineResult (), que analiza el grafo actual de las salidas PCollections y Target y presenta un plan para asegurar que cada una de las salidas se crea y luego ejecuta, volviendo sólo cuando se completan los trabajos . 

El *PipelineResult* devuelto por el método run contiene información sobre lo que se ejecutó, incluyendo el número de trabajos que se ejecutaron durante la ejecución del pipeline y los valores de los contadores Hadoop para cada una de esas etapas a través de las clases de componente *StageResult*.

El último método que debe llamarse en cualquier ejecución de pipeline Crunch es el método  done() de la interfaz PipelineResult Pipeline. El método finalizado garantizará que las salidas restantes que todavía no se hayan creado se ejecuten a través de la ejecución y limpiarán los directorios temporales que Crunch crea durante las ejecuciones para contener información de trabajos serializados y salidas intermedias.


Crunch también permite a los desarrolladores ejecutar un control más fino sobre la ejecución de la pipeline a través del método *PipelineExecution* de pipeline runAsync(). El método **runAsync** es una versión sin bloqueo del método run que devuelve una instancia de PipelineExecution que se puede utilizar para supervisar el pipeline Crunch que se está ejecutando actualmente. 

El objeto PipelineExecution también es útil para debugging pipelines de Crunch al visualizar el plan de ejecución de Crunch en formato DOT mediante su método String getPlanDotFile(). PipelineExection implementa el ListenableFuture de Guava, para que pueda adjuntar manejadores que serán llamados cuando su pipeline termine de ejecutarse.

La mayor parte del trabajo del planificador de Crunch implica decidir dónde y cuándo almacenar en memoria caché los productos intermedios entre las diferentes etapas del pipeline. 

Si descubre que el planificador de Crunch no está decidiendo de forma óptima dónde dividir dos trabajos dependientes, puedes controlar qué PCollections se utiliza como puntos de división en un pipeline mediante los métodos Iterable <T> materialize () y PCollection <T> cache() Disponible en la interfaz PCollection.

Si el planificador detecta una PCollection materializada o en caché a lo largo del camino entre dos trabajos, el planificador preferirá el PCollection ya en caché a su propia elección.

La implementación de materializar y caché varían ligeramente entre las canalizaciones de ejecución basadas en MapReduce y Spark de una manera que se explica en la sección posterior de la guía.

## Las diferentes implementaciones de pipelines 

El MRPipeline es la implementación más antigua de la interfaz de Pipeline y compila y ejecuta el DAG de PCollections en una serie de trabajos de MapReduce. MRPipeline tiene tres constructores que se utilizan comúnmente:

* MRPipeline(Class<?> jarClass) toma una referencia de clase que se utiliza para identificar el archivo jar que contiene DoFns y cualquier clase asociada que se debe pasar al clúster para ejecutar el job.
* MRPipeline(Class<?> jarClass, Configuration conf) es como el constructor de clase, pero le permite declarar una instancia de configuración para utilizarla como base del trabajo. Este es un buen constructor para utilizar en general, especialmente cuando se utiliza la interfaz de herramientas de Hadoop y la clase base configurada para declarar los principales métodos para ejecutar su pipeline.
* MRPipeline(Class<?> jarClass, String appName, Configuration conf) le permite declarar un prefijo común (dado por el argumento appName) para todos los trabajos MapReduce que se ejecutarán como parte de esta canalización de datos. Esto puede facilitar la identificación de sus trabajos en JobTracker o ApplicationMaster.

Hay una serie de parámetros de configuración útiles que se pueden utilizar para ajustar el comportamiento de MRPipeline que debe tener en cuenta:

| ﻿Name                             | Type    | Usage Notes                                                                                                                                                                                                       |
|----------------------------------|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| crunch.tmp.dir                   | string  | The base directory for Crunch to use when it writes temporary outputs for a job. Default is /tmp                                                                                                                  |
| crunch.debug                     | boolean | Enables debug mode which traps and logs any runtime exceptions and input data. Can also be enabled via enableDebug() on the Pipeline interface. False by default because it introduces a fair amount of overhead. |
| crunch.job.name.max.stack.length | integer | Controls the length of the name of the job that Crunch generates for each phase of the pipeline. Default is 60 chars.                                                                                             |
| crunch.log.job.progress          | boolean | If true Crunch will print the Map %P Reduce %P data to stdout as the jobs run. False by default.f true Crunch will print the Map %P Reduce %P data to stdout as  true Crunch will print the Map %Ptrue            |
| crunch.disable.combine.file      | boolean | By default Crunch will use CombineFileInputFormat for subclasses of FileInputFormat. This can be disabled on a per-source basis or globally.                                                                      |
| crunch.combine.file.block.size   | integer | The block size to use for the CombineFileInputFormat. Default is the dfs.block.size for the cluster.                                                                                                              |
| crunch.max.running.jobs          | integer | Controls the maximum number of MapReduce jobs that will be executed simultaneously. Default is 5.                                                                                                                 |


### SparkPipeline

El SparkPipeline es la implementación más reciente de la interfaz de Pipeline, y se agregó en Crunch 0.10.0. Tiene dos constructores por defecto:

1. SparkPipeline(String sparkConnection, String appName) que toma una cadena de conexión Spark, que es de la forma local [numThreads] para modo local o master: puerto para un clúster Spark. Este constructor creará su propia instancia JavaSparkContext para controlar la canalización Spark que ejecuta.
2. SparkPipeline(JavaSparkContext contexto, String appName) utilizará un JavaSparkContext dado directamente, en lugar de crear su propia.

Tenga en cuenta que JavaSparkContext crea su propia instancia de configuración en sí, y que actualmente no es una forma de establecerlo durante el inicio de la canalización. Esperamos y esperamos que esto será corregido en una versión futura de Spark.

Crunch delega gran parte de la ejecución del pipeline a Spark y realiza relativamente poco de las tareas de planificación de pipeline, con algunas excepciones fundamentales:

*Multiple inputs:* Crunch realiza el trabajo de abstraer la combinación del formato de entrada y los parámetros de configuración de una manera que hace que sea fácil trabajar con múltiples entradas en una tubería, incluso si son del mismo tipo y tendrían parámetros conf conflictivos , Si estuviera intentando unir dos archivos avro con esquemas diferentes, los parámetros de conf del esquema normalmente estarían en conflicto entre sí.)

*Multiple outputs:*  Spark no tiene un concepto de múltiples salidas; Cuando se escribe un conjunto de datos en disco, la tubería que crea ese conjunto de datos se ejecuta inmediatamente. Esto significa que usted necesita ser un poco listo sobre el almacenamiento en caché etapas intermedias por lo que no terminan volver a ejecutar un pipeline largo y grande varias veces con el fin de escribir un par de salidas. Crunch hace eso por ti, junto con el mismo formato de salida y el ajuste de parámetros que obtienes para múltiples entradas.

*Data serialization:* Spark utiliza serialización Java o Kryo, con un manejo especializado para Writables. Kryo no maneja bien los registros de Avro, por lo que la serialización de Crunch convierte esos registros en arrays de bytes para que no rompan sus tuberías.

*Checkpointing:* Target.WriteMode permite el chequeo de datos a través de las ejecuciones de pipeline de Spark. Esto es útil durante el desarrollo activo de la tubería, ya que la mayoría de los fallos que se producen al crear líneas de datos están en funciones definidas por el usuario que se encuentran con entradas inesperadas.

Tengamos en cuenta que el método de caché (Opciones de CacheOptions) en la interfaz PCollection expone el mismo nivel de control sobre la caché de RDD que proporciona la API de Spark, en términos de memoria vs. disco y serializado vs. datos deserializados. 

Aunque estos mismos métodos existen para la implementación de MRPipleine, la única estrategia de almacenamiento en caché que se aplica es el disco de MapReduce y la caché de serialización, las otras opciones se ignoran.

Es importante que llame al método done() en el SparkPipeline al final de su trabajo, lo que limpiará el JavaSparkContext. Usted puede obtener fracasos extraños e impredecibles si no lo hace. 

Como la implementación más reciente de Pipeline, debe esperar que SparkPipeline sea un poco duro alrededor de los bordes y no pueda manejar todos los casos de uso que MRPipeline puede manejar, aunque la comunidad de Crunch está trabajando activamente para asegurar la compatibilidad completa entre las dos implementaciones.

### MemPipeline

La implementación de MemPipeline tiene algunas propiedades interesantes. En primer lugar, a diferencia de MRPipeline, MemPipeline es un singleton; Usted no crea un MemPipeline, usted apenas consigue una referencia a él vía el método estático MemPipeline.getInstance(). 

En segundo lugar, todas las operaciones en el MemPipeline se ejecutan completamente en la memoria, no hay serialización de los datos en el disco por defecto, y el uso de PType es bastante mínimo. Esto tiene tanto ventajas como inconvenientes; 

como bueno digamos que los funcionamientos de MemPipeline son extremadamente rápidos y son una buena manera de probar la lógica interna de sus operaciones de DoFns y de pipeline. 

En el lado negativo, MemPipeline no ejercerá el código de serialización, por lo que es posible que funcione un MemPipeline bien mientras una ejecución de clúster real usando MRPipeline o SparkPipeline falle debido a algún problema de serialización de datos. Como regla, siempre debe tener pruebas de integración que ejecuten MapReduce o Spark en modo local para que pueda probar estos problemas.

Puede agregar datos al soporte de PCollections por MemPipeline de dos maneras principales. Primero, puede utilizar los métodos de lectura read(Source<T> src) de Pipeline, como lo haría para MRPipeline o SparkPipeline. 

MemPipeline requiere que cualquier source de entrada implemente la interfaz *ReadableSource* para que los datos que contienen se puedan leer en la memoria. También puede aprovechar un par de métodos factory útiles en MemPipeline que se pueden utilizar para crear PCollections de Java Iterables:

```java
 PCollection<String> data = MemPipeline.collectionOf("a", "b", "c");
  PCollection<String> typedData = MemPipeline.typedCollectionOf(Avros.strings(), "a", "b", "c");

  List<Pair<String, Long>> list = ImmutableList.of(Pair.of("a", 1L), Pair.of("b", 2L));
  PTable<String, Long> table = MemPipeline.tableOf(list);
  PTable<String, Long> typedTable = MemPipeline.typedTableOf(
      Writables.tableOf(Writables.strings(), Writables.longs()), list);
 ```     

Como puede ver, puede crear colecciones tipeadas o sin tipo, dependiendo de si proporciona o no un PType para ser utilizado con la PCollection que se cree. 

En general, proporcionar un PType es una buena idea, principalmente porque muchos de los métodos de Crunch API asumen que PCollections tiene un PType válido y no nulo disponible para trabajar.

En el lado de salida, hay algún soporte limitado para escribir el contenido de un PCollection en memoria o PTable en un archivo Avro, un archivo de Secuencia o un archivo de texto, pero el soporte aquí no es tan robusto como el soporte En el lado de lectura porque Crunch no tiene una interfaz equivalente de WritableTarget que coincida con la interfaz ReadableSourc <T> en el lado de lectura. 

A menudo, la mejor manera de verificar que el contenido de su pipeline es correcto es utilizando el método materialize() para obtener una referencia al contenido de la colección en memoria y luego verificarlos directamente, sin escribirlos en el disco.

## Unit Testing Pipelines

Para usar pipelines de datos en producción, las pruebas unitarias son una necesidad absoluta. La implementación de la interfaz Pipeline de MemPipeline tiene varias herramientas para ayudar a los desarrolladores a crear pruebas unitarias efectivas.

### Unit Testing DoFns

Muchas de las implementaciones DoFn, como MapFn y FilterFn, son muy fáciles de probar, ya que aceptan una sola entrada y devuelven una sola salida. Para DoFns de uso general, necesitamos una instancia de la interfaz *Emitter* que podamos pasar al método de proceso de DoFn y luego leer en los valores que son escritos por la función. 

La compatibilidad con este patrón es proporcionada por la clase *InMemoryEmitter*, que tiene un método de List<T> getOutput() que se puede utilizar para leer los valores que se pasaron a la instancia Emitter por una instancia DoFn:

```java
@Test
public void testToUpperCaseFn() {
  InMemoryEmitter<String> emitter = new InMemoryEmitter<String>();
  new ToUpperCaseFn().process("input", emitter);
  assertEquals(ImmutableList.of("INPUT"), emitter.getOutput());
}
```

### Testing Complex DoFns and Pipelines

Muchos de los DoFns que escribimos implican un procesamiento más complejo que requiere que nuestro DoFn se inicialice y se limpie, o que defina Counters que usamos para rastrear las entradas que recibimos.

Para garantizar que nuestros DoFns funcionen adecuadamente a lo largo de todo su ciclo de vida, lo mejor es utilizar la implementación de MemPipeline para crear instancias en memoria de PCollections y PTables que contengan una pequeña cantidad de datos de prueba y aplicar DoFns a esas PCollections para probar su Funcionalidad.

Podemos recuperar fácilmente el contenido de cualquier PCollection en memoria llamando a su método Iterable<T> materialize(), que volverá inmediatamente. También podemos rastrear los valores de los contadores que se llamaron como los DoFns se ejecutaron contra los datos de prueba llamando al método estático getCounters() en la instancia de MemPipeline y restablecer los contadores entre las ejecuciones de prueba llamando al método static clearCounters():

```java
public static class UpperCaseWithCounterFn extends DoFn<String, String> {
  @Override
  public void process(String input, Emitter<T> emitter) {
    String upper = input.toUpperCase();
    if (!upper.equals(input)) {
      increment("UpperCase", "modified");
    }
    emitter.emit(upper);
  }
}

@Before
public void setUp() throws Exception {
  MemPipeline.clearCounters();
}

@Test
public void testToUpperCase_WithPipeline() {
  PCollection<String> inputStrings = MemPipeline.collectionOf("a", "B", "c");
  PCollection<String> upperCaseStrings = inputStrings.parallelDo(new UpperCaseWithCounterFn(), Writables.strings());
  assertEquals(ImmutableList.of("A", "B", "C"), Lists.newArrayList(upperCaseStrings.materialize()));
  assertEquals(2L, MemPipeline.getCounters().findCounter("UpperCase", "modified").getValue());
}
```

### Designing Testable Data Pipelines

De la misma manera que tratamos de escribir código testeable, queremos asegurarnos de que nuestros data pipelines estén escritos de una manera que los haga fáciles de probar. 

En general, debe tratar de dividir pipelines complejas en una serie de llamadas de función que realizan un pequeño conjunto de operaciones en PCollections de entrada y devolver una o más PCollections como resultado.

Esto facilita el intercambio en diferentes implementaciones de PCollection para pruebas y ejecuciones de producción. Veamos un ejemplo que calcula una iteración del algoritmo PageRank que se toma de una de las pruebas de integración de Crunch:

```java

// Each entry in the PTable represents a URL and its associated data for PageRank computations.
public static PTable<String, PageRankData> pageRank(PTable<String, PageRankData> input, final float d) {
  PTypeFamily ptf = input.getTypeFamily();

  // Compute the outbound page rank from each of the input pages.
  PTable<String, Float> outbound = input.parallelDo(new DoFn<Pair<String, PageRankData>, Pair<String, Float>>() {
    @Override
     public void process(Pair<String, PageRankData> input, Emitter<Pair<String, Float>> emitter) {
     PageRankData prd = input.second();
      for (String link : prd.urls) {
        emitter.emit(Pair.of(link, prd.propagatedScore()));
      }
    }
  }, ptf.tableOf(ptf.strings(), ptf.floats()));

  // Update the PageRank for each URL.
  return input.cogroup(outbound).mapValues(
      new MapFn<Pair<Collection<PageRankData>, Collection<Float>>, PageRankData>() {
        @Override
        public PageRankData map(Pair<Collection<PageRankData>, Collection<Float>> input) {
          PageRankData prd = Iterables.getOnlyElement(input.first());
          Collection<Float> propagatedScores = input.second();
          float sum = 0.0f;
          for (Float s : propagatedScores) {
            sum += s;
          }
          return prd.next(d + (1.0f - d) * sum);
        }
      }, input.getValueType());
}
```

Al incorporar nuestra lógica de negocio dentro de un método estático que opera en PTables, podemos probar fácilmente nuestros cálculos de PageRank que combinan DoFns personalizado con la operación integrada de coguptos de Crunch mediante la implementación de MemPipeline para crear conjuntos de datos de prueba que podemos verificar fácilmente mediante Y, a continuación, esta misma lógica se puede ejecutar en un conjunto de datos distribuidos utilizando las implementaciones MRPipeline o SparkPipeline.

### Pipeline execution plan visualizations

Crunch proporciona herramientas para visualizar el plan de ejecución del pipeline. El método *String getPlanDotFile()* de *PipelineExecution* devuelve una visualización de formato DOT del plan de exacción. Además, si la carpeta de salida está configurada, Crunch guardará el  diagrama dotfile ejecución del pipeline:

```java
    Configuration conf =...;     
    String dotfileDir =...;

    // Set DOT files out put folder path
    DotfileUtills.setPipelineDotfileOutputDir(conf, dotfileDir);
```

Detalles adicionales del plan de ejecución Crunch se pueden exponer al habilitar el modo de depuración de ficheros como:

```java
// Requires the output folder to be set.
    DotfileUtills.enableDebugDotfiles(conf);
```

Esto producirá (y ahorrará) 4 diagramas adicionales que visualizarán las etapas internas del plan de ejecución del pipeline. Dichos diagramas son el pineage PCollection, la base del pipeline y los split graphs y la representación del nodo run-time (RTNode).

(Nota: El modo de depuración requiere que se establezca la carpeta de salida.)
