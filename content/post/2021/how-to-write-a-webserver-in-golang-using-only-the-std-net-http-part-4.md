---
title: "How to write a webserver in Golang using only the std net/http - Part 4"
subtitle: "Learn how to implement the database layer for your webserver"
date: 2021-01-31T00:02:30-04:00
draft: false
categories:
- "development"
tags:
- "golang"
- "api"
---

<!-- ![](https://images.pexels.com/photos/5582597/pexels-photo-5582597.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940) -->

The purpose of this article is to provide instructions on how utilize sessions and authentication.

<!--more-->

# Authentication & Authorization

Just as a reminder this is part 4 of the series, you'll need to finish [part 3](/posts/2021/how-to-write-a-webserver-in-golang-using-only-the-std-net-http-part-3/) before continuing.

### jwt.go

```go
// github.com/bartmika/mulberry-server/pkg/utils/jwt.go
package utils

import (
    "time"

    jwt "github.com/dgrijalva/jwt-go"
)

func GenerateJWT(hmacSecret []byte, sessionUuid string, clientUuid string) (string, error) {
    token := jwt.New(jwt.SigningMethodHS256)

    claims := token.Claims.(jwt.MapClaims)

    claims["authorized"] = true
    claims["session_uuid"] = sessionUuid
    claims["client_uuid"] = clientUuid
    claims["exp"] = time.Now().Add(time.Minute * 30).Unix()

    tokenString, err := token.SignedString(hmacSecret)

    if err != nil {
        return "", err
    }
    return tokenString, nil
}

func ProcessJWT(hmacSecret []byte, reqToken string) (map[string]string, error){
    token, err := jwt.Parse(reqToken, func(t *jwt.Token) (interface{}, error) {
        return hmacSecret, nil
    })
    if err == nil && token.Valid {
        if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
            m := make(map[string]string)
            m["session_uuid"] = claims["session_uuid"].(string)
            m["client_uuid"] = claims["client_uuid"].(string)
            // m["exp"] = claims["exp"]
            return m, nil
        } else {
            return nil, err
        }

    } else {
        return nil, err
    }
    return nil, nil
}
```

### middleware.go

```go
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

            m, err := utils.ProcessJWT(mySigningKey, reqToken)
            if err == nil {
                ctx = context.WithValue(ctx, "is_authorized", true)
                ctx = context.WithValue(ctx, "session_uuid", m["session_uuid"])
                ctx = context.WithValue(ctx, "client_uuid", m["client_uuid"])

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

func Middleware(fn http.HandlerFunc) http.HandlerFunc {
    // Attach our middleware
    fn = URLProcessorMiddleware(fn)
    fn = JWTProcessorMiddleware(fn)
    return func(w http.ResponseWriter, r *http.Request) {
        // Flow to the next middleware.
        fn(w, r)
    }
}
```

### main.go

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
    router.HandleFunc("/", controllers.Middleware(c.HandleRequests))

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

### controller.go

```go
// github.com/bartmika/mulberry-server/internal/controllers/controller.go
package controllers

import (
    "net/http"

    "github.com/bartmika/mulberry-server/internal/repositories"
)

type BaseHandler struct {
    UserRepo *repositories.UserRepo
    TsdRepo *repositories.TimeSeriesDatumRepo
}

func NewBaseHandler(u *repositories.UserRepo, tsd *repositories.TimeSeriesDatumRepo) (*BaseHandler) {
    return &BaseHandler{
        UserRepo: u,
        TsdRepo: tsd,
    }
}

func (h *BaseHandler) HandleRequests(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    // Get our URL paths which are slash-seperated.
    ctx := r.Context()
    p := ctx.Value("url_split").([]string)
    n := len(p)

    // Get our authorization information.
    isAuthorized := ctx.Value("is_authorized").(bool)

    switch {
    case n == 1 && p[0] == "version" && r.Method == http.MethodGet:
        h.getVersion(w, r)
    case n == 1 && p[0] == "login" && r.Method == http.MethodPost:
        h.postLogin(w, r)
    case n == 1 && p[0] == "register" && r.Method == http.MethodPost:
        h.postRegister(w, r)
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodGet:
        if isAuthorized {
            h.getTimeSeriesData(w, r)
        } else {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
        }
    case n == 1 && p[0] == "time-series-data" && r.Method == http.MethodPost:
        if isAuthorized {
            h.postTimeSeriesData(w, r)
        } else {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
        }
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodGet:
        if isAuthorized {
            h.getTimeSeriesDatum(w, r, p[1])
        } else {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
        }
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodPut:
        if isAuthorized {
            h.putTimeSeriesDatum(w, r, p[1])
        } else {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
        }
    case n == 2 && p[0] == "time-series-datum" && r.Method == http.MethodDelete:
        if isAuthorized {
            h.deleteTimeSeriesDatum(w, r, p[1])
        } else {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
        }
    default:
        http.NotFound(w, r)
    }
}
```

### user.go

```go
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

    // Generate a `UUID` for our session.
    uid := uuid.New().String()

    // Generate our JWT token.
    mySigningKey := ctx.Value("jwt_signing_key").([]byte)
    accessToken, err := utils.GenerateJWT(mySigningKey, uid, user.Uuid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Finally return success.
    responseData := models.LoginResponse{
        AccessToken: accessToken,
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

### tsd.go

```go
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
func (h *BaseHandler) getTimeSeriesData(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    userUuid := ctx.Value("client_uuid").(string)

    results, err := h.TsdRepo.FilterByUserUuid(ctx, userUuid)
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
    userUuid := ctx.Value("client_uuid").(string)

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
    err := h.TsdRepo.Create(ctx, uid, requestData.InstrumentUuid, requestData.Value, requestData.Timestamp, userUuid)
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
func (h *BaseHandler) getTimeSeriesDatum(w http.ResponseWriter, r *http.Request, uuid string) {
    ctx := r.Context()
    userUuid := ctx.Value("client_uuid").(string)

    // Lookup our record.
    tsd, err := h.TsdRepo.FindByUuid(ctx, uuid)
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
func (h *BaseHandler) putTimeSeriesDatum(w http.ResponseWriter, r *http.Request, uid string) {
    ctx := r.Context()
    userUuid := ctx.Value("client_uuid").(string)

    // Lookup our record and enforce account access.
    tsd, err := h.TsdRepo.FindByUuid(ctx, uid)
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
    err = h.TsdRepo.Save(ctx, tsd)
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
    userUuid := ctx.Value("client_uuid").(string)

    // Lookup our record and enforce account access.
    tsd, err := h.TsdRepo.FindByUuid(ctx, uid)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    if tsd.UserUuid != userUuid {
        http.Error(w, "Forbidden", http.StatusForbidden)
        return
    }

    if err := h.TsdRepo.DeleteByUuid(ctx, uid); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
    w.WriteHeader(http.StatusOK) // Note: https://tools.ietf.org/html/rfc7231#section-6.3.1
}
```

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
Content-Length: 292
Content-Type: application/json
Date: Sun, 31 Jan 2021 22:34:28 GMT

{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRob3JpemVkIjp0cnVlLCJjbGllbnRfdXVpZCI6IjlkZDJjYzBhLTkzNGQtNDc4OC04MzA0LTFlMGI4MmQ5YjZlNiIsImV4cCI6MTYxMjEzNDI2OCwic2Vzc2lvbl91dWlkIjoiOTk3ZDQ3NWUtNGMyZi00YWViLThlMjEtOWNlYWM3MTg3NDhjIn0.fBP01baxjC9EwO3vzeHpeRKppOkDlVYmTniZUEPXfAw"
}
{{</ highlight >}}

Please save the output the "access_token" so you can write in your console.

{{< highlight bash "linenos=false">}}
export MULBERRY_API_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRob3JpemVkIjp0cnVlLCJjbGllbnRfdXVpZCI6IjlkZDJjYzBhLTkzNGQtNDc4OC04MzA0LTFlMGI4MmQ5YjZlNiIsImV4cCI6MTYxMjEzNDI2OCwic2Vzc2lvbl91dWlkIjoiOTk3ZDQ3NWUtNGMyZi00YWViLThlMjEtOWNlYWM3MTg3NDhjIn0.fBP01baxjC9EwO3vzeHpeRKppOkDlVYmTniZUEPXfAw"
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
