---
title: "How to Start a Personal Blog With Hugo a Static Site Generator Written in Go (Part 2)"
subtitle: "Let's deploy our website to the internet with GitHub Pages and Namecheap DNS"
date: 2020-09-07T15:35:08-04:00
draft: false
author: "Bartlomiej Mika"
cover:
    image: "/img/2020/09/07-hugo-banner.jpeg"
    alt: "Hugo"
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---

The purpose of this article is to help you deploy your website to the internet.

<!--more-->

Before I begin, I am assuming the following:

* Either you have finished reading [part 1 of 2](/posts/2020/how-to-start-a-personal-blog-with-hugo-a-static-site-generator-written-in-go-part-1/) in this series.
* Or you have a Hugo site on your machine and you'd like to learn how to deploy the site

What you need for this article:

* A registered domain name
* A GitHub account

## Setup DNS

The first thing we need to do is to have the DNS information from our site point to GitHub's. Depending on your domain name service provider, you will have to read the details. If you have **Namecheap** then you read [this article](https://www.namecheap.com/support/knowledgebase/article.aspx/9645/2208/how-do-i-link-my-domain-to-github-pages).

## Setup GitHub Pages

### bartmika.github.io repo
You will need to create a git repository dedicated to your sites static content. Click the Create repository button and name it **bartmika.github.io**.

Go into your repository's settings and set the **Custom Domain** field to your domain address:

![Custom Domain Setting](/img/2020/09/3-1.png)

### bartlomiejmika-hugo repo
Great, now what we want to do is link it with our **bartlomiejmika-hugo** repository as a submodule. Once connected we will use ``hugo`` to build our static website and save it into our **bartmika.github.io** repository where GitHub will publish it onto the internet; furthermore, because we have our domain name host mapped to GitHub, our website will resolve. To begin, inside your **bartlomiejmika-hugo** repository please write:

```
cd $HOME/go/src/github.com/bartmika/bartlomiejmika-hugo
git submodule add https://github.com/bartmika/bartmika.github.io.git public
```

Create file called ``CNAME`` and populate it with your domain name:

```
echo "bartlomiejmika.com" > CNAME
```

Save your latest changes

```bash
$ git add --all;
$ git commit -m 'Created sub-module with the `bartmika.github.io` repo.';
$ git push origin master;
```

Finally, how do we generate our static website using ``hugo``? Simple, run the following command:

```bash
$ hugo --minify
```

And it's built! We are ready to go into our sub-module, commit our output and submit it to GitHub, where once it's submitted will deploy our site content onto the internet! Exciting! Here's the code:

```bash
$ cd public
$ git add --all;
$ git commit -m 'Initial commit of website content.';
$ git push origin master;
```

To confirm, please wait 24 hours for the DNS information to propagate but once it's finished you can see your site up when you enter your domain [bartlomiejmika.com](https://bartlomiejmika.com).
