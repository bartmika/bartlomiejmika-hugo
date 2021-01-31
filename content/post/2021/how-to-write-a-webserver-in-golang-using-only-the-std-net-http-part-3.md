---
title: "How to write a webserver in Golang using only the std net/http - Part 3"
subtitle: "Learn how to implement the database layer for your webserver"
date: 2021-01-30T00:02:30-04:00
draft: false
categories:
- "development"
tags:
- "golang"
- "api"
---

<!-- ![](https://images.pexels.com/photos/5582597/pexels-photo-5582597.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940) -->

The purpose of this article is to provide instructions on how to setup *postgres* database with our application.

<!--more-->

Just as a reminder this is part 3 of the series, you'll need to finish [part 2](/posts/2021/how-to-write-a-webserver-in-golang-using-only-the-std-net-http-part-1/) before continuing.

# Database Setup

Before we begin, please download and install [**postgres database**](https://www.postgresql.org/) onto your computer.

Load up your *terminal* or *postgres GUI* and run the following code:

{{< highlight sql "linenos=true">}}
drop database mulberry_db;
create database mulberry_db;
\c mulberry_db;
CREATE USER golang WITH PASSWORD '123password';
GRANT ALL PRIVILEGES ON DATABASE mulberry_db to golang;
ALTER USER golang CREATEDB;

{{</ highlight >}}

Afterwords we will setup our application structure. Copy and paste the following:

{{< highlight sql "linenos=true">}}
CREATE TABLE users (
    uuid VARCHAR (36) PRIMARY KEY,
    name VARCHAR (255) NOT NULL,
    email VARCHAR (255) UNIQUE NOT NULL,
    password_hash VARCHAR (511) NOT NULL
);
CREATE UNIQUE INDEX idx_user_uuid
ON users (uuid);
CREATE UNIQUE INDEX idx_user_email
ON users (email);

CREATE TABLE time_series_data (
    uuid VARCHAR (36) PRIMARY KEY,
    instrument_uuid VARCHAR (36) UNIQUE NOT NULL,
    value FLOAT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    user_uuid VARCHAR (36) UNIQUE NOT NULL,
    FOREIGN KEY (user_uuid) REFERENCES users(uuid)
);
CREATE UNIQUE INDEX idx_time_series_data_uuid
ON time_series_data (uuid);
{{</ highlight >}}

# Data Layer

The project structure will remain the same as from [part 2](/posts/2021/how-to-write-a-webserver-in-golang-using-only-the-std-net-http-part-1/). We will begin with updating the files in the following order:

1. mulberry-server/pkg/db/db.go
2. mulberry-server/pkg/models/user.go
3. mulberry-server/pkg/models/tsd.go
4. mulberry-server/internal/respositories/user.go
5. mulberry-server/internal/respositories/tsd.go

## Models Package

Let's begin rewriting the data layer by starting in the **db.go** file. Please note we will be utilizing a third-party library for handling communication with the **postgres** database.

### db.go

{{< highlight go "linenos=true">}}
package db

import (
    "fmt"
    "database/sql"

    _ "github.com/lib/pq"
)

func ConnectDB(databaseHost, databasePort, databaseUser, databasePassword, databaseName string) (*sql.DB, error) {
    psqlInfo := fmt.Sprintf("host=%s port=%s user=%s "+"password=%s dbname=%s sslmode=disable",
        databaseHost,
        databasePort,
        databaseUser,
        databasePassword,
        databaseName,
    )

    dbInstance, err := sql.Open("postgres", psqlInfo)
    if err != nil {
        return nil, err
    }
    err = dbInstance.Ping()
    if err != nil {
        return nil, err
    }
    return dbInstance, nil
}
{{</ highlight >}}

### user.go

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/models/user.go
package models

import (
	"context"
)

// The definition of the user record we will saving in our database.
type User struct {
    Uuid string          `json:"uuid"`
    Name string          `json:"name"`
    Email string         `json:"email"`
    PasswordHash string  `json:"password_hash"`
}

// The interface that *must* be implemented.
type UserRepository interface {
    Create(ctx context.Context, uuid string, name string, email string, passwordHash string) error
    FindByUuid(ctx context.Context, uuid string) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
    Save(ctx context.Context, user *User) error
}

// The struct used to represent the user's `register` POST request data.
type RegisterRequest struct {
    Name string     `json:"name"`
    Email string    `json:"email"`
    Password string `json:"password"`
}

// The struct used to represent the system's response when the `register` POST request was a success.
type RegisterResponse struct {
    Message string `json:"message"`
    Uuid string    `json:"uuid"`
}

// The struct used to represent the user's `login` POST request data.
type LoginRequest struct {
    Email string    `json:"email"`
    Password string `json:"password"`
}

// The struct used to represent the system's response when the `login` POST request was a success.
type LoginResponse struct {
    AccessToken string `json:"access_token"`
    Uuid string        `json:"uuid"`
}
{{</ highlight >}}

### tsd.go

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/models/tsd.go
package models

import (
    "context"
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
    Create(ctx context.Context, uuid string, instrumentUuid string, value float64, timestamp time.Time, userUuid string) error
    ListAll(ctx context.Context) ([]*TimeSeriesDatum, error)
    FilterByUserUuid(ctx context.Context, userUuid string) ([]*TimeSeriesDatum, error)
    FindByUuid(ctx context.Context, uuid string) (*TimeSeriesDatum, error)
    DeleteByUuid(ctx context.Context, uuid string) error
    Save(ctx context.Context, datum *TimeSeriesDatum) error
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

## Repositories Package
Now we will move to the **repositories** package and start with the **user.go** file:

### user.go

{{< highlight go "linenos=true">}}
package repositories

import (
    "context"
    "database/sql"
    // "encoding/json"
    "time"

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

func (r *UserRepo) Create(ctx context.Context, uuid string, name string, email string, passwordHash string) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    query := "INSERT INTO users (uuid, name, email, password_hash) VALUES ($1, $2, $3, $4)"

    stmt, err := r.db.PrepareContext(ctx, query)
    if err != nil {
        return err
    }
    defer stmt.Close()

    _, err = stmt.ExecContext(
        ctx,
        uuid,
        name,
        email,
        passwordHash,
    )
    return err
}

func (r *UserRepo) FindByUuid(ctx context.Context, uuid string) (*models.User, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    m := new(models.User)

    query := "SELECT uuid, name, email, password_hash FROM users WHERE uuid = $1"
    err := r.db.QueryRowContext(ctx, query, uuid).Scan(
        &m.Uuid,
        &m.Name,
        &m.Email,
        &m.PasswordHash,
    )
    if err != nil {
        // CASE 1 OF 2: Cannot find record with that uuid.
        if err == sql.ErrNoRows {
            return nil, nil
        } else { // CASE 2 OF 2: All other errors.
            return nil, err
        }
    }
    return m, nil
}

func (r *UserRepo) FindByEmail(ctx context.Context, email string) (*models.User, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    m := new(models.User)

    query := "SELECT uuid, name, email, password_hash FROM users WHERE email = $1"
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &m.Uuid,
        &m.Name,
        &m.Email,
        &m.PasswordHash,
    )
    if err != nil {
        // CASE 1 OF 2: Cannot find record with that email.
        if err == sql.ErrNoRows {
            return nil, nil
        } else { // CASE 2 OF 2: All other errors.
            return nil, err
        }
    }
    return m, nil
}

func (r *UserRepo) Save(ctx context.Context, m *models.User) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    query := "UPDATE users SET name = $1, email = $2, password_hash = $3 WHERE uuid = $4"
    stmt, err := r.db.PrepareContext(ctx, query)
    if err != nil {
        return err
    }
    defer stmt.Close()

    _, err = stmt.ExecContext(
        ctx,
        m.Name,
        m.Email,
        m.PasswordHash,
        m.Uuid,
    )
    return err
}
{{</ highlight >}}

### tsd.go

{{< highlight go "linenos=true">}}
package repositories

import (
    "context"
    "database/sql"
    // "encoding/json"
    "time"

    "github.com/bartmika/mulberry-server/pkg/models"
)

// TimeSeriesDatumRepo implements models.TimeSeriesDatumRepository
type TimeSeriesDatumRepo struct {
    db *sql.DB
}

func NewTimeSeriesDatumRepo(db *sql.DB) *TimeSeriesDatumRepo {
    return &TimeSeriesDatumRepo{
        db: db,
    }
}

func (r *TimeSeriesDatumRepo) Create(ctx context.Context, uuid string, instrumentUuid string, value float64, timestamp time.Time, userUuid string) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    query := "INSERT INTO time_series_data (uuid, instrument_uuid, value, timestamp, user_uuid) VALUES ($1, $2, $3, $4, $5)"

    stmt, err := r.db.PrepareContext(ctx, query)
    if err != nil {
        return err
    }
    defer stmt.Close()

    _, err = stmt.ExecContext(
        ctx,
        uuid,
        instrumentUuid,
        value,
        timestamp,
        userUuid,
    )
    return err
}

