---
title: "How to Build an API Server in Go - Part 2: Simple Database"
date: 2021-01-29T00:02:30-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "Golang"
- "API"
- "Database"
- "HOWTO"
---

![](/img/2021/common/go-banner.png)

The purpose of this post is to learn how our [basic API server](/post/2021/how-to-build-an-api-server-in-go-part-1-basic-server/) can read the body of a *request*. In addition, we will learn how to use an easy-to-use *simple database* for beginners called [scribble](https://github.com/sdomino/scribble).

<!--more-->

Let's begin on how to architect the data layer in our app. For beginners, working with a database is a little overwhelming - let's try setting up a project with a [a Simple JSON Database in Golang — Scribble](https://medium.com/@skdomino/scribble-a-tiny-json-database-in-golang-9817854deb05) and then in a later article replace it for something more standard used.

This post belongs to the following series:
1. [How to Build an API Server in Go - Part 1: Basic Server](/post/2021/how-to-build-an-api-server-in-go-part-1-basic-server/)
2. **How to Build an API Server in Go - Part 2: Simple Database**
3. [How to Build an API Server in Go - Part 3: Postgres Database](/post/2021/how-to-build-an-api-server-in-go-part-3-postgres-database/)
4. [How to Build an API Server in Go - Part 4: Access Control](/post/2021/how-to-build-an-api-server-in-go-part-4-access-control/)


# How to Integrate with a Database

We will structure our data layer according to **Approach 3: Repository Interface per Model** as mentioned in [this article](https://archive.is/XxqyT).

## New Project Structure

We will start with the following project structure. Three new packages will be introduced: **repository**, **db** and **models**.

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
{{</ highlight >}}

More details as follows:

* **db** - This is the package which handles loading up the database we are using.
* **models** - This is the package which has the all the *structs* we will be using to represent our data in the *database*. In our first example we will create a model for the *User*; in addition, we need to provide an *interface* for all the *methods* that need to be implemented in the **repositories** code.
* **repositories** - This is the package which implements the *struct* and *methods* from the **models** code.

## Enter the Data Layer

Begin by installing the dependency with running this code. For more information go ahead and review the [Scribble](https://github.com/sdomino/scribble) third-party library.

{{< highlight bash "linenos=false">}}
$ go get github.com/sdomino/scribble
{{</ highlight >}}

The **db.go** file will be implemented as follows:

{{< highlight go "linenos=true">}}
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

{{< highlight go "linenos=table">}}
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

{{< highlight go "linenos=table">}}
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

{{< highlight go "linenos=table">}}
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
    c := controllers.New(userRepo)

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

# How to Read the Body from Requests

Great, we have a primitive data-structure, now *how do we use it*? In this section we will implement our **user api endpoints** to see how to utilize our database.

The first important note to take is that we want our API endpoints to *only* accept **application/json** for the *Content-Type header* so our Golang code can take advantage of the **encoding/json** std. As a result, we can use the **Encode** and **Decode** code provided by Golang via ["JSON and Go" blog post](https://blog.golang.org/json)!

We must store the request and response data format in our **models** package. Here is the changes we made:

{{< highlight go "linenos=true">}}
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
{{</ highlight >}}

Next we'll need to handle hashing passwords, create the **password.go** file with the following contents:

{{< highlight go "linenos=true">}}
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
{{</ highlight >}}

Next let's utilize the *Encode* and *Decode* functions. Our **user.go** file will look as follows:

{{< highlight go "linenos=true">}}
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
func (c *Controller) postRegister(w http.ResponseWriter, r *http.Request) {
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
    if userFound, _ := c.UserRepo.FindByEmail(requestData.Email); userFound != nil {
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
    c.UserRepo.Create(uid, requestData.Name, requestData.Email, passwordHash)

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
func (c *Controller) postLogin(w http.ResponseWriter, r *http.Request) {
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
    user, err := c.UserRepo.FindByEmail(requestData.Email)
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
{{</ highlight >}}

With the above code there are few items to note:

* The ``json.NewDecoder`` and ``Decode`` will take the request *bytes* and populate a *struct*.
* The ``json.NewEncoder`` and ``Encode`` will take the response *struct* and populate a *bytes* array.

# Making API Calls
### Register with Success

Let's attempt to *register*. Start the server and run the following commands in your console:

{{< highlight bash "linenos=false">}}
$ http post 127.0.0.1:5000/api/v1/register email="fherbert@dune.com" \
                                        password="the-spice-must-flow" \
                                            name="Frank Herbert"
{{</ highlight >}}

The output should be as follows:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 59
Content-Type: application/json
Date: Sat, 30 Jan 2021 20:22:48 GMT

{
    "message": "You have successfully registered an account."
}
{{</ highlight >}}

### Register with Failure
Now if you rerun the same command you will see our validation successfully works:

{{< highlight bash "linenos=false">}}
HTTP/1.1 400 Bad Request
Content-Length: 20
Content-Type: text/plain; charset=utf-8
Date: Sat, 30 Jan 2021 20:23:42 GMT
X-Content-Type-Options: nosniff

Email already exists
{{</ highlight >}}

### Login with Failure (1 of 2)
Next let's try to *login*. Run the following in your console to test out the *Email does not exist* validation:

{{< highlight bash "linenos=false">}}
$ http post 127.0.0.1:5000/api/v1/login email="patreides@dune.com" \
                                     password="house-of-the-atreides"
{{</ highlight >}}

We should see an output something like:

{{< highlight bash "linenos=false">}}
HTTP/1.1 400 Bad Request
Content-Length: 21
Content-Type: text/plain; charset=utf-8
Date: Sat, 30 Jan 2021 20:29:10 GMT
X-Content-Type-Options: nosniff

Email does not exist
{{</ highlight >}}

### Login with Failure (2 of 2)
Next let's verify our invalid password validation works:

{{< highlight bash "linenos=false">}}
$ http post 127.0.0.1:5000/api/v1/login email="fherbert@dune.com" \
                                     password="house-of-harkonnen"
{{</ highlight >}}

With the following output:

{{< highlight bash "linenos=false">}}
HTTP/1.1 400 Bad Request
Content-Length: 19
Content-Type: text/plain; charset=utf-8
Date: Sat, 30 Jan 2021 20:31:08 GMT
X-Content-Type-Options: nosniff

Incorrect password
{{</ highlight >}}

### Login with Success

And for our grand finally, run the login command which works:

{{< highlight bash "linenos=false">}}
$ http post 127.0.0.1:5000/api/v1/login email="fherbert@dune.com" \
                                     password="the-spice-must-flow"
{{</ highlight >}}

Wonderful! The success output will be as follows:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 79
Content-Type: application/json
Date: Sat, 30 Jan 2021 20:35:02 GMT

{
    "access_token": "TODO: WE WILL FIGURE OUT HOW TO DO THIS IN ANOTHER ARTICLE!"
}
{{</ highlight >}}

Please note that we will discuss **access tokens** and **user authentication** in another article. For now we will only return this placeholder message.

# More code - Time-Series Data

Let us implement our *time-series datum* data-structure in our application. Remember the steps are as follows:

* Always start with the **models** package
* Next write our file in the **repositories** package
* Then update the **controller.go** and **main.go** files
* You are ready to utilize your data-structure in all your API endpoints!

## Data Layer

Start by create our **tsd.go** file in the **models** package:

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/models/tsd.go
package models

import (
    "time"
)

type TimeSeriesDatum struct {
    Uuid string `json:"uuid"`
    InstrumentUuid string `json:"instrument_uuid"`
    Value float64 `json:"value"`
    Timestamp time.Time `json:"timestamp"`
    UserUuid string `json:"user_uuid"`
}

type TimeSeriesDatumRepository interface {
    Create(uuid string, instrumentUuid string, value float64, timestamp time.Time, userUuid string) error
    ListAll() ([]*TimeSeriesDatum, error)
    FilterByUserUuid(userUuid string) ([]*TimeSeriesDatum, error)
    FindByUuid(uuid string) (*TimeSeriesDatum, error)
    DeleteByUuid(uuid string) error
    Save(datum *TimeSeriesDatum) error
}

type TimeSeriesDatumCreateRequest struct {
    InstrumentUuid string `json:"instrument_uuid"`
    Value float64 `json:"value,string"`
    Timestamp time.Time `json:"timestamp"`
    UserUuid string `json:"user_uuid"`
}

type TimeSeriesDatumCreateResponse struct {
    Uuid string `json:"uuid"`
    InstrumentUuid string `json:"instrument_uuid"`
    Value float64 `json:"value,string"`
    Timestamp time.Time `json:"timestamp"`
    UserUuid string `json:"user_uuid"`
}

type TimeSeriesDatumPutRequest struct {
    InstrumentUuid string `json:"instrument_uuid"`
    Value float64 `json:"value,string"`
    Timestamp time.Time `json:"timestamp"`
    UserUuid string `json:"user_uuid"`
}
{{</ highlight >}}

Next we need to write the **tsd.go** file inside our **repositories** package:

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/repositories/tsd.go
package repositories

import (
    "encoding/json"
    "time"

	"github.com/sdomino/scribble"
	"github.com/bartmika/mulberry-server/pkg/models"
)

// TimeSeriesDatumRepo implements models.TimeSeriesDatumRepository
type TimeSeriesDatumRepo struct {
	db *scribble.Driver
}

func NewTimeSeriesDatumRepo(db *scribble.Driver) *TimeSeriesDatumRepo {
	return &TimeSeriesDatumRepo{
		db: db,
	}
}

func (r *TimeSeriesDatumRepo) Create(uuid string, instrumentUuid string, value float64, timestamp time.Time, userUuid string) error {
	tsd := models.TimeSeriesDatum{
		Uuid: uuid,
		InstrumentUuid: instrumentUuid,
		Value: value,
		Timestamp: timestamp,
        UserUuid: userUuid,
	}
	if err := r.db.Write("time_series_data", uuid, &tsd); err != nil {
		return err
	}
	return nil
}

func (r *TimeSeriesDatumRepo) ListAll() ([]*models.TimeSeriesDatum, error) {
    var results []*models.TimeSeriesDatum
    records, err := r.db.ReadAll("time_series_data")
	if err != nil {
        return nil, err
	}

    for _, f := range records {
		tsdFound := models.TimeSeriesDatum{}
        if err := json.Unmarshal([]byte(f), &tsdFound); err != nil {
            return nil, err
		}
		results = append(results, &tsdFound)
	}
	return results, nil
}

func (r *TimeSeriesDatumRepo) FilterByUserUuid(userUuid string) ([]*models.TimeSeriesDatum, error) {
    var results []*models.TimeSeriesDatum
    records, err := r.db.ReadAll("time_series_data")
	if err != nil {
        return nil, err
	}

    for _, f := range records {
		tsdFound := models.TimeSeriesDatum{}
        if err := json.Unmarshal([]byte(f), &tsdFound); err != nil {
            return nil, err
		}
		if tsdFound.UserUuid == userUuid {
			results = append(results, &tsdFound)
		}
	}
	return results, nil
}

func (r *TimeSeriesDatumRepo) FindByUuid(uuid string) (*models.TimeSeriesDatum, error) {
    tsd := models.TimeSeriesDatum{}
	if err := r.db.Read("time_series_data", uuid, &tsd); err != nil {
		return nil, err
	}
	return &tsd, nil
}


func (r *TimeSeriesDatumRepo) DeleteByUuid(uuid string) error {
    if err := r.db.Delete("time_series_data", uuid); err != nil {
		return err
	}
	return nil
}

func (r *TimeSeriesDatumRepo) Save(tsd *models.TimeSeriesDatum) error {
	if err := r.db.Write("time_series_data", tsd.Uuid, tsd); err != nil {
		return err
	}
	return nil
}
{{</ highlight >}}

## Application Layer

Update our **controller.go** to look as follows:

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"
    "strings"

    "github.com/bartmika/mulberry-server/internal/repositories"
)

type Controller struct {
    UserRepo *repositories.UserRepo
    TsdRepo *repositories.TimeSeriesDatumRepo // NEW
}

func New(u *repositories.UserRepo, tsd *repositories.TimeSeriesDatumRepo) (*Controller) { // NEW
    return &Controller{
        UserRepo: u,
        TsdRepo: tsd, // NEW
    }
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
    case n == 1 && p[0] == "login" && r.Method == http.MethodPost:
        c.postLogin(w, r)
    case n == 1 && p[0] == "register" && r.Method == http.MethodPost:
        c.postRegister(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodGet:
        c.getTimeSeriesData(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodPost:
        c.postTimeSeriesData(w, r)
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

And finally update the **main.go** to support our new data-structure.

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

    sqldb "github.com/bartmika/mulberry-server/pkg/db"
    "github.com/bartmika/mulberry-server/internal/repositories"
    "github.com/bartmika/mulberry-server/internal/controllers"
)

func main() {
    db, err := sqldb.ConnectDB()
    if err != nil {
        log.Fatal(err)
    }

    userRepo := repositories.NewUserRepo(db)
    tsdRepo := repositories.NewTimeSeriesDatumRepo(db)

    c := controllers.New(userRepo, tsdRepo)

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

## Utilizing in our API Endpoints

Update our **tsd.go** file as follows:

{{< highlight go "linenos=true">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/controllers/tsd.go
package controllers

import (
    "fmt"
    "net/http"
    "encoding/json"

    "github.com/google/uuid"

    "github.com/bartmika/mulberry-server/pkg/models"
)

// To run this API, try running in your console:
// $ http get 127.0.0.1:5000/api/v1/time-series-data
func (c *Controller) getTimeSeriesData(w http.ResponseWriter, req *http.Request) {
    //TODO: Add filtering based on the authenticated user account. For now just list all the records.
    //      In a future article we will update this code.
    results, err := c.TsdRepo.ListAll()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Encode our results
    if err := json.NewEncoder(w).Encode(&results); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// To run this API, try running in your console:
// $ http post 127.0.0.1:5000/api/v1/time-series-data instrument_uuid="lalala" value="123" timestamp="2021-01-30T10:20:10.000Z" user_uuid="lalala"
func (c *Controller) postTimeSeriesData(w http.ResponseWriter, r *http.Request) {
    var requestData models.TimeSeriesDatumCreateRequest
    if err := json.NewDecoder(r.Body).Decode(&requestData); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // For debugging purposes only.
    fmt.Println(requestData.InstrumentUuid)
    fmt.Println(requestData.Value)
    fmt.Println(requestData.Timestamp)
    fmt.Println(requestData.UserUuid)

    // Generate a `UUID` for our record.
    uid := uuid.New().String()

    // Save to our database.
    err := c.TsdRepo.Create(uid, requestData.InstrumentUuid, requestData.Value, requestData.Timestamp, requestData.UserUuid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusCreated)

    // Return our record.
    responseData := models.TimeSeriesDatumCreateResponse{
        Uuid: uid,
        InstrumentUuid: requestData.InstrumentUuid,
        Value: requestData.Value,
        Timestamp: requestData.Timestamp,
        UserUuid: requestData.UserUuid,
    }
    if err := json.NewEncoder(w).Encode(&responseData); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// To run this API, try running in your console:
// $ http get 127.0.0.1:5000/api/v1/time-series-datum/f3e7b442-f3d4-4c2f-8f8d-d347982c1569
func (c *Controller) getTimeSeriesDatum(w http.ResponseWriter, req *http.Request, uuid string) {
    // Lookup our record.
    tsd, err := c.TsdRepo.FindByUuid(uuid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Return our record to the user as a response.
    if err := json.NewEncoder(w).Encode(&tsd); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// To run this API, try running in your console:
// $ http put 127.0.0.1:5000/api/v1/time-series-datum/f3e7b442-f3d4-4c2f-8f8d-d347982c1569 instrument_uuid="lalala" value="321" timestamp="2021-01-30T10:20:10.000Z" user_uuid="lalala"
func (c *Controller) putTimeSeriesDatum(w http.ResponseWriter, r *http.Request, uid string) {
    var requestData models.TimeSeriesDatumPutRequest
    if err := json.NewDecoder(r.Body).Decode(&requestData); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // For debugging purposes only.
    fmt.Println(uid)
    fmt.Println(requestData.InstrumentUuid)
    fmt.Println(requestData.Value)
    fmt.Println(requestData.Timestamp)
    fmt.Println(requestData.UserUuid)

    // Update our record.
    err := c.TsdRepo.Save(&models.TimeSeriesDatum{
        Uuid: uid,
        InstrumentUuid: requestData.InstrumentUuid,
        Value: requestData.Value,
        Timestamp: requestData.Timestamp,
        UserUuid: requestData.UserUuid,
    })
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// To run this API, try running in your console:
// $ http delete 127.0.0.1:5000/api/v1/time-series-datum/f3e7b442-f3d4-4c2f-8f8d-d347982c1569
func (c *Controller) deleteTimeSeriesDatum(w http.ResponseWriter, req *http.Request, uid string) {
    if err := c.TsdRepo.DeleteByUuid(uid); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
    w.WriteHeader(http.StatusOK) // Note: https://tools.ietf.org/html/rfc7231#section-6.3.1
}
{{</ highlight >}}

## Making API calls

Let's begin by making a post. Please note that the *timestamp* must be formatted using **"RFC3339" date/time standard** - For more information please read this [great article](https://archive.is/7cqtR).

## Create with Success

Run the following in your console:

{{< highlight go "linenos=false">}}
$ http post 127.0.0.1:5000/api/v1/time-series-data instrument_uuid="lalala" \
                                                             value="123" \
                                                         timestamp="2021-01-30T10:20:10.000Z" \
                                                         user_uuid="bababa"
{{</ highlight >}}

And you should get the following message:

{{< highlight go "linenos=false">}}
HTTP/1.1 201 Created
Content-Length: 145
Content-Type: application/json
Date: Sat, 30 Jan 2021 22:36:44 GMT

{
    "instrument_uuid": "lalala",
    "timestamp": "2021-01-30T10:20:10Z",
    "user_uuid": "bababa",
    "uuid": "8ca9c245-9b48-44ce-b3bf-5b6262deb92f",
    "value": "123"
}
{{</ highlight >}}

## List with Success

Run the following in your console:

{{< highlight go "linenos=false">}}
$ http get 127.0.0.1:5000/api/v1/time-series-data
{{</ highlight >}}

And you should get the following message:

{{< highlight go "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 145
Content-Type: application/json
Date: Sat, 30 Jan 2021 22:37:27 GMT

[
    {
        "instrument_uuid": "lalala",
        "timestamp": "2021-01-30T10:20:10Z",
        "user_uuid": "bababa",
        "uuid": "8ca9c245-9b48-44ce-b3bf-5b6262deb92f",
        "value": 123
    }
]
{{</ highlight >}}

## Retrieve with Success

Run the following in your console:

{{< highlight go "linenos=false">}}
$ http get 127.0.0.1:5000/api/v1/time-series-datum/8ca9c245-9b48-44ce-b3bf-5b6262deb92f
{{</ highlight >}}

And you should get the following message:

{{< highlight go "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 143
Content-Type: application/json
Date: Sat, 30 Jan 2021 22:39:04 GMT

{
    "instrument_uuid": "lalala",
    "timestamp": "2021-01-30T10:20:10Z",
    "user_uuid": "bababa",
    "uuid": "8ca9c245-9b48-44ce-b3bf-5b6262deb92f",
    "value": 123
}
{{</ highlight >}}

## Update with Success

Run the following in your console:

{{< highlight go "linenos=false">}}
$ http put 127.0.0.1:5000/api/v1/time-series-datum/8ca9c245-9b48-44ce-b3bf-5b6262deb92f \
instrument_uuid="lalala" \
          value="321" \
      timestamp="2021-01-30T10:20:10.000Z" \
      user_uuid="bababa"
{{</ highlight >}}

And you should get the following message:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 143
Content-Type: application/json
Date: Sat, 30 Jan 2021 22:42:41 GMT

{
    "instrument_uuid": "lalala",
    "timestamp": "2021-01-30T10:20:10Z",
    "user_uuid": "bababa",
    "uuid": "8ca9c245-9b48-44ce-b3bf-5b6262deb92f",
    "value": 321
}
{{</ highlight >}}

## Delete with Success

Run the following in your console:

{{< highlight bash "linenos=false">}}
$ http delete 127.0.0.1:5000/api/v1/time-series-datum/8ca9c245-9b48-44ce-b3bf-5b6262deb92f
{{</ highlight >}}

And you should get the following message:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 0
Content-Type: application/json
Date: Sat, 30 Jan 2021 22:43:32 GMT


{{</ highlight >}}

# Final Thoughts - What's next?

Woh, that's a lot! Hopefully this article was helpful for you. We still have the following to consider:

* How would our code work if we used a *real* database?
* How do we handle sessions? Access tokens?
* How do we handle background processes?
* How do we handle pagination in our list API endpoint?

Now onward to the next part of our series: [**Part 3: Postgres Database >>**](/post/2021/how-to-build-an-api-server-in-go-part-3-postgres-database/).
