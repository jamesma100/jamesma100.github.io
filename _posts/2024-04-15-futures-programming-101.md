---
layout: post
title: "Futures programming 101"
---

### Race conditions
Instead of looking at how to avoid race conditions, I think a more interesting exercise would be to _try_ to trigger a race condition.

Perhaps the most obvious way is to launch futures that modify a shared variable.
Note that futures may be scheduled on multiple cores (or on different hyperthreads on the same core), so they could run in _parallel_, not just concurrently.
```
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
import scala.util.{Failure, Success}

object Increment:
  @volatile var counter: Integer = 0
  def increment(): Unit = {
    counter += 1;
  }
  def resetToZero(): Unit = {
    counter = 0;
  }
  def main(args: Array[String]): Unit = {
    for (i <- 1 to 8) {
      Future { Thread.sleep(1000); increment() }
    }
    Thread.sleep(5000) // prevent JVM shutdown
    println(s"counter value: $counter")
  }
```

To avoid this, simply synchronize the critical section on the object:
```
def increment(): Unit = {
  this.synchronized {
    counter += 1;
  }
}
```
Another way to time a race condition could be to invoke multiple callbacks on a future.
Since there is no guarantee that they run in any predefined order, nor is there any guarantee that they run on the same CPU, they could be running concurrently.
```
resetToZero()
val sleepAndDoNothing = Future { Thread.sleep(5000) }
// spin up 8 threads (I have 8 cores on my machine)
// as soon as `sleepAndDoNothing` completes, they will all be scheduled
for (i <- 1 to 8) {
  sleepAndDoNothing.foreach { Thread.sleep(1000); _ => increment() }
}
Thread.sleep(5000)
println(s"counter value: $counter")
```

### Blocking
You can start a Future computation and have the current thread to wait the result of the Future before proceeding, similar to how a parent process might fork a child process and wait for its completion.

```
import scala.concurrent.{Await, ExecutionContext, Future}
import scala.concurrent.duration.DurationInt

val f = Future {
  Thread.sleep(5000)
  println("future f has completed")
}
// wait 12 second maximum until f completes
Await.ready(f, 12.second)
```

For instance, we might launch a separate thread that does some computation and place its result into some shared queue.
The current thread might block on that computation to finish before popping from the queue to retrieve the result.
There can be many reasons we want to block - the most common one would be to prevent overwhelming the server with excessive client requests.

```
import java.util.concurrent.{BlockingQueue, LinkedBlockingQueue}

val queue = new LinkedBlockingQueue[String]()
val f = Future {
  Thread.sleep(40000)
  queue.put("result0")
}
Await.ready(f, 50000.second)
val item = queue.take()
println(s"retrieved item: $item")
```

