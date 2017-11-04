---
layout:     post
title:      "Dependency injection of Akka Actors with Google Guice"
subtitle:   "use Google Guice with Akka Actors"
date:       2017-11-03 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/saturn-bg5.jpg"
---

I use Google Guice for dependency injection in most of my projects. I also use Akka Actors a lot for solving concurrency related tasks in my projects. I often write code like

```scala
val a = injector.getInstance(classOf[A])
val b = injector.getInstance(classOf[B])
val c = injector.getInstance(classOf[C])
val actorSystem = injector.getInstance(classOf[ActorSystem])
val myActorRef : ActorRef = actorSystem.actorOf(Props(new MyActor(a, b, c)))
```

And while this works. I still resent that now I have two ways of creating objects in my project. One via Guice approach of `injector.getInstance` and then a second approach which is specific for actors `actorOf(Props(...))` mechanism listed above. I always wished that I could use Guice consistently to get instances of actors and classes alike.

In this blog, we'll try to do just that. Lets write up a minimal actor which will be used in this example.

```scala
package com.abhi.logic
class Logic {
  def add(i: Int, j: Int) : Int = i + j
}

package com.abhi.actor
class MyActor(logic: Logic) extends Actor {
  def receive = {
  	case msg: Msg => sender() ! logic.add(msg.i, msg.j)
  }
}

case class Msg(i: Int, j: Int)
```

So I have written all my business logic in a simple class (which I can test easily) and now I am using my logic class as a dependency inside of the actor.

BTW, I am using the [codingwell/scala-guice][1] library to integrate Guice with my Scala project. Let us code up the Module required to perform all dependency injection.

```scala
class MyModule extends AbstractModule with ScalaModule{
   def configure() : Unit = {
      bind[ActorSystem].toInstance(ActorSystem())
      bind[Logic]
   }

   @Provides
   @Singleton
   @Named("MyActor")
   def getMyActor(actorSystem: ActorSystem, logic: Logic) = {
      actorSystem.actorOf(Props(new MyActor(logic)))
   }
}
```

You can see that I am using the `@Named` approach to get the right instance. This is required because all actors in the end are created as `ActorRef` so we can't use the `getInstance[T]` approach because the T is same for all actors. So we use names.

If our project had 100 actors. We will have one `@provides` method for each actor with a unique name. We can easily specify all our dependencies as parameters of this method. Since they have been bound earlier in the configure method, we get them transparently in the `@provides` method.

This also means that if tomorrow my actor starts depending on Logic1, Logic2, and Logic3 classes, I have just one place to make the change. the users of the actor don't see the dependencies.

Now we need a small utility class which will make it easy for the clients to looking the named instances.

```scala
object ActorUtil {
  implicit class ActorInjector(injector: Injector) = {
    getActor(name: String) : ActorRef = {
      injector.getInstance(Key.get(classOf[ActorRef], Names.named(name)))
    }
  }
}
```

We are creating this utility class so that our client code can easily lookup actors by their names. These names are what we have configured in the `@Provides` method for each actor.

Now let us write a client for our actors

```scala
import ActorUtil._
val injector = Guice.createInjector(new MyModule)
val myActor = injector.getActor("MyActor")
val msg = (myActor ? MyMsg(10, 20)).mapTo[Int]
```

If I was writing a class which needed the MyActor as a dependency. I could write

```scala
@Singleton
class MyClass @Inject()(@Named("MyActor") myactor: ActorRef) {
  myactor ! Msg(...)
}
```

That's pretty clean because now I can get instances of my actors using Guice mechanisms. This leads to more consistent coding style.

The full code of this example is located [here][2]

[1]:https://github.com/codingwell/scala-guice
[2]:https://github.com/abhsrivastava/ActorGuice
