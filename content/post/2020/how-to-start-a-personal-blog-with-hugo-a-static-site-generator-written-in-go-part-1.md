---
title: "How to Start a Personal Blog With Hugo a Static Site Generator Written in Go (Part 1)"
subtitle: "Let's build a static website from scratch"
date: 2020-09-07T15:35:07-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "development"
tags:
- "golang"
- "static site generator"
- "hugo"
- "static site"
---

The purpose of this article is to help you setup a personal blog as quickly as possible. These are the instructions and notes I've written down when setting up my site that I'd like to share.

<!--more-->

## Assumption
Before I begin, I am assuming you are an intermediate computer user and that you are familiar with the following technologies:

* Linux Commands
* Git / Github
* HTML

If you are an absolute beginning this article might be a challenge for you.

## What is a static site generator (SSG)?

Please read [this article](https://www.netlify.com/blog/2020/04/14/what-is-a-static-site-generator-and-3-ways-to-find-the-best-one/) to get an understanding of the **theory**. Once you finish read, continue with this article to get the **application**.

## Pre-requisites

### Install Golang

If you haven't already installed **Golang** begin by installing [Golang](https://golang.org/doc/install) onto your computer.

### Install Hugo

Please go over to the developer website for Hugo and follow along in the [installation instructions](https://gohugo.io/getting-started/installing/) page to get Hugo setup on your machine.

## Instructions
Once your prerequisites have been met, continue with instructions here.

1. Open up your favourite **terminal** and start by going into your directory where you want work from.

   We will be using the folder structure consistent with [GitHub](https://github.com) and **Golang** as follows:

      ```bash
      cd $HOME/go/src/github.com/bartmika
      ```

2. Next use ``hugo`` command to create our site. I chose the ``bartlomiejmika`` prefix and ``-hugo`` suffix to structure the name. Please replace the prefix with your websites domain name (ex: johnsmith):

      ```bash
      $ hugo new site bartlomiejmika-hugo
      ```

3. Go into the directory and initialize git repository.
      ```bash
      $ cd bartlomiejmika-hugo
      $ git init
      ```

4. Pick a theme! This is the best part, Hugo has a lot of different themes choose from. Please go to [this link](https://themes.gohugo.io/) and pick a theme you like. For this blog post I will be choosing [beautifulhugo](https://robertheaton.com/a-blogging-style-guide-vol-2/). Also please note that some themes might have different installation instructions so please follow them. Here is the instructions for the theme we selected:

      ```bash
      $ cd themes
      $ git submodule add https://github.com/halogenica/beautifulhugo.git beautifulhugo
      ```

5. Create your first blog post!

      ```bash
      $ cd ../
      $ hugo new posts/my-first-post.md
      ```

6. Copy the contents from [this sample post file](https://raw.githubusercontent.com/halogenica/beautifulhugo/master/exampleSite/content/post/2015-01-04-first-post.md) into your file located at ``content/posts/my-first-post.md``.

7. Copy the contents from [this sample configuration file](https://raw.githubusercontent.com/halogenica/beautifulhugo/master/exampleSite/config.toml) into your config file located at ``config.toml``.

8. Let's confirm our site works by running the ``hugo server`` command which will generate the site and run it. Please note everytime you make a change in your any file the ``hugo`` server will **hotreload** the site with your latest content. Also note the ``-D`` flag enables **draft** articles which you are writing to be displayed on the site:

      ```bash
      hugo server -D
      ```

9. Open your browsers and visit [http://localhost:1313/](http://localhost:1313/) to view your site. If something appears then congratulations you have setup your site! The next step is you'll need to look through the project and spend time customizing it.

10. Create a new repo in github and call it ``bartlomiejmika-hugo``, copy the HTTPS address ``https://github.com/bartmika/bartlomiejmika-hugo.git`` (see below) ![Custom Domain Setting](/img/2020/09/2-1.png)

    Add it as the remote git server for this project with the following command.

      ```bash
      $ git remote add origin https://github.com/bartmika/bartlomiejmika-hugo.git
      ```

11. Commit all your code and you should be good!

      ```bash
      $ git add --all;
      $ git commit -m 'Initial commit';
      $ git push origin master;
      ```

12. You are finished. This means you can now start writing blogs on your local machine for now. When you are ready to publish your site to the internet, please follow the instructions in **[part 2](/posts/2020/how-to-start-a-personal-blog-with-hugo-a-static-site-generator-written-in-go-part-2/)** of this series.

Thank you and have a great day!
