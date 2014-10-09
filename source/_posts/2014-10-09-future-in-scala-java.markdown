---
layout: post
title: "Futures in Scala and Java"
date: 2014-10-09 11:22:14 +0200
comments: true
categories: [java,scala,reactive,microservices]

keywords: "java, scala, futures, observable, functional, programming"
description: "Brief comparison between Java 8 futures and Scala futures"
---

Cuando se recibe una petición, por ejemplo una llamada a un API, es frecuente que para atenderla
haya que que recabar información de múltiples orígenes. Estos orígenes pueden ser directamente un
repositorio de información (un acceso JDBC), o bien, un servicio interno independiente
que gestiona una determinada funcionalidad (microservice).
Así mismo, algunas de estas informaciones serán independientes
unas de otras, pero habrá casos en las que existan dependencias entre ellas.
Por otro lado, sería muy
interesante que no se bloquearan innecesariamente recursos del sistema (reactive), que son siempre muy escasos.

Por lo tanto, el sistema más eficiente sería aquel:

  - que permitiera paralelizar todas las operaciones que fuera posible
  - que permitiera combinar los resultados de operaciones intermedias para obtener el resultado final
  - que evitara bloquear ningún recurso de manera innecesaria

En Scala esto es algo inherente al ADN del lenguaje, los _elementos básicos_ del lenguaje proporcionan lo
necesario, pero y en Java?

Supongamos un sistema de Gamificación en el que el API que devuelve el perfil del jugador proporciona la
siguiente información:

  - Datos básicos de perfil (nickname, nivel)
  - Posición en los rankings
  - Última medalla conseguida en su nivel actual

Por ejemplo, en este caso: los datos básicos del perfil y la posición en el ranking son operaciones independientes que podrían lanzarse
sobre los correspondientes servicios, y la identificación de la última medalla conseguida depende de conocer
previamente el nivel último nivel del usuario.

## Scala Flavour
Scala ofrece las __Futures__ como mecanismo para la parelización de acciones, y las operaciones
__flatMap__ y __map__ para la combinación de las mismas dentro del propio lenguaje.

Las siguientes __Futures__ modelizan las operaciones a realizar:

  - Recuperación de la posición en el Ranking [fBasicProfile]
  - Recuperación del perfil básico [fRanking]
  - Recuperación de la última medalla del nivel actual [fLastMedalInLevel], que depende del valor devuelto por [fBasicProfile]

``` scala Futures for Scala https://github.com/enqae/futuresComposition/blob/master/src/main/scala/TheFutures.scala
def fBasicProfile =
  (user: String) =>
        Future {
          doAction(s"Select BasicProfile ($user)")
          "basicProfile:MoonWalker"
        }

def fRanking =
  (user: String) =>
        Future {
          doAction(s"Select Ranking Position ($user)")
          "125"
        }

def fLastMedalInLevel =
  (basicProfile: String) =>
        Future {
          doAction(s"Select LastMedalInLevel ($basicProfile)")
          if (basicProfile.endsWith("MoonWalker"))
              "MoonConquest" else "NoMedalYet"
        }
```

<!-- more -->

### flatMap y map
El modo más básico de combinar estas _Futures_ con las operaciones flatMap/map es el siguiente:

``` scala flatMap/map Combination https://github.com/enqae/futuresComposition/blob/master/src/main/scala/Example_01_FuturesBasicFlatMap.scala
def findFullProfile(user: String): Future[String] = {

  val fRankingForUser = fRanking(user)
  val fbasicProfileForUser = fBasicProfile(user)

  fRankingForUser.flatMap(
      basicProfile =>
        fbasicProfileForUser.flatMap(
            ranking =>
              fLastMedalInLevel(basicProfile).map(
                    lastMedal =>
                       s"$basicProfile;$ranking;$lastMedal"
                                                 )
                                     )
                          )
}
```

Simple, aunque no _escalable_. Si hubiera un cuarto servicio al que acceder este mecanismo se volvería inmanejable.

### for comprenhensions
Para solucionar esto Scala ofrece una sintaxis alternativa, las __for comprenhension__, que dejan al
compilador la generación de esta estructura básica. Con una _for comprenhension_ lo anterior quedaría como:

