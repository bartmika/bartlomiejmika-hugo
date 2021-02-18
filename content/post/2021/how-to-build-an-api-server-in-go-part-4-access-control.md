---
title: "How to Build an API Server in Go - Part 4: Access Control"
date: 2021-01-31T00:02:30-04:00
draft: false
categories:
- "Web Development"
tags:
- "Golang"
- "API"
- "Security"
- "HOWTO"
---

<!-- ![](https://images.pexels.com/photos/5582597/pexels-photo-5582597.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940) -->

Learn how to protect API endpoints with access and refresh tokens using the third-party [jwt-go](https://github.com/dgrijalva/jwt-go) library.

<!--more-->

This post belongs to the following series:

1. [How to Build an API Server in Go - Part 1: Basic Server](/post/2021/how-to-build-an-api-server-in-go-part-1-basic-server/)
2. [How to Build an API Server in Go - Part 2: Simple Database](/post/2021/how-to-build-an-api-server-in-go-part-2-simple-database/)
3. [How to Build an API Server in Go - Part 3: Postgres Database](/post/2021/how-to-build-an-api-server-in-go-part-3-postgres-database/)
4. **How to Build an API Server in Go - Part 4: Access Control**

# Authentication

Just as a reminder this is part 4 of the series, you'll need to finish [part 3](/posts/2021/how-to-write-a-webserver-in-golang-using-only-the-std-net-http-part-3/) before continuing.

## Structure


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
|   │   📄controller.go
|   │   📄middleware.go
|   │   📄tsd.go
|   │   📄user.go
|   │   📄version.go
|   |
|   └───📁repositories
|       📄tsd.go
|       📄user.go
|
└───📁pkg
    |
    └───📁db
    |   📄db.go
    |
    └───📁models
    |   📄tsd.go
    |   📄user.go
    |
    └───📁utils
        📄jwt.go
        📄password.go
{{</ highlight >}}

## models package

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
}

// The struct used to represent the user's `login` POST request data.
type LoginRequest struct {
	Email string    `json:"email"`
	Password string `json:"password"`
}

// The struct used to represent the system's response when the `login` POST request was a success.
type LoginResponse struct {
	AccessToken string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

// The struct used to represent the user's `refresh token` POST request data.
type RefreshTokenRequest struct {
	Value string     `json:"value"`
}

// The struct used to represent the system's response when the `refresh token` POST request was a success.
type RefreshTokenResponse struct {
	AccessToken string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}
{{</ highlight >}}

## utils package

### jwt.go

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/pkg/utils/jwt.go
package utils

import (
    "time"

    jwt "github.com/dgrijalva/jwt-go"
)

// Generate the `access token` and `refresh token` for the secret key.
func GenerateJWTTokenPair(hmacSecret []byte, clientUuid string) (string, string, error) {
    //
    // Generate token.
    //
    token := jwt.New(jwt.SigningMethodHS256)
    claims := token.Claims.(jwt.MapClaims)
    claims["user_uuid"] = clientUuid
    claims["exp"] = time.Now().Add(time.Hour * 1).Unix()

    tokenString, err := token.SignedString(hmacSecret)
    if err != nil {
        return "", "", err
    }

    //
    // Generate refresh token.
    //
    refreshToken := jwt.New(jwt.SigningMethodHS256)
	rtClaims := refreshToken.Claims.(jwt.MapClaims)
	rtClaims["user_uuid"] = clientUuid
	rtClaims["exp"] = time.Now().Add(time.Hour * 72).Unix()

	refreshTokenString, err := refreshToken.SignedString(hmacSecret)
	if err != nil {
		return "", "", err
	}

    return tokenString, refreshTokenString, nil
}

// Validates either the `access token` or `refresh token` and returns either the
// `user_uuid` if success or error on failure.
func ProcessJWTToken(hmacSecret []byte, reqToken string) (string, error){
    token, err := jwt.Parse(reqToken, func(t *jwt.Token) (interface{}, error) {
        return hmacSecret, nil
    })
    if err == nil && token.Valid {
        if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
            user_uuid := claims["user_uuid"].(string)
            // m["exp"] := string(claims["exp"].(float64))
            return user_uuid, nil
        } else {
            return "", err
        }

    } else {
        return "", err
    }
    return "", nil
}
{{</ highlight >}}

## controllers package
### middleware.go

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/controllers/middleware.go
package controllers

import (
    "os"
    "context"
    "strings"
    "log"
    "net/http"

    "github.com/bartmika/mulberry-server/pkg/utils"
)

// Middleware will split the full URL path into slash-sperated parts and save to
// the context to flow downstream in the app for this particular request.
func URLProcessorMiddleware(fn http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Split path into slash-separated parts, for example, path "/foo/bar"
        // gives p==["foo", "bar"] and path "/" gives p==[""]. Our API starts with
        // "/api/v1", as a result we will start the array slice at "3".
        p := strings.Split(r.URL.Path, "/")[3:]

        // log.Println(p) // For debugging purposes only.

        // Open our program's context based on the request and save the
        // slash-seperated array from our URL path.
        ctx := r.Context()
        ctx = context.WithValue(ctx, "url_split", p)

        // Flow to the next middleware.
        fn(w, r.WithContext(ctx))
    }
}

