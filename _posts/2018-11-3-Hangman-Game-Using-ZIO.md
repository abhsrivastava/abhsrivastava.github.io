---
layout:     post
title:      "Building the Hangman Game using ScalaZ ZIO"
subtitle:   "Let's learn the use of IO Monad with this fun game"
date:       2018-11-03 12:00:00
author:     "Abhishek Srivastava"
header-img: "img/satelliterev4.jpg"
---

I came across this [fantastic talk][1] delivered by [John De Goes][2] for [Scala Kiev Meetup][3]. In this talk
he implements a small fun game called Hangman and in the process he teaches us the [ZIO][4] Library. 

I found the audio of the talk to be a little choppy and so I decided to write this blog and summarize my learnings from the talk.

If you are reading this article, I'll presume that you already know why we need things like ZIO. We wrap side effecting code into IO Monads to make it referentially transparent. This allows us to push all the side effects to the boundary of our application and thus the inner functional core of the program is preserved.

Let me explain the game a little before we start writing it. The computer will select a random word for you and will ask you to guess the word. You can guess one character at a time. Every time you guess a character which occurs in the word, all occurrences of that character will be revealed. If you guess all the characters which occur in the chosen word you win the game. If the number of guesses reaches 10 and you still haven't guessed all characters, you lose.

This is a good example to learn about the IO Monad because there are lots of side effects in this game. We need to collect user input like their name, each character they guess. We need to inform them whether their guess is right or wrong and in the end we need to tell them whether they won or lost the game. All these operations are side effects. So how can we write this application in a purely functional way?

```scala
sbt new scala/hello-world.g8
```

It will ask you for the project name enter "hangman". This will create a Hello World project for you. First thing to do is to replace the contents of the build.sbt with this one. The build.sbt which the hello world project generates has lots of comments and we need a simpler file. 

```scala
lazy val root = (project in file(".")).
  settings(
    inThisBuild(List(
      organization := "com.abhi",
      scalaVersion := "2.12.7",
      version      := "0.1.0-SNAPSHOT"
    )),
    name := "hangman",
    libraryDependencies ++= Seq(
      "org.scalaz" %% "scalaz-zio" % "0.3.1"
    ),
    trapExit := false
  )
```

Do a `sbt compile` here to make sure that the project compiles and all binaries are downloaded correctly. Remove the Main.scala file from the project and add Hangman.scala with the following content.

```scala
import scalaz.zio._
import scalaz.zio.console._
import java.io.IOException

object Hangman extends App {
    def run(args: List[String]) : IO[IOException, ExitStatus] = ???
}
```

Right off the bat we see a fundamental difference between this App and the regular App which we use in Scala. This doesn't have the main method which returns Unit. Instead our main method is replaced by a run method which returns a value. This value represents our entire program. The runtime will lazily evaluate this program and thus our program will run.

Our whole program is a value and it should always return this value whether our program runs successfully or it encounters an error. We have to do this by help of the redeem function of ScalaZ ZIO. It helps us return a value from our program in case of errors. The code below returns the value of ExitStatus(1) in case of errors and ExitStatus(0) in case of success. Note that the left side of the IO is Nothing. Here we are indicating that our program never throws an exception. This program will always return an ExitStatus.

```scala
    def run(args: List[String]) : IO[Nothing, ExitStatus] = {
        hangman.redeemPure(
            _ => ExitStatus.ExitNow(1),
            _ => ExitStatus.ExitNow(0)
        )
    }

    val hangman : IO[IOException, Unit] = ???
```

We need a case class which holds the state of our game. Here we capture the name of the player in the name attribute. We need to capture all the guesses which the player has made in a Set and finally the word which the player has to guess. 
We have a property which captures all the failed guesses. This helps us determine whether to terminate the game or not. We also have two properties to determine if the player won or lost.

```scala
case class State(name: String, guesses: Set[Char] = Set.empty[Char], word: String) {
    final def failures : Int = (guesses -- word.toSet).size
    final def playerLost: Boolean = failures > 10
    final def playerWon : Boolean = (word.toSet -- guesses).size == 0
}
```

So now we need a long list of words from which we will randomly pick a word which our player has to guess. For this I have put a [words.txt][5] file in my github repo which has 999 words. You can add/remove words as you like. 
You have to put this file in the src/resources folder. We need to load the contents of this file in a List and then we'll randomly pick items from this list

```scala
object Hangman extends App {
    case class State(name: String, guesses: Set[Char] = Set.empty[Char], word: String) {
        final def failures : Int = (guesses -- word.toSet).size
        final def playerLost: Boolean = failures >= 10
        final def playerWon : Boolean = (word.toSet -- guesses).size == 0
    }
    lazy val Dictionary : List[String] = scala.io.Source.fromResource("words.txt").getLines.toList
    def run(args: List[String]) : IO[Nothing, ExitStatus] = {
        hangman.redeemPure(
            _ => ExitStatus.ExitNow(1),
            _ => ExitStatus.ExitNow(0)
        )
    }

    val hangman : IO[IOException, Unit] = ???
}
```
We need a function to ask the name of the user. We do so by

