---
title: "How to write a webserver in Golang using only the std net/http - Part 1"
subtitle: "Learn how to setup a small project and make simple GET / POST / PUT / DELETE requests using only the standard library net/http"
date: 2021-01-25T00:02:30-04:00
draft: false
categories:
- "development"
tags:
- "golang"
- "api"
---

![](https://images.pexels.com/photos/5582597/pexels-photo-5582597.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940)

The purpose of this article is to provide instructions on how to setup a simple RESTful API server, in Golang, using the standard library and not any other third-party web framework. You will know how to create REST endpoints within your project that can handle **POST**, **GET**, **PUT** and **DELETE** HTTP requests.

<!--more-->

# Table of Contents
* Why the [``net/http``](https://golang.org/pkg/net/http/) Standard Library?
* What do I need?
* Requirements - What are we building?
* Design - How will it work?
  - Project Structure
  - API Endpoints
* Implementation - Let's build our code!
  - 1. Project Location
  - 2. Code repository
  - 3. Golang Modules
  - 4. Project Scaffolding
* Testing - Verify the code works
* Implement remaining API endpoints
* Graceful Shutdown
* Final Thoughts - What's next?

# Why [``net/http``](https://golang.org/pkg/net/http/) Standard Library?
A general popular opinion in web-development is to use a web-development framework for build web application - *why reinvent the wheel?*

However in Golang, this general popular opinion is not held firm. The standard library is the core package provided by Golang is sufficient for all our web application needs.

I have used (and am using) [``go-chi``](https://github.com/go-chi/chi) and [``fibre``](https://github.com/gofiber/fiber) web libraries and I would highly recommend both. For this article, I want to explore what's involved with writing an application *without* any external library and only the provided [``http``](https://golang.org/pkg/net/http/) library.

# What do I need?
1. You will need Golang version 1.15+ installed on your development machine.
2. You will need to know the basics of Golang. I am assuming you've finished learning about how Golang works and you are looking to learn how to start coding a webserver for a API restful purposes.

# Requirements - What are we building?
Let's build a web-application which allows users to handle time-series data. The user can do the following:

* Create a new time-series datum
* List time-series data
* Retrieve a single time-series datum
* Update a single timer-series datum
* Delete a single time-series datum

The name of our application will be called ``mulberry`` and it will exist in a code repository ``mulberry-server``.

And remember, for this article, we cannot use any framework!

# Design - How will it work?

### Project Structure
In Golang, there is no official project structure; as a result, the onus is on the developer to structure a correct project hierarchy. A popular solution developers choose is the [following convention](https://github.com/golang-standards/project-layout). Our application's initial structure will look as follows (and will grow with time!):

```
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
```

### API Endpoints

We will have the following API endpoints:

```text
|--------------------------------------------------------------------------------
| METHOD  | URL                               | DESCRIPTION                    |
|--------------------------------------------------------------------------------
| GET     | /api/v1/version                   | Get the application version    |
| POST    | /api/v1/login                     | Authenticate the user          |
| POST    | /api/v1/register                  | Creates an account for a user  |
| GET     | /api/v1/time-series-data          | List all the time-series data  |
| POST    | /api/v1/time-series-data          | Create a time-series datum     |
| GET     | /api/v1/time-series-datum/<uuid>  | Get the details of a datum     |
| PUT     | /api/v1/time-series-datum/<uuid>  | Update a single datum          |
| DELETE  | /api/v1/time-series-datum/<uuid>  | Delete a single datum          |
|--------------------------------------------------------------------------------
```

# Implementation - Let's build our code!
## 1. Project Location
Start in your default Golang home folder:

```bash
$ cd ~/go/src/github.com/bartmika
```

Please note:

* Replace ``bartmika`` with your username on [GitHub](https://github.com).
* If that folder does not exist, please create it now.

## 2. Code repository
Create a new code repository called ``mulberry-server`` and clone your code repository:

```bash
$ git clone https://github.com/bartmika/mulberry-server.git
$ cd mulberry-server
```

Please note:

* Replace ``bartmika`` with your username on [GitHub](https://github.com).

## 3. Golang Modules
Since we are using Golang *modules* please follow these steps to get started:

```bash
$ go mod init github.com/bartmika/mulberry-server
$ export GIT_TERMINAL_PROMPT=1
$ go mod tidy
```

After running the above code your repository should look like this:

```
📦mulberry-server
   📄go.md
```

## 4. Project Scaffolding

Before we write any API related functionality, let's structure our code with a good base to build from. The scaffolding we use is based on the "model-view-controller" pattern; in addition, the structure is my *opinion* and should not be considered *the standard*.

Start by creating the folders and the files inside each folder.

This is how the **main.go** file will look like.

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/cmd/serve/main.go
package main

import (
    "fmt"
    "net/http"

    "github.com/bartmika/mulberry-server/internal/controllers"
)

func main() {
    c := controllers.NewBaseHandler()

    router := http.NewServeMux()
    router.HandleFunc("/", c.HandleRequests)

    s := &http.Server{
        Addr: fmt.Sprintf("%s:%s", "localhost", "5000"),
        Handler: router,
    }

    if err := s.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        panic(err)
    }
}
{{</ highlight >}}

And the **controller.go** file looks like this:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"
)

type BaseHandler struct {
    //TODO: Implement in the future.
}

func NewBaseHandler() (*BaseHandler) {
    return &BaseHandler{}
}

func (h *BaseHandler) HandleRequests(w http.ResponseWriter, req *http.Request) {
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

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/controllers/version.go
package controllers

import (
    "net/http"
)

func (h *BaseHandler) getVersion(w http.ResponseWriter, req *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.Write([]byte("Mulberry Server v1.0"))
}
{{</ highlight >}}

Next update the **controller.go** with the following:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"
)

type BaseHandler struct {
    UserRepo *repositories.UserRepo
}

func NewBaseHandler() (*BaseHandler) {
    return &BaseHandler{}
}

func (h *BaseHandler) HandleRequests(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path == "/api/v1/version" {
        h.getVersion(w, r)
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

```bash
$ SERVERPORT=5000 go run cmd/serve.go;
```

If you see a message saying the server started then congratulations, you have setup the project scaffolding!

# Testing - Verify the code works

Before you begin, please install the [``httpie``](https://httpie.io/) application. Once installed, please run the following code to confirm we can make an API call:

```bash
$ http get 127.0.0.1:5000/api/v1/version
```

If you get a result somewhat similar like this, then condradulations!

```
Last login: Thu Jan 28 00:10:01 on ttys015
bmika@MACMINI-AFA2131 mulberry-server % http get 127.0.0.1:5000/api/v1/version
HTTP/1.1 200 OK
Content-Length: 6
Content-Type: application/json
Date: Fri, 29 Jan 2021 05:03:22 GMT

Mulberry Server v1.0
```

# Implement remaining API endpoints

Before we begin, you have to be aware of an issue, the issue is that the std ``net/http`` does not add any support for **URL Arguments** such as ``/api/v1/time-series-datum/:uuid``. As a result, Golang developers are torn between using a third-party package, structuring your API endpoints differently, or writing your own custom router.

For more on this topic, please see the following links:

* [Go’s std net/http is all you need … right?](https://archive.is/T4cp0)
* [How to not use an http-router in go](https://archive.is/6hptY)
* [Different approaches to HTTP routing in Go](https://archive.is/ZpZg6)

Given these options, what should we do? From the beginners perspective, the [split switch](https://benhoyt.com/writings/go-routing/#split-switch) seems like the easiest to do, in addition it's quite performant with the cost of slightly ugly looking code.

Before we begin, make sure your project hierarchy look as follows by creating the **tsd.go** file:

```
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
```

Please fill the **tsd.go** file with the following contents:

{{< highlight golang "linenos=false">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/controllers/tsd.go
package controllers

import (
    "net/http"
)

func (h *BaseHandler) getTimeSeriesData(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: List Time Series Data")) //TODO: IMPLEMENT.
}

func (h *BaseHandler) postTimeSeriesData(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: Create Series Data")) //TODO: IMPLEMENT.
}

func (h *BaseHandler) getTimeSeriesDatum(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Get Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}

func (h *BaseHandler) putTimeSeriesDatum(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Update Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}

func (h *BaseHandler) deleteTimeSeriesDatum(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Delete Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}
{{</ highlight >}}

And then create the **user.go** file:

{{< highlight golang "linenos=false">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/controllers/user.go
package controllers

import (
    "net/http"
)

func (h *BaseHandler) postLogin(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: Logging in...")) //TODO: IMPLEMENT.
}

func (h *BaseHandler) postRegister(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: Registering...")) //TODO: IMPLEMENT.
}
{{</ highlight >}}

And then update the **controller.go** file:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"
    "strings"
)

type BaseHandler struct {
    //TODO: Implement in the future.
}

func NewBaseHandler() (*BaseHandler) {
    return &BaseHandler{}
}

func (h *BaseHandler) HandleRequests(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    // Split path into slash-separated parts, for example, path "/foo/bar"
    // gives p==["foo", "bar"] and path "/" gives p==[""]. Our API starts with
    // "/api/v1", as a result we will start the array slice at "3".
    p := strings.Split(r.URL.Path, "/")[3:]
    n := len(p)

    // fmt.Println(p, n) // For debugging purposes only.

    switch {
    case n == 1 && p[0] == "version" && r.Method == http.MethodGet:
        h.getVersion(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodGet:
        h.getTimeSeriesData(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodPost:
        h.postTimeSeriesData(w, r)
    case n == 1 && p[0] == "login" && r.Method == http.MethodPost:
        h.postLogin(w, r)
    case n == 1 && p[0] == "register" && r.Method == http.MethodPost:
        h.postRegister(w, r)
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodGet:
        h.getTimeSeriesDatum(w, r, p[1])
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodPut:
        h.putTimeSeriesDatum(w, r, p[1])
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodDelete:
        h.deleteTimeSeriesDatum(w, r, p[1])
    default:
        http.NotFound(w, r)
    }
}
{{</ highlight >}}

And finally to test out making API calls, we have the following commands you can run in your console.

```bash
$ http get 127.0.0.1:5000/api/v1/version

$ http post 127.0.0.1:5000/api/v1/register email="lalal@lalal.com" password="lalalal" name="lalalalalala"

$ http post 127.0.0.1:5000/api/v1/login email="lalal@lalal.com" password="lalalal"

$ http get 127.0.0.1:5000/api/v1/time-series-data

$ http post 127.0.0.1:5000/api/v1/time-series-data instrument_id="1" timestamp="2021-01-01"

$ http get 127.0.0.1:5000/api/v1/time-series-datum/xxx

$ http put 127.0.0.1:5000/api/v1/time-series-datum/xxx instrument_id="1" timestamp="2021-01-02"

$ http delete 127.0.0.1:5000/api/v1/time-series-datum/xxx
```

If you got a ``200 OK`` response with some-sort of string message, then congratulations! You have implemented your server using only the ``net/http`` standard library.

# Graceful Shutdown

Update the **main.go** file with the following content to support graceful shutdowns of our webserver.

```go
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
    c := controllers.NewBaseHandler()

    router := http.NewServeMux()
    router.HandleFunc("/", c.HandleRequests)

	srv := &http.Server{
		Addr: fmt.Sprintf("%s:%s", "localhost", "5000"),
        Handler: router,
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
```

# Final Thoughts - What's next?

That's it! We have ourselves a basic API web-app. We have dipped our feat on the shores of the ocean of Golang web-development and the water feels welcoming. Moving forward we'll need to cover more topics such as:

* How do we read the body of a request? How do we write responses?
* How do we handle using a database?
* How do we handle login and authenticated API endpoints?
* How do we handle sessions?
* How do we handle background processes?

Now onward to [**part 2**](/posts/2021/how-to-write-a-webserver-in-golang-using-only-the-std-net-http-part-2/)!
