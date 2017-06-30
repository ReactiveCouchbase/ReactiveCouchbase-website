---
layout: home
title: ReactiveCouchbase
---

# About ReactiveCouchbase

ReactiveCouchbase is a scala driver that provides non-blocking and asynchronous I/O operations on top of <a href="http://www.couchbase.com" target="_blank">Couchbase</a>. ReactiveCouchbase is designed to avoid blocking on each database operations. Every operation returns immediately, using the elegant <a href="http://www.scala-lang.org/api/current/#scala.concurrent.Future" target="_blank">Scala Future API</a> to resume execution when it's over. With this driver, accessing the database is not an issue for performance anymore. ReactiveCouchbase is also highly focused on streaming data in and out from your Couchbase servers using the very nice <a href="http://www.reactive-streams.org/" target="_blank">ReactiveStreams</a> on top of <a href="http://doc.akka.io/docs/akka/2.4/scala/stream/index.html" target="_blank">Akka Streams</a>.

# Work in progress
 
ReactiveCouchbase RS (**ReactiveStreams edition**) is currently **under heavy development**. If you want to try it, add a resolver to your `build.sbt` file

```scala
resolvers += "reactivecouchbase-rs-snapshots" at "https://raw.github.com/ReactiveCouchbase/reactivecouchbase-rs-core/master/repository/snapshots"

resolvers += "reactivecouchbase-rs-releases" at "https://raw.github.com/ReactiveCouchbase/reactivecouchbase-rs-core/master/repository/releases"
```

or you can build it to get the nice goodies

```sh
git clone https://github.com/ReactiveCouchbase/reactivecouchbase-rs-core.git
cd reactivecouchbase-rs-core
sbt ';clean;compile;publish-local'
```

then in your project add the following dependency

```scala
libraryDependencies += "org.reactivecouchbase" %% "reactivecouchbase-rs-core" % "1.0.0-SNAPSHOT"
```

and you're ready to go

If you encounter any issue during build or usage, just <a target="_blank" href="https://github.com/ReactiveCouchbase/reactivecouchbase-rs-core/issues">let me know</a>.

# Simple example

```scala
import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import com.typesafe.config.ConfigFactory
import org.reactivecouchbase.rs.scaladsl.{N1qlQuery, ReactiveCouchbase}
import org.reactivecouchbase.rs.scaladsl.json._
import play.api.libs.json._
import akka.stream.scaladsl.Sink
import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import com.typesafe.config.ConfigFactory

object ReactiveCouchbaseTest extends App {

  val system = ActorSystem("ReactiveCouchbaseSystem")
  
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
    """.stripMargin))

  val bucket = driver.bucket("default")

  val future = for {
    _        <- bucket.insert[JsValue]("key1", 
                  Json.obj("message" -> "Hello World", "type" -> "doc"))
    doc      <- bucket.get("key1")
    exists   <- bucket.exists("key1")
    docs     <- bucket.search(
                    N1qlQuery("select message from default where type = $type")
                  .on(Json.obj("type" -> "doc")).asQueryParams)
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
import org.reactivecouchbase.rs.scaladsl._

@Singleton
class Couchbase @Inject()(configuration: Configuration, 
      lifecycle: ApplicationLifecycle) {

  private val driver = 
      ReactiveCouchbase(configuration.underlying.getConfig("reactivecouchbase"))

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
import org.reactivecouchbase.rs.scaladsl.json._
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
      .on(Json.obj("type" -> filter.getOrElse("doc")).asQueryParams)
      .asSource
      .map(Json.stringify)
      .intersperse("[", ",", "]")
    Ok.chunked(source).as("application/json")
  }
}
```

## What if I want to use a JSON lib other than Play Json ?

you can easily do that, actually everything linked to Play Json is imported from 

```scala
import org.reactivecouchbase.rs.scaladsl.json._
```

then you just have to reimplement a few things

```scala
import akka.util.ByteString
import com.couchbase.client.java.document.json.JsonObject
import org.reactivecouchbase.rs.scaladsl.json.{JsonReads, JsonWrites, JsonSuccess, QueryParams}
import foo.bar.jsonlib.{JsonNode, JsonObj}

val read: JsonReads[JsonNode] = JsonReads(bs => JsonSuccess(JsonNode.parse(bs.utf8String)))
val write: JsonWrites[JsonNode] = JsonWrites(jsv => ByteString(JsonNode.stringify(jsv)))

implicit val defaultByteStringFormat: JsonFormat[JsonNode] = JsonFormat(read, write)

implicit val defaultByteStringConverter: CouchbaseJsonDocConverter[JsonNode] = new CouchbaseJsonDocConverter[JsonNode] {
  override def convert(ref: AnyRef): JsonNode = ...
}

case class JsonObjQueryParams(query: JsonObj = ByteString.empty) extends QueryParams {
  override def isEmpty: Boolean = !query.hasValue
  override def toJsonObject: JsonObject = ...
}
```

You have a few examples at

* https://github.com/ReactiveCouchbase/reactivecouchbase-rs-core/blob/master/src/main/scala/org/reactivecouchbase/rs/scaladsl/json/package.scala
* https://github.com/ReactiveCouchbase/reactivecouchbase-rs-core/blob/master/src/main/scala/org/reactivecouchbase/rs/scaladsl/json/bytestring.scala 
* https://github.com/ReactiveCouchbase/reactivecouchbase-rs-core/blob/master/src/main/scala/org/reactivecouchbase/rs/scaladsl/json/converter.scala#L21-L31

# Projects

The core of ReactiveCouchbase RS is available on Gihtub and depends on Play Json library and Akka Streams

* <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-rs-core" target="_blank">ReactiveCouchbase RS project on GitHub</a>

# Community

* ReactiveCouchbase organisation on <a href="https://github.com/ReactiveCouchbase" target="_blank">GitHub</a>
* Tickets are on <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-rs-core/issues" target="_blank">GitHub</a>. Feel free to report any bug you find or to make pull requests.
* ReactiveCouchbase on <a href="https://groups.google.com/forum/?hl=fr#!forum/reactivecouchbase" target="_blank">Google Groups</a>

                
