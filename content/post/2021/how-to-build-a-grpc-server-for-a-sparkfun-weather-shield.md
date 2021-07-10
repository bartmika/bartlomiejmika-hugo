---
title: "How to Build a gRPC Server for a SparkFun Weather Shield"
date: 2021-07-09T18:35:46-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Open Source Software"
tags:
- "IoT"
- "gRPC"
- "Golang"
---

![](/img/2021/03-27/grpc.jpg)

We will take what we learned in the [*previous post*](/post/2021/how-to-read-data-from-a-sparkfun-weather-shield-dev-13956-in-golang/) and rewrite it to support interprocess communication of our SparkFun Weather Shield from other applications using the [`gRPC`](https://grpc.io).

<!--more-->

## Setup a Basic gRPC Server

*Before writing any business logic, start off by building the project scaffolding. The scaffolding we will use is found if you read the [gRPC Basics tutorial](https://grpc.io/docs/languages/go/basics/). Our goal is to create a simple RPC server which we can call with a client; afterwords, we'll focus on writing our business logic. If you don't know anything about gRPC then checkout [this we article on gRPC basics](/post/2021/example-of-writing-a-simple-grpc-server-in-golang-from-scratch/).*

Start off by setting up the an empty repository in [github](https://github.com/bartmika/serialreader-server) and run the following in your `terminal`:

{{< highlight bash "linenos=false">}}
cd ~/go/src/github.com/bartmika
git clone https://github.com/bartmika/serialreader-server.git
cd serialreader-server
{{</ highlight >}}

Initialize golang modules.

{{< highlight bash "linenos=false">}}
go mod init github.com/bartmika/serialreader-server
{{</ highlight >}}

Install our project's dependencies.

{{< highlight bash "linenos=false">}}
export GO111MODULE=on  # Enable module mode
go get google.golang.org/protobuf/cmd/protoc-gen-go
go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
go get google.golang.org/grpc
go get github.com/golang/protobuf/ptypes/timestamp
go get github.com/spf13/cobra
go get github.com/tarm/serial
{{</ highlight >}}

Setup our initial project structure. We will use this structure to grow our application. Please look at the folder and file names in the "Project Hierarchy" and create them in your computer as blank files.

#### Project Hierarchy
{{< highlight bash "linenos=false">}}
📁 serialreader-server
│   📄 main.go
│
└───📁 cmd
|   |
|   └───📄 hello.go
|       📄 root.go
|       📄 serve.go
|       📄 version.go
|
└───📁 internal
|   |
|   └───📄 server_impl.go
|       📄 server.go
|
└───📁 proto
    |
    └───📄 serialreader.proto

{{</ highlight >}}

What are going to do? We want our project structured in such a way that it uses the [`spf13/cobra`](https://github.com/spf13/cobra) package so we can easily commands in our application.

Please copy and paste the following code into those empty files.

#### (1 of 8) main.go
{{< highlight go "linenos=false">}}
package main // github.com/bartmika/serialreader-server/main.go

import (
	"github.com/bartmika/serialreader-server/cmd"
)

func main() {
	cmd.Execute()
}
{{</ highlight >}}

#### (2 of 8) cmd/root.go
{{< highlight go "linenos=false">}}
package cmd // github.com/bartmika/serialreader-server/cmd/root.go

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
	Use:   "serialreader-server",
	Short: "Serve time-series data",
	Long:  `Serve time-series data from a connected Arduino device with an attached 'SparkFun Weather Shield' device over gRPC.`,
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

#### (3 of 8) cmd/version.go
{{< highlight go "linenos=false">}}
package cmd // github.com/bartmika/serialreader-server/cmd/version.go

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
		fmt.Println("serialreader-server v1.0")
	},
}
{{</ highlight >}}

#### (4 of 8) proto/serialreader.proto
{{< highlight go "linenos=false">}}
syntax = "proto3"; // github.com/bartmika/serialreader-server/proto/serialreader.proto

option go_package = "github.com/bartmika/serialreader-server";

package proto;


service SerialReader {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
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
{{</ highlight >}}

#### (5 of 8) internal/server.go
{{< highlight go "linenos=false">}}
package internal // github.com/bartmika/serialreader-server/internal/server.go

import (
	"fmt"
	"log"
	"net"

	"google.golang.org/grpc"

	pb "github.com/bartmika/serialreader-server/proto"
)

type SerialReaderServer struct {
	port               int
	grpcServer         *grpc.Server
}

func New(port int) *SerialReaderServer {
	return &SerialReaderServer{
		port:               port,
		grpcServer:         nil,
	}
}

// Function will consume the main runtime loop and run the business logic
// of the application.
func (s *SerialReaderServer) RunMainRuntimeLoop() {
	// Open a TCP server to the specified localhost and environment variable
	// specified port number.
	lis, err := net.Listen("tcp", fmt.Sprintf(":%v", s.port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	// Initialize our gRPC server using our TCP server.
	grpcServer := grpc.NewServer()

	// Save reference to our application state.
	s.grpcServer = grpcServer

	// For debugging purposes only.
	log.Printf("gRPC server is running.")

	// Block the main runtime loop for accepting and processing gRPC requests.
	pb.RegisterSerialReaderServer(grpcServer, &SerialReaderServerImpl{
		// DEVELOPERS NOTE:
		// We want to attach to every gRPC call the following variables...
		// ...
	})
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

// Function will tell the application to stop the main runtime loop when
// the process has been finished.
func (s *SerialReaderServer) StopMainRuntimeLoop() {
	log.Printf("Starting graceful shutdown now...")

	// Finish any RPC communication taking place at the moment before
	// shutting down the gRPC server.
	s.grpcServer.GracefulStop()
}
{{</ highlight >}}

#### (6 of 8) internal/server_impl.go
{{< highlight go "linenos=false">}}
package internal // github.com/bartmika/serialreader-server/internal/server_impl.go

import (
	"context"
	"log"

	pb "github.com/bartmika/serialreader-server/proto"
)

type SerialReaderServerImpl struct {
	pb.SerialReaderServer
}

func (s *SerialReaderServerImpl) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

{{</ highlight >}}

#### (7 of 8) cmd/serve.go
{{< highlight go "linenos=false">}}
package cmd // github.com/bartmika/serialreader-server/cmd/serve.go

import (
	"os"
	"os/signal"
	"syscall"

	"github.com/spf13/cobra"

	server "github.com/bartmika/serialreader-server/internal"
)

var (
	port                     int
)

func init() {
	// The following are optional and will have defaults placed when missing.
	serveCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port to run this server on")

	// Make this sub-command part of our application.
	rootCmd.AddCommand(serveCmd)
}

func doServe() {
	// Setup our server.
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
	Long:  `Run the gRPC server to allow other services to access the serial reader`,
	Run: func(cmd *cobra.Command, args []string) {
		doServe()
	},
}

{{</ highlight >}}

#### (8 of 8) cmd/hello.go
{{< highlight go "linenos=false">}}
package cmd // github.com/bartmika/serialreader-server/cmd/hello.go

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/spf13/cobra"
	"google.golang.org/grpc"
	// "google.golang.org/grpc/credentials"

	pb "github.com/bartmika/serialreader-server/proto"
)

var (
	name string
)

func init() {
	// The following are required.
	helloCmd.Flags().StringVarP(&name, "name", "n", "Anonymous", "The name to send the server.")
	helloCmd.MarkFlagRequired("name")

	// The following are optional and will have defaults placed when missing.
	helloCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port of our server.")
	rootCmd.AddCommand(helloCmd)
}

func doHello() {
	// Set up a direct connection to the gRPC server.
	conn, err := grpc.Dial(
		fmt.Sprintf(":%v", port),
		grpc.WithInsecure(),
		grpc.WithBlock(),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}

	// Set up our protocol buffer interface.
	client := pb.NewSerialReaderClient(conn)
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
{{</ highlight >}}

## Usage of the Basic gRPC Server

In your primary **terminal**, please run the server by entering the following **command**:

{{< highlight bash "linenos=false">}}
go run main.go serve
{{</ highlight >}}

You should see something like this:

![](/img/2021/07-09/a1.png)

If you see this then you have successfully setup the basic gRPC server. In a another **terminal** window or tab, please execute the following **command** to verify our server is working.


{{< highlight bash "linenos=false">}}
go run main.go hello --name="Bartlomiej"
{{</ highlight >}}

You should see something like this:

![](/img/2021/07-09/a2.png)

Good stuff, our server works! Now let's build a gRPC service definition over our serial reader.

## Integrate the SparkFun Weather Shield Reader
*Before we begin, I am assuming you have successfully read through our [previous article](/post/2021/how-to-read-data-from-a-sparkfun-weather-shield-dev-13956-in-golang/) before continuing. If you did not the next steps will note make sense.*

Take a look at the following folder structure and create the necessary empty files:

#### Project Hierarchy
{{< highlight bash "linenos=false">}}
📁 serialreader-server
│   📄 main.go
│
└───📁 cmd
|   |
|   └───📄 get_data.go (*)
|       📄 hello.go
|       📄 root.go
|       📄 serve.go
|       📄 version.go
|
└───📁 internal
|   |
|   └───📄 arduino_reader.go (*)
|       📄 server_impl.go
|       📄 server.go
|
└───📁 proto
    |
    └───📄 serialreader.proto

{{</ highlight >}}

Notice the asterisks (*), those are the new files you will need to create.

Please copy and paste the following files. It's important to note you need to override the entire file when pasting!

#### (1 of 6) proto/serialreader.proto
{{< highlight proto "linenos=false">}}
syntax = "proto3"; // github.com/bartmika/serialreader-server/proto/serialreader.proto

option go_package = "github.com/bartmika/serialreader-server";

package proto;

import "google/protobuf/timestamp.proto";

service SerialReader {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
    rpc GetSparkFunWeatherShieldData (GetTimeSeriesData) returns (SparkFunWeatherShieldTimeSeriesData) {}
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

// --- POLLING ENDPOINT ---

message GetTimeSeriesData {}

message SparkFunWeatherShieldTimeSeriesData {
    bool status = 1;
    google.protobuf.Timestamp timestamp = 2;
    float humidityValue = 3;
    string humidityUnit = 4;
    float temperatureValue = 5;
    string temperatureUnit = 6;
    float pressureValue = 7;
    string pressureUnit = 8;
    float temperatureBackupValue = 9;
    string temperatureBackupUnit = 10;
    float altitudeValue = 11;
    string altitudeUnit = 12;
    float illuminanceValue = 13;
    string illuminanceUnit = 14;
    float soilMoistureValue = 15;
    string soilMoistureUnit = 16;
}
{{</ highlight >}}

#### (2 of 6) internal/arduino_reader.go
{{< highlight go "linenos=false">}}
package internal // github.com/bartmika/serialreader-server/internal/arduino_reader.go

import (
	"fmt"
	"encoding/json"
	"log"
	"time"

	"github.com/tarm/serial"
)

const RX_BYTE = "1"

// The time-series data structure used to store all the data that will be
// returned by the `SparkFun Weather Shield` Arduino device.
type TimeSeriesData struct {
	Status                     string  `json:"status,omitempty"`
	Runtime                    int     `json:"runtime,omitempty"`
	Id                         int     `json:"id,omitempty"`
	HumidityValue              float32 `json:"humidity_value,omitempty"`
	HumidityUnit               string  `json:"humidity_unit,omitempty"`
	TemperatureValue           float32 `json:"temperature_primary_value,omitempty"`
	TemperatureUnit            string  `json:"temperature_primary_unit,omitempty"`
	PressureValue              float32 `json:"pressure_value,omitempty"`
	PressureUnit               string  `json:"pressure_unit,omitempty"`
	TemperatureBackupValue     float32 `json:"temperature_secondary_value,omitempty"`
	TemperatureBackupUnit      string  `json:"temperature_secondary_unit,omitempty"`
	AltitudeValue              float32 `json:"altitude_value,omitempty"`
	AltitudeUnit               string  `json:"altitude_unit,omitempty"`
	IlluminanceValue           float32 `json:"illuminance_value,omitempty"`
	IlluminanceUnit            string  `json:"illuminance_unit,omitempty"`
	Timestamp                  int64   `json:"timestamp,omitempty"`
}

// The abstraction of the `SparkFun Weather Shield` reader.
type ArduinoReader struct {
	serialPort *serial.Port
}

// Constructor used to intialize the serial reader designed to communicate
// with Arduino configured for the `SparkFun Weather Shield` settings.
func NewArduinoReader(devicePath string) *ArduinoReader {
	log.Printf("READER: Attempting to connect Arduino device...")
	c := &serial.Config{Name: devicePath, Baud: 9600}
	s, err := serial.OpenPort(c)
	if err != nil {
		log.Fatal(err)
	}

	// DEVELOPERS NOTE:
	// The following code will warm up the Arduino device before we are
	// able to make calls to the external sensors.
	log.Printf("READER: Waiting for Arduino external sensors to warm up")
	ar := &ArduinoReader{serialPort: s}
	ar.GetSparkFunWeatherShieldData()
	time.Sleep(5 * time.Second)
	ar.GetSparkFunWeatherShieldData()
	time.Sleep(5 * time.Second)
	return ar
}

// Function returns the JSON data of the instrument readings from our Arduino
// device configured for the `SparkFun Weather Shield` settings.
func (ar *ArduinoReader) GetSparkFunWeatherShieldData() *TimeSeriesData {
	// DEVELOPERS NOTE:
	// (1) The external device (Arduino) is setup to standby idle until it
	//     receives a poll request from this code, once a poll request has
	//     been submitted then all the sensors get polled and their data is
	//     returned.
	// (2) Please look at the following code to understand how the external
	//     device works in: <TODO ADD>
	//
	// (3) The reason for design is as follows:
	//     (a) The external device does not have a real-time clock
	//     (b) We don't want to add any real-time clock shields because
	//         extra hardware means it costs more.
	//     (c) We don't want to write complicated code of synching time
	//         from this code because it will make the code complicated.
	//     (d) Therefore we chose to make sensor polling be event based
	//         and this code needs to send a "poll request".

	// STEP 1:
	// We need to send a single byte to the external device (Arduino) which
	// will trigger a polling event on all the sensors.
	n, err := ar.serialPort.Write([]byte(RX_BYTE))
	if err != nil {
		log.Fatal(err)
	}

	// STEP 2:
	// The external device will poll the device, we need to make our main
	// runtime loop to be blocked so we wait until the device finishes and
	// returns all the sensor measurements.
	buf := make([]byte, 1028)
	n, err = ar.serialPort.Read(buf)
	if err != nil {
		log.Fatal(err)
	}

	// STEP 3:
	// Check to see if ANY data was returned from the external device, if
	// there was then we load up the string into a JSON object.
	var tsd TimeSeriesData
	err = json.Unmarshal(buf[:n], &tsd)
	if err != nil {
		return nil
	}
	tsd.Timestamp = time.Now().Unix()
	return &tsd
}

// Function used to print to the console the time series data.
func PrettyPrintTimeSeriesData(tsd *TimeSeriesData) {
	fmt.Println("Status: ", tsd.Status)
	fmt.Println("Runtime: ", tsd.Runtime)
	fmt.Println("Status: ", tsd.Id)
	fmt.Println("HumidityValue: ", tsd.HumidityValue)
	fmt.Println("HumidityUnit: ", tsd.HumidityUnit)
	fmt.Println("TemperatureValue: ", tsd.TemperatureValue)
	fmt.Println("TemperatureUnit: ", tsd.TemperatureUnit)
	fmt.Println("PressureValue: ", tsd.PressureValue)
	fmt.Println("PressureUnit: ", tsd.PressureUnit)
	fmt.Println("TemperatureBackupValue: ", tsd.TemperatureBackupValue)
	fmt.Println("TemperatureBackupUnit: ", tsd.TemperatureBackupUnit)
	fmt.Println("AltitudeValue: ", tsd.AltitudeValue)
	fmt.Println("AltitudeUnit: ", tsd.AltitudeUnit)
	fmt.Println("IlluminanceValue: ", tsd.IlluminanceValue)
	fmt.Println("IlluminanceUnit: ", tsd.IlluminanceUnit)
	fmt.Println("Timestamp: ", tsd.Timestamp)
}
{{</ highlight >}}

#### (3 of 6) internal/server.go
{{< highlight go "linenos=false">}}
package internal // github.com/bartmika/serialreader-server/internal/server.go

import (
	"fmt"
	"log"
	"net"

	"google.golang.org/grpc"

	pb "github.com/bartmika/serialreader-server/proto"
)

type SerialReaderServer struct {
	port               int
	arduinoDevicePath  string
	arduinoReader             *ArduinoReader
	grpcServer         *grpc.Server
}

func New(arduinoDevicePath string, port int) *SerialReaderServer {
	return &SerialReaderServer{
		port:               port,
		arduinoDevicePath:  arduinoDevicePath,
		arduinoReader:             nil,
		grpcServer:         nil,
	}
}

// Function will consume the main runtime loop and run the business logic
// of the application.
func (s *SerialReaderServer) RunMainRuntimeLoop() {
	// Open a TCP server to the specified localhost and environment variable
	// specified port number.
	lis, err := net.Listen("tcp", fmt.Sprintf(":%v", s.port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	// Establish our device connection.
	arduinoReader := NewArduinoReader(s.arduinoDevicePath)

	// Initialize our gRPC server using our TCP server.
	grpcServer := grpc.NewServer()

	// Save reference to our application state.
	s.grpcServer = grpcServer
	s.arduinoReader = arduinoReader

	// For debugging purposes only.
	log.Printf("gRPC server is running.")

	// Block the main runtime loop for accepting and processing gRPC requests.
	pb.RegisterSerialReaderServer(grpcServer, &SerialReaderServerImpl{
		// DEVELOPERS NOTE:
		// We want to attach to every gRPC call the following variables...
		arduinoReader: arduinoReader,
	})
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

// Function will tell the application to stop the main runtime loop when
// the process has been finished.
func (s *SerialReaderServer) StopMainRuntimeLoop() {
	log.Printf("Starting graceful shutdown now...")

	s.arduinoReader = nil

	// Finish any RPC communication taking place at the moment before
	// shutting down the gRPC server.
	s.grpcServer.GracefulStop()
}
{{</ highlight >}}

#### (4 of 6) internal/server_impl.go
{{< highlight go "linenos=false">}}
package internal // github.com/bartmika/serialreader-server/internal/server_impl.go

import (
	"context"
	"log"

	"github.com/golang/protobuf/ptypes"

	pb "github.com/bartmika/serialreader-server/proto"
)

type SerialReaderServerImpl struct {
	arduinoReader *ArduinoReader
	pb.SerialReaderServer
}

func (s *SerialReaderServerImpl) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func (s *SerialReaderServerImpl) GetSparkFunWeatherShieldData(ctx context.Context, in *pb.GetTimeSeriesData) (*pb.SparkFunWeatherShieldTimeSeriesData, error) {
	datum := s.arduinoReader.GetSparkFunWeatherShieldData()
	return &pb.SparkFunWeatherShieldTimeSeriesData{
		Status: true,
		Timestamp: ptypes.TimestampNow(), // Note: https://godoc.org/github.com/golang/protobuf/ptypes#Timestamp
		HumidityValue: datum.HumidityValue,
		HumidityUnit: datum.HumidityUnit,
		TemperatureValue: datum.TemperatureValue,
		TemperatureUnit: datum.TemperatureUnit,
		PressureValue: datum.PressureValue,
		PressureUnit: datum.PressureUnit,
		TemperatureBackupValue: datum.TemperatureBackupValue,
		TemperatureBackupUnit: datum.TemperatureBackupUnit,
		AltitudeValue: datum.AltitudeValue,
		AltitudeUnit: datum.AltitudeUnit,
		IlluminanceValue: datum.IlluminanceValue,
		IlluminanceUnit: datum.IlluminanceUnit,
	}, nil
}
{{</ highlight >}}

#### (5 of 6) cmd/serve.go
{{< highlight go "linenos=false">}}
package cmd // github.com/bartmika/serialreader-server/cmd/serve.go

import (
	"os"
	"os/signal"
	"syscall"

	"github.com/spf13/cobra"

	server "github.com/bartmika/serialreader-server/internal"
)

var (
	port                     int
	arduinoDevicePath        string
)

func init() {
	// The following are required.
	serveCmd.Flags().StringVarP(&arduinoDevicePath, "arduino_path", "f", "/dev/cu.usbmodem14201", "The location of the connected arduino device on your computer.")
	serveCmd.MarkFlagRequired("arduino_path")

	// The following are optional and will have defaults placed when missing.
	serveCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port to run this server on")

	// Make this sub-command part of our application.
	rootCmd.AddCommand(serveCmd)
}

func doServe() {
	// Setup our server.
	server := server.New(arduinoDevicePath, port)

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
	Long:  `Run the gRPC server to allow other services to access the serial reader`,
	Run: func(cmd *cobra.Command, args []string) {
		doServe()
	},
}
{{</ highlight >}}

#### (6 of 6) cmd/get_data.go
{{< highlight go "linenos=false">}}
package cmd // github.com/bartmika/serialreader-server/cmd/get_data.go

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/spf13/cobra"
	"google.golang.org/grpc"
	// "google.golang.org/grpc/credentials"

	pb "github.com/bartmika/serialreader-server/proto"
)

func init() {
	// The following are optional and will have defaults placed when missing.
	getDataCmd.Flags().IntVarP(&port, "port", "p", 50051, "The port of our server.")
	rootCmd.AddCommand(getDataCmd)
}

func doGetData() {
	// Set up a direct connection to the gRPC server.
	conn, err := grpc.Dial(
		fmt.Sprintf(":%v", port),
		grpc.WithInsecure(),
		grpc.WithBlock(),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}

	// Set up our protocol buffer interface.
	client := pb.NewSerialReaderClient(conn)
	defer conn.Close()

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	// Perform our gRPC request.
	tsd, err := client.GetSparkFunWeatherShieldData(ctx, &pb.GetTimeSeriesData{})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}

	// Print out the gRPC response.
	log.Println("Server Response:")
	fmt.Println("Status: ", tsd.Status)
	fmt.Println("HumidityValue: ", tsd.HumidityValue)
	fmt.Println("HumidityUnit: ", tsd.HumidityUnit)
	fmt.Println("TemperatureValue: ", tsd.TemperatureValue)
	fmt.Println("TemperatureUnit: ", tsd.TemperatureUnit)
	fmt.Println("PressureValue: ", tsd.PressureValue)
	fmt.Println("PressureUnit: ", tsd.PressureUnit)
	fmt.Println("TemperatureBackupValue: ", tsd.TemperatureBackupValue)
	fmt.Println("TemperatureBackupUnit: ", tsd.TemperatureBackupUnit)
	fmt.Println("AltitudeValue: ", tsd.AltitudeValue)
	fmt.Println("AltitudeUnit: ", tsd.AltitudeUnit)
	fmt.Println("IlluminanceValue: ", tsd.IlluminanceValue)
	fmt.Println("IlluminanceUnit: ", tsd.IlluminanceUnit)
	fmt.Println("Timestamp: ", tsd.Timestamp)
}

var getDataCmd = &cobra.Command{
	Use:   "get_data",
	Short: "Poll data from the gRPC server",
	Long:  `Connect to the gRPC server and poll the time series data. Command used to test out that the server is running.`,
	Run: func(cmd *cobra.Command, args []string) {
		doGetData()
	},
}
{{</ highlight >}}

## Usage of our Serial Data Reader gRPC Server

In your primary **terminal**, please run the server by entering the following **command**:

{{< highlight bash "linenos=false">}}
go run main.go serve -f="/dev/cu.usbmodem14401"
{{</ highlight >}}

Please note that the value `/dev/cu.usbmodem14401` was taken from the **Arduino IDE**, if you don't know how to get this value, please [read this article](/post/2021/how-to-read-data-from-a-sparkfun-weather-shield-dev-13956-in-golang/).

In another **terminal window** or **terminal tab**, please execute the following **command** to verify our server is working.

{{< highlight bash "linenos=false">}}
go run main.go get_data
{{</ highlight >}}

You should see something like this:

![](/img/2021/07-09/a3.png)

If you see the above, congratulations! You have written a gRPC server overtop your serial reader so you can now write an unlimited applications to poll the data.


## Where do we go from here?

You know how to read from the device and you now know how to serve it across many localhost applications or remote applications using [`gRPC`](https://grpc.io). The next logic step is figuring out *how* to store the time-series data we retrieved. In the [next post](/post/2021/how-to-build-a-grpc-server-over-tstorage-to-create-tstorage-server/) we will discuss building a [`gRPC`](https://grpc.io) server for storing and retrieving data.
