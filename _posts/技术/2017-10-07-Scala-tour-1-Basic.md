---
layout: post
title: Scala官方tour备忘-1
category: 技术
tags: Scala,tour
keywords: Scala
description: 
---

Here is the [Tutor Basic](http://docs.scala-lang.org/tour/basics.html).

Go to https://scalafiddle.io.

#### <font color='#DC322F'>Expressions

- Values (***Values cannot be re-assigned.***)
```
val x = 1+1  //equals val x: Int = 1 + 1
println(x)  //2
x = 3   // This does not compile.
```

- Variables
```
var x = 1+1
x = 3   // This compiles because "x" is declared with the "var" keyword.
println(x*x)    //9
```

#### Blocks
```
println({
    val x = 1+1
    x + 1
)}  //3
```

#### Functions
```
val addOne = (x:Int) => x+1
println(addOne(1))  //2

//multiple parameters.
val add = (x:Int, y:Int) => x+y
println(add(1,2))   //3

//no parameters.
//来自《银河漫游指南》的梗
val getTheAnswer=() => 42
println(getTheAnswer()) //42
```

#### Methods

> //method的定义用def的关键字，function不用 
>
> Methods are defined with the `def` keyword. `def` is followed by a *name*, *parameter lists*, a *return type*, and a *body*.

```Scala
def add(x:Int, y:Int):Int = x+y
println(add(1,2))   //3

//multiple parameter lists.
def addThenMultiply(x:Int, y:Int)(multiplier:Int):Int = {
    (x+y)*multiplier
}
println(addThenMultiply(1,2)(3))    //9
```

#### Classes

//返回值类型是Unit，Unit是什么类型？【见第二部分】

//所有的方法都需要有返回值，有点类似Java里面的void？
> `Unit` is a value type which carries ***no meaningful information***. There is exactly one instance of Unit which can be declared literally like so: `()`. All functions ***must return*** something so sometimes Unit is a useful return type. 

```Scala
class Greeter(prefix:String, suffix:String){
    def greet(name:String): Unit =
        println(prefix + name + suffix)
}

val greeter = new Greeter("hello, ", "!")
greeter.greet("Scala developer")    //hello, Scala developer!
```

#### Case Classes [importance]

> Scala has a special type of class called a “case” class. By default, case classes are immutable and compared by value. You can define case classes with the `case class` keywords.
>
> You can instantiate case classes without new keyword.

这里看tour的内容，不是很理解

#### Objects [importance]

//可以把Object当做是对应类的单例
> Objects are single instances of their own definitions. You can think of them as singletons of their own classes.

```Scala
object IdFactory{
    private var counter = 0
    def create():Int = {
        counter += 1
        counter
    }
}

val newId:Int = IdFactory.create()
println(newId)  //1
val newerId = IdFactory.create()
println(newerId)    //2
```

#### Traits [importance]

//带方法体的接口？
> Traits are types containing certain fields and methods. Multiple traits can be combined.

```Scala
trait Greeter{
    def greet(name:String): Unit = 
        println("hello, " + name + "!")
}

class DefaultGreeter extends Greeter

class CustomizableGreeter(prefix:String, postfix:String) extends Greeter {
    //重新方法
    override def greet(name:String): Unit = {
        println(prefix + name + postfix)
    }
}

val greeter = new DefaultGreeter()
greeter.greet("Scala developer")    //hello, Scala developer!

val customGreeter = new CustomizableGreeter("How are you,", "?")
customGreeter.greet("Scala developer")  //How are you, Scala developer?
```

#### Main Method

> The main method is an entry point of a program. The Java Virtual Machine requires a main method to be named `main` and take one argument, an array of strings.

```
object Main {
    def main(args: Array[String]): Unit =
        println("hello, Scala developer!")
}
```
