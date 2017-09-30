---
layout:     post
title:      "Scanning Cassandra Tables with Alpakka"
subtitle:   "Scan entire Cassandra Tables with ease with Alpakka"
date:       2017-09-29 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/saturn-bg4.jpg"
---

I often have to write code to scan entire cassandra tables. The table in question here is more than 100 million rows where each row contains a hashmap with 300 keys on average.

We have a legacy code which uses raw jdbc code using Cassandra jdbc driver and while it works the code is exteremely unwieldy because we have to write mutable data structures inside of a for loop which gets executed by means of a callback. (Yuck!)

There are plenty of folks out there who use Spark for such purposes, but I like to work without the need for bulky things like Hadoop clusters. 

I came accross a technology called Alpakka today and it made scanning Cassandra tables a breeze.

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

That's it. Akka Steams API also gives you nice operations like fold on the flow so that you can neatly aggregate data.

Writing back into cassandra is already pretty easy .. thanks to the Akka Streams Sink.

```scala
   def cassandraSink(session: Session) : Sink[Foo, Future[Done]] = {
      implicit val s = session
      import scala.concurrent.ExecutionContext.Implicits.global
      val stmt = session.prepare("update foo set name = ? where id = ?")
      val binder = (foo: Foo, statement: PreparedStatement) => statement.bind(foo.name, java.lang.Long.valueOf(foo.id))
      CassandraSink[Foo](parallelism = 10, stmt, binder)
   }

```

Now think sink can be attached to any Flow which emits a Foo.

Alpakka project can be found at [Alpakka](https://developer.lightbend.com/docs/alpakka/current/) and at [Github](https://github.com/akka/alpakka)