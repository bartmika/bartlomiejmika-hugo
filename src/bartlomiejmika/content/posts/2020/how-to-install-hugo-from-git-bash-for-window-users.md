---
title: "How to Install Hugo From Git Bash for Window Users"
subtitle: "Learn how to build and install Hugo from source code"
date: 2020-09-09T22:11:07-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "Golang"
- "Static Site Generator"
- "Hugo"
- "Static Site"
- "HOWTO"
cover:
    image: "/img/2020/09/07-hugo-banner.jpeg"
    alt: "Hugo"
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---

The purpose of this article is to help beginners understand how to do the [advanced install](https://github.com/gohugoio/hugo#build-and-install-the-binaries-from-source-advanced-install) of [``hugo``](https://github.com/gohugoio/hugo) static site generator (SSG).

<!--more-->
# Introduction

In the previous [articles](/posts/2020/how-to-start-a-personal-blog-with-hugo-a-static-site-generator-written-in-go-part-1/) I wrote them under the assumption that the user was for *MacOS* or *Linux*, but what about *Windows*? If you are a *Windows* user then this article is for you!

# Instructions

1. Go to the [following link](https://golang.org/dl/) and download the latest version of **Golang** programming language. When this article was written we installed version **1.15.2**.

2. Once you download the file, install the executable and you'll have **Golang** ready on your system.

3. Go the [following link](https://git-scm.com/downloads) and download the latest version of **Git**. When this article was written we installed version **2.28.0**.

4. Once you download the file, install the executable and you'll have **Git** ready on your system.

5. Click **Start** in the menu and search for the keyword **Git Bash**, once you find it please click on the application.

6. You should see the terminal loaded up.

7. Before we install [``hugo``](https://github.com/gohugoio/hugo), we will need to setup our ``go`` local environment:

    ```bash
    cd ~/
    mkdir go
    cd go
    mkdir bin
    mkdir pkg
    mkdir src
    cd src
    mkdir github.com
    ```

8. Let's install [``hugo``](https://github.com/gohugoio/hugo) via this command:

    ```bash
    cd ~/go/src/github.com/
    mkdir gohugoio
    cd gohugoio
    git clone https://github.com/gohugoio/hugo.git
    cd hugo
    go install
    ```

9. Confirm our [``hugo``](https://github.com/gohugoio/hugo) application has been installed:

    ```bash
    $GOPATH/bin/hugo
    ```

10. Once confirmed you are ready to continue! The next few sections we will create the folder of our [GitHub](https://github.com) username, my username is ["bartmika"](https://github.com/bartmika) so I am writing with it, please use your own username and follow along:

    ```bash
    cd ~/go/src/github.com
    mkdir bartmika
    cd bartmika
    ```

11. Create our site. My recommendation is use the format "your-domain" + "-hugo" were "your-domain" is the name of your website and "-hugo" indicates this is a [``hugo``](https://github.com/gohugoio/hugo) project. Please note, my website domain is [bartlomiejmika.com](https://bartlomiejmika.com) so I will be writing with "bartlomiejmika-hugo" with the following:

    ```bash
    $GOPATH/bin/hugo new site bartlomiejmika-hugo
    cd bartlomiejmika-hugo
    ```

12. Create your first post.

    ```bash
    $GOPATH/bin/hugo new posts/hello-world.md
    ```

13. Copy the contents from [this sample post file](https://raw.githubusercontent.com/halogenica/beautifulhugo/master/exampleSite/content/posts/2015-01-04-first-post.md) into your file located at ``content/posts/hello-world.md``.


14. Let's confirm our site works by running the ``hugo server`` command which will generate the site and run it. Please note everytime you make a change in your any file the [``hugo``](https://github.com/gohugoio/hugo) server will **hotreload** the site with your latest content. Also note the ``-D`` flag enables **draft** articles which you are writing to be displayed on the site:

      ```bash
      hugo server -D
      ```

15. Open your browsers and visit [http://localhost:1313/](http://localhost:1313/) to view your site. If something appears then congratulations you have setup your site! The next step is you'll need to look through the project and spend time customizing it.

Thank you for reading, if you want to learn how create a website with a beautiful theme then please another article I've written titled ["How to Start a Personal Blog With Hugo a Static Site Generator Written in Go"](/posts/2020/how-to-start-a-personal-blog-with-hugo-a-static-site-generator-written-in-go-part-1/).

Before I finish, I'd like to say if you like the open source [``hugo``](https://github.com/gohugoio/hugo) project please add a **star** to show your support and if you really appreciate it then please consider [sponsoring](https://github.com/sponsors/bep).
