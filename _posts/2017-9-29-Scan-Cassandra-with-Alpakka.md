---
layout:     post
title:      "Scanning Cassandra Tables with Alpakka"
subtitle:   "Scan entire Cassandra Tables with ease with Alpakka"
date:       2017-09-29 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/saturn-bg.jpg"
---

I often have to write code to scan entire cassandra tables. The table in question here is more than 100 million rows where each row contains a hashmap with 300 keys on average.

We have a legacy code which uses raw jdbc code using Cassandra jdbc driver and while it works the code is exteremely unwieldy because we have to write mutable data structures inside of a for loop which gets executed by means of a callback. (Yuck!)

There are plenty of folks out there who use Spark for such purposes, but I like to work without the need for bulky things like Hadoop clusters. 

I came accross a technology called Alpakka today and it made scanning Cassnadra tables a breeze.

Here is my build.sbt

```scala
libraryDependencies ++= Seq(
   "com.lightbend.akka" %% "akka-stream-alpakka-cassandra" % "0.11"
)

```

And now let us scan the table

```scala
  val stmt = new SimpleStatement("select id, name from foo").setFetchSize(100)
  val source = CassandraSource(stmt)
  val flow = Flow[Row, NotUsed].map{row => Foo(row.getLong(0), row.getString(1))}
  val sink = Sink.foreach[Foo](println)
  val graph = RunnableGraph.fromGraph(GraphDSL.create(sink){implicit builder => 
  	s => 
  	souce ~> flow ~> s.in
  	ClosedShape
  }) 
  val future = graph.run()
  Await(future, Duration.Inf)
```

That's it. This code is heavenly as compared the the 100s of lines of jdbc code which I use currently. Very fast as well.

Alpakka project can be found at [Alpakka](https://developer.lightbend.com/docs/alpakka/current/) and at [Github](https://github.com/akka/alpakka)