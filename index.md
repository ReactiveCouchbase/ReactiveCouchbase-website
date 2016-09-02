---
layout: home
title: ReactiveCouchbase
---

About Reactive Couchbase  
==========================

ReactiveCouchbase (former play2-couchbase) is a scala driver that provides non-blocking and asynchronous I/O operations on top of <a href="http://www.couchbase.com">Couchbase</a>. ReactiveCouchbase is designed to avoid blocking on each database operations. Every operation returns immediately, using the elegant <a href="http://www.scala-lang.org/api/current/#scala.concurrent.Future">Scala Future API</a> to resume execution when it's over. With this driver, accessing the database is not an issue for performance anymore. ReactiveCouchbase is also highly focused on streaming data in and out from your Couchbase servers using the very nice <a href="http://www.playframework.com/documentation/2.2.x/Iteratees">Play framework Iteratee API</a>.

Use ReactiveCouchbase from other environments
=======================

* from Java, using the <a href="java.html">Java wrapper</a>
* from Play 2 application, using the <a href="play.html">Play plugin</a>

A simple example
=======================
 
Here you'll see how to use the basic features of ReactiveCouchbase.
 
### Prerequisites 

 
We assume that you got a running Couchbase instance. If not, get the latest Couchbase binaries and install it (<a href="http://www.couchbase.com/download">http://www.couchbase.com/download</a>). You can also populate your default bucket with the beer sample provided with Couchbase.
 
### Set up your project dependencies 

 
ReactiveCouchbase is available on a private Maven repository. If you use SBT, you just have to edit build.sbt and add the following

```scala
resolvers += "ReactiveCouchbase Releases" at "https://raw.github.com/ReactiveCouchbase/repository/master/releases/"
 
libraryDependencies ++= Seq(
  "org.reactivecouchbase" %% "reactivecouchbase-core" % "0.3"
)
```

or if you feel adventurous, you can use snapshots

```scala
resolvers += "ReactiveCouchbase Snapshots" at "https://raw.github.com/ReactiveCouchbase/repository/master/snapshots/"
 
libraryDependencies ++= Seq(
  "org.reactivecouchbase" %% "reactivecouchbase-core" % "0.4-SNAPSHOT"
)
```
                 
### Connect to the default bucket 

                 
You can get a connection to the default bucket like this

```scala
// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
 
object Application extends App {
 
  // get a driver instance driver    
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")
 
  // do something here with the default bucket  

  // shutdown the driver (only at app shutdown)
  driver.shutdown()
}
```

You'll need an application.conf file in the project resources 

```
couchbase {
  buckets = [{
    host="127.0.0.1"
    port="8091"
    base="pools"
    bucket="default"
    user=""
    pass=""
    timeout="0"
  }]
}
```
                 
                 
You can also provide your own ActorSystem, Logging system and configuration programmatically to the driver when you create it.
    
### Insert some documents

Using JSON blobs

```scala
// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
// import the implicit JsObject reader and writer 
import org.reactivecouchbase.CouchbaseRWImplicits.documentAsJsObjectReader
import org.reactivecouchbase.CouchbaseRWImplicits.jsObjectToDocumentWriter
import scala.concurrent.Future
import play.api.libs.json._

object Application extends App {

  // get a driver instance driver  
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")

  // creates a JSON document
  val document = Json.obj(
    "name" -> "John",
    "surname" -> "Doe",
    "age" -> 42,
    "address" -> Json.obj(
      "number" -> "221b",
      "street" -> "Baker Street",
      "city" -> "London"
    )
  )
  
  // persist the JSON doc with the key 'john-doe', using implicit 'jsObjectToDocumentWriter' for serialization
  bucket.set[JsObject]("john-doe", document).onSuccess {
    case status => println(s"Operation status : ${status.getMessage}")
  }

  // shutdown the driver (only at app shutdown)
  driver.shutdown()
}
```

Using typesafe objects

```scala

// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
import scala.concurrent.Future
import play.api.libs.json._

case class Address(number: String, street: String, city: String)
case class Person(name: String, surname: String, age: Int, address: Address)

object Application extends App {

  // get a driver instance driver  
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")

  // provide implicit Json formatters
  implicit val addressFmt = Json.format[Address]
  implicit val personFmt = Json.format[Person]

  // creates a JSON document
  val document = Person("John", "Doe", 42, Address("221b", "Baker Street", "London"))

  // persist the Person instance with the key 'john-doe', using implicit 'personFmt' for serialization
  bucket.set[Person]("john-doe", document).onSuccess {
    case status => println(s"Operation status : ${status.getMessage}")
  }

  // shutdown the driver (only at app shutdown)
  driver.shutdown()
}

```
### Fetch a document

```scala

// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
import scala.concurrent.Future
import play.api.libs.json._

case class Address(number: String, street: String, city: String)
case class Person(name: String, surname: String, age: Int, address: Address)

object Application extends App {

  // get a driver instance driver  
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")

  // provide implicit Json formatters
  implicit val addressFmt = Json.format[Address]
  implicit val personFmt = Json.format[Person]

  // get the Person instance with the key 'john-doe', using implicit 'personFmt' for deserialization
  bucket.get[Person]("john-doe").map { opt =>
    println(opt.map(person => s"Found John : ${person}").getOrElse("Cannot find object with key 'john-doe'"))
  }

  // shutdown the driver (only at app shutdown)
  driver.shutdown()
}
``` 

### Run a simple query 
                 
Assuming your default bucket is filled with the beer sample data, let's print all the beers 

```scala
// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
// import the implicit JsObject reader  
import org.reactivecouchbase.CouchbaseRWImplicits.documentAsJsObjectReader
import com.couchbase.client.protocol.views.Query
import play.api.libs.iteratee.Iteratee
import scala.concurrent.Future
import play.api.libs.json._

object Application extends App {

  // get a driver instance driver  
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")

  // search all docs from the view 'brewery_beers'   
  // then enumerate it and print each document on the fly  
  // streaming style
  bucket.searchValues[JsObject]("beers", "brewery_beers")
      (new Query().setIncludeDocs(true))
        .enumerate.apply(Iteratee.foreach { doc =>
    println(s"One more beer for the road  : ${Json.prettyPrint(doc)}")
  })

  // search the whole list of docs from the view 'brewery_beers'   
  val futureList: Future[List[JsObject]] = 
    bucket.searchValues[JsObject]("beers", "brewery_beers")
      (new Query().setIncludeDocs(true)).toList
  // when the query is done, run over all the docs and print them  
  futureList.map { list =>
    list.foreach { doc =>
      println(s"One more beer for the road : ${Json.prettyPrint(doc)}")
    }
  }
  // shutdown the driver (only at app shutdown)
  driver.shutdown()
}
```
                 

Samples
======================

You can find some application samples on <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-play/tree/master/samples">ReactiveCouchbase's GitHub</a>

You can also find starter kits to quickly bootstrap projects

* <a href="https://github.com/ReactiveCouchbase/reactivecouchbase-starter-kit">Pure Scala starter kit</a>
* <a href="https://github.com/ReactiveCouchbase/play-java-starter-kit">Play Java starter kit</a>
* <a href="https://github.com/ReactiveCouchbase/play-scala-starter-kit">Play Scala starter kit</a>
* <a href="https://github.com/ReactiveCouchbase/play-crud-starter-kit">Pure Scala CRUD starter kit</a>
* <a href="https://github.com/arcane86/ReactiveCouchbase-SecureSocial-Sample">SecureSocial sample using Play</a> by <a href="https://github.com/arcane86">arcane86</a>

Projects
======================

ReactiveCouchbase is composed of 3 Projects

### ReactiveCouchbase Core

The core of ReactiveCouchbase. Depends on Play Iteratees library, Play JSON library and Akka

* <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-core">GitHub</a>
* <a href="doc/reactivecouchbase-core">ScalaDoc</a>

### ReactiveCouchbase Event Store

A simple Event Store for eventsourcing application using Couchbase as event log

* <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-es">GitHub</a>
* <a href="doc/reactivecouchbase-es">ScalaDoc</a>

### ReactiveCouchbase Play 2 plugin

You can absolutely use ReactiveCouchbase from a Play 2 application. 
The Play plugin for ReactiveCouchbase include some nice features to boost your productivity and let you focus on app writing :

* Integration with the native Json API of the framework
* Integration with the native stream API of the framework for realtime webapps
* Integration with the Action API
* Driver and connections management handled automatically by the plugin
* Automatic CRUD HTTP services for fast prototyping
* Integration with Java
* Couchbase as cache 
* Design document synchronisation per bucket at startup (Evolutions module)
* Sample data synchronisation per bucket at startup (Fixture module)

For more information about it, go to the <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-play/blob/master/README.md">ReactiveCouchbase-play's GitHub project</a>

* <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-play">GitHub</a>
* <a href="doc/reactivecouchbase-play">ScalaDoc</a>

Community
======================

* ReactiveCouchbase organisation on <a href="https://github.com/ReactiveCouchbase">GitHub</a>
* Tickets are on <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-core/issues">GitHub</a> (<a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-core/issues">1</a>, <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-play/issues">2</a>, <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-es/issues">3</a>). Feel free to report any bug you find or to make pull requests.
* ReactiveCouchbase on <a href="https://groups.google.com/forum/?hl=fr#!forum/reactivecouchbase">Google Groups</a>

                
