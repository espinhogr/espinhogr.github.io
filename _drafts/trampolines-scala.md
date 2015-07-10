---
title: Trampolines in scala for newbies
---

This is a test post

{% highlight scala %}
import scalaz.Free.Trampoline
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
