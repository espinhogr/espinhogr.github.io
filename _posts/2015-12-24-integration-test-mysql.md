---
title: "Integration test with Docker"
category: test
tags:
 - test
 - integration
 - MySQL
 - docker
 - dao
---

## How should I test a Data Access Object (DAO)? ##

The answer to this question has always been a big doubt for me in the past. I'm sure everybody sooner or later reached the point where you need to test some code interacting with the DB. Unit testing code that talks with other code is quite easy (depending on how well you architect your application) but when you reach the borders, testing the interaction with other systems gets more complex. For instance let's say you are writing Java code using JDBC, what should you test about a method executing a query on the DB and returning the `ResultSet`? Should you test the `executeQuery` method has been called with the right query? Should you mock the `Statement` object to return the `ResultSet` you expect? All these approaches looks silly to me, both of them are not actually testing the query I'm using works, they just make sure I wrote the query as _I suppose_ works and the DB is returning what _I suppose_ will return. If you're thinking

> "That's vert true but in my method I have a more complex logic to test"

then maybe you are missing an abstraction layer in your code.   
The way to go in my opinion is to test your code against a real instance, make sure that your code does what it is supposed to do.


### Sub-optimal solutions ###

If you search online, you can see a lot of resources telling you to use H2 but what I don't like is that if I want to test apples I don't want to test oranges, I know it's a good approximation but I want to test apples! Writing code is similar to [Rock Balancing](https://en.wikipedia.org/wiki/Rock_balancing), it's an art, get something wrong and everything can collapse.   
Someone, instead likes to write stubs and recreate the behaviour of the external system you are testing against to. This is something that doesn't make sense in my opinion apart for really sporadic cases. You should focus more on implementing something new rather than implementing something that already exists.

## Solution with Docker ##

As the use of containers became more and more invasive I said:

>"Why not using Docker for integration tests?"

It provides everything we need:

- __Repeatable tests__ For each and every build we have a dedicated empty MySQL instance
- __Real environment__ We test against the same version we have in production
- __No dedicated hardware__ The docker image runs on the same machine you execute the tests 
- __Easy integration__ Interacting with docker is quite easy and you need only few lines of code when you understand how it works.

The amazing part of this approach is that it's not limited to MySQL but it can be used to test against any external system you have a docker image of, the configuration is pretty similar.   
Compared to H2 approach, using MySQL directly gives you also the advantage of being able to test SQL procedures/triggers you have in your DB (even though I'm not a big fan of them).

## How to implement it ##

The fastest ways to integrate with docker I found online were the following:

- __Via Maven__
- __Programmatically__

I think making the phase of running a docker image as part of the building process is quite interesting, therefore using the Maven integration seemed the best option. Another advantage of this approach is that you can use different profiles for different operating systems.

> In OSX, docker starts a VirtualBox VM where a linux instance is hosted. This will be the place where your images will run, only on linux your container will share the guest-box resources. You need to consider this when you want to access a system via network protocol as we will see later.

