---
layout: post
title: "Akismet y Groovy"
date: 2010-02-03 20:22:37 +0200
comments: true
categories:
- lenguajes
- open source
- software
---

[Akismet](http://akismet.com/) es el servicio de detección de spam que utiliza WordPress, pero no solamente es utilizable desde WordPress, sino que cualquier aplicación puede utilizarlo a través del [API REST](http://akismet.com/development/api/) que ofrece. Akismet no es un servicio gratuito, pero las modalidades de uso que ofrece son bastante asequibles, teniendo en cuenta los ahorros que puede suponer el no tener que tratar con el spam en tu sistema (o al menos, no tanto como sin él).
El API es muy simple, son únicamente cuatro operaciones:

  * Verificar la clave de uso (verifiy-key)
  * Comprobar un comentario (comment-check)
  * Notificar un falso negativo (submit-spam)
  * Notificar un falso positico (submit-ham)

Bueno, esto está bien, pero lo que realmente quería, era aprovechar este sencillo API y un wrapper sobre él en Groovy ([AkismetDrops](https://github.com/enqae/akismet-client)), para ver por encima alguna de las características que, desde mi punto de vista, hacen de Groovy algo interesante:

<!--more-->

#### Integración con Java
Groovy se compila a bytecode y su sintaxis es muy similar a la de Java, ello hace que la utilización de cualquier librería Java existente sea trivial:

{% codeblock %}
import org.apache.commons.logging.Log
import org.apache.commons.logging.LogFactory
...
Log logger = LogFactory.getLog(this.class.name)
...
{% endcodeblock %}

#### Sintaxis cómoda para el manejo de listas y mapas
Las listas y mapas probablemente sean uno de los elementos más utilizados en los programas, por lo que una sintaxis que simplifique su uso, sería muy apreciada:

Declarar una List

{% codeblock %}
  def validMethods = ['isThisCommentSpam': "comment-check",
                      'thisCommentShouldBeSpam': "submit-spam",
                      'thisCommentIsFalsePositive': "submit-ham"]
{% endcodeblock %}

Declarar e iterar por los valores de un Map

{% codeblock %}
  def dataToPost = [:] // Map
  ...
  def queryString = [] // List

  parameters.each {k, v ->
     queryString << "${k}=${URLEncoder.encode(v)}"
  }
{% endcodeblock %}  

#### Sentencias de control de flujo potenciadas
Las sentencias **if** y **switch** están muy potenciadas, especialmente la sentencia switch, que contrariamente a lo que ocurre en Java, que solamente admite enteros, en Groovy admite casi cualquier cosa (cadenas, mapas, rangos…):

  * power switch

{% codeblock %}
  def requiredParameters = [AkismetComment.BLOG,
                            AkismetComment.USER_IP, AkismetComment.USER_AGENT,
                            AkismetComment.COMMENT_CONTENT]
  ...
  ...

  switch (property.key) {
      case "class":
      case "metaClass":
        break
      case "otherData":
        property.value?.each {k, v ->
          if (v)
            dataToPost.put(k, v?.toString())
         }
        break
      case requiredParameters:
        if (property.value?.trim()) {
          dataToPost."${property.key}" = property.value.toString()
        } else {
          if (property.key == 'blog') {
            dataToPost.blog = blog
          } else {
            throw new IllegalArgumentException("Required parameter: ${property.key}")
          }
        }
        break
      case nonRequiredParameters:
        if (property.value?.trim()) {
          dataToPost."${property.key}" = property.value
        }
  }
{% endcodeblock %}  


#### Identity/with

Esto en realidad es casi una curiosidad, pero si vas a realizar múltiples operaciones sobre un objeto, el tener que escribir el nombre del objeto para cada operación puede ser un poco tedioso, y aquí la identity nos ahorra unas pulsaciones:

{% codeblock %}
      Writer writer = new OutputStreamWriter(connection.outputStream)
      writer.with {
        ...
        write queryString.join('&') // writer.wrie
        flush() //writer.flush
        close() //writer.close
      }
{% endcodeblock %}


#### Closures

Esta es una de las características más importantes de Groovy, que le proporcionan las capacidades de los lenguajes funcionales. Si trabajas con Groovy, tienes que manejarte con las closures porque están en la base del lenguaje. Es cierto, que requieren un cierto tiempo para acostumbrarse, pero luego se hacen imprescindibles.

{% codeblock %}
 Closure response200 = {text ->
                            Boolean result = (text == 'true') ;
                            if (logger.isDebugEnabled()) {
                              if (!result) {
                                if (text != 'false') {
                                  logger.debug("Message from Akismet: ${text}")
                                }
                              }
                            }
                            return result
                        }
...

 def validMethodsActions = ['isThisCommentSpam': response200,
                             'thisCommentShouldBeSpam': response200Log,
                             'thisCommentIsFalsePositive': response200Log]
...
{% endcodeblock %}


#### Comportamiento dinámico

Groovy es un lenguaje dinámico, es decir, es posible modificar su comportamiento durante la propia ejecución, y esto va desde modificar ligeramente el comportamiento de determinados métodos de un objeto, hasta generar nuevos métodos para un objeto en tiempo de ejecución, existe incluso una forma de atender a métodos que no existen. Y esto para qué sirve, bueno al principio puede ser un poco complicado de ver pero esta característica es la que da soporte entre otras cosas a un framework como Grails (que es una de las cosas que hacen especialmente intersante a Groovy).

  * AOP fácil:

{% codeblock %}
// Declara la clase como interceptable
class Akismet implements GroovyInterceptable {
  ...
  // Declara los métodos del API vacíos
  Boolean isApiKeyValid() {}
  Boolean isThisCommentSpam(AkismetComment comment) {}
  Boolean thisCommentShouldBeSpam(AkismetComment comment) {}
  def thisCommentIsFalsePositive(AkismetComment comment) {}
  ...

  def validMethods = ['isThisCommentSpam': "comment-check",
                      'thisCommentShouldBeSpam': "submit-spam",
                      'thisCommentIsFalsePositive': "submit-ham"]
  ...
  // Declara el interceptor - Todas las llamadas vendrán por aquí
  def invokeMethod(String name, args) {

    if (validMethods[name]) {
      // Si es uno de los métodos del API se tratarán aquí enviándolos a Akismet

      ...
    } else {
      // Si no es uno de los métodos del API, miramos si es un método que existe o no

      def otherMethod = Akismet.metaClass.getMetaMethod(name, args)
      if (otherMethod) {

        // El método existe y cedemos el control a su implementación ...
        return otherMethod.invoke(this, args)
      } else {

        // Se solicita un método que no existe, y cedemos el control al gestor por defecto, aunque
        // podríamos haber hecho cualquier otra cosa...
        Akismet.metaClass.invokeMissingMethod(this, name, args)
      }
    }
  }

  ...
}
{% endcodeblock %}

Esto es solamente rascar un poco la superficie, quedan fuera muchas cosas (rangos, bucles simplificados, categories, builders…). Un par de libros que tratan el tema con muchísima más amplitud son [Groovy in Action](http://www.manning.com/koenig/) y [Programming Groovy](http://pragprog.com/titles/vslg/programming-groovy).

Pero... para poder hacer todo esto desde Java, qué hay que hacer? ... pues aunque parezca mentira, solamente hay que incluir un JAR... y, bueno, no se me han olvidado los ;  ... se pueden poner, pero no son obligatorios.
