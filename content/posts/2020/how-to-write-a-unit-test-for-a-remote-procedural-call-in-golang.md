---
title: "How to Write a Unit Test for a Remote Procedural Call in Golang"
date: 2020-08-13T23:14:55-04:00
draft: false
categories:
- "development"
tags:
- "go"
- "golang"
- "rpc"
- "unit testing"
---

Recently I have been learning about **remote procedural calls** (RPCs) in **Golang* and realized I was unable to find an easy example on how to write a unit test for RPCs. In this post, I'll explain how I figured out a solution.

<!--more-->

## Assumption
Before I begin, I am assuming you are just starting to learn about **remote procedural calls** (RPCs) and you don't know how you would go about writing a single unit test.

## What is a remote procedural call?
According to [this Wikipedia article](https://en.wikipedia.org/wiki/Remote_procedure_call):

> a **remote procedure call (RPC)** is when a computer program causes a procedure (subroutine) to execute in a different address space (commonly on another computer on a shared network), which is coded as if it were a normal (local) procedure call, without the programmer explicitly coding the details for the remote interaction. That is, the programmer writes essentially the same code whether the subroutine is local to the executing program, or remote.

So in essence you can write a bunch of **functions** in your application, serve these functions over the network and other programmers can call your functions locally on their computer and not worry about the network interaction - *how cool is that*?

## How does RPC work in Golang?

I will skip this section and refer you to an excellent resource I used to learn how RPC works in Golang. Please checkout [Chapter 13 Remote Procedure Call via
"Network Programming with Go" by Levon](https://tumregels.github.io/Network-Programming-with-Go/rpc/go_rpc.html).

## How to Unit Test?

Let's pretend you have written the following RPC methods which you want to serve in your RPC server:

```go
// mathrpc.go
package mathrpc

type Args struct {
    A, B int
}

type Quotient struct {
    Quo, Rem int
}

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
    *reply = args.A * args.B
    return nil
}

func (t *Arith) Divide(args *Args, quo *Quotient) error {
    if args.B == 0 {
        return error.String("divide by zero")
    }
    quo.Quo = args.A / args.B
    quo.Rem = args.A % args.B
    return nil
}
```

How would you unit test the above code? You can write code as follows:

```go
//mathrpc_test.go
package mathrpc

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestMultiplyWithSuccess(t *testing.T) {
	// Setup our test.
	arith := new(Arith)
	args := Args{3, 2}
	var reply int

	// Perform our operation.
	err := arith.Multiply(&args, &reply)

    // Perform our validation
	assert.NoError(t, err)
	assert.Equal(t, 6, reply, "Did not multiply correctly!`")
}

func TestDivideWithSuccess(t *testing.T) {
	// Setup our test.
	arith := new(Arith)
	args := Args{17, 8}
	var reply Quotient

	// Perform our operation.
	err := arith.Divide(&args, &reply)

    // Perform our validation
	assert.NoError(t, err)
	assert.Equal(t, 2, reply.Quo, "Did not return correct quotient`")
    assert.Equal(t, 125, reply.Rem, "Did not return correct remainder`")
}
```

As you can see, it's not too difficult to write a unit test on an RPC; in essence, you can treat the RPC function as if was a plain function and write a plain unit test on it.