func JWTProcessorMiddleware(fn http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()

        // Read our application's signing key and attach it to the application
        // context so it can flow downstream in all our applications.
        mySigningKey := []byte(os.Getenv("MULBERRY_APP_SIGNING_KEY"))
        ctx = context.WithValue(ctx, "jwt_signing_key", mySigningKey)

        reqToken := r.Header.Get("Authorization")

        if reqToken != "" {
            // Special thanks to "poise" via https://stackoverflow.com/a/44700761
            splitToken := strings.Split(reqToken, "Bearer ")
            reqToken = splitToken[1]

            // log.Println(reqToken) // For debugging purposes only.

            user_uuid, err := utils.ProcessJWTToken(mySigningKey, reqToken)
            if err == nil {
                ctx = context.WithValue(ctx, "is_authorized", true)
                ctx = context.WithValue(ctx, "user_uuid", user_uuid)

                // Flow to the next middleware with our JWT token saved.
                fn(w, r.WithContext(ctx))
                return
            }
            log.Println("JWTProcessorMiddleware | ProcessJWT | err", err)
        }

        // Flow to the next middleware without anything done.
        ctx = context.WithValue(ctx, "is_authorized", false)
        fn(w, r.WithContext(ctx))
    }
}

func ChainMiddleware(fn http.HandlerFunc) http.HandlerFunc {
    // Attach our middleware
    fn = URLProcessorMiddleware(fn)
    fn = JWTProcessorMiddleware(fn)
    return func(w http.ResponseWriter, r *http.Request) {
        // Flow to the next middleware.
        fn(w, r)
    }
}
{{</ highlight >}}

### controller.go

{{< highlight go "linenos=true">}}
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"

    "github.com/bartmika/mulberry-server/internal/repositories"
)

type Controller struct {
    UserRepo *repositories.UserRepo
    TsdRepo *repositories.TimeSeriesDatumRepo
}

func New(u *repositories.UserRepo, tsd *repositories.TimeSeriesDatumRepo) (*Controller) {
    return &Controller{
        UserRepo: u,
        TsdRepo: tsd,
    }
}

