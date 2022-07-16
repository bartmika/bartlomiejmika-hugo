---
title: "Useful Resources that Helped me Learn Golang over the Years"
date: 2021-04-08T00:29:06-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "Golang"
cover:
    image: "/img/2021/07-09/red_mulberries_germination_with_sparkfun_weather_shield.jpg"
    alt: ""
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---

The purpose of this post is to share my catalogue of useful links I've gathered in my quest to better understand Golang. If I've used a webpage to solve a problem, or if I used a link to learn something, then you'll find a link of it here.

<!--more-->

*Please note this is an article I'll be periodically updating so feel free to check back often. If you find any links you've used which you would like to recommend, please don't hesitate and write them in the comments below. Also please note, this list is my personal list and is by no means any definitive list.*

# Basics - Learn Golang from Scratch
### golangbot.com
This is *the* resource I used to learn Golang from scratch. When asked were did you learn Golang, I *always* recommend this resource. Visit webpage via [golangbot.com](https://golangbot.com/learn-golang-series/).

### Honourable Mentions:
These look like very promising resources I have not had enough time to read in-depth:
- [Learn go with Tests](https://quii.gitbook.io/learn-go-with-tests/)
- [Take your first steps with Go](https://docs.microsoft.com/en-us/learn/paths/go-first-steps/)
- [Play with Go](https://play-with-go.dev/)
- [The Ultimate Go Study Guide](https://github.com/hoanhan101/ultimate-go)
- [Tutorial Edge](https://tutorialedge.net)

# Engineering & Architecture
### Project Layout
When organizing project hierarchy of my Golang app, I always refer to the [project layout](https://github.com/golang-standards/project-layout).

### How I Organize Structs in Go Projects
I officially use the organization in this in my projects and can confirm having success with it. The TL;DR is that:
* Use *IDO* to store structures which no other Golang project will access (Used by frontend, ex: React app)
* Use *TDO* to share data structures between projects. (Used by micro-services).
* Use *Model* to store structures used in the data layer for your app to access the database.
* Write code to convert from *TDO* to *Model* and vice versa. Keep a strict separation.

[Read article](https://www.dudley.codes/posts/2021.02.23-golang-struct-organization/).

### Thoughts on the repository pattern in Golang
I have participated in projects with no ORM and thus had to write a data layer with SQL queries to communicate with a database. I found success using the **repository pattern** in Golang.

[Read article](https://adodd.net/posts/go-ddd-repository-pattern/).

### Honourable Mentions

* [What is the recommended Go Project folder structure? (found via /r/golang/)](https://www.reddit.com/r/golang/comments/8g26il/what_is_the_recommended_go_project_folder/)

# Web Development (Standard Library):

### Go’s std net/http is all you need … right?
If you've always used frameworks for web-development and you want to better understand why Golang can potentially be used without frameworks then this is a great article start! After reading it I better understood the PRO/CON of using the standard library instead of a framework.
[Read article](https://medium.com/@joeybloggs/gos-std-net-http-is-all-you-need-right-1c5555a9f2f6)

### Different approaches to HTTP routing in Go
Incredible article explaining the challenge of writing your own http router and various solutions. I have learned to use the **Split switch algorithm** for my projects when using the Go std. Understanding the http routing algorithms is *essential* in helping you write web-applications utilizing Golang std versus using a framework.
[Read article](https://benhoyt.com/writings/go-routing/)

# Web Development (Web Frameworks)
The only libraries (aka Frameworks) that I am familiar with:

### Chi
A library which utilizes the standard library but provides conveniences.

- [Visit Repository](https://github.com/go-chi/chi)

### Fiber
A library which is doesn't use the standard library router but instead is built on top of the [fasthttp](https://github.com/valyala/fasthttp) router.

- [Read developer documentation](https://docs.gofiber.io/)
- [Visit Repository](https://github.com/gofiber/fiber)

# Database:
- https://medium.com/avitotech/how-to-work-with-postgres-in-go-bad2dabd13e4
- How do you organize db and caching layer? via https://www.reddit.com/r/golang/comments/6o8rzt/how_do_you_organize_db_and_caching_layer/
- Null in databases and JSON marshalling - https://stackoverflow.com/questions/33072172/how-can-i-work-with-sql-null-values-and-json-in-a-good-way and https://github.com/Ellerbach/Golang-Json-serialize-deserialize

# Unit Tests & Mocking
- Mocking Golang with Interfaces In Real Life via https://archive.is/9x49s
- Unit Test (SQL) in Golang via https://archive.is/mKCsG
- Unit test a CLI via https://www.bradcypert.com/testing-a-cobra-cli-in-go/
- “Testify” Library via https://github.com/stretchr/testify

# Networking
- “Gobs of Data” via https://blog.golang.org/gob
- “Gob Package” via https://golang.org/pkg/encoding/gob/
- “Unix Domain Sockets in Go” via https://golangnews.org/2019/02/unix-domain-sockets-in-go/
- “Network Programming in Go / Gob Package” via https://tumregels.github.io/Network-Programming-with-Go/dataserialisation/the_gob_package.html
- “Network Programming in Go / RPC” via https://tumregels.github.io/Network-Programming-with-Go/rpc/
- “Why use Unix Domain Sockets over IP” via https://github.com/nixcloud/ip2unix#problem-statement
- x via https://anna.elde.codes/blog/golang-unix-socket-setup/
- grace https://medium.com/pantomath/how-we-use-grpc-to-build-a-client-server-system-in-go-dd20045fa1c2
- Graceful shutdown in Go http server via https://medium.com/honestbee-tw-engineer/gracefully-shutdown-in-go-http-server-5f5e6b83da5a

# Security
- “Implementing RSA Encryption and Signing in Golang (With Examples)” via https://www.sohamkamani.com/golang/rsa-encryption/
- Save and load crypto/rsa PrivateKey to and from the disk via https://stackoverflow.com/a/13555138

# QA
- Tools to Improve Your Golang Code Quality via https://dzone.com/articles/tools-to-improve-your-golang-code-quality
- golangci-lint via https://github.com/golangci/golangci-lint

# Timekeeping
- https://github.com/jinzhu/now

# Preparing for Job Interviews
* https://www.hackerrank.com/dashboard
* https://www.interviewbit.com/
* http://leetcode.com


# News Sites to Follow:
* https://golangdocs.com/
* https://www.reddit.com/r/golang/

# TODO:
https://stackoverflow.com/questions/22688906/go-naming-conventions-for-const
