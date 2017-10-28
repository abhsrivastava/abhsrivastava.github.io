---
layout:     post
title:      "Replace SprayTestKit with a simple Scala DSL"
subtitle:   "Build a custom DSL through functional programming"
date:       2017-10-27 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/saturn-bg4.jpg"
---

I work on a Scala application which has several hundered test cases written in [SprayTestKit][1]. One of the challenges in migrating this application to a new technology is that the moment we replace spray all our test cases break. (because test cases depend on spray). While everyone is quite eager to replace spray with latest and greatest technoogy stack, almost no one wants to rewrite these test cases.

I wanted a way to decouple our test cases from SprayTestKit without having to write these all the test cases.

Let us look at the definintion of a typical SprayTestKit test case which uses the DSL specified by the kit

```scala
Get("/foo/bar", "{'name': 'test', 'age': 20}") ~> cookie ~> apiRoute ~> check {
   responseAs[String] must contain("Say hello")
}
```

I really like this a lot because this makes it so easy to write test cases for web services. We will develop a DSL which looks similar to this, but doesn't need spray framework. I will also make certain improvements to this DSL. The second parameter to the HTTP Verb methods takes in a string which must be the JSON representation of the request body. this forces me to call `asJson` methods again and again on my request objects. I will let my library handle the json conversion. So I will just pass my case classes as the second parameter to the HTTP Verb method. Second change is that the apiRoute parameter is a little redundant because the URL to the HTTP Verb method is the one which decides where the call will be made. So why to specify the same information twice?

So our DSL will look like this

```scala
Get("/foo/bar", Input("test", 20)) ~> addCookie("name", "value") ~> check { resp => 
	val output = responseAs[Output](resp.body)
	assert(outupt === Output("Hello World foo"))
}
```

Here you can assume that we are calling a web service written in any programming launage which takes json representation of SayHello as a input parameter and returns json representatino of SayHelloResponse. 

Another change I am made to the DSL is that the test cases doesn't need to `Cookie` object because this will directly tie the test case to the Cookie object provided by my HTTP Library. Instead I use a function which takes two strings, and I will build the cookie internally. This means that my test cases don't need to directly touch the Http library objects. This will enable me to easily switch my HTTP Library without changing my test cases again.


So let's get the easy part out. We need an enumeration which contains all the HTTP Status codes. I simply searched the web and created a simple scala enum which contains all the codes. this enum called StatusCodes can be found [here][2].

The first part of our DSL is the HTTP Verb function which creates the Request object.

```scala
object WebServiceTestKit {
   private def toUri(url: String): Uri = Uri.unsafeFromString(url)
   private def toRequest(url: String, method: Method) = Request(uri = toUri(url), method = method)
   // HTTP Verbs
   def Post[T](url: String, t: T) : Request = toRequest(url, Method.POST)
   def Get[T](url: String, t: T) : Request = toRequest(url, Method.GET)
   def Put[T](url: String, t: T) : Request = toRequest(url, Method.PUT)
   def Delete[T](url: String, t: T) : Request = toRequest(url, Method.DELETE)
}
```

Great to with this, we created the first part of the DSL. We can create the Request object. However we need a way to convert our parameter of type T to JSON.

We will import Circe dependencies and then add the following code

```scala
object WebServiceTestKit {
   val client = PooledHttp1Client() 	
   private def toUri(url: String): Uri = Uri.unsafeFromString(url)
   private def toRequest(url: String, method: Method) = Request(uri = toUri(url), method = method)
   // HTTP Verbs
   def Post[T](url: String, t: T)(implicit e: Encoder[T]) : Request = toRequest(url, Method.POST).withBody(t.asJson.noSpaces).unsafeRun()
   def Get[T](url: String, t: T)(implicit e: Encoder[T]) : Request = toRequest(url, Method.GET).withBody(t.asJson.noSpaces).unsafeRun()
   def Put[T](url: String, t: T)(implicit e: Encoder[T]) : Request = toRequest(url, Method.PUT).withBody(t.asJson.noSpaces).unsafeRun()
   def Delete[T](url: String, t: T)(implicit e: Encoder[T]) : Request = toRequest(url, Method.DELETE).withBody(t.asJson.noSpaces).unsafeRun()
}
```

