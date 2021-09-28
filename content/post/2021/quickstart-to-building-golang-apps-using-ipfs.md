---
title: "Quickstart to Building Golang Apps using IPFS"
date: 2021-09-26T23:58:04-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Development"
tags:
- "Golang"
- "IPFS"
---

![](/img/2021/09-26/ipfs-logo.webp)

Do you want to write a Golang app which will use [IPFS](http://ipfs.io)? The purpose of this post is to get you up and running [IPFS](http://ipfs.io) as quickly as possible so you can see what's involved.

<!--more-->

## Context

I wanted to experiment with [IPFS](http://ipfs.io/) with Golang but had difficulty with the [provided tutorial](https://github.com/ipfs/go-ipfs/tree/c9cc09f6f7ebe95da69be6fa92c88e4cb245d90b/docs/examples/go-ipfs-as-a-library). I spent some time researching and wrote some useful code samples you can use to quickly get started running IPFS for programming.

## Before you begin

Please install [IPFS Desktop](http://docs.ipfs.ioinstall/ipfs-desktop/) before proceeding with the code samples. Once installed, start the desktop app and if it connects you are ready to begin.

Also I am assuming you have a home directory where you put your Golang source code you are working on. Mien will be: ``~/go/src/github.com/bartmika``.

## IPFS Quickstart

There will be three code samples you can use to get working. The purpose of the three examples are as follows:

* **Example 1** - Demonstrate how to send and receive simple text data over the IPFS network using your local IPFS daemon.

* **Example 2** - Demonstrate how to send and receive JSON encoded data over the IPFS network using your local IPFS daemon.

* **Example 3** - Demonstrate how I setup the [Go IPFS as a Library tutorial](https://github.com/ipfs/go-ipfs/tree/c9cc09f6f7ebe95da69be6fa92c88e4cb245d90b/docs/examples/go-ipfs-as-a-library).

* **Example 4** - Demonstrate how to use publish/subscription code via IPFS network. This functionality is typically found in chatting apps for example.

### Example 1: Send and Receive Text Data over IPFS

1. Create your project in your home `golang` directory.

    ```bash
    cd ~/go/src/github.com/bartmika
    mkdir ipfs-example1
    cd ipfs-example1
    ```

2. Initialize our `golang` project:

    ```bash
    go mod init github.com/bartmika/ipfs-example1
    ```

3. Get our only dependency.

    ```bash
    go get github.com/ipfs/go-ipfs-api
    ```

4. When the dependencies update, your `go.mod` file should look something like this:

    ```go
    module github.com/bartmika/ipfs-example1

    go 1.16

    require github.com/ipfs/go-ipfs-api v0.2.0
    ```

5. Create a `main.go` file and populate it with the following code sample:

    ```go
    package main

    import (
    	// "context"
    	"bytes"
    	"fmt"
    	"os"
    	"strings"

    	shell "github.com/ipfs/go-ipfs-api"
    )

    func main() {
    	//
    	// Connect to your local IPFS deamon running in the background.
    	//

    	// Where your local node is running on localhost:5001
    	sh := shell.NewShell("localhost:5001")

    	//
    	// Add the file to IPFS
    	//

    	cid, err := sh.Add(strings.NewReader("hello world!"))
    	if err != nil {
    		fmt.Fprintf(os.Stderr, "error: %s", err)
    		os.Exit(1)
    	}
    	fmt.Printf("added %s\n", cid)

    	//
    	// Get the data from IPFS and save the contents to a file.
    	//

    	out := fmt.Sprintf("%s.txt", cid)
    	err = sh.Get(cid, out)
    	if err != nil {
    		fmt.Fprintf(os.Stderr, "error: %s", err)
    		os.Exit(1)
    	}

    	//
    	// Get the data from IPFS and output the contents into `string` format
    	// and output into the terminal console.
    	//

    	data, err := sh.Cat(cid)
    	if err != nil {
    		fmt.Fprintf(os.Stderr, "error: %s", err)
    		os.Exit(1)
    	}

    	// ...so we convert it to a string by passing it through
    	// a buffer first. A 'costly' but useful process.
    	// https://golangcode.com/convert-io-readcloser-to-a-string/
    	buf := new(bytes.Buffer)
    	buf.ReadFrom(data)
    	newStr := buf.String()
    	fmt.Printf("data %s", newStr)

    }
    ```

6. Run the code.

    ```bash
    go run main.go
    ```

7. You should see code being outputted. Spend time and review the code. Where do you go next? Read the [API documentation](https://github.com/ipfs/go-ipfs-api).


### Example 2. Send and Receive JSON Data over IPFS

1. Create your project in your home `golang` directory.

    ```bash
    cd ~/go/src/github.com/bartmika
    mkdir ipfs-example2
    cd ipfs-example2
    ```

2. Initialize our `golang` project:

    ```bash
    go mod init github.com/bartmika/ipfs-example2
    ```

3. Get our only dependency.

    ```bash
    go get github.com/ipfs/go-ipfs-api
    ```

4. Create a `main.go` file and populate it with the following code sample:

    ```go
    package main

    import (
    	// "context"
    	"bytes"
    	"encoding/json"
    	"fmt"
    	"os"

    	shell "github.com/ipfs/go-ipfs-api"
    )

    // TimeSeriesDatum is the structure used to store a single time series data
    type TimeSeriesDatum struct {
    	Id    uint64 `json:"id"`
    	Value uint64 `json:"value"`
    }

    func main() {
    	//
    	// Connect to your local IPFS deamon running in the background.
    	//

    	// Where your local node is running on localhost:5001
    	sh := shell.NewShell("localhost:5001")

    	//
    	// Add the file to IPFS
    	//

    	tsd := &TimeSeriesDatum{
    		Id:    1,
    		Value: 123,
    	}
    	tsdBin, _ := json.Marshal(tsd)
    	reader := bytes.NewReader(tsdBin)

    	cid, err := sh.Add(reader)
    	if err != nil {
    		fmt.Fprintf(os.Stderr, "error: %s", err)
    		os.Exit(1)
    	}
    	fmt.Printf("added %s\n", cid)

    	//
    	// Get the data from IPFS and output the contents into `struct` format.
    	//

    	data, err := sh.Cat(cid)
    	if err != nil {
    		fmt.Fprintf(os.Stderr, "error: %s", err)
    		os.Exit(1)
    	}

    	// ...so we convert it to a string by passing it through
    	// a buffer first. A 'costly' but useful process.
    	// https://golangcode.com/convert-io-readcloser-to-a-string/
    	buf := new(bytes.Buffer)
    	buf.ReadFrom(data)
    	newStr := buf.String()

    	res := &TimeSeriesDatum{}
    	json.Unmarshal([]byte(newStr), &res)
    	fmt.Println(res)
    }
    ```


5. Run the code.

    ```bash
    go run main.go
    ```

6. You should see code being outputted. Spend time and review the code. Where do you go next? Read the [API documentation](https://github.com/ipfs/go-ipfs-api).


### Example 3. Go IPFS as a Library

I tried the running the [Use go-ipfs as a library to spawn a node and add a file code](https://github.com/ipfs/go-ipfs/tree/c9cc09f6f7ebe95da69be6fa92c88e4cb245d90b/docs/examples/go-ipfs-as-a-library) (on Sept 25th 2021) and ran into various errors. After some researching here is what I did to get it working locally.


1. Create your project in your home `golang` directory.

    ```bash
    cd ~/go/src/github.com/bartmika
    mkdir ipfs-example3
    cd ipfs-example3
    ```

2. Initialize our `golang` project:

    ```bash
    go mod init github.com/bartmika/ipfs-example3
    ```

3. I ran into various problems with the go packages and finally go it working with the following contents. Update your `go.mod` file to look as follows:

    ```go
    module github.com/bartmika/ipfs-example

    go 1.16

    require (
	    github.com/ipfs/go-ipfs v0.7.0
	    github.com/ipfs/go-ipfs-config v0.13.0
	    github.com/ipfs/go-ipfs-files v0.0.8
	    github.com/ipfs/interface-go-ipfs-core v0.4.0
	    github.com/libp2p/go-libp2p-core v0.6.1
	    github.com/libp2p/go-libp2p-peerstore v0.2.6
	    github.com/multiformats/go-multiaddr v0.3.1
    )
    ```


4. Create a `main.go` file and populate it with the [code found here](https://github.com/ipfs/go-ipfs/blob/c9cc09f6f7ebe95da69be6fa92c88e4cb245d90b/docs/examples/go-ipfs-as-a-library/main.go).

5. Run the code.

    ```bash
    go run main.go
    ```

6. You should see code being outputted. Spend time and review the code.


### Example 4: Publish and Subscribe to Text Data over IPFS

1. Create your project in your home `golang` directory.

    ```bash
    cd ~/go/src/github.com/bartmika
    mkdir ipfs-example4
    cd ipfs-example4
    ```

2. Initialize our `golang` project:

    ```bash
    go mod init github.com/bartmika/ipfs-example4
    ```

3. Get our only dependency.

    ```bash
    go get github.com/ipfs/go-ipfs-api
    ```

4. When the dependencies update, your `go.mod` file should look something like this:

    ```go
    module github.com/bartmika/ipfs-example4

    go 1.16

    require github.com/ipfs/go-ipfs-api v0.2.0
    ```

5. Create a `main.go` file and populate it with the following code sample:

    ```go
    package main

    import (
    	"log"
    	"time"
    	// "context"

    	shell "github.com/ipfs/go-ipfs-api"
    )

    func main() {
    	//
    	// Before we begin...
    	//

    	// Make sure that `pubsub` is enabled by either running:
    	// (a) ipfs daemon --enable-pubsub-experiment
    	// (b) Or enable `pubsub` in your `IPFS Desktop`

    	// (c) Create a topic we are to follow.
    	topic := "some-test-topic-created-by-bart" // You can change to your own value!

    	// (d) Please read more about `pubsub` via the following links:
    	// - Take a look at pubsub on IPFS via http://blog.ipfs.io.ipns.localhost:8080/25-pubsub/

    	//
    	// Connect to your local IPFS deamon running in the background.
    	//

    	// Where your local node is running on localhost:5001
    	sh := shell.NewShell("localhost:5001")

    	//
    	// Subscribe to a `topic` in IPFS.
    	//

    	subscription, err := sh.PubSubSubscribe(topic)
    	if err != nil {
    		log.Fatal(err)
    	}
    	defer subscription.Cancel()

    	//
    	// Publish to the `topic` through IPFS.
    	//

    	go func() {
    		log.Println("Publisher is about to begin...")
    		time.Sleep(2 * time.Second)

    		sh.PubSubPublish(topic, "Hello world!!")
    		log.Println("Finished publishing.")
    	}()

    	//
    	// Wait to receive published data in the topic.
    	//

    	log.Println("Waiting to receive data from publisher on topic...")
    	res, err := subscription.Next()
    	if err != nil {
    		log.Fatal(err)
    	}
    	log.Println("Successfully received from published!")

    	//
    	// Decode the string data.
    	//

    	log.Println("Received From:", res.From)
    	str := string(res.Data)
    	log.Println("Received Data:", str)
    	log.Println("Received Seqno:", res.Seqno)
    	log.Println("Received TopicIDs:", res.TopicIDs)
    }
    ```

6. Run the code.

    ```bash
    go run main.go
    ```

7. You should see code being outputted. Spend time and review the code. Where do you go next? Read the [API documentation](https://github.com/ipfs/go-ipfs-api).


## Misc: How do I delete all my pinned items?

You may have noticed in your **IPFS Desktop** that the **Settings** navigation has a **PINNING SERVICES** section saying we have some pinned items. We can remove these items.

Simply run the following code in your terminal:

```bash
ipfs pin ls --type recursive | cut -d' ' -f1 | xargs -n1 ipfs pin rm
```

Special thanks to [jclay](https://stackoverflow.com/users/2609013/jclay) via [stackoverflow](https://stackoverflow.com/a/43118023) for providing the above commands.
