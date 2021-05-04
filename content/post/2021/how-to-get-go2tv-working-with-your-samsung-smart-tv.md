---
title: "How to Get Go2TV Working With Your Samsung Smart TV"
date: 2021-04-15T00:22:03-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Open Source Software"
tags:
- "Golang"
- "IoT"
---

![](/img/2021/common/go-banner.png)

A recent reddit post on [/r/golang](https://www.reddit.com/r/golang/) titled [Go2TV - cast videos to UPnP/DLNA MediaRenderers
](https://www.reddit.com/r/golang/comments/mqu6r0/go2tv_cast_videos_to_upnpdlna_mediarenderers/) caught my interest. In this blog post I'll try to get it working with my Samsung TV.

<!--more-->

To begin, I followed the link which leads to the public github repo - [Go2TV](https://github.com/alexballas/Go2TV). The documentation looks good so I proceed to do the following.

First I check to see if their version of Golang matches my own. I run:

{{< highlight bash "linenos=false">}}
$ go version
{{</ highlight >}}

I am using the older ``Go 1.15`` version but this software requires ``Go 1.16``, I swiftly upgrade my Go via [this link](https://golang.org/dl/).
If you have anything below 16.0 then you must download the latest version of Golang before proceeding.

Next I clone the project locally on my computer:

{{< highlight bash "linenos=false">}}
$ cd ~/go/src/github.com/
$ mkdir alexballas
$ cd alexballas
$ git clone https://github.com/alexballas/Go2TV.git
$ cd Go2TV
{{</ highlight >}}

*Please note, I have my Golang home directory in the ``~/go/src/github.com`` location.*

After looking through the instructions and the ``Makefile``, I decide to build it locally by running these commands:

{{< highlight bash "linenos=false">}}
$ make build
$ cd build
$ chmod u+x go2tv
{{</ highlight >}}

Next I load up the help details of the app:

{{< highlight bash "linenos=false">}}
go2tv -h
{{</ highlight >}}

Alright, I turn on my TV and run the following command:

{{< highlight bash "linenos=false">}}
$ go2tv -l
{{</ highlight >}}

The output I get is as follows:

![go2tv -l](/img/2021/04-15/go2tv_-l.png)

Interesting! Copy the movie you want to stream to your TV into the ``build`` folder.

{{< highlight bash "linenos=false">}}
$ go2tv -v tech-presentation.mov
{{</ highlight >}}

On the Samsung TV it will ask for permission, grant it permission.

On the computer I see this:

![go2tv running](/img/2021/04-15/go2tv_running_1.png)

And on my TV I see:

![go2tv running](/img/2021/04-15/go2tv_running_2.jpg)

Hurray! Exciting to see something written in Golang communicating with my TV. Great job and thank you to [Alex Ballas](https://github.com/alexballas) for writing this awesome little app.
