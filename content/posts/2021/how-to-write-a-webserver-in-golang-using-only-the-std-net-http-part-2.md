---
title: "How to write a webserver in Golang using only the std net/http - Part 2"
subtitle: "Learn how to implement the database layer for your webserver"
date: 2021-01-29T00:02:30-04:00
draft: false
categories:
- "development"
tags:
- "golang"
- "api"
---

![](https://images.pexels.com/photos/5582597/pexels-photo-5582597.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940)

The purpose of this article is to provide instructions on how to read body of *request* and use a *primitive database*.

<!--more-->

Let's begin on how to architect the data layer in our app. For beginners, working with a database is a little overwhelming - let's try setting up a project with a [a Simple JSON Database in Golang — Scribble](https://medium.com/@skdomino/scribble-a-tiny-json-database-in-golang-9817854deb05) and then in a later article replace it for something more standard used.

Just as a reminder this is part 2 of the series, you'll need to finish [part 2](/posts/2021/how-to-write-a-webserver-in-golang-using-only-the-std-net-http-part-1/) before continuing.

# How to Integrate with a Database

We will structure our data layer according to **Approach 3: Repository Interface per Model** as mentioned in [this article](https://archive.is/XxqyT).

## New Project Structure

We will start with the following project structure. Three new packages will be introduced: **repository**, **db** and **models**.

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
|   │      📄tsd.go
|   │      📄user.go
|   │      📄version.go
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
    |   📄user.go
    |
    └───📁utils
        📄password.go
```

More details as follows:

* **db** - This is the package which handles loading up the database we are using.
* **models** - This is the package which has the all the *structs* we will be using to represent our data in the *database*. In our first example we will create a model for the *User*; in addition, we need to provide an *interface* for all the *methods* that need to be implemented in the **repositories** code.
* **repositories** - This is the package which implements the *struct* and *methods* from the **models** code.

## Enter the Data Layer

Begin by installing the dependency with running this code. For more information go ahead and review the [Scribble](https://github.com/sdomino/scribble) third-party library.

```bash
$ go get github.com/sdomino/scribble
```

The **db.go** file will be implemented as follows:

{{< highlight golang "linenos=false">}}
package db

import (
    "github.com/sdomino/scribble"
)

func ConnectDB() (*scribble.Driver, error) {
    // The location of our db.
    dir := "./my_database"

    // a new scribble driver, providing the directory where it will be writing to,
    // and a qualified logger if desired
    db, err := scribble.New(dir, nil)
    if err != nil {
        return nil, err
    }

    return db, nil
}
{{</ highlight >}}

Next we will create a sample **user.go** file with the structure we can use in the **models** package. Please note, we are putting this file in the **pkg** folder in case we want to use this file in the future in another project. If you don't know what the **pkg** is then please read about it [here](https://stackoverflow.com/questions/47369621/what-is-the-use-of-pkg-directory-in-go#:~:text=The%20pkg%20directory%20contains%20Go,that%20object%20into%20many%20executables.).

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/models/user.go
package models

// The definition of the user record we will saving in our database.
type User struct {
    Uuid string          `json:"uuid"`
    Name string          `json:"name"`
    Email string         `json:"email"`
    PasswordHash string  `json:"password_hash"`
}

// The interface that *must* be implemented in the `repositories` package.
type UserRepository interface {
    Create(uuid string, name string, email string, passwordHash string) error
    FindByUuid(uuid string) (*User, error)
    FindByEmail(email string) (*User, error)
    Save(user *User) error
}
{{</ highlight >}}

Now we will implement the **repositories** package which is responsible for storing *structs* and their implementation of the interfaces in the **models** package. Remember once a struct has implemented all the required methods of an interface, then we can call those methods on them; as a result, this makes testing much easier.

{{< highlight golang "linenos=false">}}
// github.com/bartmika/mulberry-server/internal/repositories/user.go
package repositories

import (
    "encoding/json"

    "github.com/sdomino/scribble"
    "github.com/bartmika/mulberry-server/pkg/models"
)

// UserRepo implements models.UserRepository
type UserRepo struct {
    db *scribble.Driver
}

func NewUserRepo(db *scribble.Driver) *UserRepo {
    return &UserRepo{
        db: db,
    }
}

func (r *UserRepo) Create(uuid string, name string, email string, passwordHash string) error {
    u := models.User{
        Uuid: uuid,
        Name: name,
        Email: email,
        PasswordHash: passwordHash,
    }
    if err := r.db.Write("users", uuid, u); err != nil {
        return err
    }
    return nil
}

func (r *UserRepo) FindByUuid(uuid string) (*models.User, error) {
    u := models.User{}
    if err := r.db.Read("users", uuid, &u); err != nil {
        return nil, err
    }
    return &u, nil
}

func (r *UserRepo) FindByEmail(email string) (*models.User, error) {
    records, err := r.db.ReadAll("users")
    if err != nil {
        return nil, err
    }

    for _, f := range records {
        userFound := models.User{}
        if err := json.Unmarshal([]byte(f), &userFound); err != nil {
            return nil, err
        }
        if userFound.Email == email {
            return &userFound, nil
        }
    }
    return nil, nil
}

func (r *UserRepo) Save(user *models.User) error {
    if err := r.db.Write("users", user.Uuid, user); err != nil {
        return err
    }
    return nil
}
{{</ highlight >}}

Update the **main.go** file, notice the **NEW** comments to help see what we changed different.

{{< highlight golang "linenos=false">}}
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

    sqldb "github.com/bartmika/mulberry-server/pkg/db" // NEW: Import our database loader.
    "github.com/bartmika/mulberry-server/internal/repositories" // NEW: Load our model interfaces.
    "github.com/bartmika/mulberry-server/internal/controllers"
)

func main() {
    // NEW: Let us connect to our `scribble` database so our `repositories` can use it.
    db := sqldb.ConnectDB()

    // NEW: We define all the repositories that we are using.
    userRepo := repositories.NewUserRepo(db)

    // NEW: Load our repositories into our controller.
    c := controllers.NewBaseHandler(userRepo)

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
{{</ highlight >}}

# How to Read the Body from Requests

Great, we have a primitive data-structure, now *how do we use it*? In this section we will implement our **user api endpoints** to see how to utilize our database.

The first important note to take is that we want our API endpoints to *only* accept **application/json** for the *Content-Type header* so our Golang code can take advantage of the **encoding/json** std. As a result, we can use the **Encode** and **Decode** code provided by Golang via ["JSON and Go" blog post](https://blog.golang.org/json)!

We must store the request and response data format in our **models** package. Here is the changes we made:

```go
// github.com/bartmika/mulberry-server/internal/models/user.go
package models

// The definition of the user record we will saving in our database.
type User struct {
	Uuid string          `json:"uuid"`
	Name string          `json:"name"`
	Email string         `json:"email"`
	PasswordHash string  `json:"password_hash"`
}

// The interface that *must* be implemented in the `repositories` package.
type UserRepository interface {
	Create(uuid string, name string, email string, passwordHash string) error
	FindByUuid(uuid string) (*User, error)
	FindByEmail(email string) (*User, error)
	Save(user *User) error
}


// NEW: The struct used to represent the user's `register` POST request data.
type RegisterRequest struct {
    Name string          `json:"name"`
    Email string         `json:"email"`
    Password string      `json:"password"`
}

// NEW: The struct used to represent the system's response when the `register` POST request was a success.
type RegisterResponse struct {
    Message string          `json:"message"`
}

// NEW: The struct used to represent the user's `login` POST request data.
type LoginRequest struct {
    Email string         `json:"email"`
    Password string      `json:"password"`
}

// NEW: The struct used to represent the system's response when the `login` POST request was a success.
type LoginResponse struct {
    AccessToken string    `json:"access_token"`
}
```

Next we'll need to handle hashing passwords, create the **password.go** file with the following contents:

```go
// github.com/bartmika/mulberry-server/pkg/utils/password.go
package utils

import (
    "golang.org/x/crypto/bcrypt"
)

// Function takes the plaintext string and returns a hash string.
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

// Function checks the plaintext string and hash string and returns either true
// or false depending.
func CheckPasswordHash(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

Next let's utilize the *Encode* and *Decode* functions. Our **user.go** file will look as follows:

```go
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/controllers/user.go
package controllers

import (
    "fmt"
    "net/http"
    "encoding/json"

    "github.com/google/uuid"

    "github.com/bartmika/mulberry-server/pkg/models"
    "github.com/bartmika/mulberry-server/pkg/utils"
)

// To run this API, try running in your console:
// $ http post 127.0.0.1:5000/api/v1/register email="fherbert@dune.com" password="the-spice-must-flow" name="Frank Herbert"
func (h *BaseHandler) postRegister(w http.ResponseWriter, r *http.Request) {
    // Initialize our array which will store all the results from the remote server.
    var requestData models.RegisterRequest

    // Read the JSON string and convert it into our golang stuct else we need
    // to send a `400 Bad Request` errror message back to the client,
    err := json.NewDecoder(r.Body).Decode(&requestData) // [1]
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // For debugging purposes, print our output so you can see the code working.
    fmt.Println(requestData.Name)
    fmt.Println(requestData.Email)
    fmt.Println(requestData.Password)

    // Lookup the email and if it is not unique we need to generate a `400 Bad Request` response.
    if userFound, _ := h.UserRepo.FindByEmail(requestData.Email); userFound != nil {
        http.Error(w, "Email alread exists", http.StatusBadRequest)
        return
    }

    // Generate a `UUID` for our record.
    uid := uuid.New().String()

    // Secure our password.
    passwordHash, err := utils.HashPassword(requestData.Password)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Save our new user account.
    h.UserRepo.Create(uid, requestData.Name, requestData.Email, passwordHash)

    // Generate our response.
    responseData := models.RegisterResponse{
        Message: "You have successfully registered an account.",
    }
    if err := json.NewEncoder(w).Encode(&responseData); err != nil {  // [2]
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// To run this API, try running in your console:
// $ http post 127.0.0.1:5000/api/v1/login email="fherbert@dune.com" password="the-spice-must-flow"
func (h *BaseHandler) postLogin(w http.ResponseWriter, r *http.Request) {
    var requestData models.LoginRequest

    err := json.NewDecoder(r.Body).Decode(&requestData)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // For debugging purposes, print our output so you can see the code working.
    fmt.Println(requestData.Email)
    fmt.Println(requestData.Password)

    // Lookup the user in our database, else return a `400 Bad Request` error.
    user, err := h.UserRepo.FindByEmail(requestData.Email)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    if user == nil {
        http.Error(w, "Email does not exist", http.StatusBadRequest)
        return
    }

    // Verify the inputted password and hashed password match.
    passwordMatch := utils.CheckPasswordHash(requestData.Password, user.PasswordHash)
    if passwordMatch == false {
        http.Error(w, "Incorrect password", http.StatusBadRequest)
        return
    }

    // Finally return success.
    responseData := models.LoginResponse{
        AccessToken: "TODO: WE WILL FIGURE OUT HOW TO DO THIS IN ANOTHER ARTICLE!",
    }
    if err := json.NewEncoder(w).Encode(&responseData); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// SPECIAL THANKS:
// [1][2]: Learned from:
// a. https://blog.golang.org/json
// b. https://stackoverflow.com/questions/21197239/decoding-json-using-json-unmarshal-vs-json-newdecoder-decode
```

With the above code there are few items to note:

* The ``json.NewDecoder`` and ``Decode`` will take the request *bytes* and populate a *struct*.
* The ``json.NewEncoder`` and ``Encode`` will take the response *struct* and populate a *bytes* array.

## API Calling
### Register with Success

Let's attempt to *register*. Start the server and run the following commands in your console:

```bash
$ http post 127.0.0.1:5000/api/v1/register email="fherbert@dune.com" \
                                        password="the-spice-must-flow" \
                                            name="Frank Herbert"
```

The output should be as follows:

```
HTTP/1.1 200 OK
Content-Length: 59
Content-Type: application/json
Date: Sat, 30 Jan 2021 20:22:48 GMT

{
    "message": "You have successfully registered an account."
}
```

### Register with Failure
Now if you rerun the same command you will see our validation successfully works:

```
HTTP/1.1 400 Bad Request
Content-Length: 20
Content-Type: text/plain; charset=utf-8
Date: Sat, 30 Jan 2021 20:23:42 GMT
X-Content-Type-Options: nosniff

Email already exists
```

### Login with Failure (1 of 2)
Next let's try to *login*. Run the following in your console to test out the *Email does not exist* validation:

```
$ http post 127.0.0.1:5000/api/v1/login email="patreides@dune.com" \
                                     password="house-of-the-atreides"
```

We should see an output something like:

```
HTTP/1.1 400 Bad Request
Content-Length: 21
Content-Type: text/plain; charset=utf-8
Date: Sat, 30 Jan 2021 20:29:10 GMT
X-Content-Type-Options: nosniff

Email does not exist
```

### Login with Failure (2 of 2)
Next let's verify our invalid password validation works:

```
$ http post 127.0.0.1:5000/api/v1/login email="fherbert@dune.com" \
                                     password="house-of-harkonnen"
```

With the following output:

```
HTTP/1.1 400 Bad Request
Content-Length: 19
Content-Type: text/plain; charset=utf-8
Date: Sat, 30 Jan 2021 20:31:08 GMT
X-Content-Type-Options: nosniff

Incorrect password
```

### Login with Success

And for our grand finally, run the login command which works:

```
$ http post 127.0.0.1:5000/api/v1/login email="fherbert@dune.com" \
                                     password="the-spice-must-flow"
```

Wonderful! The success output will be as follows:

```
HTTP/1.1 200 OK
Content-Length: 79
Content-Type: application/json
Date: Sat, 30 Jan 2021 20:35:02 GMT

{
    "access_token": "TODO: WE WILL FIGURE OUT HOW TO DO THIS IN ANOTHER ARTICLE!"
}
```