This code compiles because its our job to pass the ecoder for type T as an implicit to the HTTP Verb methods. We will use Circe automatic type derivation to pass this ecoder without writing one.

Now let's go for the second part of the DSL. We need a way to attach headers and cookies to this request object. We will use some clever functional programming so that we can build our DSL.

```scala
val addHeader : (String, String) => Request => Request = (name, value) => (req: Request) => req.putHeaders(Header(name, value))
val addCookie: (String, String) => Request => Request = (name, value) => (req: Request) => req.putHeaders(org.http4s.headers.Cookie(Cookie(name, value)))
```

OK looks cryptic. Here we have a addHeader function. Which takes two strings as a parameter and returns another function as an output. the output function takes a http request object as input and returns a http request object as output (the output has the headers and cookies attached)

Now we need to build our `~>` method which chains the output of the HTTP Verb method to the add Header method. We also need to apply our operation in infix style because our code is like `Post(...) ~> addHeader("foo", "bar")`. To meet this requirement we write an implicit class inside our WebServiceTestKit

```scala
implicit class RequestOps(request: Request) {
  def ~>(f: Request => Request) : Request = f(request)
}
```

When we apply the `~>` operator on the Request object the compilation will fail. this will cause the compiler to look for an implicit conversion. it will convert our Request object to the RequestOps type. Luckily our functions like 'addHeader' and 'addCookie' are of type Request => Request. So we can easily do `Post(...) ~> addheader(...)`.

So the last piece now is the check method. We will apply the same stragety as we did for adding headers. We will have a function called check. Which takes in a function as a input parameter. This input parameter function accepts a Respnose object and then returns another function as output. the output function is of type Request => Unit. 

```scala
case class WebTestKitResponse(status: StatusCodes.Value, body: String)
val check : (WebTestKitResponse => Unit) => Request => Unit = (f) => (req: Request) => {
   val response = client.fetch[WebTestKitResponse](req) {
      case Successful(resp) => resp.as[String].map(b => WebTestKitResponse(StatusCodes.fromInt(200), b))
      case fail => fail.as[String].map(f => WebTestKitResponse(StatusCodes.fromInt(fail.status.code), f))
   }.unsafeRun()
   f(response)
}
```


Our current `~>` operator cannot be used for check function. because check function produces a function of type `Request => Unit`. but ~> expects `Request => Request`. So we will overload the `~>` method as

```scala
implicit class RequestOps(request: Request) {
  def ~>(f: Request => Request) : Request = f(request)
  def ~>(f: Request => Unit) : Unit = f(request)  
}
```

That's it. now we can write a simple client program that uses our DSL

```scala
import com.abhi.webservice.testkit.WebServiceTestKit._
import io.circe.generic.auto._

case class SayHello(name: String)
case class SayHelloResponse(msg: String)

object Example extends App {
   Post("/foo/bar", SayHello("abhishek")) ~> addHeader("auth", "xxxyyyy") ~> check { resp =>
      assert(resp.status == StatusCodes.OK)
      val response = responseAs[SayHelloResponse](resp.body)
      assert(response.msg == "Hello World abhishek")
   }
}
```

It's imporant not to forget to import `import io.circe.generic.auto._` because that gives us the Ecoder of the cases classes for free. Otherwise we will have to manually write our encoders and that won't be fun at all.

This DSL is not a drop in replacement for the SprayTestKit because we made some changes, so I still had to spend 1 day doing search and replace in my code to make minor code changes. But in the end all my 400+ test cases ran and i removed the dependency on spray client and spray test kit from my code.

I am uploading my code [here][3].


[1]: https://github.com/spray/spray/tree/master/spray-testkit/src
[2]: https://github.com/abhsrivastava/WebServiceTestKit/blob/master/src/main/scala/com/abhi/webservice/testkit/StatusCodes.scala
[3]: https://github.com/abhsrivastava/WebServiceTestKit
