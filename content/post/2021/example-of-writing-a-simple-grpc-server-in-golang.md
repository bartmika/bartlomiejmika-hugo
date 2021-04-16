---
title: "Example of Writing a Simple gRPC Server in Golang"
date: 2021-03-27T14:10:03-04:00
draft: true
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "Golang"
- "gRPC"
---

How do you write a Golang server using [``gRPC``](https://grpc.io/docs/languages/go/quickstart/) from scratch? Heres how to do it.

<!--more-->

# Introduction
*Before we begin, I'd like to mention that the code below was taken from the [``/example``](https://github.com/grpc/grpc-go/tree/master/examples) folder in the [``gRPC``](https://grpc.io/docs/languages/go/quickstart/) package.*

As a textile learner, I like to write things out even if provided - I don't feel confident being handed code and be expected to learn - I learn better by writing it from scratch; as a result, the [``gRPC`` documentation](https://grpc.io/docs/languages/go/quickstart/) has not been good to me, to remedy this issue, I've written this article to help me understand better by writing out the steps from scratch.

# Instructions

Let's begin by going to our ``golang`` home directory.
{{< highlight bash "linenos=false">}}
$ cd ~/go/src/github.com/bartmika
{{</ highlight >}}

Create our project and initialize our module.
{{< highlight bash "linenos=false">}}
$ mkdir simple-grpc
$ cd simple-grpc
$ go mod init github.com/bartmika/simple-grpc
{{</ highlight >}}

Get our project dependencies
{{< highlight bash "linenos=false">}}
$ export GO111MODULE=on  # Enable module mode
$ go get google.golang.org/protobuf/cmd/protoc-gen-go
$ go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
{{</ highlight >}}

Before we begin, make sure we export our path (if you haven't done this before).
{{< highlight bash "linenos=false">}}
$ export PATH="$PATH:$(go env GOPATH)/bin"
{{</ highlight >}}

To get started with [``gRPC``](https://grpc.io/docs/languages/go/quickstart/) we will need to setup our ``proto`` file. The file is responsible for defining the interface of our project. **Proto** is a short form for **protocol buffers** - [Would you like to know more?](https://developers.google.com/protocol-buffers)
{{< highlight bash "linenos=false">}}
$ mkdir proto
$ vi proto/greeter.proto
{{</ highlight >}}

Copy and paste the following code into the file you have open.
{{< highlight proto "linenos=false">}}
syntax = "proto3";

option go_package = "github.com/bartmika/simple-server"; // Don't forget to change!

package helloworld;

// The greeting service definition.
service Greeter {
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

Now that we defined our interface, we'll need to generate our client and server files.
{{< highlight bash "linenos=false">}}
$ protoc --go_out=. --go_opt=paths=source_relative \
   --go-grpc_out=. --go-grpc_opt=paths=source_relative \
   proto/greeter.proto
{{</ highlight >}}

We have our server and client code generated. We can now write our own ``golang`` code to *utilize* them. Start by creating our server.
{{< highlight bash "linenos=false">}}
$ mkdir -p cmd/server
$ touch cmd/server/main.go
{{</ highlight >}}

Copy and paste the following code into our server:
{{< highlight go "linenos=false">}}
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"

    pb "github.com/bartmika/simple-grpc/proto"
)

const (
    port = ":50051"
)

// server is used to implement helloworld.GreeterServer.
type server struct {
    pb.UnimplementedGreeterServer
}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
    log.Printf("Received: %v", in.GetName())
    return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func main() {
    lis, err := net.Listen("tcp", port)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    s := grpc.NewServer()
    pb.RegisterGreeterServer(s, &server{})
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
{{</ highlight >}}

Next create our client file.
{{< highlight bash "linenos=false">}}
$ mkdir -p cmd/client
$ touch cmd/client/main.go
{{</ highlight >}}

Copy and paste the following code into our server:
{{< highlight go "linenos=false">}}
package main

import (
    "context"
    "log"
    "os"
    "time"

    "google.golang.org/grpc"

    pb "github.com/bartmika/simple-grpc/proto"
)

const (
    address     = "localhost:50051"
    defaultName = "world"
)

func main() {
    // Set up a connection to the server.
    conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()
    c := pb.NewGreeterClient(conn)

    // Contact the server and print out its response.
    name := defaultName
    if len(os.Args) > 1 {
        name = os.Args[1]
    }
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()
    r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    log.Printf("Greeting: %s", r.GetMessage())
}
{{</ highlight >}}

Finally, run the server and client in seperate terminals via the commands:

{{< highlight bash "linenos=false">}}
$ go run cmd/server
$ go run cmd/client "Bart"
{{</ highlight >}}
