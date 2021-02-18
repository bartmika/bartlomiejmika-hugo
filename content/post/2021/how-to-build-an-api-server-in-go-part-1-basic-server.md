---
title: "How to Build an API Server in Go - Part 1: Basic Server"
date: 2021-01-25T00:02:30-04:00
draft: false
categories:
- "Web Development"
tags:
- "Golang"
- "API"
- "HOWTO"
---

<!-- ![](https://images.pexels.com/photos/5582597/pexels-photo-5582597.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940) -->

The purpose of this post is to provide instructions on how to setup a simple RESTful API server, in Golang, using only the [*net/http* package](https://golang.org/pkg/net/http/) and not any other third-party web framework. You will learn how to create REST endpoints within your project that can handle **POST**, **GET**, **PUT** and **DELETE** HTTP requests. This is the first post in a multi-post series.

<!--more-->

This post belongs to the following series:
1. **How to Build an API Server in Go - Part 1: Basic Server**
2. [How to Build an API Server in Go - Part 2: Simple Database](/post/2021/how-to-build-an-api-server-in-go-part-2-simple-database/)
3. [How to Build an API Server in Go - Part 3: Postgres Database](/post/2021/how-to-build-an-api-server-in-go-part-3-postgres-database/)
4. [How to Build an API Server in Go - Part 4: Access Control](/post/2021/how-to-build-an-api-server-in-go-part-4-access-control/)

# Why [``net/http``](https://golang.org/pkg/net/http/) Standard Library? {#1}
A general popular opinion in web-development is to use a web-development framework for build web application - *why reinvent the wheel?*

However in Golang, this general popular opinion is not held firm. The standard library is the core package provided by Golang is sufficient for all our web application needs.

I have used (and am using) [``go-chi``](https://github.com/go-chi/chi) and [``fibre``](https://github.com/gofiber/fiber) web libraries and I would highly recommend both. For this article, I want to explore what's involved with writing an application *without* any external library and only the provided [``http``](https://golang.org/pkg/net/http/) library.

# What do I need? {#2}
1. You will need Golang version 1.15+ installed on your development machine.
2. You will need to know the basics of Golang. I am assuming you've finished learning about how Golang works and you are looking to learn how to start coding a webserver for a API restful purposes.

# Requirements - What are we building? {#3}
Let's build a web-application which allows users to handle time-series data. The user can do the following:

* Create a new time-series datum
* List time-series data
* Retrieve a single time-series datum
* Update a single timer-series datum
* Delete a single time-series datum

The name of our application will be called ``mulberry`` and it will exist in a code repository ``mulberry-server``.

And remember, for this article, we cannot use any framework!

# Design - How will it work?  {#4}

## Project Structure {#4a}
In Golang, there is no official project structure; as a result, the onus is on the developer to structure a correct project hierarchy. A popular solution developers choose is the [following convention](https://github.com/golang-standards/project-layout). Our application's initial structure will look as follows (and will grow with time!):

{{< highlight bash "linenos=false">}}
📦mulberry-server
│   📄README.md
│   📄Makefile
│
└───📁cmd
│   |
|   └───📁serve
|       📄main.go
│
└───📁internal
    │
    └───📁controllers
        📄controller.go
{{</ highlight >}}

## API Endpoints {#4b}

We will have the following API endpoints:

{{< highlight bash "linenos=false">}}
|----------------------------------------------------------------------------
| METHOD | URL                              | DESCRIPTION                   |
|----------------------------------------------------------------------------
| GET    | /api/v1/version                  | Get the application version   |
| POST   | /api/v1/login                    | Authenticate the user         |
| POST   | /api/v1/register                 | Creates an account for a user |
| GET    | /api/v1/time-series-data         | List all the time-series data |
| POST   | /api/v1/time-series-data         | Create a time-series datum    |
| GET    | /api/v1/time-series-datum/<uuid> | Get the details of a datum    |
| PUT    | /api/v1/time-series-datum/<uuid> | Update a single datum         |
| DELETE | /api/v1/time-series-datum/<uuid> | Delete a single datum         |
|----------------------------------------------------------------------------
{{</ highlight >}}

# Implementation - Let's build our code! {#5}
## 1. Project Location {#5a}
Start in your default Golang home folder:

{{< highlight bash "linenos=false">}}
$ cd ~/go/src/github.com/bartmika
{{</ highlight >}}

Please note:

* Replace ``bartmika`` with your username on [GitHub](https://github.com).
* If that folder does not exist, please create it now.

## 2. Code repository
Create a new code repository called ``mulberry-server`` and clone your code repository:

{{< highlight bash "linenos=false">}}
$ git clone https://github.com/bartmika/mulberry-server.git
$ cd mulberry-server
{{</ highlight >}}

Please note:

* Replace ``bartmika`` with your username on [GitHub](https://github.com).

## 3. Golang Modules
Since we are using Golang *modules* please follow these steps to get started:

{{< highlight bash "linenos=false">}}
$ go mod init github.com/bartmika/mulberry-server
$ export GIT_TERMINAL_PROMPT=1
$ go mod tidy
{{</ highlight >}}

After running the above code your repository should look like this:

{{< highlight bash "linenos=false">}}
📦mulberry-server
   📄go.md
{{</ highlight >}}

## 4. Project Scaffolding

Before we write any API related functionality, let's structure our code with a good base to build from. The scaffolding we use is based on the "model-view-controller" pattern; in addition, the structure is my *opinion* and should not be considered *the standard*.

Start by creating the folders and the files inside each folder.

This is how the **main.go** file will look like.

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/cmd/serve/main.go
package main

import (
    "fmt"
    "net/http"

    "github.com/bartmika/mulberry-server/internal/controllers"
)

func main() {
    c := controllers.New()

    mux := http.NewServeMux()
    mux.HandleFunc("/", c.HandleRequests)

    s := &http.Server{
        Addr: fmt.Sprintf("%s:%s", "localhost", "5000"),
        Handler: mux,
    }

    if err := s.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        panic(err)
    }
}
{{</ highlight >}}

And the **controller.go** file looks like this:

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"
)

type Controller struct {
    //TODO: Implement in the future.
}

func New() (*Controller) {
    return &Controller{}
}

func (c *Controller) HandleRequests(w http.ResponseWriter, req *http.Request) {
    //TODO: Implement in the future
    //--------------------------------------------------------------------------------
    // GET     | /api/v1/version                   | Get the application version    |
    // POST    | /api/v1/login                     | Authenticate the user          |
    // POST    | /api/v1/register                  | Creates an account for a user  |
    // GET     | /api/v1/time-series-data          | List all the time-series data  |
    // GET     | /api/v1/time-series-data          | List all the time-series data  |
    // POST    | /api/v1/time-series-data          | Create a time-series datum     |
    // GET     | /api/v1/time-series-datum/<uuid>  | Get the details of a datum     |
    // PUT     | /api/v1/time-series-datum/<uuid>  | Update a single datum          |
    // DELETE  | /api/v1/time-series-datum/<uuid>  | Delete a single datum          |
    //--------------------------------------------------------------------------------
    return
}
{{</ highlight >}}

That's it, we've structured our project which will grow nicely.

Let us learn how to create our first API endpoint and then call it. We will be implementing the ``/api/v1/version`` URL path which supports *GET* requests.

Create a new file called **version.go** with the contents of:

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/controllers/version.go
package controllers

import (
    "net/http"
)

func (c *Controller) getVersion(w http.ResponseWriter, req *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.Write([]byte("Mulberry Server v1.0"))
}
{{</ highlight >}}

Next update the **controller.go** with the following:

{{< highlight go "linenos=table">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"
)

type Controller struct {
    UserRepo *repositories.UserRepo
}

func New() (*Controller) {
    return &Controller{}
}

func (c *Controller) HandleRequests(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path == "/api/v1/version" {
        c.getVersion(w, r)
        return
    }
    //TODO: Implement in the future
    //--------------------------------------------------------------------------------
    // GET     | /api/v1/version                   | Get the application version    |
    // POST    | /api/v1/login                     | Authenticate the user          |
    // POST    | /api/v1/register                  | Creates an account for a user  |
    // GET     | /api/v1/time-series-data          | List all the time-series data  |
    // GET     | /api/v1/time-series-data          | List all the time-series data  |
    // POST    | /api/v1/time-series-data          | Create a time-series datum     |
    // GET     | /api/v1/time-series-datum/<uuid>  | Get the details of a datum     |
    // PUT     | /api/v1/time-series-datum/<uuid>  | Update a single datum          |
    // DELETE  | /api/v1/time-series-datum/<uuid>  | Delete a single datum          |
    //--------------------------------------------------------------------------------

    http.NotFound(w, r)
    return
}
{{</ highlight >}}

Finally start the server and confirm everything works:

{{< highlight bash "linenos=false">}}
$ go run cmd/serve/main.go;
{{</ highlight >}}

If you see a message saying the server started then congratulations, you have setup the project scaffolding!

# Testing - Verify the code works

Before you begin, please install the [``httpie``](https://httpie.io/) application. Once installed, please run the following code to confirm we can make an API call:

{{< highlight bash "linenos=false">}}
$ http get 127.0.0.1:5000/api/v1/version
{{</ highlight >}}

If you get a result somewhat similar like this, then congratulations! Please note if you are using Windows with ``Git bash for Windows`` and it hangs, then please [visit this url for a solution](https://stackoverflow.com/a/41192693).

{{< highlight bash "linenos=false">}}
Last login: Thu Jan 28 00:10:01 on ttys015
bmika@MACMINI-AFA2131 mulberry-server % http get 127.0.0.1:5000/api/v1/version
HTTP/1.1 200 OK
Content-Length: 6
Content-Type: application/json
Date: Fri, 29 Jan 2021 05:03:22 GMT

Mulberry Server v1.0
{{</ highlight >}}

If you are *not* using [``httpie``](https://httpie.io/) you can use ``curl``. Here is what you would write in your terminal:

{{< highlight bash "linenos=false">}}
$ curl -X GET \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  "http://127.0.0.1:5000/api/v1/version"
{{</ highlight >}}

# Implement remaining API endpoints

Before we begin, you have to be aware of an issue, the issue is that the std ``net/http`` does not add any support for **URL Arguments** such as ``/api/v1/time-series-datum/:uuid``. As a result, Golang developers are torn between using a third-party package, structuring your API endpoints differently, or writing your own custom router.

For more on this topic, please see the following links:

* [Go’s std net/http is all you need … right?](https://archive.is/T4cp0)
* [How to not use an http-router in go](https://archive.is/6hptY)
* [Different approaches to HTTP routing in Go](https://archive.is/ZpZg6)

Given these options, what should we do? From the beginners perspective, the [split switch](https://benhoyt.com/writings/go-routing/#split-switch) seems like the easiest to do, in addition it's quite performant with the cost of slightly ugly looking code.

Before we begin, make sure your project hierarchy look as follows by creating the **tsd.go** file:

{{< highlight bash "linenos=false">}}
📦mulberry-server
│   📄README.md
│   📄Makefile
│
└───📁cmd
│   |
|   └───📁serve
|       📄main.go
│
└───📁internal
    │
    └───📁controllers
        📄controller.go
        📄version.go
        📄tsd.go
        📄user.go
{{</ highlight >}}

Please fill the **tsd.go** file with the following contents:

{{< highlight go "linenos=true">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/controllers/tsd.go
package controllers

import (
    "net/http"
)

func (c *Controller) getTimeSeriesData(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: List Time Series Data")) //TODO: IMPLEMENT.
}

func (c *Controller) postTimeSeriesData(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: Create Series Data")) //TODO: IMPLEMENT.
}

func (c *Controller) getTimeSeriesDatum(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Get Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}

func (c *Controller) putTimeSeriesDatum(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Update Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}

func (c *Controller) deleteTimeSeriesDatum(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Delete Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}
{{</ highlight >}}

And then create the **user.go** file:

{{< highlight go "linenos=true">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/controllers/user.go
package controllers

import (
    "net/http"
)

func (c *Controller) postLogin(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: Logging in...")) //TODO: IMPLEMENT.
}

func (c *Controller) postRegister(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: Registering...")) //TODO: IMPLEMENT.
}
{{</ highlight >}}

And then update the **controller.go** file:

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"
    "strings"
)

type Controller struct {
    //TODO: Implement in the future.
}

func New() (*Controller) {
    return &Controller{}
}

func (c *Controller) HandleRequests(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    // Split path into slash-separated parts, for example, path "/foo/bar"
    // gives p==["foo", "bar"] and path "/" gives p==[""]. Our API starts with
    // "/api/v1", as a result we will start the array slice at "3".
    p := strings.Split(r.URL.Path, "/")[3:]
    n := len(p)

    // fmt.Println(p, n) // For debugging purposes only.

    switch {
    case n == 1 && p[0] == "version" && r.Method == http.MethodGet:
        c.getVersion(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodGet:
        c.getTimeSeriesData(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodPost:
        c.postTimeSeriesData(w, r)
    case n == 1 && p[0] == "login" && r.Method == http.MethodPost:
        c.postLogin(w, r)
    case n == 1 && p[0] == "register" && r.Method == http.MethodPost:
        c.postRegister(w, r)
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodGet:
        c.getTimeSeriesDatum(w, r, p[1])
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodPut:
        c.putTimeSeriesDatum(w, r, p[1])
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodDelete:
        c.deleteTimeSeriesDatum(w, r, p[1])
    default:
        http.NotFound(w, r)
    }
}
{{</ highlight >}}

And finally to test out making API calls, we have the following commands you can run in your console.

{{< highlight bash "linenos=false">}}
$ http get 127.0.0.1:5000/api/v1/version

$ http post 127.0.0.1:5000/api/v1/register email="lalal@lalal.com" password="lalalal" name="lalalalalala"

$ http post 127.0.0.1:5000/api/v1/login email="lalal@lalal.com" password="lalalal"

$ http get 127.0.0.1:5000/api/v1/time-series-data

$ http post 127.0.0.1:5000/api/v1/time-series-data instrument_id="1" timestamp="2021-01-01"

$ http get 127.0.0.1:5000/api/v1/time-series-datum/xxx

$ http put 127.0.0.1:5000/api/v1/time-series-datum/xxx instrument_id="1" timestamp="2021-01-02"

$ http delete 127.0.0.1:5000/api/v1/time-series-datum/xxx
{{</ highlight >}}

If you got a ``200 OK`` response with some-sort of string message, then congratulations! You have implemented your server using only the ``net/http`` standard library.

If you want to use the ``curl`` command then run the following:

{{< highlight bash "linenos=false">}}
$ curl -X GET \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  "http://127.0.0.1:5000/api/v1/version"

$ curl -X POST \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  -d '{"email":"lalal@lalal.com","password":"lalalal"}' \
  "http://127.0.0.1:5000/api/v1/login"

$ curl -X POST \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  -d '{"email":"lalal@lalal.com","password":"lalalal","name":"lalalal"}' \
  "http://127.0.0.1:5000/api/v1/register"

$ curl -X GET \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  "http://127.0.0.1:5000/api/v1/time-series-data"

$ curl -X POST \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  -d '{"instrument_id":1,"timestamp":"2021-01-01"}' \
  "http://127.0.0.1:5000/api/v1/time-series-data"

$ curl -X GET \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  "http://127.0.0.1:5000/api/v1/time-series-datum/xxx"

$ curl -X PUT \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  -d '{"instrument_id":1,"timestamp":"2021-01-01"}' \
  "http://127.0.0.1:5000/api/v1/time-series-datum/xxx"

$ curl -X DELETE \
  -H "Content-type: application/json" \
  -H "Accept: application/json" \
  "http://127.0.0.1:5000/api/v1/time-series-datum/xxx"

{{</ highlight >}}

# Graceful Shutdown

Update the **main.go** file with the following content to support graceful shutdowns of our webserver.

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/cmd/serve/main.go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/bartmika/mulberry-server/internal/controllers"
)

func main() {
    c := controllers.New()

    mux := http.NewServeMux()
    mux.HandleFunc("/", c.HandleRequests)

    srv := &http.Server{
        Addr: fmt.Sprintf("%s:%s", "localhost", "5000"),
        Handler: mux,
    }

    done := make(chan os.Signal, 1)
    signal.Notify(done, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

    go runMainRuntimeLoop(srv)

    log.Print("Server Started")

    // Run the main loop blocking code.
    <-done

    stopMainRuntimeLoop(srv)
}

func runMainRuntimeLoop(srv *http.Server) {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("listen: %s\n", err)
    }
}

func stopMainRuntimeLoop(srv *http.Server) {
    log.Printf("Starting graceful shutdown now...")

    // Execute the graceful shutdown sub-routine which will terminate any
    // active connections and reject any new connections.
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer func() {
        // extra handling here
        cancel()
    }()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Server Shutdown Failed:%+v", err)
    }
    log.Printf("Graceful shutdown finished.")
    log.Print("Server Exited")
}
{{</ highlight >}}

# Final Thoughts - What's next?

That's it! We have ourselves a basic API web-app. We have dipped our feat on the shores of the ocean of Golang web-development and the water feels welcoming. Moving forward we'll need to cover more topics such as:

* How do we read the body of a request? How do we write responses?
* How do we handle using a database?
* How do we handle login and authenticated API endpoints?
* How do we handle sessions?
* How do we handle background processes?

Now onward to the next part of our series: [**Part 2: Simple Database >>**](/post/2021/how-to-build-an-api-server-in-go-part-2-simple-database/).
