---
title: "TPoller Server Open Source Time Series Data Polling Server Written in Golang"
date: 2021-07-12T12:00:04-04:00
draft: true
---


1. Every connected device has a `reader` application.
2. Every reader application runs as a gRPC server with an open service definition that other apps can access to read data
3. Every reader application must provide connect to the `tstorage-server` and always be connected.
4. Every reader application must have a `poll` RPC which will read the data from the device and send the data to `tstorage-server`

1. `tpoller-server` will poll data from whatever `reader` application it has configured
2. `tpoller-server` will connect to each `reader` device.
3. `tpoller-server` will call the `poll` RPC command so the device polls data and saves it to `tstorage-server`.



tweatherpoll-server
1. Connects to device
2. Connects to `tstorage-server`
3. Provides a simple 'poll' RPC command to be called by external app
4. When 'poll' is called, server will poll data from the connected device and save the time series data to `tstorage-server`.

tpoller-server
1. You choose the internval for polling. Right now only supports every one minute
2. You choose the compatible poll apps
3. You start the server
4. Server will make `poll` RPC requests to the chosen compatible poll apps
