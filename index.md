---
layout: home
title: ReactiveCouchbase
---

# About ReactiveCouchbase  

ReactiveCouchbase is a scala driver that provides non-blocking and asynchronous I/O operations on top of <a href="http://www.couchbase.com" target="_blank">Couchbase</a>. ReactiveCouchbase is designed to avoid blocking on each database operations. Every operation returns immediately, using the elegant <a href="http://www.scala-lang.org/api/current/#scala.concurrent.Future" target="_blank">Scala Future API</a> to resume execution when it's over. With this driver, accessing the database is not an issue for performance anymore. ReactiveCouchbase is also highly focused on streaming data in and out from your Couchbase servers using the very nice <a href="http://www.reactive-streams.org/" target="_blank">ReactiveStreams</a> on top of <a href="http://doc.akka.io/docs/akka/2.4/scala/stream/index.html" target="_blank">Akka Streams</a>.

# Work in progress
 
ReactiveCouchbase v2 (**ReactiveStreams edition**) is currently a work in progress. You can build the driver on your machine to try it :

```sh
git clone https://github.com/ReactiveCouchbase/reactivecouchbase-rs-core.git
cd reactivecouchbase-rs-core
sbt ';clean;compile;publish-local'
```

then in your project add the dependency

```scala
libraryDependencies += "org.reactivecouchbase" % "reactivecouchbase-core" % "2.0.0-SNAPSHOT"
```

# Simple example

```scala
import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import com.typesafe.config.ConfigFactory
import org.reactivecouchbase.scaladsl.{N1qlQuery, ReactiveCouchbase}
import play.api.libs.json.Json
import akka.stream.scaladsl.Sink
import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import com.typesafe.config.ConfigFactory

object ReactiveCouchbaseTest extends App {

  implicit val system = ActorSystem("ReactiveCouchbaseSystem")
  implicit val materializer = ActorMaterializer.create(system)
  implicit val ec = system.dispatcher

  val driver = ReactiveCouchbase(ConfigFactory.parseString(
    """
      |buckets {
      |  default {
      |    name = "default"
      |    hosts = ["127.0.0.1"]
      |  }
      |}
    """.stripMargin), system)

  val bucket = driver.bucket("default")

  val future = for {
    _        <- bucket.insert("key1", 
                  Json.obj("message" -> "Hello World", "type" -> "doc"))
    doc      <- bucket.get("key1")
    exists   <- bucket.exists("key1")
    docs     <- bucket.search(
                    N1qlQuery("select message from default where type = $type")
                  .on(Json.obj("type" -> "doc")))
                  .asSeq
    messages <- bucket.search(
                    N1qlQuery("select message from default where type = 'doc'"))
                  .asSource.map(doc => (doc \ "message").as[String].toUpperCase)
                  .runWith(Sink.seq[String])
    _        <- driver.terminate()
  } yield (doc, exists, docs)

  future.map {
    case (_, _, docs) => println(s"found $docs")
  }

}
```

# What about a PlayFramework plugin ?

I don't think you actually need a plugin, if you want to use it from Play Framework, you can define a service to access your buckets like the following :


```scala
import javax.inject._
import play.api.inject.ApplicationLifecycle
import play.api.Configuration
import org.reactivecouchbase.scaladsl._

@Singleton
class Couchbase @Inject()(configuration: Configuration, lifecycle: ApplicationLifecycle) {

  private val driver = ReactiveCouchbase(configuration.underlying.getConfig("reactivecouchbase"))

  def bucket(name: String): Bucket = driver.bucket(name)

  lifecycle.addStopHook { () =>
    driver.terminate()
  }
}
```

so you can define a controller like the following

```scala
import javax.inject._
import scala.concurrent.ExecutionContext
import play.api.mvc._
import akka.stream.Materializer
import play.api.libs.json._

@Singleton
class ApiController @Inject()(couchbase: Couchbase)
    (implicit ec: ExecutionContext, materializer: Materializer) extends Controller {

  def eventsBucket = couchbase.bucket("events")

  def events(filter: Option[String] = None) = Action {
    val source = eventsBucket
      .search(N1qlQuery(
        "select id, payload, date, params, type from events where type = $type"
      )
      .on(Json.obj("type" -> filter.getOrElse("doc")))
      .asSource
      .map(Json.stringify)
    Ok.chunked(source.intersperse("[", ",", "]")).as("application/json")
  }
}
```

# Projects

The core of ReactiveCouchbase is available on Gihtub and depends on Play JSON library and Akka Streams

* <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-rs-core" target="_blank">ReactiveCouchbase project on GitHub</a>

# Community

* ReactiveCouchbase organisation on <a href="https://github.com/ReactiveCouchbase" target="_blank">GitHub</a>
* Tickets are on <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-rs-core/issues" target="_blank">GitHub</a>. Feel free to report any bug you find or to make pull requests.
* ReactiveCouchbase on <a href="https://groups.google.com/forum/?hl=fr#!forum/reactivecouchbase" target="_blank">Google Groups</a>

                
