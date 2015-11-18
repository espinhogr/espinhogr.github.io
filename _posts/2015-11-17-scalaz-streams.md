---
title: "Scalaz streams: use case"
category: scala
tags:
 - scala
 - streams
 - functional programming
 - reactive
---

## Preface

Few days ago I came across to a use case I think is perfect to showcase how to create/use a stream. As part of my daily job I had to create a report from multiple databases sources and join together the data (ETL). In this article I will focus only on the MySQL data part to not go off-topic but this can be easily extended to any database. The problem I was trying to solve was reading the data and processing it without loading the whole dataset in memory. Streams are an amazing abstraction to do this.

## Push vs Pull Streams
When you approach streaming processing the main thing you need to know is whether you are using a push or a pull stream. Sometimes you have the choice over the stream type, sometimes you are forced to use the one your domain needs.

The main difference here is that if you imagine the classical pattern producer-consumer you have two cases:

  - _Push_ --> The producer is producing messages independently from the consumer. If the consumer is faster than the producer everything is fine, if the producer is faster then you have a problem because you are flooding the consumer and it will end with it drowning. This effect is called _backpressure_ and you have to deal with it (unless the stream implementation you're using already provides a solution for that).
  - _Pull_ <-- The producer produces new messages only when the consumer asks for them. Here the consumer has control on the flow speed therefore you don't experience _backpressure_.

Here I refer to messages as the values that your stream produces but obviously it can contain anything.

## Scalaz streams
Scalaz streams are _pull_ streams and the producer abstraction is mapped on the trait `Process[+F[_], +O]`. In this trait, type `O` is the type of values the producer produces and `F` is an effect that is triggered when new values are requested from the consumer. Usually `F` is `Task` because we want to load data asynchronously.

## Use case
To simplify the example I will read a wide number of records from MySQL and I will print them to the `stdout`. We don't care too much what we are reading and how for the purpose of the article. I'm using Slick for this example but you can actually use any library to read from the database.

First thing, we need a utility method to convert a `Future[A]` into a scalaz `Task[A]`. Let's do some chores.
{% highlight scala %}
import scalaz._
import scalaz.Scalaz._
import scalaz.concurrent.Task
import scala.concurrent.Future

object ImplicitFuture {

  implicit class FutureOps[A](val inner: Future[A]) extends AnyVal {

    // The conversion will happen in the global ExecutionContext because
    // it's trivial
    import scala.concurrent.ExecutionContext.Implicits.global

    def asScalaz: Task[A] = Task.async { cb: (Throwable \/ A ⇒ Unit) ⇒
      inner.onComplete({
        case scala.util.Failure(t) ⇒ cb(-\/(t))
        case scala.util.Success(a) ⇒ cb(\/-(a))
      })
    }
  }
}
{% endhighlight %}

This code is quite easy to understand, it basically lifts a `Future[A]` to a `FutureOps[A]` and allows you to call the method `asScalaz` on it to convert it to a `Task[A]`. If you don't know why it extends AnyVal you should read [this](http://www.scala-lang.org/api/current/index.html#scala.AnyVal), it's all about runtime performance.

When the chores are done we are ready to implement our stream. The function is recursive for convenience. I'm really sorry for the indentation but scala has really long types and I didn't want to omit them to provide a better explanation.
{% highlight scala %}
import slick.jdbc.JdbcBackend
import slick.lifted.Query
import scalaz.stream.Process

class StreamBuilder(db: JdbcBackend.Database) {

    // This says how many records you want to retrieve each time
    // you call the database and you "fill" the stream. Is called
    // PAGE_SIZE because you can imagine it as paginating the DB result.
    val PAGE_SIZE = 5000

    private[this] def executeQuery(
      query: Query[MyTable, MyRecord, Seq],
      offset: Int, prevExtract: Int): Process[Task, MyRecord] = {
        import ImplicitFuture._

        // Termination condition: if you read less than PAGE_SIZE in the
        // last call it means that you are in the last page
        if (prevExtract < PAGE_SIZE)

          // Stop producing and close the stream
          Process.halt
        else {

          // This makes a call to the DB via Slick and gets only the
          // records where your stream arrived.
          val queryFuture: Future[Seq[MyRecord]] =
            db.run(query.drop(offset).take(PAGE_SIZE).result)

          // Here we are preparing the next step for the stream, in
          // case our query fails we want to return a failure in the
          // stream, if it works we return everything that the query
          // returned and we concatenate to the next step.
          Process.awaitOr(queryFuture.result).asScalaz) {
            failure => Process.fail(failure.asThrowable)
          } {
            entries =>
            Process.emitAll(entries).asInstanceOf[Process[Task, MyRecord]] ++
              executeQuery(query, offset + PAGE_SIZE, entries.size)
          }
        }
    }

    // This is the main function, the entry point for the recursion.
    // Here you pass the query you want to execute on the DB that
    // returns the sequence of MyRecord you want in the stream.
    def selectAsStream(
      query: Query[MyTable, MyRecord, Seq]): Process[Task, MyRecord] =
        executeQuery(query, 0, PAGE_SIZE)
}
{% endhighlight %}

The code, given a `Query`, returns a `Process[Task, MyRecord]` that is a stream of `MyRecord` values. Every time the stream doesn't have any value to serve, it queries the DB and gives you the result back `PAGE_SIZE` by `PAGE_SIZE`.

Now it's time to write the code that reads from the stream.
{% highlight scala %}
import slick.jdbc.JdbcBackend
import slick.lifted.Query

object Test extends App {
  // I omit the code to create the DB connection because it's not
  // the purpose of this article
  val db: JdbcBackend.Database = createDBInstance()

  // Same thing for writing a query with Slick
  val query: Query[MyTable, MyRecord, Seq] = createQuery()

  new StreamBuilder(db).selectAsStream(query).map(println).run.run
}
{% endhighlight %}

What happens in this code is that we create a `StreamBuilder` and we tell it to create a stream for us. With the `map` function we are telling the stream that it has a new step down the line: `println`. This is a trivial step but we can obviously put anything there. With the double `run` call we start the stream and the first `Task` that produces values.
What this code produces is a printing on the `stdout` of all the records retrieved from the DB without making your heap explode in case you have a lot of them.

You might be tempted to write
{% highlight scala %}
new StreamBuilder(db).selectAsStream(query).runLog.run.foreach(println)
{% endhighlight %}
And this would make your heap blow up because in this case you are loading all the data upfront and then you are printing it. This is wrong because it goes against the concept of stream, here you are downloading the whole stream and then you are elaborating it. What you have to do is consider each step of your elaboration as a block downstream and applying it with a `map` before starting the stream itself.

## Other resources
Whist studying how scalaz stream works I appreciated a lot this [article](https://www.chrisstucchio.com/blog/2014/scalaz_streaming_tutorial.html) and also the examples you have on [scalaz codebase](https://github.com/scalaz/scalaz-stream/blob/master/src/main/scala/scalaz/stream/Process.scala).