func (r *TimeSeriesDatumRepo) ListAll(ctx context.Context) ([]*models.TimeSeriesDatum, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    query := "SELECT uuid, instrument_uuid, value, timestamp, user_uuid FROM time_series_data"

    rows, err := r.db.QueryContext(ctx, query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var s []*models.TimeSeriesDatum
    for rows.Next() {
        m := new(models.TimeSeriesDatum)
        err = rows.Scan(
            &m.Uuid,
            &m.InstrumentUuid,
            &m.Value,
            &m.Timestamp,
            &m.UserUuid,
        )
        if err != nil {
            return nil, err
        }
        s = append(s, m)
    }
    err = rows.Err()
    if err != nil {
        return nil, err
    }
    return s, err
}

func (r *TimeSeriesDatumRepo) FilterByUserUuid(ctx context.Context, userUuid string) ([]*models.TimeSeriesDatum, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    query := "SELECT uuid, instrument_uuid, value, timestamp, user_uuid FROM time_series_data WHERE user_uuid = $1"

    rows, err := r.db.QueryContext(ctx, query, userUuid)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var s []*models.TimeSeriesDatum
    for rows.Next() {
        m := new(models.TimeSeriesDatum)
        err = rows.Scan(
            &m.Uuid,
            &m.InstrumentUuid,
            &m.Value,
            &m.Timestamp,
            &m.UserUuid,
        )
        if err != nil {
            return nil, err
        }
        s = append(s, m)
    }
    err = rows.Err()
    if err != nil {
        return nil, err
    }
    return s, err
}

func (r *TimeSeriesDatumRepo) FindByUuid(ctx context.Context, uuid string) (*models.TimeSeriesDatum, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    m := new(models.TimeSeriesDatum)

    query := "SELECT uuid, instrument_uuid, value, timestamp, user_uuid FROM time_series_data WHERE uuid = $1"
    err := r.db.QueryRowContext(ctx, query, uuid).Scan(
        &m.Uuid,
        &m.InstrumentUuid,
        &m.Value,
        &m.Timestamp,
        &m.UserUuid,
    )
    if err != nil {
        // CASE 1 OF 2: Cannot find record with that uuid.
        if err == sql.ErrNoRows {
            return nil, nil
        } else { // CASE 2 OF 2: All other errors.
            return nil, err
        }
    }
    return m, nil
}

func (r *TimeSeriesDatumRepo) DeleteByUuid(ctx context.Context, uuid string) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    query := "DELETE FROM time_series_data WHERE uuid = $1;"

    _, err := r.db.Exec(query, uuid)
    if err != nil {
        return err
    }
    return nil
}

