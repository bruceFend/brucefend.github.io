---
layout: post
title: Java Turorials 学习
category: 技术
tags: Java
keywords: Java
description: 
---


开始静下心来看Java的官方指导文档，然后是Android。

## 1、About the Java Technology

**Java technology is both a programming language and a platform.**
【Java既是一门编程语言，又是一个平台。】

### The Java Programming Language
In the Java programming language, all source code is first written in plain text files ending with the *.java* extension. 
Those source files are then compiled into *.class* files by the *javac* compiler. 
A *.class* file does not contain code that is native to your processor; it instead contains bytecodes 
— *the machine language* of the Java Virtual Machine (Java VM). 
The *java* launcher tool then runs your application with an instance of the Java Virtual Machine.


### The Java Platform

A *platform* is the hardware or software environment in which a program runs. 
We've already mentioned some of the most popular platforms like Microsoft Windows, Linux, Solaris OS, and Mac OS. 
Most platforms can be described as a combination of the operating system and underlying hardware. 
The Java platform differs from most other platforms in that it's a software-only platform that runs on top of 
other hardware-based platforms.The Java platform has two components:

- The *Java Virtual Machine*
- The *Java Application Programming Interface* (API)

![The API and Java Virtual Machine insulate the program from the underlying hardware.](http://docs.oracle.com/javase/tutorial/figures/getStarted/getStarted-jvm.gif)

## 2、[Object-Oriented Programming Concepts](http://docs.oracle.com/javase/tutorial/java/concepts/index.html)

### Object
Software objects are conceptually similar to real-world objects: they too consist of **state** and related **behavior**. An object stores its state in *fields* (variables in some programming languages) and exposes its behavior through *methods* (functions in some programming languages). 

Methods operate on an object's internal state and serve as the primary mechanism for object-to-object communication. Hiding internal state and requiring all interaction to be performed through an object's methods is known as **data encapsulation** — a fundamental principle of object-oriented programming.

### Class
 A **class** is the *blueprint* from which individual objects are created.

### Inheritance
Object-oriented programming allows classes to **inherit** commonly used state and behavior from other classes. 
In the Java programming language, each class is allowed to have *one* direct superclass, and each superclass has the potential for an *unlimited* number of subclasses:

![2](http://docs.oracle.com/javase/tutorial/figures/java/concepts-bikeHierarchy.gif)

### Interface
In its most common form, an interface is *a group of related **methods*** with empty bodies.

Implementing an interface allows a class to become more formal about the behavior it promises to provide. Interfaces form a contract between the class and the outside world, and this contract is enforced at build time by the compiler. If your class claims to implement an interface, all methods defined by that interface must appear in its source code before the class will successfully compile.

### Package
A package is a namespace that organizes a set of related classes and interfaces. 

## 3、[Essential Classes](http://docs.oracle.com/javase/tutorial/essential/index.html)

### 3.1 Exception

**Definition**: An *exception* is an event, which occurs during the execution of a program, that disrupts the normal flow of the program's instructions.

#### 1. Check Exception
These are exceptional conditions that a well-written application **should anticipate** and recover from.
#### 2. Error
These are exceptional conditions that are **external** to the application, and that the application usually **cannot** anticipate or recover from. 
#### 3. Runtime Exception
These are exceptional conditions that are internal to the application, and that the application **usually cannot** anticipate or recover from. 


### 3.1.1 [The finally Block](http://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html)
【finally部分的代码，不一定会被执行。关键是看该线程是否还“活着”】

**The *finally* block *always* executes when the try block exits.**
This ensures that the *finally* block is executed even if an unexpected exception occurs. 

But *finally* is useful for more than just exception handling — it allows the programmer to avoid having cleanup code accidentally bypassed by a *return*, *continue*, or *break*. Putting cleanup code in a *finally* block is always a good practice, even when no exceptions are anticipated.

---
**Note**: If the JVM exits while the *try* or *catch* code is being executed, 
then the *finally* block **may not** execute. 

Likewise, if the thread executing the *try* or *catch* code is interrupted or killed, 
the *finally* block **may not** execute even though the application as a whole continues.

> 当一个线程在执行 *try* 语句块或者 *catch* 语句块时被打断（interrupted）或者被终止（killed），与其相对应的 *finally* 语句块可能不会执行

---

The runtime system **always executes** the statements within the *finally* block regardless of what happens within the *try* block. So it's the perfect place to perform cleanup.

*finally* 语句块应该是**在控制转移语句之前执行**，控制转移语句除了 *return* 外，还有 *break* 和 *continue*。另外，*throw* 语句也属于控制转移语句。
<font color=red>**重要**</font>

虽然 return、throw、break 和 continue 都是控制转移语句，但是它们之间是有区别的。
其中 return 和 throw **把程序控制权转交给它们的调用者（invoker）**，
而 break 和 continue 的控制权是**在当前方法内转移**。

### 3.1.2 [The try-with-resources Statement](http://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
The *try-with-resources* statement is a try statement that declares one or more resources. A *resource* is an object that **must be closed** after the program is finished with it. 

【可以自动关闭对应的资源，如文件流等】

The *try-with-resources* statement ensures that each *resource* is closed at the end of the statement. Any object that implements *java.lang.AutoCloseable*, which includes all objects which implement *java.io.Closeable*, can be used as a resource.


```
static String readFirstLineFromFile(String 
path) throws IOException {

    **try (BufferedReader br =**

                   **new BufferedReader(new FileReader(path))) {**
        return br.readLine();
    }
}
```

### 3.1.3 [How to Throw Exceptions](http://docs.oracle.com/javase/tutorial/essential/exceptions/throwing.html)

![3](http://docs.oracle.com/javase/tutorial/figures/essential/exceptions-throwable.gif)


### 3.1.4 [Unchecked Exceptions — The Controversy](http://docs.oracle.com/javase/tutorial/essential/exceptions/runtime.html)
【**未经检查的错误，要尽量避免**】

One *Exception* subclass, *RuntimeException*, is reserved for exceptions that indicate incorrect use of an API. An example of a runtime exception is *NullPointerException*, which occurs when a method tries to access a member of an object through a *null* reference. 

The section Unchecked Exceptions — The Controversy discusses why **most applications shouldn't throw runtime exceptions or subclass *RuntimeException*.**

The next question might be: "If it's so good to document a method's API, including the exceptions it can throw, why not specify runtime exceptions too?" 

++Runtime exceptions represent problems that are the result of a programming problem, and as such, **the API client code cannot reasonably be expected to recover from them or to handle them in any way**.++ 

【 **几种常见的 *RuntimeException*** 】

Such problems include **arithmetic exceptions**, such as dividing by zero; **pointer exceptions**, such as trying to access an object through a null reference; and **indexing exceptions**, such as attempting to access an array element through an index that is too large or too small.

Runtime exceptions can occur anywhere in a program, and in a typical one they can be very numerous. **Having to add runtime exceptions in every method declaration would reduce a program's clarity.** Thus, the compiler does not require that you catch or specify runtime exceptions (although you can).

**Generally speaking, do not throw a RuntimeException or create a subclass of RuntimeException simply because you don't want to be bothered with specifying the exceptions your methods can throw.**

### 3.1.5 [Advantages of Exceptions](http://docs.oracle.com/javase/tutorial/essential/exceptions/advantages.html)

#### 1.Separating Error-Handling Code from "Regular" Code

#### 2.Propagating Errors Up the Call Stack

#### 3.Grouping and Differentiating Error Types
