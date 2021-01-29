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
* Why [``http``](https://golang.org/pkg/net/http/) Standard Library?
* What do I need?
* Requirements - What are we building?
* Design - How will it work?
  - a. Project Structure
  - b. API
  - c. Command Console
  - d. Coding Style
* Implementation - Let's build our code!
  - a. Project Location
  - b. Code repository
  - c. Golang Modules
  - d. Project Scaffolding
  - e. API Scaffolding
* Testing - Verify the code works
* Implement remaining API endpoints
* Final Thoughts - What's next?

# Why [``http``](https://golang.org/pkg/net/http/) Standard Library?
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

## a. Project Structure
In Golang, there is no official project structure; as a result, the onus is on the developer to structure a correct project hierarchy. A popular solution developers choose is the [following convention](https://github.com/golang-standards/project-layout). Our application's structure will look as follows:

```
📦mulberry-server
│   📄README.md
│   📄Makefile
│
└───📁cmd
|      📄serve.go
│
└───📁internal
    │
    └───📁api
    │      📄api.go
    │
    └───📁manager
    │      📄manager.go
    │
    └───📁models
           📄model.go
```

## b. API

We will have the following API endpoints:

```text
|--------------------------------------------------------------------------------
| METHOD  | URL                               | DESCRIPTION                    |
|--------------------------------------------------------------------------------
| GET     | /api/v1/version                   | Get the application version    |
| GET     | /api/v1/time-series-data          | List all the time-series data  |
| POST    | /api/v1/time-series-data          | Create a time-series datum     |
| GET     | /api/v1/time-series-datum/<uuid>  | Get the details of a datum     |
| PUT     | /api/v1/time-series-datum/<uuid>  | Update a single datum          |
| DELETE  | /api/v1/time-series-datum/<uuid>  | Delete a single datum          |
|--------------------------------------------------------------------------------
```

## c. Command Console

We should be able to run the following console commands:

```bash
go run main.go
```

## d. Coding Style

We will run the `gofmt` application on this code repository.

At the end of the project, please be sure to run:

```bash
$ go fmt $HOME/go/src/github.com/bartmika/mulberry-server/...
```

# Implementation - Let's build our code!
## a. Project Location
Start in your default Golang home folder:

```bash
$ cd ~/go/src/github.com/bartmika
```

Please note:

* Replace ``bartmika`` with your username on [GitHub](https://github.com).
* If that folder does not exist, please create it now.

## b. Code repository
Create a new code repository called ``mulberry-server`` and run the following code:

```bash
$ git clone https://github.com/bartmika/mulberry-server.git
$ cd mulberry-server
```

## c. Golang Modules
Since we are using Golang ``modules`` please follow these steps to get started:

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

## d. Project Scaffolding

Before we write any API related functionality, let's structure our code with a good base to build from. The scaffolding we use is based on the "model-view-controller" pattern; in addition, the structure is my *opinion* and should not be considered *the standard* but rather the way I would write.

Begin with our ``serve.go`` file with the following contents:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/cmd/serve.go
package main

import (
    "log"
    // "net/http"
    "os"
    "os/signal"
    "syscall"

    "github.com/bartmika/mulberry-server/internal/manager"
)

func main() {
    manager, initErr := manager.NewManager()
    if initErr != nil {
        log.Fatalf("NewManager error: %s\n", initErr)
    }

    // Integrate this program with the signal handler provided by the operating
    // system through the go routine.
    done := make(chan os.Signal, 1)
    signal.Notify(done, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

    // Create a new go routine which will become our main runtime loop because
    // we will halt this current main runtime loop with a go channel.
    go func() {
        log.Println("Server running.")
        if execErr := manager.RunMainRuntimeLoop(); execErr != nil {
            log.Fatalf("main error: %s\n", execErr)
        }
    }()

    // Halt the main runtime loop until the program receives a shutdown signal.
    <-done

    // Execute the graceful shutdown sub-routine which will terminate any
    // active connections and reject any new connections.
    manager.StopMainRuntimeLoop()
}
{{</ highlight >}}

Study the code and you should take away the following benefits:

* Core application logic is handled someplace else, so our currently file makes use of the easy-to-use functions.
* We integrate with the OS to handle termination commands so our application *gracefully shuts down* when terminated.

Next we need to create our application manager which is in essence the *controller*. This file will be responsible for starting up and managing the (a) database and (b) API.

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/manager/manager.go
package manager

import (
    "log"
)

type Manager struct {
    //TODO: api
    //TODO: database
}

func NewManager() (*Manager, error) {
    //TODO: Database initialization code.
    // ...

    //TODO: API handler initialization code.
    // ...

    //TODO: Server initialization code.
    // ...

    //TODO: Save to our struct.
    return &Manager{}, nil
}

func (m *Manager) RunMainRuntimeLoop() error {
    //TODO
    return nil
}

func (m *Manager) StopMainRuntimeLoop() error {
    log.Println("Starting graceful shutdown")

    //TODO

    log.Println("Finished gaceful shutdown")
    return nil
}
{{</ highlight >}}

Next we'll create an ``APIServer`` structure. All our API functions will be methods belonging to this struct. This section can be considered both the *view* and *controller*.

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/api/api.go
package api

type APIServer struct {
    //TODO: Implement properties
}

func NewAPIServer () (*APIServer, error) {
    return nil, nil
}

//TODO: Implement functions
{{</ highlight >}}

And finally the *model* layer, we will create the ``Store`` structure so all our database methods will be utilizing it.

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/models/models.go
package models

type Store struct {
    //TODO: Implement properties.
}

func NewStore() (*Store) {
    return &Store{
        //TODO: Finish...
    }
}

//TODO: Implement functions.
{{</ highlight >}}

Copy and paste the above code to confirm no compilation errors. Study the code. Notice the following key points:

1. If we want to add a new *model* then we create a file in the ``mulberry-server/internal/models`` folder. For example: "user.go" would be created and all the methods would be using the ``Store`` struct as the pointer receiver.
2. If we want to add another *api* then we create a file in the ``mulberry-server/internal/api`` folder and update the ``api.go`` file with our new URL routes.
3. The ``internal`` folder means anything inside of it cannot be called from other Golang code, in essence all our code is *private*. Some developers move the "models" folder in a ``pkg`` folder to make the code accessible outside the package - this is a good idea when you are building microservices.

## e. API Scaffolding

Since we've never used the ``net/http`` standard library, let's write some prototype code that we can learn and play around with. The following code is used for learning purposes; afterwords, in a different tutorial, we will replace the code with real code using a database, etc.

Let us learn how to create our first API endpoint, the ``/api/v1/version`` URL path which supports ``GET`` requests.

To begin, we need to update ``manager.go`` as follows:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/manager/manager.go
package manager

import (
    "log"      // STEP 1: Import the following...
    "net/http" // (1)
    "os"       // (2)
    "context"  // (3)
    "time"     // (4)

    "github.com/bartmika/mulberry-server/internal/api"
)

type Manager struct {
    // STEP 2: Our manager will handle the following...
    server *http.Server
    //TODO: database field in the future...
}

func NewManager() (*Manager, error) {
    //TODO: Database initialization.
    // ...

    // STEP 3: Initialize our API handler.
    as, err := api.NewAPIServer() //TODO: Integrate with the database in the future.
    if err != nil {
        log.Fatal(err)
    }

    // STEP 4: Initialize our router.
    router := http.NewServeMux()

    // STEP 5: Define URL paths that the `RouteHandler` will handle.
    router.HandleFunc("/", as.HandleRequests)

    port := os.Getenv("SERVERPORT")
    if port == "" {
        log.Fatal("Not specified the port.")
    }

    // STEP 6: Initialize the server instance that will run the specified
    //         port for the specified routes.
    srv := &http.Server{
		Addr:    "localhost:"+port,
		Handler: router,
	}

    // STEP 7: Finally initialize our struct which will manage these services.
    return &Manager{
        server: srv, // Step 4
    }, nil
}

func (m *Manager) RunMainRuntimeLoop() error {
    // STEP 8
    if err := m.server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        return err
    }
    return nil
}

func (m *Manager) StopMainRuntimeLoop() error {
    log.Println("Starting graceful shutdown")

    // STEP 9
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer func() {
		//TODO: Database shutdown code in the future...
        // . . .

		cancel()
	}()
	if err := m.server.Shutdown(ctx); err != nil {
		log.Fatalf("Server Shutdown Failed:%+v", err)
	}

    log.Println("Finished gaceful shutdown")
    return nil
}
{{</ highlight >}}

Next we need to update the API as follows:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/api/api.go
package api

import (
    "fmt"
    "net/http"
)

type APIServer struct {
    //TODO: Implement properties
}

func NewAPIServer () (*APIServer, error) {
    return nil, nil
}

func (as *APIServer) RouteHandler(w http.ResponseWriter, req *http.Request) {
    if req.URL.Path == "/api/v1/version" {
        if req.Method == http.MethodGet {
            as.getVersionHandler(w, req)
            return
        }
    }

    //TODO: Implement more functions...

    // DEVELOPERS NOTE:
    // PLEASE NOTE WE WILL BE IMPLEMENTING THE API ENDPOINTS OF OUR APP IN THE FUTURE:
    //--------------------------------------------------------------------------------
    // GET     | /api/v1/time-series-data          | List all the time-series data  |
    // POST    | /api/v1/time-series-data          | Create a time-series datum     |
    // GET     | /api/v1/time-series-datum/<uuid>  | Get the details of a datum     |
    // PUT     | /api/v1/time-series-datum/<uuid>  | Update a single datum          |
    // DELETE  | /api/v1/time-series-datum/<uuid>  | Delete a single datum          |
    //--------------------------------------------------------------------------------

    http.Error(w, fmt.Sprintf("Unexpected method and URL path`, got %v", req.Method), http.StatusMethodNotAllowed)
    return
}
{{</ highlight >}}

And finally we will need to create a new file called ``version_api.go`` with the contents of:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/api/version_api.go
package api

import (
    "net/http"
)

func (as *APIServer) getVersionHandler(w http.ResponseWriter, req *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.Write([]byte("Mulberry Server v1.0"))
}
{{</ highlight >}}

In summary the project hierarchy should look as follows:

```
📦mulberry-server
│   📄README.md
│   📄Makefile
│
└───📁cmd
|      📄serve.go
│
└───📁internal
    │
    └───📁api
    │      📄api.go
    │      📄version_api.go
    │
    └───📁manager
    │      📄manager.go
    │
    └───📁models
           📄model.go
```

Finally start the server and confirm everything works:

```bash
$ SERVERPORT=5656 go run cmd/serve.go;
```

If you see a message saying the server started then congratulations, you have setup the project scaffolding!

# Testing - Verify the code works

Before you begin, please install the [``httpie``](https://httpie.io/) application. Once installed, please run the following code to confirm we can make an API call:

```bash
$ http get 127.0.0.1:5656/api/v1/version
```

If you get a result somewhat similar like this, then condradulations!

```
Last login: Thu Jan 28 00:10:01 on ttys015
bmika@MACMINI-AFA2131 mulberry-server % http get 127.0.0.1:5656/api/v1/version
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

Before we begin, make sure your project hierarchy should look as follows by creating the ``tsd_api.go`` file in your project:

```
📦mulberry-server
│   📄README.md
│   📄Makefile
│
└───📁cmd
|      📄serve.go
│
└───📁internal
    │
    └───📁api
    │      📄api.go
    │      📄tsd_api.go
    │      📄version_api.go
    │
    └───📁manager
    │      📄manager.go
    │
    └───📁models
           📄model.go
```

Therefore, the code will look like this:

{{< highlight golang "linenos=false">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/api/api.go
package api

import (
    "fmt"
    "strings"
    "net/http"
)

type APIServer struct {
    //TODO: Implement properties
}

func NewAPIServer () (*APIServer, error) {
    return nil, nil //TODO: Implement
}

func (as *APIServer) HandleRequests(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    // Split path into slash-separated parts, for example, path "/foo/bar"
    // gives p==["foo", "bar"] and path "/" gives p==[""]. Our API starts with
    // "/api/v1", as a result we will start the array slice at "3".
    p := strings.Split(r.URL.Path, "/")[3:]
    n := len(p)

    // fmt.Println(p, n) // For debugging purposes only.

    switch {
    case n == 1 && p[0] == "version" && r.Method == http.MethodGet:
        as.getVersionHandler(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodGet:
        as.getTimeSeriesDataHandler(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodPost:
        as.postTimeSeriesDatumHandler(w, r)
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodGet:
        as.getTimeSeriesDatumHandler(w, r, p[1])
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodPut:
        as.putTimeSeriesDatumHandler(w, r, p[1])
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodDelete:
        as.deleteTimeSeriesDatumHandler(w, r, p[1])
    default:
        // Defensive Code: The catch-all code which will error if user did not
        //                 make a request that our function supports.
        http.Error(w, fmt.Sprintf("Unexpected method and URL path`, got %v", r.Method), http.StatusMethodNotAllowed)
    }
}
{{</ highlight >}}

And

{{< highlight golang "linenos=false">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/api/tsd_api.go
package api

import (
    "net/http"
)

func (as *APIServer) getTimeSeriesDataHandler(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: List Time Series Data")) //TODO: IMPLEMENT.
}

func (as *APIServer) postTimeSeriesDatumHandler(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("TODO: Create Series Data")) //TODO: IMPLEMENT.
}

func (as *APIServer) getTimeSeriesDatumHandler(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Get Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}

func (as *APIServer) putTimeSeriesDatumHandler(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Update Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}

func (as *APIServer) deleteTimeSeriesDatumHandler(w http.ResponseWriter, req *http.Request, uuid string) {
    w.Write([]byte("TODO: Delete Series Datum with UUID: " + uuid)) //TODO: IMPLEMENT.
}
{{</ highlight >}}

## Testing

And finally to test out making API calls, we have the following commands you can run in your console.

```bash
$ http get 127.0.0.1:5656/api/v1/version

$ http get 127.0.0.1:4112/api/v1/time-series-data

$ http post 127.0.0.1:4112/api/v1/time-series-data instrument_id="1" timestamp="2021-01-01"

$ http get 127.0.0.1:4112/api/v1/time-series-datum/xxx

$ http put 127.0.0.1:4112/api/v1/time-series-datum/xxx instrument_id="1" timestamp="2021-01-02"

$ http delete 127.0.0.1:4112/api/v1/time-series-datum/xxx
```

# Final Thoughts - What's next?

That's it! We have ourselves a basic API web-app. We have dipped our feat on the shores of the ocean of Golang web-development and the water feels welcoming. Moving forward we'll need to cover more topics such as:

* How do we read ``post``/``put`` requests? How do we write responses?
* How do we handle using a database?
* How do we handle login and authenticated API endpoints?
* How do we handle sessions?
* How do we handle background processes?