```scala
val getName : IO[IOException, String] = for {
    _ <- putStrLn("What is your name: ")
    name <- getStrLn
} yield name
```

Why to write code this way rather than simply doing a `StdLn.readLine`? With this approach we are only expressing the intent to read a line. We are not actually reading a line. The code to read the line is inside of IO and will be lazily evaluated at the edge of our application. And that's the whole point of this application that we don't person side effects inside our code we wrap then in the IO data structure and then evaluate them outside of the code. Our code just returns a value of type `IO[IOException, String]` it doesn't actually read anything from the console.

The above code is a little verbose. We can shorten it by re-writing it as 

```scala
val getName : IO[IOException, String] = putStrLn("What is your name: ") *> getStrLn
```

We are telling the runtime that we don't care about the output of PutStrLn. We evaluate the expressions in a sequence but discard the value of the first expression and keep only the output of the second.

We need a function to read the character being guessed by the player. This is very similar to getName. The only difference is that we have to do validation on the input data and if the validation fails we need to show an error and then ask for input again. 

```scala
val getChoice : IO[IOException, Char] = for {
    line <- putStrLn(s"Please enter a letter") *> getStrLn
    char <- line.toLowerCase.trim.headOption match {
        case None => putStrLn(s"You did not enter a character") *> getChoice
        case Some(x) => IO.now(x)
    }
} yield char
```

We also need a functional random number generator. This is also an interesting problem. Functional programs are required to return the same output value for a specific input. Like `add(2, 2)` will always return a 4. However, what to do about the random number generator? by design it's supposed to return a different value each time it's called with the same max value.

Once again we use the IO. We are using the `sync` method to wrap the scala.util.Random into the IO. Now our program doesn't return a random number. it returns a value of type IO which lazily generates random numbers at the time of evaluation. No matter how many times you call the function above it will always return a value of type IO[Nothing, Int]. Thus, this code is referentially transparent because it returns our intent of generating a random number. Not the random number itself.

```scala
def nextInt(max: Int) : IO[Nothing, Int] = IO.sync(scala.util.Random.nextInt(max))
```

Now that we have our random number generator, we need logic to randomly pick a word from out Dictionary

```scala
val chooseWord: IO[IOException, String] = for {
    rand <- nextInt(Dictionary.length)
} yield Dictionary.lift(rand).getOrElse("Bug in the program!")
```

Since IO is a Monad we can use the scala for comprehension to unwrap the values. 

We finally need one last helper method before we start writing the logic of the game. We need a method to render the State of the game. When the game starts we show " - " for each character. the player can count these and determine the total number of characters in the word. Now each time the player guesses the character correctly we display all occurrences of that character. Others are still rendered as " - ". We also show all characters the player has guessed so far.

```scala
def renderState(state: State) : IO[IOException, Unit] = {
    val word = state.word.toList.map(c => 
        if (state.guesses.contains(c)) s" $c " else "   "
    ).mkString("")
    val line = List.fill(state.word.length)(" - ").mkString("")
    val guesses = " Guesses: " + state.guesses.mkString("")
    val text = word + "\n" + line + "\n\n" + guesses + "\n"
    putStrLn(text)
}
```

OK. So, now we have all our helper methods in place. So let's write the main logic of the game. We start the game by greeting the player and then asking the player his/her name. Once we have that we randomly pick the word which the player has to guess and we render the create and render the initial state of the game. We finally start the game loop. The game loop asks the player to guess a new character.

```scala
val hangman : IO[IOException, Unit] = for {
    _ <- putStrLn("Welcome to purely functional hangman")
    name <- getName
    _ <- putStrLn(s"Welcome $name. Let's begin!")
    word <- chooseWord
    state = State(name, Set(), word)
    _ <- renderState(state)
    _ <- gameLoop(state)
} yield()
def gameLoop(state: State) : IO[IOException, State] = ???
```

The game loop is the method which decides whether it's time to terminate the game or keep playing. If the decision is to keep playing then it recursively calls itself.

```scala
def gameLoop(state: State) : IO[IOException, State] = {
    for {
        guess <- getChoice
        state <- IO.now(state.copy(guesses = state.guesses + guess))
        _ <- renderState(state)
        loop <- if (state.playerWon) putStrLn(s"Congratulations ${state.name} you won the game!").const(false)
                else if (state.playerLost) putStrLn(s"Sorry ${state.name} you lost the game. The word was ${state.word}").map(_ => false).const(false)
                else if (state.word.contains(guess)) putStrLn(s"You guessed correctly!").const(true)
                else putStrLn(s"That's wrong. but keep trying!").const(true)
        state <- if (loop) gameLoop(state) else IO.now(state)
    } yield state
}
```

The game is pretty cute and I had a fun time playing it with my children. I hope to teach them the IO Monad :)

The complete code of this game is located in my [github][6] repo.

[1]: https://www.youtube.com/watch?v=XONTFZ4afY0
[2]: https://twitter.com/jdegoes
[3]: https://www.meetup.com/meetup-group-kyiv-scala-group/
[4]: https://github.com/scalaz/scalaz-zio
[5]: https://github.com/abhsrivastava/hangman/blob/master/src/main/resources/words.txt
[6]: https://github.com/abhsrivastava/hangman

