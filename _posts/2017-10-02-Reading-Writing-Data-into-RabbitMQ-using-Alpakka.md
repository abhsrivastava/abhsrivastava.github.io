---
layout:     post
title:      "Reading and Writing data into RabbitMQ using Alpakka"
subtitle:   "Part III of the series on Alpakka"
date:       2017-10-2 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/saturn-bg3.jpg"
---

In [Part 1](https://abhsrivastava.github.io/2017/09/29/Scan-Cassandra-with-Alpakka/) of this series we learnt how to use Alpakka Cassandra connector to scan entire cassandra table. In [part 2](https://abhsrivastava.github.io/2017/10/02/Alpkka-File-CSV-Elastic/) of the series we saw how to use Elastic Connector of Alpakka to load data into ElasticSearch indexes. In part 3 we will look at how to read and write data into RabbitMQ using Alpakka.


We need to setup RabbitMQ for our quick prototype. Here are the steps which I used to stand up a RabbitMQ Test environment in minutes.

```shell
  brew install rabbitmq
  vi /usr/local/etc/rabbitmq/rabbitmq-env.conf
```

I changed the contents of this file to 

```
CONFIG_FILE=/usr/local/etc/rabbitmq/rabbitmq
NODE_IP_ADDRESS=abhisheks-mini
NODENAME=rabbit@abhisheks-mini
```

Basically I removed the references to `127.0.0.1` and `localhost` and replaced with the machine name. I think this required if you'll connect to the RabbitMQ Remotely.

now let us start the rabbitmq by `brew services start rabbitmq`. We can check whether the service started successfully by `rabbitmqctl status`

BTW: most command line utilities are not in the PATH by default. So I had to edit my .bash_profile and add `/usr/local/sbin` directory to PATH variable.

Now, we need to setup our exchange, queue and its bindings.

```
rabbitmqctl add_user abhi abhi
rabbitmqctl set_user_tags abhi administrator
rabbitmqctl set_permissions -p abhi ".*" ".*" ".*"
rabbitmqctl add_vhost myvhost
rabbitmqctl set_permissions -p myvhost abhi ".*" ".*" ".*"
rabbitmqadmin declare exchange --vhost=myvhost name=exchange type=direct --user=abhi --password=abhi
rabbitmqadmin declare queue --vhost=myvhost name=queue durable=true --user=abhi --password=abhi
rabbitmqadmin declare binding --vhost=myvhost source=exchange destination=queue destination_type=queue routing_key="foobar" --user=abhi --password=abhi
```

Now if all the above steps go through successfully, then we can test our queue by publishing a test message

```
rabbitmqadmin publish exchange=exchange --vhost=myvhost routing_key="foobar" --user=abhi --password=abhi payload="Hello World"
```

If this says "message published" then we are ready to roll.

So now let us pull in all the Alpakka connectors we need for this example. We need the file and CSV connectors to load our countrycapital.csv file and we need the AMQP connector for reading/writing data from/to RabbitMQ.

```scala
name := "AlpakkaAMQP"
version := "1.0"
scalaVersion := "2.12.3"
libraryDependencies ++= Seq(
   "com.lightbend.akka" %% "akka-stream-alpakka-amqp" % "0.13",
   "com.lightbend.akka" %% "akka-stream-alpakka-csv" % "0.13",
   "com.lightbend.akka" %% "akka-stream-alpakka-file" % "0.13"
)
```

So first we will write some data into RabbitMQ. Jumping straight to the meat, let us crate the Akka Streams sink.

```scala
   val queueName = "myqueue"
   val queueDeclaration = QueueDeclaration(queueName, durable = true)
   val uri = "amqp://abhi:abhi@abhisheks-mini:5672/myvhost"
   val settings = AmqpSinkSettings(AmqpConnectionUri(uri))
      .withRoutingKey("foobar")
      .withExchange("exchange")
      .withDeclarations(queueDeclaration)
   val amqpSink = AmqpSink.simple(settings)
```

Quite straightfoward. We declare our queue, we create a settings object which contains the URI of the location where we want to connect. Ofcourse, we need to specify our routing key and our exchange.

This gives us a Akka Streams Sink which accepts a stream of objects of type `akka.util.ByteString`

This is really convenient because we have already seen how to read our file as a stream of Bytestream in previous two blog entries.

```scala
   val resource = getClass.getResource("/countrycapital.csv")
   val path = Paths.get(resource.toURI)
   val source = FileTailSource.lines(path, 8092, 100 millis).map{x => println(x); x}.map(ByteString(_))
```

So now we have a source of type ByteSream and we have a sink which accepts a type of ByteStream. Time to build our graph and execute it.

```scala
   val graph = RunnableGraph.fromGraph(GraphDSL.create(amqpSink){implicit builder =>
      s =>
         import GraphDSL.Implicits._
         source ~> s.in
         ClosedShape
   })
   val future = graph.run()
   future.onComplete { _ =>
      actorSystem.terminate()
   }
   Await.result(actorSystem.whenTerminated, Duration.Inf)
```

Now in order to read the data we just wrote into our queue. we need to build a Akka Streams source for RabbitMQ.

```scala
   val queueName = "queue"
   val queueDeclaration = QueueDeclaration(queueName, durable = true)
   val uri = "amqp://abhi:abhi@abhisheks-mini:5672/myvhost"
   val amqpUri = AmqpConnectionUri(uri)
   val namedQueueSourceSettings = NamedQueueSourceSettings(amqpUri, queueName).withDeclarations(queueDeclaration)
   val source = AmqpSource.atMostOnceSource(namedQueueSourceSettings, bufferSize = 10)
```

The code above is again straight forward. we provide our queue name, erver name, port and vhost. This gives us a source which reads data of type `IncommingMessage` we need to convert it into String. This can easily be done by our 3 flows (we have already used these in our previous blog entries)

```scala
   val flow1 = Flow[IncomingMessage].map(msg => msg.bytes)
   val flow2 = Flow[ByteString].map(_.utf8String)
   val sink = Sink.foreach[String](println)
```

Once again. connect all the flows in a graph and execute it and you'll see all the records being read from the RabbitMQ queue.

The entire example is available at my [github](https://github.com/abhsrivastava/AlpakkaAMQP)