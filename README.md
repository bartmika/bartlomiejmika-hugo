# bartlomiejmika-hugo
The source code of the [bartlomiejmika.com](https://bartlomiejmika.com) website written using [Hugo](https://gohugo.io/), a static content generator written in Go.

## Installation

To begin, clone this repository

```
git clone https://github.com/bartmika/bartlomiejmika-hugo.git
```

Once cloned, you'll want to install any dependencies you have. Here is for MacOS:

```
brew install hugo
```

Now to start the server locally, run the following.

```
hugo serve -D
```

In your browser, enter the path [http://localhost:1313](http://localhost:1313).

## Usage

### New Post

Here are a few examples:

```
hugo new posts/my-first-post.md
hugo new posts/2020/07/24/how-to-setup-peercoin-on-raspberry-pi-for-headless-minting.md
hugo new posts/2020/08/20/how-to-use-surge-sh-in-multi-client-projects.md
hugo new posts/2020/08/28/lets-build-a-basic-react-native-app.md
hugo new posts/2020/how-to-start-a-personal-blog-with-hugo-a-static-site-generator-written-in-go.md
hugo new posts/2020/how-to-install-hugo-from-git-bash-for-windows.md
hugo new posts/2020/what-are-some-best-practises-in-software-engineering.md
```

### Deployment

```
hugo --minify
cd public
git add --all;
git commit -m 'Latest commit...';
git push origin master;
```
