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
Scalaz streams are _pull_ streams and the producer abstraction is mapped on the `Process[+F[_], +O]` trait. In the trait type `O` is the type of values the producer produces and `F` is an effect that is triggered when new values are requested from the consumer. Usually `F` is `Task` because we want to load data asynchronously. 


## Other resources
Whist studying how scalaz stream work I appreciated a lot this [article](https://www.chrisstucchio.com/blog/2014/scalaz_streaming_tutorial.html) and also the examples you have on [scalaz codebase](https://github.com/scalaz/scalaz-stream/blob/master/src/main/scala/scalaz/stream/Process.scala).
