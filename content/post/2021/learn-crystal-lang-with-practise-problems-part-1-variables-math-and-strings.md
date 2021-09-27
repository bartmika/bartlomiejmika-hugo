---
title: "Learn Crystal with Practise Problems - Part 1: Variables Math and Strings"
date: 2021-09-08T23:38:40-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Development"
tags:
- "Crystal-Lang"
---

![](/img/2021/09-08/jason-d-k65_6C4hu2E-unsplash.jpg)

Are you interested in learning [#Crystal-Lang](/tags/crystal-lang/) but having difficulty because you are a [tactile learner](https://en.wikipedia.org/wiki/Kinesthetic_learning)? The purpose of this tutorial is to provide you practise problems you can try out at your leisure.

<!--more-->

Recently on [HackerNews](https://news.ycombinator.com/item?id=27903288) I discovered **crystal-lang**, or sometimes shortened down to just **crystal**, a programming language that fancies itself as *slick as ruby and fast as C* had just become versioned to **1.1.0** via the post titled ["Crystal 1.1.0 is released! (2021)"](https://crystal-lang.org/2021/07/16/1.1.0-released.html). Now that crystal-lang is becoming production ready and becoming more adapted by industry, I thought about learning this language.

The practise problems in this post are to be tried after you have installed the **crystal-lang** compiler and read the following developer docs:

* [Getting Started](https://crystal-lang.org/reference/getting_started/index.html)
* [Hello World](https://crystal-lang.org/reference/tutorials/basics/10_hello_world.html)
* [Variables](https://crystal-lang.org/reference/tutorials/basics/20_variables.html)
* [Math](https://crystal-lang.org/reference/tutorials/basics/30_math.html)
* [Strings](https://crystal-lang.org/reference/tutorials/basics/40_strings.html)
* [Control Flow](https://crystal-lang.org/reference/tutorials/basics/50_control_flow.html)
* [Methods](https://crystal-lang.org/reference/tutorials/basics/60_methods.html)

**Absolutely do not proceed if you have not read the above! You have been warned.**.

According to the documentation, once you finish reading the [tutorials](https://crystal-lang.org/reference/tutorials/basics/index.html) section, you are to move on to reading the [language specifications](https://crystal-lang.org/reference/syntax_and_semantics/index.html) section. This is fine but I like to play with the language as I learn to cement the fundamentals; as a result, I am writing this series of blog posts with some practise problems you can try out to assist you with learning **crystal-lang**.

With all this mind, let's begin!

## Before we begin ... let's talk about flowcharts

What is a flowchart? We define it as follows:

> A flowchart is a diagram that represents all the necessary steps for solving a specific task.
> - [Introduction to computer programming with flowcharts](https://joaoventura.net/blog/2020/flowcharts/)

Flowcharts are a valuable technical communication tool I'd like to use for you here. [Would you like to learn more](https://joaoventura.net/blog/2020/flowcharts/)?

# Exercise Problems
## Problem #1
### Description:
Please write a console application which will accept your name as an input, and print out your name back to the console.

### Flowchart:
![Problem #1](/img/2021/09-08/learn-crystal-part1-p1.jpg)

### Learning Notes:
- Notice the developer documentation you read did not mention how to get data from the console? This will require some leg work on your end!
- Don't forget to use [*interpolation*](https://crystal-lang.org/reference/tutorials/basics/40_strings.html#interpolation).

### Solution:
Try to write the code yourself and when you are ready, look below to see the solution I wrote. To execute this code, run the following in your terminal:

```bash
$ crystal p1.cr
```

And the source code is as follows:

{{< highlight crystal "linenos=false">}}
# p1.cr
#------
# The purpose of this program is to accept user name from the console and
# print the name back out to the console.

puts "Please enter your name"

name = gets

puts "Hi #{name}"
{{</ highlight >}}

## Problem #2
### Description
Please write a console application which will accept only a single *integer*, multiple it by 2, and print the result.

### Flowchart:
![Problem #1](/img/2021/09-08/learn-crystal-part1-p2.jpg)

### Learning Notes:
#### Note 1 - How to do type conversions?
If you started writing something like this:

```
puts "Please any integer value"

i = gets

i = i * 2

puts "#{i}"
```

You may notice you get the following error when compiling:

![Problem #1](/img/2021/09-08/learn-crystal-part1-p2-n1.png)

We run into our first problem because we did not read the [language specifications](https://crystal-lang.org/reference/syntax_and_semantics/index.html) - The [`gets`](https://crystal-lang.org/api/1.1.1/IO.html#gets(chomp=true):String?-instance-method) functions returns a `(String | null)` data type while we need `(Int)`.

To move forward:

* We need to convert from `(String | Null)` to `(String)` data type. [See API docs](https://crystal-lang.org/api/1.1.1/String.html#to_s:String-instance-method).
* We need to convert `(String)` to `(Int32)` data type. [See API docs](https://crystal-lang.org/api/1.1.1/String.html#to_i(base:Int=10,whitespace:Bool=true,underscore:Bool=false,prefix:Bool=false,strict:Bool=true,leading_zero_is_octal:Bool=false)-instance-method).

**Before proceeding any further, please read the entire ["type reflections"](https://crystal-lang.org/reference/syntax_and_semantics/type_reflection.html#type-reflection) section.**

As a result our code will look as follows:

```
puts "Please any integer value"

i = gets

i = i.to_s.to_i  # Force convert (String | Nil) to (String) then (Int32).
i = i * 2

puts "#{i}"
```

If you run that code you will see it working.

#### Note 2 - How to capture non-integers?

If you run your code and input an alphabetic value (ex: "A") then you'll notice the following error:

![Problem #1](/img/2021/09-08/learn-crystal-part1-p2-n2.png)

As a result, you'll need to learn [`exception handling`](https://crystal-lang.org/reference/syntax_and_semantics/exception_handling.html) from the developers documentation and try applying it what you learn.

**Before proceeding any further, please read the entire ["exception handling"](https://crystal-lang.org/reference/syntax_and_semantics/exception_handling.html) section.**

Once you finish reading, you should be able to apply correct fix to get the solution as follows...

### Solution:
{{< highlight crystal "linenos=false">}}
puts "Please any integer value"

i = gets

begin
    i = i.to_s.to_i  # Force convert (String | Nil) to String then Int32.
    i = i * 2
    puts "#{i}"
rescue
    puts "Not integer"
end
{{</ highlight >}}


## Problem #3
### Description
Please write a console application which will ask for your name, with simple validation, and print your name back into the console.

### Flowchart:
![Problem #1](/img/2021/09-08/learn-crystal-part1-p3.jpg)

### Learning Notes:
#### Note 1 - How do we repeatedly execute code?

How do we handle repetition or looping? We'll need some-sort of `for`, `do/while`, or `while` syntax to enable our code to repeat itself. Looking through the [language specifications](https://crystal-lang.org/reference/syntax_and_semantics/index.html), we see a [`while`](https://crystal-lang.org/reference/syntax_and_semantics/while.html) syntax is available to us!

**Before proceeding any further, please read the entire ["while loop"](https://crystal-lang.org/reference/syntax_and_semantics/while.html) section.**

### Solution:
{{< highlight crystal "linenos=false">}}
name = ""

while name == ""
    puts "Please enter name"
    name = gets

    name = name.to_s # Force (String | Null) to (String) format

    if name.empty?
        puts "Error - no name entered"
    end
end

puts "Hello #{name}"
{{</ highlight >}}

## Problem #4
### Description
Please write an application which will calculate *simple interest*. According to [Investopedia](https://www.investopedia.com/terms/s/simple_interest.asp), simple interest is defined as follows:

> Simple Interest = P × I × N
>
> where:
>
> P = principle
>
> I = daily interest rate
>
> N = number of days between payments
​
### Solution:
{{< highlight crystal "linenos=false">}}
# All numbers and text copied from:
# https://www.investopedia.com/terms/s/simple_interest.asp
#
# For example, say a student obtains a simple-interest loan to pay one year of
# college tuition, which costs $18,000, and the annual interest rate on the loan
# is 6%. The student repays the loan over three years.
# The amount of simple interest paid is:

p = 18_000
i = 0.06 # Note: Convert from percent (6%) to rate (0.06) for our computation.
n = 3

si = p * i * n

puts "Interest #{si}"

total = p + si

puts "Total Debt #{total}"
{{</ highlight >}}

# Final Notes:
Cover photo by [Harrison Broadbent](https://unsplash.com/@harrisonbroadbent) on [Unsplash](https://unsplash.com).
