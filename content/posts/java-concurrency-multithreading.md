---
title: 'Java Concurrency Multithreading'
date: 2024-07-23T20:25:44-04:00
draft: false
tags:
  - java
  - concurrency
  - multithreading
  - interview
categories: notes
keywords:
  - java
  - concurrency
  - multithreading
---

## Thread

### Thread Subclass

Here is an example of creating a Java `Thread` subclass:

```java
public class MyThread extends Thread {
	public void run(){
		System.out.println("MyThread running");
	}
}
```

To create and start the above thread you can do like this:

```java
MyThread myThread = new MyThread();
myTread.start();
```

You can also create an anonymous subclass of Thread like this:

```java
Thread thread = new Thread(){
	public void run(){
		System.out.println("Thread Running");
	}
}

thread.start();
```

### Runnable Interface Implementation

#### Java Class Implements Runnable

```java
public class MyRunnable implements Runnable {
  public void run(){
    System.out.println("MyRunnable running");
  }
}
```

#### Anonymous Implementation of Runnable

```java
Runnable myRunnable =
    new Runnable(){
        public void run(){
            System.out.println("Runnable running");
        }
    }
```

#### Java Lambda Implementation of Runnable

```java
Runnable runnable =
        () -> { System.out.println("Lambda Runnable running"); };
```

#### Starting a Thread With a Runnable

```java
Runnable runnable = new MyRunnable(); // or an anonymous class, or lambda...

Thread thread = new Thread(runnable);
thread.start();
```

### The Java Memory Model

Here is a diagram illustrating the call stack and local variables stored on the thread stacks, and objects stored on the heap:

![memory model](java-memory-model.png)

Here is a simplified diagram of modern computer hardware architecture:

![hardware memory](hardware-memory-architecture.png)

#### Bridging The Gap Between The Java Memory Model And The Hardware Memory Architecture

The hardware memory architecture does not distinguish between thread stacks and heap. On the hardware, both the thread stack and the heap are located in main memory.

![hardware-memory-gap](hardwaare-memory-gap.png)

When objects and variables can be stored in various different memory areas in the computer, certain problems may occur. The two main problems are:

- Visibility of thread updates (writes) to shared variables.
- Race conditions when reading, checking and writing shared variables.

#### Visibility of Shared Objects

If two or more threads are sharing an object, without the proper use of either volatile declarations or synchronization, updates to the shared object made by one thread may not be visible to other threads.

![variable-visibility](variable-visibility.png)

The `volatile` keyword can make sure that a given variable is read directly from main memory, and always written back to main memory when updated.

### Race Conditions

If two or more threads share an object, and more than one thread updates variables in that shared object, race conditions may occur.

### Synchronized keyword, blocks and methods in Java

- > **Synchronized blocks are reentrant in Java.**

- > **Synchronized blocks can only blocks threads running on same virtual machine.**

#### Limitations

- Only ne thread can enter a synchronized block at a time.
- There is no guarantee about the sequence in which waiting thread gets access to the synchronized block.
  - starvation is possible

#### Performance Overhead

- **Low overhead** - when sync block is uncontested (not already locked)
- **Higher overhead** - when sync block is contested (already locked by another thread)

#### Synchronized Instance methods

Uses `MyCounter` **instance** as monitor object.

```java
public class MyCounter {

  private int count = 0;

  public synchronized void add(int value){
      this.count += value;
  }
  public synchronized void subtract(int value){
      this.count -= value;
  }
}
```

#### Synchronized Static Methods

Uses `MyCounter.class` as monitor object.

```java
public static MyStaticCounter{

  private static int count = 0;

  public static synchronized void add(int value){
    count += value;
  }

  public static synchronized void subtract(int value){
    count -= value;
  }
}
```

#### Synchronized Blocks in Instance Methods

- Using instance object as monitor object.

```java
public void add(int value){

    synchronized(this){
       this.count += value;
    }
}
```

The following two examples are both synchronized on the instance they are called on. They are therefore equivalent with respect to synchronization:

```java
public class MyClass {

    public synchronized void log1(String msg1, String msg2){
       log.writeln(msg1);
       log.writeln(msg2);
    }


    public void log2(String msg1, String msg2){
       synchronized(this){
          log.writeln(msg1);
          log.writeln(msg2);
       }
    }
}
```

#### Synchronized Blocks in Static Methods

These methods are synchronized on the class object of the class the methods belong to:

```java
public class MyClass {

    public static synchronized void log1(String msg1, String msg2){
       log.writeln(msg1);
       log.writeln(msg2);
    }


    public static void log2(String msg1, String msg2){
       synchronized(MyClass.class){
          log.writeln(msg1);
          log.writeln(msg2);
       }
    }
}
```

#### Synchronized Blocks in Lambda Expressions

It is even possible to use synchronized blocks inside a Java Lambda Expression as well as inside anonymous classes.

