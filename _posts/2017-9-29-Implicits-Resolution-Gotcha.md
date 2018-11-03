---
layout:     post
title:      "Implicit Resolution Gotcha"
subtitle:   "While working on Slick and interesting scenario of implicit resolution came up"
date:       2017-09-29 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/saturn-bg3.jpg"
---

While writing some slick code today an interesting problem poped up. Here is the code I was working on

```scala
object MyApp extends App {
  implicit val result = GetResult(r => Foo(r.<<, r.<<))
  val query = """select id, name from foo""".as[Foo]
  ...
}
case class Foo(id: String, name: String)
```

This compiles and works fine, but I never write my implicits inline becuase they just add noise to the code.

So I copy and pasted my implicit definition into the companion object

```scala
object MyApp extends App {
  import Foo.result	
  val query = """select id, name from foo""".as[Foo]
  ...
}
case class Foo(id: String, name: String)
object Foo {
	implicit val result = GetResult(r => Foo(r.<<, r.<<))	
}
```

Oddly, I get an error saying

```
  could not find implicit value for parameter rconv: slick.jdbc.GetResult[Foo]
```

Now that's pretty surprising because I am importing the variable directly. It turns out that the implicit resolution rules state that the return type of implicits must be specified explictly when importing them from an external place.

So the correct code was

```scala
object MyApp extends App {
  import Foo.result	
  val query = """select id, name from foo""".as[Foo]
  ...
}
case class Foo(id: String, name: String)
object Foo {
	implicit val result : GetResult[Foo] = GetResult(r => Foo(r.<<, r.<<))	
}
```

