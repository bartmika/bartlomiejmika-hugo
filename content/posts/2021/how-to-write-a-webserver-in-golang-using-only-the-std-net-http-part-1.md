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
    - Part 1: Data Layer
    - Part 2: Application Layer
    - Part 3: Enter the ``net/http`` standard library
* Testing - Verify the code works
* Implement remaining API endpoints
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
|   │
|   └───📁controllers
|   │      📄controller.go
|   |
|   └───📁repositories
|          📄user.go
|
└───📁pkg
    |
    └───📁db
    |   📄db.go
    |
    └───📁models
        📄user.go
```

### API Endpoints

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

### Part 1: Data Layer

We will structure our data layer according to **Approach 3: Repository Interface per Model** as mentioned in [this article](https://archive.is/XxqyT).

Begin with our **db.go** file which we will implement in a future article:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/pkg/db/db.go
package db

import (
    "database/sql"
)

func ConnectDB() *sql.DB {
    return &sql.DB{} //TODO: Implement in future.
}
{{</ highlight >}}

Next we will create a sample **user.go** file with the structure we can use in the **models** package. Please note, we are putting this file in the **pkg** folder in case we want to use this file in the future in another project.

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/pkg/models/user.go
package models

type User struct {
    Name string //TODO: Will fill out in the future.
}

type UserRepository interface {
    FindByID(ID int) (*User, error)
    Save(user *User) error
    //TODO: Will fill out in the future.
}
{{</ highlight >}}

Now we will implement the **repositories** package which is responsible for storing *structs* and their implementation of the interfaces in the **models** package. Remember once a struct has implemented all the required methods of an interface, then we can call those methods on them; as a result, this makes testing much easier.

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/respositories/user.go
package repositories

import (
    "database/sql"

    "github.com/bartmika/mulberry-server/pkg/models"
)

// UserRepo implements models.UserRepository
type UserRepo struct {
    db *sql.DB
}

func NewUserRepo(db *sql.DB) *UserRepo {
    return &UserRepo{
        db: db,
    }
}

func (r *UserRepo) FindByID(ID int) (*models.User, error) {
    return &models.User{}, nil //TODO: Implement in the future.
}

// Save ..
func (r *UserRepo) Save(user *models.User) error {
    return nil //TODO: Implement in the future.
}
{{</ highlight >}}

### Part 2: Application Layer

This is how the **main.go** file will look like.

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/cmd/serve/main.go
package main

import (
    "fmt"
    "net/http"

    sqldb "github.com/bartmika/mulberry-server/pkg/db"
    "github.com/bartmika/mulberry-server/internal/repositories"
    "github.com/bartmika/mulberry-server/internal/controllers"
)

func main() {
    db := sqldb.ConnectDB() //TODO: Implement in the future.

    userRepo := repositories.NewUserRepo(db)

    c := controllers.NewBaseHandler(userRepo)

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

    "github.com/bartmika/mulberry-server/internal/repositories"
)

type BaseHandler struct {
    UserRepo *repositories.UserRepo
}

func NewBaseHandler(u *repositories.UserRepo) (*BaseHandler) {
    return &BaseHandler{
        UserRepo: u,
    }
}

func (h *BaseHandler) HandleRequests(w http.ResponseWriter, req *http.Request) {
    //TODO: Implement in the future
    //--------------------------------------------------------------------------------
    // GET     | /api/v1/version                   | Get the application version    |
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

* Every time you want to add a new entity, you create a file in the *models*, *repositories* package and then write the API endpoint in the *controllers* package.
* If you want to swap the database from *mysql* to *postgres* then you can update the *db.go* file and everything works fine.

### Part 3: Enter the ``net/http`` standard library

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

    "github.com/bartmika/mulberry-server/internal/repositories"
)

type BaseHandler struct {
    UserRepo *repositories.UserRepo
}

func NewBaseHandler(u *repositories.UserRepo) (*BaseHandler) {
    return &BaseHandler{
        UserRepo: u,
    }
}

func (h *BaseHandler) HandleRequests(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path == "/api/v1/version" {
        h.getVersion(w, r)
        return
    }
    //TODO: Implement in the future
    //--------------------------------------------------------------------------------
    // GET     | /api/v1/version                   | Get the application version    |
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
|   │
|   └───📁controllers
|   │      📄controller.go
|   |      📄version.go
|   |      📄tsd.go
|   |
|   └───📁repositories
|          📄user.go
|
└───📁pkg
    |
    └───📁db
    |   📄db.go
    |
    └───📁models
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

And then update the **controller.go** file:

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"
    "strings"

    "github.com/bartmika/mulberry-server/internal/repositories"
)

type BaseHandler struct {
    UserRepo *repositories.UserRepo
}

func NewBaseHandler(u *repositories.UserRepo) (*BaseHandler) {
    return &BaseHandler{
        UserRepo: u,
    }
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

$ http get 127.0.0.1:5000/api/v1/time-series-data

$ http post 127.0.0.1:5000/api/v1/time-series-data instrument_id="1" timestamp="2021-01-01"

$ http get 127.0.0.1:5000/api/v1/time-series-datum/xxx

$ http put 127.0.0.1:5000/api/v1/time-series-datum/xxx instrument_id="1" timestamp="2021-01-02"

$ http delete 127.0.0.1:5000/api/v1/time-series-datum/xxx
```

If you got a ``200 OK`` response with some-sort of string message, then congratulations! You have implemented your server using only the ``net/http`` standard library.

# Final Thoughts - What's next?

That's it! We have ourselves a basic API web-app. We have dipped our feat on the shores of the ocean of Golang web-development and the water feels welcoming. Moving forward we'll need to cover more topics such as:

* How do we read the body of a request? How do we write responses?
* How do we handle using a database?
* How do we handle login and authenticated API endpoints?
* How do we handle sessions?
* How do we handle background processes?