``` scala for comprenhension Combination https://github.com/enqae/futuresComposition/blob/master/src/main/scala/Example_00_FuturesForComprenhension.scala
def findFullProfile(user: String): Future[String] = {

  val fRankingForUser = fRanking(user)
  val fbasicProfileForUser = fBasicProfile(user)

  for {

       ranking <- fRankingForUser
       basicProfile <- fbasicProfileForUser
       lastMedal <- fLastMedalInLevel(basicProfile)

      } yield s"$basicProfile;$ranking;$lastMedal"
}
```

Mucho más simple, y sobre todo _escalable_. La inclusión de una petición a un nuevo servicio no altera
significativamente la estructura del programa.

### launching y resultados
El proceso de lanzamiento de estas peticiones tendría la forma:

``` scala launching process https://github.com/enqae/futuresComposition/blob/master/src/main/scala/Example_00_Futures.scala

print("StartingMainProcessing Action")

findFullProfile("JaneNone")
      .map {
            fp => {
                print("EndingFullProfileProcessing Action")
                println(s"FullProfile: $fp")
                  }
           }

print("EndingMainProcessing Action")

// Just waiting all work to be done
waitFor(5)
```

Si miramos la ejecución de estos procesos obtendríamos algo similar a lo siguiente:

``` console request API execution
(1- launching)
[TH]. 39 ... StartingMainProcessing Action  
[TH]. 40 ... Select BasicProfile (JaneNone)
[TH]. 32 ... Select Ranking Position (JaneNone)
[TH]. 39 ... EndingMainProcessing Action

(2- parallel execution)
[TH]. 32 ... Select Ranking Position (JaneNone) - Step[1]
[TH]. 40 ... Select BasicProfile (JaneNone) - Step[1]
[TH]. 32 ... Select Ranking Position (JaneNone) - Step[2]
[TH]. 32 ... Select Ranking Position (JaneNone) - Step[3]
[TH]. 40 ... Select BasicProfile (JaneNone) - Step[2]
[TH]. 32 ... Select Ranking Position (JaneNone) - Step[4]
[TH]. 32 ... Select Ranking Position (JaneNone) - Step[5]
[TH]. 40 ... Select BasicProfile (JaneNone) - Step[3]
[TH]. 40 ... Select BasicProfile (JaneNone) - Step[4]
[TH]. 40 ... Select BasicProfile (JaneNone) - Step[5]

(3- combination)
[TH]. 32 ... Select Last...Level (basicP...Walker)
[TH]. 32 ... Select Last...Level (basicP...Walker) - Step[1]
[TH]. 32 ... Select Last...Level (basicP...Walker) - Step[2]
[TH]. 32 ... Select Last...Level (basicP...Walker) - Step[3]
[TH]. 32 ... Select Last...Level (basicP...Walker) - Step[4]
[TH]. 32 ... Select Last...Level (basicP...Walker) - Step[5]

(4- results available)
[TH]. 32 ... EndingFullProfileProcessing Action
FullProfile: basicProfile:MoonWalker;125;MoonConquest
```

  - (1 - launching)
    * el proceso de la petición se inicia en el Thread-39
    * se lanzan las futures [fBasicProfileForUser] que comienza a ejecutarse en el Thread-40 y
    [fRankingForUser] que comienza a ejecutarse en el Thread-32. Ambas de forma paralela.
    * el proceso de lanzamiento concluye sin efectuar ningún bloqueo

  - (2 - parallel execution)
    * ambas futures continuan ejecutándose hasta que hay un resultado disponible

  - (3 - combination)
    * el resultado de [fBasicProfileForUser] está disponible y puede utilizarse para solicitar el
    cálculo de [fLastMedalInLevel]

  - (4 - results available)
    * los resultados de las tres están disponibles y es posible retornar la respuesta

Por lo tanto, se solicitan las informaciones de forma paralela, no hay bloqueos y se pueden combinar
los resultados intermedios.

La operación ```waitFor``` es simplemente a efectos del ejemplo. En un caso real estaríamos utilizando un
framework adecuado, como por ejemplo Play.

