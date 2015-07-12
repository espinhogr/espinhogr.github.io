---
title: Trampolines in scala for newbies
category: scala
tags:
 - scala
 - trampolines
 - functional programming
 - monad
 - tail recursion
---

## Preface 

This post doesn't want to be a comprehensive explanation of what Tramplines are in functional-programming/scala but only an overview on how to use them and a starting point to study more and have a deeper understanding.
I'm writing what I learned from my personal experience with the hope to help people that are trying to learn scala coming from a java world. I will be
intentionally sloppy to not make things heavy and to easy the learning process.

## What are trampolines?

Trampolines are one of the several way you can use a Free Monad. At the moment you don't have to worry too much about it. This is just to let you know that
you will use a Monad in the article but with as less pain as possible.

## When do I need a Trampoline?

One of the cases in which you may need a Trampoline is to jump in a pool, but I think this is not why you are here. 
Sticking to the programming context, you may need it if you are using recursion in your scala code. As long as you use tail recursion with one function 
everything is fine as the compiler will optimize it for you and transforme it in a loop, issues come when you are trying to use tail recursive calls between two
functions or when you don't have a tail recursive call.

The case of having two functions is well explained in [scalaz documentation](http://eed3si9n.com/learning-scalaz/Stackless+Scala+with+Free+Monads.html)
but I haven't found any example of creating a Trampoline over a non-tail recursion, that's why I'm writing this post.

In few words you need a Trampoline whenever your code could blow up your JVM stack because the depth of recursion.

## Why does the Trampoline solve this problem?

The trampoline solves this problem because it basically builds an in-memory structure that can be solved with a tail recursion (that can be optimized to a loop). 
Without entering the details, it does the magic by not executing the function you are calling but concatenating them and executing them in a "loop" afterward. 
This deferring - in scalaz - is called _suspension_ because you are suspending the execution of that function and you will _resume_ it later. `resume` is
the function that does the magic.

## Use case

### The problem

We are trying to sum 1 to each element of a list and we expect to have another list as output of our algorithm. To be clear if we enter `List(1,2)` we expect as
output another list `List(2,3)`. We solve this problem with a non-tail recursive call in order to highlight the problem. Here is the code:

{% highlight scala %}
object TramplineTest extends App {
  println(recursiveFunction((1 to 2).toList))

  def recursiveFunction(l:List[Int]): List[Int] = {
    if(l.isEmpty) Nil
    else l.head + 1 :: recursiveFunction(l.tail)
  }
}
{% endhighlight %}

This code works fine for a small amount of element in the list but as soon as we increase the number of elements it blows up the stack. In this very moment we have to think about using a `Trampoline`.

### Walking toward the solution

As we said before, the Trampoline is a particular case of a Free Monad and this leads us to the Free monad implementation. If you are curious you can look into
scalaz sourcecode [Free.scala](https://github.com/scalaz/scalaz/blob/series/7.2.x/core/src/main/scala/scalaz/Free.scala) and see that a Free Monad has only 3 
instances that you can use to create a trampoline:
	
- `Return`: This just contains a value that you want to return, therefore you usually use it at the leaves of your recursion.
- `Suspend`: This contains a step of a computation that you want to execute. Technically it contains a function with arity 0 that will be executed later. You need to use this
 whenever you are calling your function recursively.
- `GoSub`: This is the most tricky one, it basically contains two objects: the next step to execute and a function to apply over that step. This is extremely useful
 if you need to do something after your function returned (i.e. `l.head + 1 ::`).

Obviously the wise developer of scalaz made this instances private and you cannot use them directly. This is why you have the following functions:

- `done`: Creates a Return object
- `suspend`: Creates a Suspend object
- `flatMap`: Creates a GoSub object

With the functions just presented you can create the equivalent of the AST for your recursion but you still need a way to interpret it in order to have the
result. Luckily this is not something that you have to implement, it's already there and it's a method of `Trampoline` class. The `run` method. Calling this 
method you will automatically execute the code and behind the scene calling the `resume` method.

### The solution

This is how we can use the knowledge we gained to solve the recursion problem.

{% highlight scala %}
import scalaz.Free.Trampoline
import scalaz.Trampoline

object TramplineTest extends App {

  println(recursiveFunction((1 to 3000000).toList).run)

  def recursiveFunction(l:List[Int]): Trampoline[List[Int]] = {
    if(l.isEmpty)
      Trampoline.done(Nil)
    else {
      Trampoline.suspend(recursiveFunction(l.tail)).flatMap { list =>
        Trampoline.done(l.head + 1 :: list)
      }
    }
  }
}
{% endhighlight %}

As you can see we just applied the functions that we discussed before `done`, `suspend` and `flatMap`, we declared the returning type of the function
as a `Trampoline` and then we called `run` on the root call.

Hope this helps newbies like me to understand better `Trampolines` and how to use them.