func (r *TimeSeriesDatumRepo) Save(ctx context.Context, m *models.TimeSeriesDatum) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    query := "UPDATE time_series_data SET instrument_uuid = $1, value = $2, timestamp = $3, user_uuid = $4 WHERE uuid = $5"
    stmt, err := r.db.PrepareContext(ctx, query)
    if err != nil {
        return err
    }
    defer stmt.Close()

    _, err = stmt.ExecContext(
        ctx,
        m.InstrumentUuid,
		m.Value,
        m.Timestamp,
        m.UserUuid,
        m.Uuid,
    )
    return err
}
{{</ highlight >}}

# Application Layer

We need to update the **controllers** package to support [contexts](https://golang.org/pkg/context/). The following files need to be updated:

1. mulberry-server/internal/controllers/user.go
2. mulberry-server/internal/controllers/tsd.go
3. mulberry-server/cmd/serve/main.go

## Controllers Package

Begin with the **user.go** file:

### user.go

{{< highlight go "linenos=true">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/controllers/user.go
package controllers

import (
    "encoding/json"
    "fmt"
    "net/http"

    "github.com/google/uuid"

    "github.com/bartmika/mulberry-server/pkg/models"
    "github.com/bartmika/mulberry-server/pkg/utils"
)

// To run this API, try running in your console:
// $ http post 127.0.0.1:5000/api/v1/register email="fherbert@dune.com" password="the-spice-must-flow" name="Frank Herbert"
func (h *BaseHandler) postRegister(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

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
    if userFound, _ := h.UserRepo.FindByEmail(ctx, requestData.Email); userFound != nil {
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
    if err := h.UserRepo.Create(ctx, uid, requestData.Name, requestData.Email, passwordHash); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Generate our response.
    responseData := models.RegisterResponse{
        Message: "You have successfully registered an account.",
        Uuid: uid,
    }
    if err := json.NewEncoder(w).Encode(&responseData); err != nil {  // [2]
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// To run this API, try running in your console:
// $ http post 127.0.0.1:5000/api/v1/login email="fherbert@dune.com" password="the-spice-must-flow"
func (h *BaseHandler) postLogin(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

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
    user, err := h.UserRepo.FindByEmail(ctx, requestData.Email)
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
        Uuid: user.Uuid,
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

### tsd.go

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
func (h *BaseHandler) getTimeSeriesData(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    //TODO: Add filtering based on the authenticated user account. For now just list all the records.
    //      In a future article we will update this code.
    results, err := h.TsdRepo.ListAll(ctx)
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
func (h *BaseHandler) postTimeSeriesData(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
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
    err := h.TsdRepo.Create(ctx, uid, requestData.InstrumentUuid, requestData.Value, requestData.Timestamp, requestData.UserUuid)
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
func (h *BaseHandler) getTimeSeriesDatum(w http.ResponseWriter, r *http.Request, uuid string) {
    ctx := r.Context()

    // Lookup our record.
    tsd, err := h.TsdRepo.FindByUuid(ctx, uuid)
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
func (h *BaseHandler) putTimeSeriesDatum(w http.ResponseWriter, r *http.Request, uid string) {
    ctx := r.Context()

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
    tsd := models.TimeSeriesDatum{
        Uuid: uid,
        InstrumentUuid: requestData.InstrumentUuid,
        Value: requestData.Value,
        Timestamp: requestData.Timestamp,
        UserUuid: requestData.UserUuid,
    }
    err := h.TsdRepo.Save(ctx, &tsd)
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
// $ http delete 127.0.0.1:5000/api/v1/time-series-datum/f3e7b442-f3d4-4c2f-8f8d-d347982c1569
func (h *BaseHandler) deleteTimeSeriesDatum(w http.ResponseWriter, r *http.Request, uid string) {
    ctx := r.Context()

    if err := h.TsdRepo.DeleteByUuid(ctx, uid); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
    w.WriteHeader(http.StatusOK) // Note: https://tools.ietf.org/html/rfc7231#section-6.3.1
}
{{</ highlight >}}

## Main Package

And finally with the **main.go** file:

### main.go

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
    // Get our environment variables which will used to configure our application.
    databaseHost := os.Getenv("MULBERRY_DB_HOST")
    databasePort := os.Getenv("MULBERRY_DB_PORT")
    databaseUser := os.Getenv("MULBERRY_DB_USER")
    databasePassword := os.Getenv("MULBERRY_DB_PASSWORD")
    databaseName := os.Getenv("MULBERRY_DB_NAME")

    db, err := sqldb.ConnectDB(databaseHost, databasePort, databaseUser, databasePassword, databaseName)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    userRepo := repositories.NewUserRepo(db)
    tsdRepo := repositories.NewTimeSeriesDatumRepo(db)

    c := controllers.NewBaseHandler(userRepo, tsdRepo)

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

# Testing
## Starting the Server
We are going to use *environment variables* to load configuration settings for our application.

{{< highlight bash "linenos=false">}}
export MULBERRY_DB_HOST=localhost
export MULBERRY_DB_PORT=5432
export MULBERRY_DB_USER=golang
export MULBERRY_DB_PASSWORD=123password
export MULBERRY_DB_NAME=mulberry_db
{{</ highlight >}}

And now you can run the code:

{{< highlight bash "linenos=false">}}
go run cmd/serve/main.go
{{</ highlight >}}

## Making API Calls

Let's begin by making a post. Please note that the *timestamp* must be formatted using **"RFC3339" date/time standard** - For more information please read this [great article](https://archive.is/7cqtR).

### Register API

Let's attempt to *register*. Start the server and run the following commands in your console:

{{< highlight bash "linenos=false">}}
$ http post 127.0.0.1:5000/api/v1/register email="fherbert@dune.com" \
                                        password="the-spice-must-flow" \
                                            name="Frank Herbert"
{{</ highlight >}}

The output should be as follows:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 105
Content-Type: application/json
Date: Sun, 31 Jan 2021 05:56:09 GMT
{
    "message": "You have successfully registered an account.",
    "uuid": "9dd2cc0a-934d-4788-8304-1e0b82d9b6e6"
}
{{</ highlight >}}

### Login API

And for our grand finally, run the login command which works:

{{< highlight bash "linenos=false">}}
$ http post 127.0.0.1:5000/api/v1/login email="fherbert@dune.com" \
                                     password="the-spice-must-flow"
{{</ highlight >}}

Wonderful! The success output will be as follows:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 125
Content-Type: application/json
Date: Sun, 31 Jan 2021 05:56:52 GMT
{
    "access_token": "TODO: WE WILL FIGURE OUT HOW TO DO THIS IN ANOTHER ARTICLE!",
    "uuid": "9dd2cc0a-934d-4788-8304-1e0b82d9b6e6"
}
{{</ highlight >}}


### Create API

Run the following in your console:

{{< highlight go "linenos=false">}}
$ http post 127.0.0.1:5000/api/v1/time-series-data instrument_uuid="lalala" \
                                                             value="123" \
                                                         timestamp="2021-01-30T10:20:10.000Z" \
                                                         user_uuid="9dd2cc0a-934d-4788-8304-1e0b82d9b6e6"
{{</ highlight >}}

And you should get the following message:

{{< highlight go "linenos=false">}}
HTTP/1.1 201 Created
Content-Length: 175
Content-Type: application/json
Date: Sun, 31 Jan 2021 05:58:06 GMT

{
    "instrument_uuid": "lalala",
    "timestamp": "2021-01-30T10:20:10Z",
    "user_uuid": "9dd2cc0a-934d-4788-8304-1e0b82d9b6e6",
    "uuid": "a8f355f4-3bb4-4741-b98b-a0dbcc7a34ff",
    "value": "123"
}
{{</ highlight >}}

### List API

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
        "user_uuid": "9dd2cc0a-934d-4788-8304-1e0b82d9b6e6",
        "uuid": "a8f355f4-3bb4-4741-b98b-a0dbcc7a34ff",
        "value": 123
    }
]
{{</ highlight >}}

### Retrieve API

Run the following in your console:

{{< highlight go "linenos=false">}}
$ http get 127.0.0.1:5000/api/v1/time-series-datum/a8f355f4-3bb4-4741-b98b-a0dbcc7a34ff
{{</ highlight >}}

And you should get the following message:

{{< highlight go "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 173
Content-Type: application/json
Date: Sun, 31 Jan 2021 06:00:47 GMT

{
    "instrument_uuid": "lalala",
    "timestamp": "2021-01-30T10:20:10Z",
    "user_uuid": "9dd2cc0a-934d-4788-8304-1e0b82d9b6e6",
    "uuid": "a8f355f4-3bb4-4741-b98b-a0dbcc7a34ff",
    "value": 123
}
{{</ highlight >}}

### Update API

Run the following in your console:

{{< highlight go "linenos=false">}}
$ http put 127.0.0.1:5000/api/v1/time-series-datum/a8f355f4-3bb4-4741-b98b-a0dbcc7a34ff \
instrument_uuid="lalala" \
          value="321" \
      timestamp="2021-01-30T10:20:10.000Z" \
      user_uuid="9dd2cc0a-934d-4788-8304-1e0b82d9b6e6"
{{</ highlight >}}

And you should get the following message:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 173
Content-Type: application/json
Date: Sun, 31 Jan 2021 06:01:14 GMT
{
    "instrument_uuid": "lalala",
    "timestamp": "2021-01-30T10:20:10Z",
    "user_uuid": "9dd2cc0a-934d-4788-8304-1e0b82d9b6e6",
    "uuid": "a8f355f4-3bb4-4741-b98b-a0dbcc7a34ff",
    "value": 321
}
{{</ highlight >}}

### Delete API

Run the following in your console:

{{< highlight bash "linenos=false">}}
$ http delete 127.0.0.1:5000/api/v1/time-series-datum/a8f355f4-3bb4-4741-b98b-a0dbcc7a34ff
{{</ highlight >}}

And you should get the following message:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 0
Content-Type: application/json
Date: Sun, 31 Jan 2021 06:01:54 GMT
{{</ highlight >}}

# What's next?

That's it for this article. Hope you enjoyed it. What's next?

* How do we handle sessions? Access tokens?
* How do we handle background processes?
* How do we handle pagination in our list API endpoint?
