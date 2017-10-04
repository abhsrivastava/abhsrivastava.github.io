---
layout:     post
title:      "Stream Avro Records into Kafka using Avro4s and Akka Streams Kafka"
subtitle:   "Continuing our quest to learn Akka Streams, we'll stream some Avro records into a Kafka Topic and then read them as well"
date:       2017-10-2 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/saturn-bg4.jpg"
---

Continuing our quest to learn Akka Streams, we'll take our same old [countrycapital.csv](https://github.com/icyrockcom/country-capitals/blob/master/data/country-list.csv) and then we'll convert each line into an AVRO Record and then write into a Kafka topic using Akka streams.

Why [Avro](https://avro.apache.org/). Avro is binary data serialization format. It is more concise than Json and is very interoperable. Almost all popular languages have Avro bindings and therfore can easily and efficiently read the data from Kafka topic.

Without any further ado. Here is our build.sbt

```scala
name := "KafkaAvro"
version := "1.0"
scalaVersion := "2.12.3"
libraryDependencies ++= Seq(
   "com.sksamuel.avro4s" %% "avro4s-core" % "1.8.0",
   "com.typesafe.akka" %% "akka-stream" % "2.5.6",
   "com.lightbend.akka" %% "akka-stream-alpakka-csv" % "0.13",
   "com.lightbend.akka" %% "akka-stream-alpakka-file" % "0.13",
   "com.typesafe.akka" %% "akka-stream-kafka" % "0.17"
)
```

Now let's get some fundamentals out of the way. Basically we need to parse our CSV and then convert it into a stream of CountryCapital objects. I won't explain the code here because I already explained this [here](https://abhsrivastava.github.io/2017/10/02/Alpkka-File-CSV-Elastic/)

```scala
  implicit val actorSystem = ActorSystem()
  implicit val actorMaterializer = ActorMaterializer()
  val resource = getClass.getResource("/countrycapital.csv")
  val path = Paths.get(resource.toURI)
  val source = FileIO.fromPath(path)
  val flow1 = CsvParsing.lineScanner()
  val flow2 = Flow[List[ByteString]].map(list => list.map(_.utf8String))
  val flow3 = Flow[List[String]].map(list => CountryCapital(list(0), list(1)))
```

So now let us convert our CountryCapital stream of objects into Avro Records

```scala
  val flow4 = Flow[CountryCapital].map { cc =>
    val baos = new ByteArrayOutputStream()
    val output = AvroOutputStream.binary[CountryCapital](baos)
    output.write(cc)
    output.close()
    val result = baos.toByteArray
    baos.close()
    result
  }
```

In the code above i used Avro4s Serializer to convert the CountryCapital object into a strem of bytes containing the Avro Record.

Now that we have the Stream of Bytes, its easy to write it into a Kafka topic

```scala
val producerSettings = ProducerSettings(actorSystem, new ByteArraySerializer, new ByteArraySerializer)
      .withBootstrapServers("abhisheks-mini:9093")
  val flow5 = Flow[Array[Byte]].map{array =>
    new ProducerRecord[Array[Byte], Array[Byte]]("test", array)
  }
  val sink = Producer.plainSink(producerSettings)
```

Here we are using the [Akka Streams Kafka](https://doc.akka.io/docs/akka-stream-kafka/current/home.html) library to create a akka stream Sink which acts as a Kafka Producer. We need to convert our stream of Array of bytes into a Producer Record. the producer record also contains the name of our topic which is "test" in my case. 

Usually Kafka runs on port `9092`. I am running it on `9093` in order to avoid port conflict with ElasticSearch (which I also run on the same server).

Now that we have our Sink. All we need to do is to build our Runnable Graph and execute it

```scala
val graph = RunnableGraph.fromGraph(GraphDSL.create(sink){implicit builder =>
  s =>
    import GraphDSL.Implicits._
    source ~> flow1 ~> flow2 ~> flow3  ~> flow4 ~> flow5 ~> s.in
    ClosedShape
})
val future = graph.run()
future.onComplete { _ =>
  actorSystem.terminate()
}
Await.result(actorSystem.whenTerminated, Duration.Inf)
```


Reading Avro records from Kafka topic is also very straigtforward. The code is exactly the same as above. The only difference is that instead of a Producer, we are creating a Consumer

```scala
   val consumerSettings = ConsumerSettings(actorSystem, new ByteArrayDeserializer(), new ByteArrayDeserializer)
      .withBootstrapServers("abhisheks-mini:9093")
      .withGroupId("abhi")
      .withProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")
   val source = Consumer.committableSource(consumerSettings, Subscriptions.topics("test"))
   val flow1 = Flow[ConsumerMessage.CommittableMessage[Array[Byte], Array[Byte]]].map{ msg => msg.record.value()}
   val flow2 = Flow[Array[Byte]].map{ array =>
      val bais = new ByteArrayInputStream(array)
      val input = AvroInputStream.binary[CountryCapital](bais)
      input.iterator.toSeq.head
   }
   val sink = Sink.foreach[CountryCapital](println)
   val graph = RunnableGraph.fromGraph(GraphDSL.create(sink){ implicit builder =>
      s =>
         import GraphDSL.Implicits._
         source ~> flow1 ~> flow2 ~> s.in
         ClosedShape
   })
   val future = graph.run()
   future.onComplete { _ =>
      actorSystem.terminate()
   }
```

Once the items are read from the kafka topic, they are read as Array of bytes. We use the Avro4s Deserializer this time, to convert array of bytes into CountryCapital object.

The whole example can be found at my [github](https://github.com/abhsrivastava/KafkaAvro).