```java
import java.util.function.Consumer;

public class SynchronizedExample {

  public static void main(String[] args) {

    Consumer<String> func = (String param) -> {

      synchronized(SynchronizedExample.class) {
        System.out.println(
            Thread.currentThread().getName() +
                    " step 1: " + param);
        try {
          Thread.sleep( (long) (Math.random() * 1000));
        } catch (InterruptedException e) {
          e.printStackTrace();
        }

        System.out.println(
            Thread.currentThread().getName() +
                    " step 2: " + param);
      }
    };

    Thread thread1 = new Thread(() -> {
        func.accept("Parameter");
    }, "Thread 1");

    Thread thread2 = new Thread(() -> {
        func.accept("Parameter");
    }, "Thread 2");

    thread1.start();
    thread2.start();
  }
}
```

### Volatile Keyword

- Use `volatile` for flags or variables that signal events or state changes between threads.
- It provides visibility guarantees, but not atomicity (indivisibility) for complex operations. If atomicity is required, consider synchronization mechanisms like `synchronized`.
- `volatile` can improve performance compared to synchronization in scenarios where only visibility is needed.

In this below scenario, there's a chance that the consumer thread might keep reading an outdated value of `finished` from its local CPU cache due to compiler optimizations. This could lead to the consumer waiting indefinitely even though the producer has already set the flag to `true`.

```java
public class TaskCompletion {
    private boolean finished = false; // Not volatile

    public void setFinished() {
        finished = true; // Producer sets the flag
    }

    public boolean isFinished() {
        return finished; // Consumer checks the flag
    }

    public static void main(String[] args) {
        TaskCompletion taskCompletion = new TaskCompletion();

        Thread producer = new Thread(() -> taskCompletion.setFinished());
        Thread consumer = new Thread(() -> {
            while (!taskCompletion.isFinished()) {
                // Busy waiting (inefficient)
            }
            System.out.println("Task completed!");
        });

        producer.start();
        consumer.start();
    }
}
```

### ThreadLocal

The Java `ThreadLocal` class enables you to create variables that can only be read and written by the same thread. Thus, even if two threads are executing the same code, and the code has a reference to the same `ThreadLocal` variable, the two threads cannot see each other's `ThreadLocal` variables. Thus, the Java ThreadLocal class provides a simple way to make code **thread safe** that would not otherwise be so.

```java
ThreadLocal<String> threadLocal = new ThreadLocal<>();

Thread thread1 = new Thread(() -> {
	threadLocal.set("Thread 1"); // setting only sets in the current thread
	System.out.println(threadLocal.get());
	threadLocal.remove(); // removing only removes from the current thread
	System.out.println(threadLocal.get());
});
Thread thread2 = new Thread(() -> {
	threadLocal.set("Thread 2");
	System.out.println(threadLocal.get());
	try {
		Thread.sleep(1000);
	} catch (InterruptedException e) {
	}
	System.out.println(threadLocal.get());
	threadLocal.remove();
	System.out.println(threadLocal.get());
});

thread1.start();
thread2.start();
```

```java
public class MyDateFormatter {

    private ThreadLocal<SimpleDateFormat> simpleDateFormatThreadLocal = new ThreadLocal<>();

    public String format(Date date) {
        SimpleDateFormat simpleDateFormat = getThreadLocalSimpleDateFormat();
        return simpleDateFormat.format(date);
    }


    private SimpleDateFormat getThreadLocalSimpleDateFormat() {
        SimpleDateFormat simpleDateFormat = simpleDateFormatThreadLocal.get();
        if(simpleDateFormat == null) {
            simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            simpleDateFormatThreadLocal.set(simpleDateFormat);
        }
        return simpleDateFormat;
    }
}
```

### Thread Pool

A thread pool is a pool threads that can be "reused" to execute tasks, so that each thread may execute more than one task. A thread pool is an alternative to creating a new thread for each task you need to execute.

![thread-pool](thread-pool.png)

### Locks Introduction

The Java `Lock` interface, `java.util.concurrent.locks.Lock`, represents a concurrent lock which can be used to guard against race conditions inside critical sections.

```java
class Example {

	public static void main(String[] args) {
		Lock lock = new ReentrantLock();
		lock.lock();
		// do something
		lock.unlock();
	}
}
```

#### Fail-safe Lock and Unlock

```java
Lock lock = new ReentrantLock();

try{
    lock.lock();
      //critical section
} finally {
    lock.unlock();
}
```

### Java ExecutorService

```java
ExecutorService executorService = Executors.newFixedThreadPool(10);

executorService.execute(new Runnable() {
    public void run() {
        System.out.println("Asynchronous task");
    }
});

executorService.shutdown();
```

#### Java ExecutorService Implementations

Since ExecutorService is an interface, you need to its implementations in order to make any use of it. The ExecutorService has the following implementation in the java.util.concurrent package:

- ThreadPoolExecutor
- ScheduledThreadPoolExecutor

#### Creating an ExecutorService

```java
ExecutorService executorService1 = Executors.newSingleThreadExecutor();
ExecutorService executorService2 = Executors.newFixedThreadPool(10);
ExecutorService executorService3 = Executors.newScheduledThreadPool(10);
```