func (c *Controller) HandleRequests(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    // Get our URL paths which are slash-seperated.
    ctx := r.Context()
    p := ctx.Value("url_split").([]string)
    n := len(p)

    // Get our authorization information.
    isAuthorized := ctx.Value("is_authorized").(bool)

    switch {
    case n == 1 && p[0] == "version" && r.Method == http.MethodGet:
        c.getVersion(w, r)
    case n == 1 && p[0] == "login" && r.Method == http.MethodPost:
        c.postLogin(w, r)
    case n == 1 && p[0] == "register" && r.Method == http.MethodPost:
        c.postRegister(w, r)
    case n == 1 && p[0] == "refresh-token" && r.Method == http.MethodPost:
        c.postRefreshToken(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodGet:
        if isAuthorized {
            c.getTimeSeriesData(w, r)
        } else {
            http.Error(w, "Unauthorized - access token expired or invalid", http.StatusUnauthorized)
        }
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodPost:
        if isAuthorized {
            c.postTimeSeriesData(w, r)
        } else {
            http.Error(w, "Unauthorized - access token expired or invalid", http.StatusUnauthorized)
        }
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodGet:
        if isAuthorized {
            c.getTimeSeriesDatum(w, r, p[1])
        } else {
            http.Error(w, "Unauthorized - access token expired or invalid", http.StatusUnauthorized)
        }
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodPut:
        if isAuthorized {
            c.putTimeSeriesDatum(w, r, p[1])
        } else {
            http.Error(w, "Unauthorized - access token expired or invalid", http.StatusUnauthorized)
        }
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodDelete:
        if isAuthorized {
            c.deleteTimeSeriesDatum(w, r, p[1])
        } else {
            http.Error(w, "Unauthorized - access token expired or invalid", http.StatusUnauthorized)
        }
    default:
        http.NotFound(w, r)
    }
}
{{</ highlight >}}

### user.go

{{< highlight go "linenos=true">}}
// FILE LOCATION: github.com/bartmika/mulberry-server/internal/controllers/user.go
package controllers

import (
    "encoding/json"
    "log"
    "net/http"

    "github.com/google/uuid"

    "github.com/bartmika/mulberry-server/pkg/models"
    "github.com/bartmika/mulberry-server/pkg/utils"
)

// To run this API, try running in your console:
// $ http post 127.0.0.1:5000/api/v1/register email="fherbert@dune.com" password="the-spice-must-flow" name="Frank Herbert"
func (c *Controller) postRegister(w http.ResponseWriter, r *http.Request) {
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

    // // For debugging purposes, print our output so you can see the code working.
    // fmt.Println(requestData.Name)
    // fmt.Println(requestData.Email)
    // fmt.Println(requestData.Password)

    // Lookup the email and if it is not unique we need to generate a `400 Bad Request` response.
    if userFound, _ := c.UserRepo.FindByEmail(ctx, requestData.Email); userFound != nil {
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
    if err := c.UserRepo.Create(ctx, uid, requestData.Name, requestData.Email, passwordHash); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

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
    ctx := r.Context()

    var requestData models.LoginRequest

    err := json.NewDecoder(r.Body).Decode(&requestData)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // // For debugging purposes, print our output so you can see the code working.
    // fmt.Println(requestData.Email)
    // fmt.Println(requestData.Password)

    // Lookup the user in our database, else return a `400 Bad Request` error.
    user, err := c.UserRepo.FindByEmail(ctx, requestData.Email)
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

    // Generate our JWT token.
    mySigningKey := ctx.Value("jwt_signing_key").([]byte)
    accessToken, refreshToken, err := utils.GenerateJWTTokenPair(mySigningKey, user.Uuid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Finally return success.
    responseData := models.LoginResponse{
        AccessToken: accessToken,
        RefreshToken: refreshToken,
    }
    if err := json.NewEncoder(w).Encode(&responseData); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// To run this API, try running in your console:
// $ http post 127.0.0.1:5000/api/v1/refresh-token value="xxx"
func (c *Controller) postRefreshToken(w http.ResponseWriter, r *http.Request) {
    var requestData models.RefreshTokenRequest

    err := json.NewDecoder(r.Body).Decode(&requestData)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // // For debugging purposes, print our output so you can see the code working.
    log.Println(requestData.Value)

    ctx := r.Context()
    mySigningKey := ctx.Value("jwt_signing_key").([]byte)

    // Verify our refresh token.
    user_uuid, err := utils.ProcessJWTToken(mySigningKey, requestData.Value)
    if err != nil {
        http.Error(w, "Unauthorized - refresh token expired or invalid", http.StatusUnauthorized)
        return
    }

    // Generate our JWT token.
    accessToken, refreshToken, err := utils.GenerateJWTTokenPair(mySigningKey, user_uuid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Finally return success.
    responseData := models.RefreshTokenResponse{
        AccessToken: accessToken,
        RefreshToken: refreshToken,
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
    // "fmt"
    "net/http"
    "encoding/json"

    "github.com/google/uuid"

    "github.com/bartmika/mulberry-server/pkg/models"
)

// To run this API, try running in your console:
// $ http get 127.0.0.1:5000/api/v1/time-series-data
func (c *Controller) getTimeSeriesData(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    userUuid := ctx.Value("user_uuid").(string)

    results, err := c.TsdRepo.FilterByUserUuid(ctx, userUuid)
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
    ctx := r.Context()
    userUuid := ctx.Value("user_uuid").(string)

    var requestData models.TimeSeriesDatumCreateRequest
    if err := json.NewDecoder(r.Body).Decode(&requestData); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // // For debugging purposes only.
    // fmt.Println(requestData.InstrumentUuid)
    // fmt.Println(requestData.Value)
    // fmt.Println(requestData.Timestamp)
    // fmt.Println(requestData.UserUuid)

    // Generate a `UUID` for our record.
    uid := uuid.New().String()

    // Save to our database.
    err := c.TsdRepo.Create(ctx, uid, requestData.InstrumentUuid, requestData.Value, requestData.Timestamp, userUuid)
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
        UserUuid: userUuid,
    }
    if err := json.NewEncoder(w).Encode(&responseData); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
}

// To run this API, try running in your console:
// $ http get 127.0.0.1:5000/api/v1/time-series-datum/f3e7b442-f3d4-4c2f-8f8d-d347982c1569
func (c *Controller) getTimeSeriesDatum(w http.ResponseWriter, r *http.Request, uuid string) {
    ctx := r.Context()
    userUuid := ctx.Value("user_uuid").(string)

    // Lookup our record.
    tsd, err := c.TsdRepo.FindByUuid(ctx, uuid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Enforce account access.
    if tsd.UserUuid != userUuid {
        http.Error(w, "Forbidden", http.StatusForbidden)
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
    ctx := r.Context()
    userUuid := ctx.Value("user_uuid").(string)

    // Lookup our record and enforce account access.
    tsd, err := c.TsdRepo.FindByUuid(ctx, uid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    if tsd.UserUuid != userUuid {
        http.Error(w, "Forbidden", http.StatusForbidden)
        return
    }

    var requestData models.TimeSeriesDatumPutRequest
    if err := json.NewDecoder(r.Body).Decode(&requestData); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // // For debugging purposes only.
    // fmt.Println(uid)
    // fmt.Println(requestData.InstrumentUuid)
    // fmt.Println(requestData.Value)
    // fmt.Println(requestData.Timestamp)
    // fmt.Println(requestData.UserUuid)

    // Update our record.
    tsd = &models.TimeSeriesDatum{
        Uuid: uid,
        InstrumentUuid: requestData.InstrumentUuid,
        Value: requestData.Value,
        Timestamp: requestData.Timestamp,
        UserUuid: userUuid,
    }
    err = c.TsdRepo.Save(ctx, tsd)
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
func (c *Controller) deleteTimeSeriesDatum(w http.ResponseWriter, r *http.Request, uid string) {
    ctx := r.Context()
    userUuid := ctx.Value("user_uuid").(string)

    // Lookup our record and enforce account access.
    tsd, err := c.TsdRepo.FindByUuid(ctx, uid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    if tsd.UserUuid != userUuid {
        http.Error(w, "Forbidden", http.StatusForbidden)
        return
    }

    if err := c.TsdRepo.DeleteByUuid(ctx, uid); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
    w.WriteHeader(http.StatusOK) // Note: https://tools.ietf.org/html/rfc7231#section-6.3.1
}
{{</ highlight >}}

## serve package

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

    c := controllers.New(userRepo, tsdRepo)

    mux := http.NewServeMux()
    mux.HandleFunc("/", controllers.ChainMiddleware(c.HandleRequests))

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

# Testing
## Starting the Server
We are going to use *environment variables* to load configuration settings for our application.

{{< highlight bash "linenos=false">}}
export MULBERRY_DB_HOST=localhost
export MULBERRY_DB_PORT=5432
export MULBERRY_DB_USER=golang
export MULBERRY_DB_PASSWORD=123password
export MULBERRY_DB_NAME=mulberry_db
export MULBERRY_APP_SIGNING_KEY=PLEASE_REPLACE_ME
{{</ highlight >}}

And now you can run the code:

{{< highlight bash "linenos=false">}}
go run cmd/serve/main.go
{{</ highlight >}}


In another terminal run the following code to make a login call:

{{< highlight bash "linenos=false">}}
bmika@MACMINI-AFA2131 mulberry-server % http post 127.0.0.1:5000/api/v1/login email="fherbert@dune.com" password="the-spice-must-flow"
{{</ highlight >}}

The return should look as follows:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 385
Content-Type: application/json
Date: Mon, 01 Feb 2021 04:27:46 GMT
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MTIxNTcyNjYsInVzZXJfdXVpZCI6IjlkZDJjYzBhLTkzNGQtNDc4OC04MzA0LTFlMGI4MmQ5YjZlNiJ9.ZKe5DargCrHZcAQQ71M46uUr0TWk9UYkiURijaKBABA",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MTI0MTI4NjYsInVzZXJfdXVpZCI6IjlkZDJjYzBhLTkzNGQtNDc4OC04MzA0LTFlMGI4MmQ5YjZlNiJ9.odXNQd1hm3cLPI9_e2jfjYXjgjf7wfxQ8hCx3-qqZYY"
}
{{</ highlight >}}

Please save the output the "access_token" so you can write in your console.

{{< highlight bash "linenos=false">}}
export MULBERRY_API_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MTIxNTcyNjYsInVzZXJfdXVpZCI6IjlkZDJjYzBhLTkzNGQtNDc4OC04MzA0LTFlMGI4MmQ5YjZlNiJ9.ZKe5DargCrHZcAQQ71M46uUr0TWk9UYkiURijaKBABA"
{{</ highlight >}}

Next run the following call to a protected API:

{{< highlight bash "linenos=false">}}
http get 127.0.0.1:5000/api/v1/time-series-data Authorization:"Bearer $MULBERRY_API_TOKEN"
{{</ highlight >}}

And your result should look like this:

{{< highlight bash "linenos=false">}}
HTTP/1.1 200 OK
Content-Length: 175
Content-Type: application/json
Date: Sun, 31 Jan 2021 22:57:40 GMT
[
    {
        "instrument_uuid": "lalala",
        "timestamp": "2021-01-30T10:20:10Z",
        "user_uuid": "9dd2cc0a-934d-4788-8304-1e0b82d9b6e6",
        "uuid": "923148e4-5dd9-4ec7-bf04-4a50f880c7db",
        "value": 123
    }
]
{{</ highlight >}}
