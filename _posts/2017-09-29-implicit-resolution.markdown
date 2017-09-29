---
layout: post
title:  "Implicit Resolution and Traits"
date:   2017-09-29 15:21:30 -0500
categories: Scala Implicits
---
An interesting gotcha came up today while writing some slick code. 

{% highlight scala %}
implicit val getResult = GetResult(r => (r.<<, r.<<))
val query = sql"""select id, name from foo""".as[Foo]
case class Foo(id: Long, name: String)
{% endhighlight %}

However I don't like to define my implicits inline because they just add noise to the code. So I tried to refactor the code like

{% highlight scala %}
import Foo._
val query = sql"""select id, name from foo""".as[Foo]
case class Foo(id: Long, name: String)
object Foo {
  implicit val getResult = GetResult(r => (r.<<, r.<<))
}
{% endhighlight %}

However I got an error

**Could not find implicit value for parameter rconv: slick.jdbc.GetResult[Foo]**

Now that was pretty puzzing because I copy pasted my implicit definition into the object and the object is clearly imported in place of the inline implicit definition. I even tried using Traits instead of an object

{% highlight scala %}
object MyApp extends Conversions {
  val query = sql"""select id, name from foo""".as[Foo]
}
case class Foo(id: Long, name: String)
trait Conversions {
  implicit val getResult = GetResult(r => (r.<<, r.<<))
}
{% endhighlight %}

Humm... still the same error.

After some amount of research found that if the implicits are declared externally, then we have to define their type explicitly. The following fix finally fixed the code.

{% highlight scala %}
import Foo._
val query = sql"""select id, name from foo""".as[Foo]
case class Foo(id: Long, name: String)
object Foo {
  implicit val getResult : GetResult[Foo] = GetResult(r => (r.<<, r.<<))
}
{% endhighlight %}