### async block
Uno de los aspectos que más persigue Scala es en ofrecer mecanismos que permitan simplificar la paralelización
de operaciones, y en este sentido, en la próxima versión se incluirá una simplicación nueva: los bloques [```async```](http://docs.scala-lang.org/sips/pending/async.html).

Utilizando bloques ```async``` quedaría de la forma:

``` scala async Block https://github.com/enqae/futuresComposition/blob/master/src/main/scala/Example_02_FuturesAsync.scala
val fRankingForUser = fRanking(user)
val fbasicProfileForUser = fBasicProfile(user)

async {
   val ranking = await(fRankingForUser)
   val basicProfile = await(fbasicProfileForUser)
   val lastMedal = await(fLastMedalInLevel(basicProfile))
   s"$basicProfile;$ranking;$lastMedal"
}
```

que es una construcción más _natural_ que la _for comprenhension_ para los procedentes de lenguajes imperativos.


## Java Flavour
En Java las cosas no son exactamente iguales. Hasta Java 7 no existía de forma nativa nada similar a lo ofrecido
por Scala. Existían las _Futures_ pero la extración del resultado siempre implicaba una operación bloqueante: ```get```.

``` java futures Java 7 https://github.com/enqae/futuresComposition/blob/master/src/main/java/Example_00_FutureBlockingGet.java

// BLOCKING Operation
String ranking = fRanking.get();

// BLOCKING Operation
String basicProfile = fBasicProfile.get();
Future<String> fLastMedalInLevel = lastMedalInLevel.apply(basicProfile);

// BLOCKING Operation
String lastMedal = fLastMedalInLevel.get();
```

Java 8, ofrece un nuevo tipo de _Future_: __CompletableFuture__.
[_CompletableFuture_](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) es
un _Monad_ (al igual que [_Optional_](/blog/2014/09/20/optionally-no-nulls/) y _Stream_), y ofrece las mismas operaciones básicas que ofrecía Scala:

  - flatMap -> thenCompose/thenComposeAsync
  - map -> thenApply/thenApplyAsync

_CompletableFuture_ tiene un comportamiento no bloqueante y ofrece alrededor de 50 métodos diferentes.

En el caso de Java, las _Futures_ que utilizaríamos serían las siguientes:

``` java Futures for Java
CompletableFuture<String> fRanking =
  CompletableFuture.supplyAsync(() -> {
      Misc.doAction("Select Ranking " + user);
      return "125";
    });

CompletableFuture<String> fBasicProfile =
  CompletableFuture.supplyAsync(() -> {
      Misc.doAction("Select BasicProfile " + user);
      return "BasicProfile:MoonWalker";
    });

Function<String, CompletableFuture<String>> lastMedalInLevel =
    (String basicProfile) ->
            CompletableFuture.supplyAsync(() -> {
                Misc.doAction("Select LastMedalInLevel");
                return
                    basicProfile.endsWith("MoonWalker")
                    ? "MoonConquest"
                    : "NoMedalYet";
            });
```

y utilizando las operaciones ```thenComposeAsync``` y ```thenApplyAsync``` obtendríamos:

``` java flatMap/map Combination https://github.com/enqae/futuresComposition/blob/master/src/main/java/Example_02_FutureComposableFlatMap.java
fRanking
  .thenComposeAsync(
      ranking -> fBasicProfile
          .thenComposeAsync(
              basicProfile ->
                    lastMedalInLevel.apply(basicProfile)
                        .thenApplyAsync(
                            lm -> String.format("%s;%s;%s", basicProfile, ranking, lm)
                                       )
                          )
                  );
```

que es extremadamente parecido a lo que obteníamos en el caso de Scala. Al igual que ocurría en Scala,
esta solución es poco _escalable_, ya que si hubiera que introducir una nueva _Future_, el código
comenzaría a estar excesivamente _anidado_.

Scala solucionaba esto con las _for comprenhension_ y los bloques _async_, y en el caso de Java existe
la operación ```allOf```, que toma una lista de _Futures_ y paraleliza su ejecución. Utilizando ```allOf```
 obtendríamos:

``` java allOf Combination https://github.com/enqae/futuresComposition/blob/master/src/main/java/Example_03_FutureComposableFlatMap.java
CompletableFuture.allOf(fBasicProfile, fRanking)
  .thenComposeAsync(
     na -> {
        // All futures ARE completed at this time
        String basicProfile = fBasicProfile.join();
        String ranking = fRanking.join();

        return lastMedalInLevel.apply(basicProfile)
                  .thenApplyAsync(
                      lm ->
                          String.format("%s;%s;%s", basicProfile, ranking, lm)
                                 );
            }
                  );
```

Más manejable, aunque algo menos que en el caso de Scala.

En ambos casos el proceso de lanzamiento de las operaciones sería de la forma:

``` java launching process
Misc.print("StartingMainProcessing Action");

findFullProfile("JaneNone")
        .thenAcceptAsync(fp -> {
            Misc.print("EndingFullProfileProcessing Action");
            Misc.print(String.format("FullProfile: %s", fp));
        });

Misc.print("End of Main Processing");

// Just waiting all work to be done
Misc.waitFor(5);
```

Muy similar al caso de Scala (```thenAcceptAsync``` es simplemente un método de terminación, que en Java
es necesario para extraer los valores).

La ejecución de estos procesos ofrece un patrón muy similar al anteriormente visto:

``` console request API execution
(1- launching)
[TH]. 336 ... StartingMainProcessing Action
[TH]. 329 ... Select Ranking JaneNone
[TH]. 329 ... Select Ranking JaneNone - Step[1]
[TH]. 337 ... Select BasicProfile JaneNone
[TH]. 337 ... Select BasicProfile JaneNone - Step[1]
[TH]. 336 ... End of Main Processing

(2- parallel execution)
[TH]. 329 ... Select Ranking JaneNone - Step[2]
[TH]. 329 ... Select Ranking JaneNone - Step[3]
[TH]. 337 ... Select BasicProfile JaneNone - Step[2]
[TH]. 337 ... Select BasicProfile JaneNone - Step[3]
[TH]. 329 ... Select Ranking JaneNone - Step[4]
[TH]. 337 ... Select BasicProfile JaneNone - Step[4]
[TH]. 329 ... Select Ranking JaneNone - Step[5]
[TH]. 337 ... Select BasicProfile JaneNone - Step[5]

(3- Combination)
[TH]. 329 ... Select LastMedalInLevel
[TH]. 329 ... Select LastMedalInLevel - Step[1]
[TH]. 329 ... Select LastMedalInLevel - Step[2]
[TH]. 329 ... Select LastMedalInLevel - Step[3]
[TH]. 329 ... Select LastMedalInLevel - Step[4]
[TH]. 329 ... Select LastMedalInLevel - Step[5]

(4- results available)
[TH]. 337 ... EndingFullProfileProcessing Action
[TH]. 337 ... FullProfile: BasicProfile:MoonWalker;125;MoonConquest
```

## resumiendo
_CompletableFuture_ es uno de los elementos introducidos en Java 8 que actualizan la plataforma y la acercan
a las necesidades actuales de las aplicaciones. No es tan potente y flexible como son las soluciones que
ofrece Scala, pero es una gran mejora.

## Nota: Java 7 y 6
Si bien en versiones anteriores a Java 8 no existía en el lenguaje ningún elemento que permitiera realizar este
tipo de operaciones, si hay librerías como por ejemplo [RxJava](https://github.com/ReactiveX/RxJava) que permiten realizar un aproximación similar.
En RxJava quedaría de la forma:

``` java using RxJava https://github.com/enqae/futuresComposition/blob/master/src/main/java/Example_01_RxJavaNonBlockingComposition.java
...
Observable<String> oBasicProfile = Observable.from(fBasicProfile);
Observable<String> oRanking = Observable.from(fRanking);
Observable<String> oLastMedal =
      oBasicProfile
           .flatMap(
                bp ->
                   Observable.from(lastMedalInLevel.apply(bp)));

return
    Observable
      .zip(
        oBasicProfile,
        oRanking,
        oLastMedal,
        (basicProfile, ranking, lastMedal)
            -> String.format("%s;%s;%s", basicProfile, ranking, lastMedal))
      .subscribeOn(Schedulers.computation());
...
```

También con las operaciones paralelizadas, combinadas y no bloqueantes.
