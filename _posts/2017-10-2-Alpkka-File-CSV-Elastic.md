---
layout:     post
title:      "Alpakka File CSV and ElasticSearch Connectors"
subtitle:   "Let us look at the File, CSV and ElasticSearch Connectors of Alpakka."
date:       2017-10-2 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/saturn-bg2.jpg"
---

In [part 1](https://abhsrivastava.github.io/2017/09/29/Scan-Cassandra-with-Alpakka/) of this series I had shown how to use the Alpakka Cassandra Connector to scan the entire cassandra table.

Today we will look at 3 connectors. The File Connector, the CSV Connector and finally the ElasticSearch connector.

The scenario we will use is that you have a list of countries and capitals available as a [csv file](https://github.com/icyrockcom/country-capitals/blob/master/data/country-list.csv) and we have to load these into ElasticSearch.

We will read the files using the File and CSV Connectors and then we will use ElasticSearch connector to create a Sink into ElasticSearch.

The build.sbt file is as follows

```scala
name := "ElasticAkkaStreams"
version := "1.0"
scalaVersion := "2.12.3"
libraryDependencies ++= Seq(
   "com.lightbend.akka" %% "akka-stream-alpakka-elasticsearch" % "0.13",
   "com.lightbend.akka" %% "akka-stream-alpakka-csv" % "0.13",
   "com.lightbend.akka" %% "akka-stream-alpakka-file" % "0.13"
)
mainClass in run := Some("com.abhi.ElasticAkkaStreams")
```

Nothing surprising here. We are just importing the connectors we need.

Now Let us first  read the files as a stream of strings.

```scala
  val resource = getClass.getResource("/countrycapital.csv")
  val path = Paths.get(resource.toURI)
  val source = FileIO.fromPath(path)
```

I have copied the country capitial csv in the resources folder. In order to read it we need to build the path to the resource and finally feed that path to the FileTailSource. The FileTailSource gives us a stream of Strings (one for each line).

Now We need to feed these lines to the CSV Connector for tokenization. the CSV Flow only accepts `akka.util.ByteString` so we need to convert our line to a ByteString

```scala
 val flow1 = Flow[String].map(ByteString(_))
 val flow2 = CsvParsing.lineScanner
 val flow3 = Flow[List[ByteString]].map(_.utf8String)
```

In the code above, we first converted our String (containing a line) into a ByteString, then we gave that ByteString to the CSVParser. Which split the line into tokens (of type List[ByteString]) then we converted each token back to a string. At this point we can load the data into a domain object of `CountryCapital`.

```scala
case class CountryCapital(country: String, capital: String)
val flow4 = Flow[List[String]].map(list => CountryCapital(list(0), list(1)))
```

The ElasticSearch Connector gives us a Sink to ElasticSearch. but that Sink accepts a stream of objects of type `IncommingMessage[JsObject]`. Here the type `JsObject` is coming from the spray-json library.

So we need to convert our domain object into the type `IncomingMessage[JsObject]` In order to facilitate this conversion we need to import a few classes which make it seamless to convert our CountryCaptial case class into spray-json JsObject.

```scala
  import DefaultJsonProtocol._
  implicit val format = jsonFormat2(CountryCapital)
  val flow5 = Flow[CountryCapital].map{cc => IncommingMessage[JsObject](Some(UUID.randomUUID.toString), cc.asJson.asJsObject)}
```  

Now that wasn't bad. Finally we can use Alpakka Elastic and build our sink

```scala
implicit val client = RestClient.builder(new HttpHost("abhisheks-mini", 9200)).build()
val sinkSettings = ElasticsearchSinkSettings(bufferSize = 100000, retryInterval = 5000, maxRetry = 100)
 val sink = ElasticsearchSink("myindex", "mytype", sinkSettings)
```


The client object provides the server name and the port of ElasticSearch. Apart from that, the sink obviously needs the index and the type where ther data will be loaded.

Finally we need to build a Runnable Graph

```scala
implicit val actorSystem = ActorSystem()
implicit val actorMaterializer = ActorMaterializer()
val graph = RunnableGraph.fromGraph(GraphDSL.create(sink){implicit builder => 
	s =>
	import GraphDSL.Implicits._ 
	source ~> flow1 ~> flow2 ~> flow3 ~> flow4 ~> flow5 ~> s.in
	ClosedShape
})
```
and then execute the graph

```scala
val future = graph.run()
future.onComplete { _ => actorSystem.terminate()}
Await.result(actorSystem.whenTerminated, Duration.Inf)
```

Now in order to check whether the data is loaded or not. Issue the following GET request to your elasticsearch REST endpoint

```
http://abhisheks-mini:9200/myindex/_search?pretty=true&q=*:*
```

In my case it showed me 231 rows.

One big Gotcha was that when you inject IncomingMessgaes into Elastic make sure that the ID is a GUID. For me the data load operaiton just loaded 51 rows when I was trying to use Country as a key.

The whole application can be found at my [github](https://github.com/abhsrivastava/ElasticAlpakkaStreams).