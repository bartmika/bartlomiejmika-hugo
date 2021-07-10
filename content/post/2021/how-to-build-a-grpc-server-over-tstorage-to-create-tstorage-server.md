---
title: "How to Build a gRPC Server over tstorage to create tstorage-server"
date: 2021-07-09T23:45:08-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Open Source Software"
tags:
- "gRPC"
- "Golang"
---

![](/img/2021/07-09/harrison-broadbent-nePxBIvqUlU-unsplash.jpg)

Did you just finish reading the [**gRPC Basics tutorial**](https://grpc.io/docs/languages/go/basics/) and you don't know where to begin with using it? In this post I'll explain how to take a fast time-series database called [`tstorage`](https://github.com/nakabonne/tstorage) and write a gRPC server and client with it.

<!--more-->

Recently I read an article from [HackerNews](https://news.ycombinator.com/item?id=27730854) titled [Write a time-series database engine from scratch](https://nakabonne.dev/posts/write-tsdb-from-scratch/) where the author, [Ryo Nakao](https://nakabonne.dev/about/), details the lessons he learned from implementing a fast **time-series database** of his own. In the article he mentions that he open sourced it as [`nakabonne/tstorage`](https://github.com/nakabonne/tstorage).

Investigating the **repository**, it looks well thought out and pretty easy to use; as a result, I thought about using it for personal projects. The challenge with the package is *interprocess communication* (IPC) as I want to have multiple small applications communicate with `tstorage`.

To get around the *IPC* issue, I decide to write a consumable **gRPC service definition** overtop the package so other local or remote applications can use it. I am calling this application [`tstorage-server`](https://github.com/bartmika/tstorage-server).

The purpose of this article is describe how to create a **list**, **create** and **remove** endpoints overtop `tstorage`. The benefit is if you want to create more [`gRPC`](https://grpc.io) servers in the future, you can reference this article.

This article is broken down into the following sections:

* Part 1. Setup the Project
* Part 2. Setup Simple Protocol Buffers
* Part 3. Setup Simple gRPC Server
* Part 4. Setup Simple gRPC Client
* Part 5. Setup `tstorage` library with our Project |
* Part 6. Server Endpoints
* Part 7. Client Subcommands
* Part 8. Usage examples

## Assumptions {#2}
Please note the following:

* Your developer machine is either MacOS or Linux
* You have `golang 1.16` installed or greater
* You are proficient with `git` and the `terminal`.
* You have basic understanding of the Golang language.
* You have a intermediate understanding of `gRPC`, and if not then please read my previous article title ["Example of Writing a Simple gRPC Server in Golang from Scratch"](/post/2021/example-of-writing-a-simple-grpc-server-in-golang-from-scratch/) which will get you prepped for this article.

## Part 1. Setup the Project {#3}
*Before writing any business logic, structure the project so it can grow in a logic and predictable manner.*

Start off by setting up the an empty **repository** in [github](https://github.com/bartmika/tstorage-server) and run the following in your `terminal`:

{{< highlight bash "linenos=false">}}
cd ~/go/src/github.com/bartmika
git clone https://github.com/bartmika/tstorage-server.git
cd tstorage-server
{{</ highlight >}}

Initialize golang modules.

{{< highlight bash "linenos=false">}}
go mod init github.com/bartmika/tstorage-server
{{</ highlight >}}

Install our project's dependencies.

{{< highlight bash "linenos=false">}}
export GO111MODULE=on  # Enable module mode
go get google.golang.org/protobuf/cmd/protoc-gen-go
go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
go get google.golang.org/grpc
go get github.com/golang/protobuf/ptypes/timestamp
go get github.com/nakabonne/tstorage
go get github.com/spf13/cobra
{{</ highlight >}}

Setup our initial project structure. We will use this structure to grow our application. Please look at the folder and file names in the "Project Hierarchy" and create them in your computer as blank files.

#### Project Hierarchy
{{< highlight bash "linenos=false">}}
📁 tstorage-server
│   📄 main.go
│
└───📁 cmd
    |
    └───📄 root.go
        📄 version.go

{{</ highlight >}}

What are going to do? We want our project structured in such a way that it uses the [`spf13/cobra`](https://github.com/spf13/cobra) package so we can easily commands in our application.

To begin, please copy and paste the following code into those empty files.

#### (1 of 3) main.go
{{< highlight go "linenos=false">}}
package main // github.com/bartmika/tstorage-server/main.go

import (
    "github.com/bartmika/tstorage-server/cmd"
)

func main() {
    cmd.Execute()
}
{{</ highlight >}}

#### (2 of 3) cmd/root.go
{{< highlight go "linenos=false">}}
package cmd // github.com/bartmika/tstorage-server/cmd/root.go

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
	Use:   "tstorage-server",
	Short: "Time-series data storage gRPC server",
	Long: `The purpose of this application is to provide a local
	       database tailored for fast time-series data storage and be
		   accessible with remote procedure calls (gRPC).`,
	Run: func(cmd *cobra.Command, args []string) {
		// Do nothing...
	},
}

func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
{{</ highlight >}}

#### (3 of 3) cmd/version.go
{{< highlight go "linenos=false">}}
package cmd // github.com/bartmika/tstorage-server/cmd/version.go

import (
	"fmt"

	"github.com/spf13/cobra"
)

func init() {
	rootCmd.AddCommand(versionCmd)
}

var versionCmd = &cobra.Command{
	Use:   "version",
	Short: "Print the version number",
	Long:  `Print the current version that this server is on.`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("tstorage-server v1.0")
	},
}
{{</ highlight >}}

Once you finished, in your `terminal` please write the following.

{{< highlight bash "linenos=false">}}
go run main.go version
{{</ highlight >}}

If you setup everything correctly, there should be no errors. Good job!

## Part 2. Setup Simple Protocol Buffers {#4}

Now that we have our **application** structured to use [`spf13/cobra`](https://github.com/spf13/cobra) as our project's scaffolding, we can start by creating our code. We will begin with **protocol buffers**, we need to define a service in `.proto` file.

To begin, create the following folders and files in the your project (notice the asterisks for the new files):

#### Project Hierarchy

{{< highlight bash "linenos=false">}}
📁 tstorage-server
│   📄 README.md
│   📄 main.go
│   📄 Makefile
│
└───📁 cmd
│   |
|   └───📄 root.go
|       📄 version.go
|
└───📁 proto (*)
     │
     └──📄 tstorage.proto (*)

{{</ highlight >}}

Please copy and paste the content of this file.

#### (1 of 1) proto/tstorage.proto
{{< highlight proto "linenos=false">}}
syntax = "proto3"; // github.com/bartmika/tstorage-server/proto/tstorage.proto

option go_package = "github.com/bartmika/tstorage-server";

package proto;

// The greeting service definition.
service TStorage {
    // Sends a greeting
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
    string name = 1;
}

// The response message containing the greetings
message HelloReply {
    string message = 1;
}
{{</ highlight >}}

In your **terminal**, make sure we export our path (if you haven’t done this before) by writing the following:

{{< highlight bash "linenos=false">}}
export PATH="$PATH:$(go env GOPATH)/bin"
{{</ highlight >}}

Run the following to generate our new **gRPC interface**.

{{< highlight bash "linenos=false">}}
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative proto/tstorage.proto
{{</ highlight >}}

After running the **command** your structure should look as follows (notice the asterisks). Please look through the new files to get a feel for the code.

#### Project Hierarchy

{{< highlight bash "linenos=false">}}
📁 tstorage-server
│   📄 README.md
│   📄 main.go
│   📄 Makefile
│
└───📁 cmd
│   |
|   └───📄 root.go
|       📄 version.go
|
└───📁 proto
     │
     └──📄 tstorage_grpc.pb.go (*)
        📄 tstorage.proto
        📄 tstorage.pb.go (*)

{{</ highlight >}}

What does this code do?

* **tstorage.pb.go**, which contains all the protocol buffer code to populate, serialize, and retrieve request and response message types.
* **tstorage_grpc.pb.go**, which contains the following:
  * An interface type (or stub) for clients to call with the methods defined in the TStorage service.
  * An interface type for servers to implement, also with the methods defined in the TStorage service.

## Part 3. Setup Simple gRPC Server {#5}

Please create the following blank files (notice asterisks):

#### Project Hierarchy
{{< highlight bash "linenos=false">}}
📁 tstorage-server
│   📄 README.md
│   📄 main.go
│   📄 Makefile
│
└───📁 cmd
│   |
|   └───📄 root.go
|       📄 serve.go (*)
|       📄 version.go
│
└───📁 internal (*)
|    │
|    └──📄 server_impl.go (*)
|       📄 server.go (*)
|
└───📁 proto (*)
     │
     └──📄 tstorage_grpc.pb.go
        📄 tstorage.proto
        📄 tstorage.pb.go

{{</ highlight >}}

Populate the following

#### (1 of 3) cmd/serve.go

```go
package cmd // github.com/bartmika/tstorage-server/cmd/serve.go

import (
	"os"
	"os/signal"
	"syscall"

	"github.com/spf13/cobra"

	server "github.com/bartmika/tstorage-server/internal"
)

var (
	port int
)

func init() {
	serveCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port to run this server on")
	serveCmd.MarkFlagRequired("port")
	rootCmd.AddCommand(serveCmd)
}

func doServe() {
	server := server.New(port)

	// DEVELOPERS CODE:
	// The following code will create an anonymous goroutine which will have a
	// blocking chan `sigs`. This blocking chan will only unblock when the
    // golang app receives a termination command; therfore the anyomous
    // goroutine will run and terminate our running application.
    //
    // Special Thanks:
    // (1) https://gobyexample.com/signals
    // (2) https://guzalexander.com/2017/05/31/gracefully-exit-server-in-go.html
    //
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
    go func() {
        <-sigs // Block execution until signal from terminal gets triggered here.
        server.StopMainRuntimeLoop()
    }()
	server.RunMainRuntimeLoop()
}

var serveCmd = &cobra.Command{
	Use:   "serve",
	Short: "Run the gRPC server",
	Long:  `Run the gRPC server to allow other services to access the storage application`,
	Run: func(cmd *cobra.Command, args []string) {
		doServe()
	},
}
```

#### (2 of 3) internal/server.go
```go
package internal // github.com/bartmika/tstorage-server/internal/server.go

import (
    "fmt"
    "log"
    "net"

    "google.golang.org/grpc"

    pb "github.com/bartmika/tstorage-server/proto"
)

type TStorageServer struct {
    port: port
    grpcServer *grpc.Server
}

func New(port int) (*TStorageServer) {
    return &TStorageServer{
        port: port,
        grpcServer: nil,
    }
}

// Function will consume the main runtime loop and run the business logic
// of the application.
func (app *TStorageServer) RunMainRuntimeLoop() {
   // Open a TCP server to the specified localhost and environment variable
   // specified port number.
   lis, err := net.Listen("tcp", fmt.Sprintf(":%v",s.port))
   if err != nil {
       log.Fatalf("failed to listen: %v", err)
   }

   // Initialize our gRPC server using our TCP server.
   grpcServer := grpc.NewServer()

   // Save reference to our application state.
   app.grpcServer = grpcServer

   // For debugging purposes only.
   log.Printf("gRPC server is running.")

   // Block the main runtime loop for accepting and processing gRPC requests.
   pb.RegisterTStorageServer(grpcServer, &TStorageServerImpl{
       // DEVELOPERS NOTE:
       // We want to attach to every gRPC call the following variables...
       // ...
       // ...
   })
   if err := grpcServer.Serve(lis); err != nil {
       log.Fatalf("failed to serve: %v", err)
   }
}

// Function will tell the application to stop the main runtime loop when
// the process has been finished.
func (app *TStorageServer) StopMainRuntimeLoop() {
    log.Printf("Starting graceful shutdown now...")

    // Finish any RPC communication taking place at the moment before
    // shutting down the gRPC server.
    app.grpcServer.GracefulStop()
}
```

#### (3 of 3) internal/server_impl.go
```go
package internal // github.com/bartmika/tstorage-server/internal/server_impl.go

import (
    "context"
    "log"

    pb "github.com/bartmika/tstorage-server/proto"
)

type TStorageServerImpl struct{
    pb.TStorageServer
}

func (s *TStorageServerImpl) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
    log.Printf("Received: %v", in.GetName())
    return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}
```

You now have implemented a basic server.

## Part 4. Setup Simple gRPC Client  {#6}

Please create the following blank files:

#### Project Hierarchy
{{< highlight bash "linenos=false">}}
📁 tstorage-server
│   📄 README.md
│   📄 main.go
│   📄 Makefile
│
└───📁 cmd
│   |
|   └───📄 hello.go (*)
|       📄 root.go
|       📄 serve.go
|       📄 version.go
│
└───📁 internal
|    │
|    └──📄 server_impl.go
|       📄 server.go
|
└───📁 proto
     │
     └──📄 tstorage_grpc.pb.go
        📄 tstorage.proto
        📄 tstorage.pb.go

{{</ highlight >}}

Next populate the following:

#### cmd/hello.go
```go
package cmd // github.com/bartmika/tstorage-server/cmd/hello.go

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/spf13/cobra"
	"google.golang.org/grpc"
	// "google.golang.org/grpc/credentials"

	pb "github.com/bartmika/tstorage-server/proto"
)

var (
	name string
)

func init() {
	helloCmd.Flags().StringVarP(&name, "name", "n", "Anonymous", "The name to send the server.")
	helloCmd.MarkFlagRequired("name")
	helloCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port of our server.")
	helloCmd.MarkFlagRequired("port")
	rootCmd.AddCommand(helloCmd)
}

func doHello() {
	// Set up a direct connection to the gRPC server.
	conn, err := grpc.Dial(
		fmt.Sprintf(":%v",port),
		grpc.WithInsecure(),
		grpc.WithBlock(),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}

	// Set up our protocol buffer interface.
	client := pb.NewTStorageClient(conn)
	defer conn.Close()

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

	// Perform our gRPC request.
    r, err := client.SayHello(ctx, &pb.HelloRequest{Name: name})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }

	// Print out the gRPC response.
    log.Printf("Server Response: %s", r.GetMessage())
}

var helloCmd = &cobra.Command{
	Use:   "hello",
	Short: "Send hello message to gRPC server",
	Long:  `Connect to the gRPC server and send a hello message. Command used to test out that the server is running.`,
	Run: func(cmd *cobra.Command, args []string) {
		doHello()
	},
}
```

Run the following **command** in your **terminal**.

```
go run main.go hello --name="Bart" --port=50051
```

You should get a response as follows:

```
2021/07/07 23:22:23 Server Response: Hello Bart
```

## Part 5. Setup `tstorage` library with our Project {#7}

#### (1 of 2) cmd/serve.go
```go
package cmd // github.com/bartmika/tstorage-server/cmd/serve.go

import (
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/spf13/cobra"

	server "github.com/bartmika/tstorage-server/internal"
	"github.com/bartmika/tstorage-server/utils"
)

var (
	port int
	dataPath string
	timestampPrecision string
	partitionDurationInHours int
	writeTimeoutInSeconds int
)

func init() {
	// The following are required.
	serveCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port to run this server on")
	serveCmd.MarkFlagRequired("port")

	// The following are optional and will have defaults placed when missing.
	serveCmd.Flags().StringVarP(&dataPath, "dataPath", "d", "./tsdb", "The location to save the database files to.")
	serveCmd.Flags().StringVarP(&timestampPrecision, "timestampPrecision", "t", "s", "The precision of timestamps to be used by all operations. Options: ")
	serveCmd.Flags().IntVarP(&partitionDurationInHours, "partitionDurationInHours", "b", 1, "The timestamp range inside partitions.")
	serveCmd.Flags().IntVarP(&writeTimeoutInSeconds, "writeTimeoutInSeconds", "w", 30, "The timeout to wait when workers are busy (in seconds).")

	// Make this sub-command part of our application.
	rootCmd.AddCommand(serveCmd)
}

func doServe() {
	// Convert the user inputted integer value to be a `time.Duration` type.
	partitionDuration := time.Duration(partitionDurationInHours) * time.Hour
	writeTimeout := time.Duration(writeTimeoutInSeconds) * time.Second

    // Setup our server.
	server := server.New(port, dataPath, timestampPrecision, partitionDuration, writeTimeout)

	// DEVELOPERS CODE:
	// The following code will create an anonymous goroutine which will have a
	// blocking chan `sigs`. This blocking chan will only unblock when the
    // golang app receives a termination command; therfore the anyomous
    // goroutine will run and terminate our running application.
    //
    // Special Thanks:
    // (1) https://gobyexample.com/signals
    // (2) https://guzalexander.com/2017/05/31/gracefully-exit-server-in-go.html
    //
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
    go func() {
        <-sigs // Block execution until signal from terminal gets triggered here.
        server.StopMainRuntimeLoop()
    }()
	server.RunMainRuntimeLoop()
}

var serveCmd = &cobra.Command{
	Use:   "serve",
	Short: "Run the gRPC server",
	Long:  `Run the gRPC server to allow other services to access the storage application`,
	Run: func(cmd *cobra.Command, args []string) {
		// Defensive code. Make sure the user selected the correct `timestampPrecision`
		// choices before continuing execution of our command.
        okTimestampPrecision := []string{"ns", "us", "ms", "s"}
		if utils.Contains(okTimestampPrecision, timestampPrecision) == false {
			log.Fatal("Timestamp precision must be either one of the following: ns, us, ms, or s.")
		}

		// Execute our command with our validated inputs.
		doServe()
	},
}
```

#### (2 of 2) internal/server.go
```go
package internal // github.com/bartmika/tstorage-server/internal/server.go

import (
    "fmt"
    "log"
    "net"
    "time"

    "google.golang.org/grpc"
    "github.com/nakabonne/tstorage"

    pb "github.com/bartmika/tstorage-server/proto"
)


type TStorageServer struct {
    port int
    dataPath string
    timestampPrecision tstorage.TimestampPrecision
    partitionDuration time.Duration
    writeTimeout time.Duration
    storage tstorage.Storage
    grpcServer *grpc.Server
}

func New(port int, dataPath string, timestampPrecision string, partitionDuration time.Duration, writeTimeout time.Duration) (*TStorageServer) {
    // Conver to the format that is accepted by the library.
    var tsp tstorage.TimestampPrecision
    switch timestampPrecision {
    case "ns":
        tsp = tstorage.Nanoseconds
    case "us":
        tsp = tstorage.Microseconds
    case "ms":
        tsp = tstorage.Milliseconds
    case "s":
        tsp = tstorage.Seconds
    }

    return &TStorageServer{
        port: port,
        dataPath: dataPath,
        timestampPrecision: tsp,
        partitionDuration: partitionDuration,
        writeTimeout: writeTimeout,
        storage: nil,
        grpcServer: nil,
    }
}


// Function will consume the main runtime loop and run the business logic
// of the application.
func (s *TStorageServer) RunMainRuntimeLoop() {
   // Open a TCP server to the specified localhost and environment variable
   // specified port number.
   lis, err := net.Listen("tcp", fmt.Sprintf(":%v",s.port))
   if err != nil {
       log.Fatalf("failed to listen: %v", err)
   }

   // Initialize our gRPC server using our TCP server.
   grpcServer := grpc.NewServer()

   // Initialize our fast time-series database.
    storage, _ := tstorage.NewStorage(
        tstorage.WithDataPath(s.dataPath),
		tstorage.WithTimestampPrecision(s.timestampPrecision),
        tstorage.WithPartitionDuration(s.partitionDuration),
        tstorage.WithWriteTimeout(s.writeTimeout),
    )

   // Save reference to our application state.
   s.grpcServer = grpcServer
   s.storage = storage

   // For debugging purposes only.
   log.Printf("gRPC server is running.")

   // Block the main runtime loop for accepting and processing gRPC requests.
   pb.RegisterTStorageServer(grpcServer, &TStorageServerImpl{
       // DEVELOPERS NOTE:
       // We want to attach to every gRPC call the following variables...
       storage: s.storage,
   })
   if err := grpcServer.Serve(lis); err != nil {
       log.Fatalf("failed to serve: %v", err)
   }
}


// Function will tell the application to stop the main runtime loop when
// the process has been finished.
func (s *TStorageServer) StopMainRuntimeLoop() {
    log.Printf("Starting graceful shutdown now...")

    // Finish our database operations running.
    s.storage.Close()

    // Finish any RPC communication taking place at the moment before
    // shutting down the gRPC server.
    s.grpcServer.GracefulStop()
}
```

## Part 6. Server Endpoints {#8}

Please override the following files.

#### (1 of 2) proto/tstorage.proto
```proto
syntax = "proto3"; // github.com/bartmika/tstorage-server/proto/tstorage.proto

option go_package = "github.com/bartmika/tstorage-server";

package proto;

import "google/protobuf/timestamp.proto";


// The tstorage service definition.
service TStorage {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
    rpc InsertRow (TimeSeriesDatum) returns (InsertResponse) {}
    rpc InsertRows (stream TimeSeriesDatum) returns (InsertResponse) {}
    rpc Select (Filter) returns (stream DataPoint) {}
}

// --- HELLO ENDPOINT ---

// The request message containing the user's name.
message HelloRequest {
    string name = 1;
}

// The response message containing the greetings
message HelloReply {
    string message = 1;
}

// --- COMMON TO ALL ENDPOINTS ---

message DataPoint {
    double value = 3;
    google.protobuf.Timestamp timestamp = 4;
}

message Label {
    string name = 1;
    string value = 2;
}

message TimeSeriesDatum {
    string metric = 1;
    repeated Label labels = 2;
    double value = 3;
    google.protobuf.Timestamp timestamp = 4;
}

// --- INSERT ENDPOINT ---

message InsertResponse {
    string message = 1;
    bool status = 2;
}

// --- SELECT ENDPOINT ---

message Filter {
    string metric = 1;
    repeated Label labels = 2;
    google.protobuf.Timestamp start = 3;
    google.protobuf.Timestamp end = 4;
}

message SelectResponse {
    repeated DataPoint points = 1;
}
```

In your **terminal** please write the following:

```bash
protoc --go_out=.      --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  proto/tstorage.proto
```

Afterwards update the following file:

#### (2 of 2) github.com/bartmika/tstorage-server/internal/server_impl.go
```go
package internal

import (
    "context"
    "log"
    "io"

    "github.com/nakabonne/tstorage"
    tspb "github.com/golang/protobuf/ptypes/timestamp"

    pb "github.com/bartmika/tstorage-server/proto"
)

type TStorageServerImpl struct{
	storage tstorage.Storage
    pb.TStorageServer
}

func (s *TStorageServerImpl) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
    log.Printf("Received: %v", in.GetName())
    return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func (s *TStorageServerImpl) InsertRow(ctx context.Context, in *pb.TimeSeriesDatum) (*pb.InsertResponse, error) {
    // // For debugging purposes only.
    // log.Println("Metric", in.Metric)
    // log.Println("Value", in.Value)
    // log.Println("Timestamp", in.Timestamp)
    // log.Println("Labels", in.Labels)

    // Generate our labels, if there are any.
    labels := []tstorage.Label{}
    for _, label := range in.Labels {
        labels = append(labels, tstorage.Label{Name: label.Name, Value: label.Value,})
    }

    // Generate our datapoint.
    dataPoint := tstorage.DataPoint{Timestamp: in.Timestamp.Seconds, Value: in.Value}

    err := s.storage.InsertRows([]tstorage.Row{
	    {
		    Metric:    in.Metric,
		    Labels:    labels,
		    DataPoint: dataPoint,
	    },
    })
    return &pb.InsertResponse{Message: "Created"}, err
}

func (s *TStorageServerImpl) InsertRows(stream pb.TStorage_InsertRowsServer) error {
    // // For debugging purposes only.
    // log.Println("Metric", in.Metric)
    // log.Println("Value", in.Value)
    // log.Println("Timestamp", in.Timestamp)
    // log.Println("Labels", in.Labels)

    // Wait and receieve the stream from the client.
    for {
        datum, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&pb.InsertResponse{
                Message: "Created",
            })
        }
        if err != nil {
            return err
        }

        // Generate our labels, if there are any.
        labels := []tstorage.Label{}
        for _, label := range datum.Labels {
            labels = append(labels, tstorage.Label{Name: label.Name, Value: label.Value,})
        }

        // Generate our datapoint.
        dataPoint := tstorage.DataPoint{Timestamp: datum.Timestamp.Seconds, Value: datum.Value}

        err = s.storage.InsertRows([]tstorage.Row{
    	    {
    		    Metric:    datum.Metric,
    		    Labels:    labels,
    		    DataPoint: dataPoint,
    	    },
        })
    }


    return nil
}

func (s *TStorageServerImpl) Select(in *pb.Filter, stream pb.TStorage_SelectServer) error {
    // // For debugging purposes only.
    // log.Println("Metric", in.Metric)
    // log.Println("Labels", in.Labels)
    // log.Println("Start", in.Start.Seconds)
    // log.Println("End", in.End.Seconds)

    // Generate our labels, if there are any.
    labels := []tstorage.Label{}
    for _, label := range in.Labels {
        labels = append(labels, tstorage.Label{Name: label.Name, Value: label.Value,})
    }

    points, err := s.storage.Select(in.Metric, labels, in.Start.Seconds, in.End.Seconds)
    if err != nil {
        return err
    }


    for _, point := range points {
        ts := &tspb.Timestamp{
    	    Seconds: point.Timestamp,
    	    Nanos: 0,
    	}
        dataPoint := &pb.DataPoint{Value: point.Value, Timestamp: ts,}
        if err := stream.Send(dataPoint); err != nil {
            return err
        }
    }

    return nil
}
```

## Part 7. Client Commands {#9}

Please create the following blank files:

#### Project Hierarchy
{{< highlight bash "linenos=false">}}
📁 tstorage-server
│   📄 README.md
│   📄 main.go
│   📄 Makefile
│
└───📁 cmd
│   |
|   └───📄 hello.go
|       📄 insert_row.go (*)
|       📄 insert_rows.go (*)
|       📄 root.go
|       📄 select.go (*)
|       📄 serve.go
|       📄 version.go
│
└───📁 internal
|    │
|    └──📄 server_impl.go
|       📄 server.go
|
└───📁 proto
     │
     └──📄 tstorage_grpc.pb.go
        📄 tstorage.proto
        📄 tstorage.pb.go

{{</ highlight >}}

Next populate the following:

#### (1 of 3) cmd/insert_row.go

```go
package cmd // github.com/bartmika/tstorage-server/cmd/insert_row.go

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/spf13/cobra"
	"google.golang.org/grpc"
	// "google.golang.org/grpc/credentials"

	tspb "github.com/golang/protobuf/ptypes/timestamp"

	pb "github.com/bartmika/tstorage-server/proto"
)

var (
	metric string
	value float64
	tsv int64
)

func init() {
	// The following are required.
	insertRowCmd.Flags().StringVarP(&metric, "metric", "m", "", "The metric to attach to the TSD.")
	insertRowCmd.MarkFlagRequired("metric")
	insertRowCmd.Flags().Float64VarP(&value, "value", "v", 0.00, "The value to attach to the TSD.")
	insertRowCmd.MarkFlagRequired("value")
	insertRowCmd.Flags().Int64VarP(&tsv, "timestamp", "t", 0, "The timestamp to attach to the TSD.")
	insertRowCmd.MarkFlagRequired("timestamp")

	// The following are optional and will have defaults placed when missing.
	insertRowCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port of our server.")
	rootCmd.AddCommand(insertRowCmd)
}

func doInsertRow() {
	// Set up a direct connection to the gRPC server.
	conn, err := grpc.Dial(
		fmt.Sprintf(":%v",port),
		grpc.WithInsecure(),
		grpc.WithBlock(),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}

	// Set up our protocol buffer interface.
	client := pb.NewTStorageClient(conn)
	defer conn.Close()

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

	ts := &tspb.Timestamp{
	    Seconds: tsv,
	    Nanos: 0,
	}

	// Generate our labels.
	labels := []*pb.Label{}
	labels = append(labels, &pb.Label{Name: "Source", Value:"Command"})

	// Perform our gRPC request.
    r, err := client.InsertRow(ctx, &pb.TimeSeriesDatum{Labels: labels, Metric: metric, Value: value, Timestamp: ts,})
    if err != nil {
        log.Fatalf("could not add: %v", err)
    }

	// Print out the gRPC response.
    log.Printf("Server Response: %s", r.GetMessage())
}

var insertRowCmd = &cobra.Command{
	Use:   "insert_row",
	Short: "Insert single datum",
	Long:  `Connect to the gRPC server and sends a single time-series datum.`,
	Run: func(cmd *cobra.Command, args []string) {
		doInsertRow()
	},
}
```

#### (2 of 3) cmd/insert_rows.go

```go
package cmd // github.com/bartmika/tstorage-server/cmd/insert_rows.go

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/spf13/cobra"
	"google.golang.org/grpc"
	// "google.golang.org/grpc/credentials"

	tspb "github.com/golang/protobuf/ptypes/timestamp"

	pb "github.com/bartmika/tstorage-server/proto"
)

func init() {
	// The following are required.
	insertRowsCmd.Flags().StringVarP(&metric, "metric", "m", "", "The metric to attach to the TSD.")
	insertRowsCmd.MarkFlagRequired("metric")
	insertRowsCmd.Flags().Float64VarP(&value, "value", "v", 0.00, "The value to attach to the TSD.")
	insertRowsCmd.MarkFlagRequired("value")
	insertRowsCmd.Flags().Int64VarP(&tsv, "timestamp", "t", 0, "The timestamp to attach to the TSD.")
	insertRowsCmd.MarkFlagRequired("timestamp")

	// The following are optional and will have defaults placed when missing.
	insertRowsCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port of our server.")
	rootCmd.AddCommand(insertRowsCmd)
}

func doInsertRows() {
	// Set up a direct connection to the gRPC server.
	conn, err := grpc.Dial(
		fmt.Sprintf(":%v",port),
		grpc.WithInsecure(),
		grpc.WithBlock(),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}

	// Set up our protocol buffer interface.
	client := pb.NewTStorageClient(conn)
	defer conn.Close()

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

	ts := &tspb.Timestamp{
	    Seconds: tsv,
	    Nanos: 0,
	}

	// Generate our labels.
	labels := []*pb.Label{}
	labels = append(labels, &pb.Label{Name: "Source", Value:"Command"})

	// Perform our gRPC request.
    r, err := client.InsertRow(ctx, &pb.TimeSeriesDatum{Labels: labels, Metric: metric, Value: value, Timestamp: ts,})
    if err != nil {
        log.Fatalf("could not add: %v", err)
    }

	// Print out the gRPC response.
    log.Printf("Server Response: %s", r.GetMessage())
}

var insertRowsCmd = &cobra.Command{
	Use:   "insert_rows",
	Short: "Insert single datum using streaming",
	Long:  `Connect to the gRPC server and send a time-series datum using the streaming RPC.`,
	Run: func(cmd *cobra.Command, args []string) {
		doInsertRows()
	},
}
```

#### (3 of 3) cmd/select.go

```go
package cmd // github.com/bartmika/tstorage-server/cmd/select.go

import (
	"context"
	"fmt"
	"io"
	"log"
	"time"

	"github.com/spf13/cobra"
	"google.golang.org/grpc"
	// "google.golang.org/grpc/credentials"

	tspb "github.com/golang/protobuf/ptypes/timestamp"

	pb "github.com/bartmika/tstorage-server/proto"
)

var (
	start int64
	end int64
)

func init() {
	// The following are required.
	selectCmd.Flags().StringVarP(&metric, "metric", "m", "", "The metric to filter by")
	selectCmd.MarkFlagRequired("metric")
	selectCmd.Flags().Int64VarP(&start, "start", "s", 0, "The start timestamp to begin our range")
	selectCmd.MarkFlagRequired("start")
	selectCmd.Flags().Int64VarP(&end, "end", "e", 0, "The end timestamp to finish our range")
	selectCmd.MarkFlagRequired("end")

	// The following are optional and will have defaults placed when missing.
	selectCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port of our server.")
	rootCmd.AddCommand(selectCmd)
}

func doSelectRow() {
	// Set up a direct connection to the gRPC server.
	conn, err := grpc.Dial(
		fmt.Sprintf(":%v",port),
		grpc.WithInsecure(),
		grpc.WithBlock(),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}

	// Set up our protocol buffer interface.
	client := pb.NewTStorageClient(conn)
	defer conn.Close()

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

    // Convert the unix timestamp into the protocal buffers timestamp format.
	sts := &tspb.Timestamp{
	    Seconds: start,
	    Nanos: 0,
	}
	ets := &tspb.Timestamp{
	    Seconds: end,
	    Nanos: 0,
	}

	// Generate our labels.
	labels := []*pb.Label{}
	labels = append(labels, &pb.Label{Name: "Source", Value:"Command"})

	// Perform our gRPC request.
    stream, err := client.Select(ctx, &pb.Filter{Labels: labels, Metric: metric, Start: sts, End: ets,})
    if err != nil {
        log.Fatalf("could not select: %v", err)
    }

    // Handle our stream of data from the server.
	for {
		dataPoint, err := stream.Recv()
		if err == io.EOF {
			break;
		}
		if err != nil {
			log.Fatalf("error with stream: %v", err)
		}

		// Print out the gRPC response.
	    log.Printf("Server Response: %s", dataPoint)
	}
}

var selectCmd = &cobra.Command{
	Use:   "select",
	Short: "List data",
	Long:  `Connect to the gRPC server and return list of results based on a selection filter.`,
	Run: func(cmd *cobra.Command, args []string) {
		doSelectRow()
	},
}
```

## Part 8. Usage Examples {#10}

### Server Start
To begin please start the server. For all the sub-commands `insert_row`, `insert_rows`, and `select` to work, this server must be running or else you will get an error.

```
go run main.go serve
```

If you get a message saying:

```
2021/07/09 12:48:05 gRPC server is running.
```

Then success!

### Insert Row
In a new **terminal tab** or **terminal window**, please run the follow:

Insert a new time series datum, try:

```
go run main.go insert_row --metric="Bart" --value=123.4 --timestamp=1600000001
```

If you get a response like this then success then you have successfully created a record.

```
2021/07/09 23:54:05 Server Response: Created
```

### Insert Row (with streams)
To verify the client connects and produces the output we want, run the following:

```
go run main.go insert_rows --metric="Bart" --value=123.4 --timestamp=1600000002
```

If you get a response like this then success!

```
2021/07/08 23:54:05 Server Response: Created
```

### Select
To verify the client connects and produces the output we want, run the following:

```bash
go run main.go select --metric="Bart" --start=1600000000 --end=1600000006
```

If you get a response like this then success!

```bash
2021/07/09 00:27:11 Server Response: [value:123.4  timestamp:{seconds:1600000001} value:123.4  timestamp:{seconds:1600000002} value:123.4  timestamp:{seconds:1600000003} value:123.4  timestamp:{seconds:1600000004} value:123.4  timestamp:{seconds:1600000005}]
```

You have successfully wrote a [`gRPC`](https://grpc.io) server overtop a fast time-series database.

Cover photo by [Harrison Broadbent](https://unsplash.com/@harrisonbroadbent) on [Unsplash](https://unsplash.com).
