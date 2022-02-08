---
title: "How to Perform RSA Encryption in Javascript (React.js) and Golang"
date: 2022-02-07T00:45:18-05:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "Golang"
- "JavaScript"
- "Security"
- "HOWTO"
- "Code Snippets"
---

![RSA Encryption](/img/2022/02/07/fly-d-4Tu-sIOXeA0-unsplash.jpg)

Do you want your **React.js (web)** frontend app talking to your **Golang** backend server? In this post, I'll explain how I got cross-devices RSA encryption working in JavaScript and Golang.
<!--more-->

# 1. Introduction
To begin, you'll need to familiarize yourself with the basic understanding of **RSA encryption**. I found this article a good starting point and would recommend you read it before proceeding further: [Implementing RSA Encryption and Signing in Golang (With Examples)](https://www.sohamkamani.com/golang/rsa-encryption/). Do not proceed any further until you understand the basics!

Next, when dealing with security for your app, it's a good idea to step back and get an overall bigger picture. I found this article to be a phenomenal resource in helping paint this bigger picture with client-side encryption using cross-platform RSA encryption. Please read this article before proceeding further: [Why & How to build Client-Side Encryption in React.Js and beyond](https://www.linkedin.com/pulse/building-client-side-encryption-reactjs-ricky-s) (archive: [link](https://archive.is/caaan)).

In using RSA encryption for Golang, I found the documentation to be mostly available with no need for third-party packages as the standard package library was good enough; however, this was not the case with JavaScript. I found this link which gives a good summary of the available encryption libraries to use for JavaScript: [JavaScript Crypto Libraries](https://gist.github.com/jo/8619441).

After some research, the library I settled on was [`jsencrypt`](https://github.com/travist/jsencrypt) because of the easy-to-understand documentation. With some experimenting, and thanks to [this stack overflow](https://stackoverflow.com/a/36181645) answer, I was able to figure it out.

# 2. Encrypt in JS, Decrypt with Go

## 2.1. React.js - Client Side Encryption

**Step 1:** Create the react app:

```shell
$ npx create-react-app rsa-frontend
```

**Step 2:** Go into your folder and install our dependencies.

```shell
$ cd rsa-frontend
$ npm install jsencrypt
```

**Step 3:** Next you'll need a *private* and *public* keys. I will copy and paste the ones provided by [`jsencrypt`](https://github.com/travist/jsencrypt) documentation.

Replace your `App.js` code so it will look like this:

```javascript
import { JSEncrypt } from "jsencrypt";


function App() {
    // Start our encryptor.
    var encrypt = new JSEncrypt();

    // Copied from https://github.com/travist/jsencrypt
    var publicKey = `
    -----BEGIN PUBLIC KEY-----
    MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDlOJu6TyygqxfWT7eLtGDwajtN
    FOb9I5XRb6khyfD1Yt3YiCgQWMNW649887VGJiGr/L5i2osbl8C9+WJTeucF+S76
    xFxdU6jE0NQ+Z+zEdhUTooNRaY5nZiu5PgDB0ED/ZKBUSLKL7eibMxZtMlUDHjm4
    gwQco1KRMDSmXSMkDwIDAQAB
    -----END PUBLIC KEY-----`;

    // Copied from https://github.com/travist/jsencrypt
    var privateKey = `
    -----BEGIN RSA PRIVATE KEY-----
    MIICXQIBAAKBgQDlOJu6TyygqxfWT7eLtGDwajtNFOb9I5XRb6khyfD1Yt3YiCgQ
    WMNW649887VGJiGr/L5i2osbl8C9+WJTeucF+S76xFxdU6jE0NQ+Z+zEdhUTooNR
    aY5nZiu5PgDB0ED/ZKBUSLKL7eibMxZtMlUDHjm4gwQco1KRMDSmXSMkDwIDAQAB
    AoGAfY9LpnuWK5Bs50UVep5c93SJdUi82u7yMx4iHFMc/Z2hfenfYEzu+57fI4fv
    xTQ//5DbzRR/XKb8ulNv6+CHyPF31xk7YOBfkGI8qjLoq06V+FyBfDSwL8KbLyeH
    m7KUZnLNQbk8yGLzB3iYKkRHlmUanQGaNMIJziWOkN+N9dECQQD0ONYRNZeuM8zd
    8XJTSdcIX4a3gy3GGCJxOzv16XHxD03GW6UNLmfPwenKu+cdrQeaqEixrCejXdAF
    z/7+BSMpAkEA8EaSOeP5Xr3ZrbiKzi6TGMwHMvC7HdJxaBJbVRfApFrE0/mPwmP5
    rN7QwjrMY+0+AbXcm8mRQyQ1+IGEembsdwJBAN6az8Rv7QnD/YBvi52POIlRSSIM
    V7SwWvSK4WSMnGb1ZBbhgdg57DXaspcwHsFV7hByQ5BvMtIduHcT14ECfcECQATe
    aTgjFnqE/lQ22Rk0eGaYO80cc643BXVGafNfd9fcvwBMnk0iGX0XRsOozVt5Azil
    psLBYuApa66NcVHJpCECQQDTjI2AQhFc1yRnCU/YgDnSpJVm1nASoRUnU8Jfm3Oz
    uku7JUXcVpt08DFSceCEX9unCuMcT72rAQlLpdZir876
    -----END RSA PRIVATE KEY-----`

    // Assign our encryptor to utilize the public key.
    encrypt.setPublicKey(publicKey);

    // Perform our encryption based on our public key - only private key can read it!
    var encrypted = encrypt.encrypt("Hello world!");

    // Decrypt with the private key...
    var decrypt = new JSEncrypt();
    decrypt.setPrivateKey(privateKey);
    var uncrypted = decrypt.decrypt(encrypted);

    return (
        <div>
          <h1>Ciphertext</h1>
          <p>{encrypted}</p>
          <p><b>Copy and paste the above ciphertext into your Golang app</b></p>
          <h1>Plaintext</h1>
          <p>{uncrypted}</p>
        </div>
    );
}

export default App;
```

## 2.2. Golang - Server Side Encryption
**Step 1:** Create your Golang app (change according to your configurations):

```shell
cd ~/go/src/github.com/bartmika
mkdir rsa-backend
cd rsa-backend
go mod init github.com/bartmika/rsa-backend
```

**Step 2:** Create a `private.pem` file in your `github.com/bartmika/rsa-backend` location and fill it with the following content:

```text
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQDlOJu6TyygqxfWT7eLtGDwajtNFOb9I5XRb6khyfD1Yt3YiCgQ
WMNW649887VGJiGr/L5i2osbl8C9+WJTeucF+S76xFxdU6jE0NQ+Z+zEdhUTooNR
aY5nZiu5PgDB0ED/ZKBUSLKL7eibMxZtMlUDHjm4gwQco1KRMDSmXSMkDwIDAQAB
AoGAfY9LpnuWK5Bs50UVep5c93SJdUi82u7yMx4iHFMc/Z2hfenfYEzu+57fI4fv
xTQ//5DbzRR/XKb8ulNv6+CHyPF31xk7YOBfkGI8qjLoq06V+FyBfDSwL8KbLyeH
m7KUZnLNQbk8yGLzB3iYKkRHlmUanQGaNMIJziWOkN+N9dECQQD0ONYRNZeuM8zd
8XJTSdcIX4a3gy3GGCJxOzv16XHxD03GW6UNLmfPwenKu+cdrQeaqEixrCejXdAF
z/7+BSMpAkEA8EaSOeP5Xr3ZrbiKzi6TGMwHMvC7HdJxaBJbVRfApFrE0/mPwmP5
rN7QwjrMY+0+AbXcm8mRQyQ1+IGEembsdwJBAN6az8Rv7QnD/YBvi52POIlRSSIM
V7SwWvSK4WSMnGb1ZBbhgdg57DXaspcwHsFV7hByQ5BvMtIduHcT14ECfcECQATe
aTgjFnqE/lQ22Rk0eGaYO80cc643BXVGafNfd9fcvwBMnk0iGX0XRsOozVt5Azil
psLBYuApa66NcVHJpCECQQDTjI2AQhFc1yRnCU/YgDnSpJVm1nASoRUnU8Jfm3Oz
uku7JUXcVpt08DFSceCEX9unCuMcT72rAQlLpdZir876
-----END RSA PRIVATE KEY-----
```

**Step 3:** Create a `main.go` file in your `github.com/bartmika/rsa-backend` location and fill it with the following content:

```golang
package main

import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/x509"
    "encoding/base64"
    "encoding/pem"
    "errors"
    "io/ioutil"
    "log"
)

// Copied from https://segmentfault.com/q/1010000002505932
func RsaDecrypt(privateKey []byte, ciphertext []byte) ([]byte, error) {
    block, _ := pem.Decode(privateKey)
    if block == nil {
        return nil, errors.New("private key error!")
    }
    priv, err := x509.ParsePKCS1PrivateKey(block.Bytes)
    if err != nil {
        return nil, err
    }
    return rsa.DecryptPKCS1v15(rand.Reader, priv, ciphertext)
}

func main() {
    // Copy the ciphertext from the `react.js` application you built.
    encrypted := "aLP7V4YwfwHJoxl3Tk+cEmvTG0R63cBwE+NtlerUG47m8vnXnKqSFY7QDAcW4+SV8J5Y+eAgOFrKuTxV+iZ5OLq2Rtp1JIHvR8/yQKDiuGeVFWNHaOb8rTSDfW/RiL2FpB6ZvCcpFUWUr86l13f7yKNdxwMcZpZcSD0VXfNsS0I="

    // Special thanks to: https://stackoverflow.com/a/53077471
    privateKey, _ := ioutil.ReadFile("private.pem")
    cipherText, _ := base64.StdEncoding.DecodeString(encrypted)
    originText, _ := RsaDecrypt(privateKey, []byte(cipherText))
    log.Println(string(originText))
    // OUTPUT:
    // Hello world!
}
```

# 3 Encrypt with Go, Decrypt in JS

## 3.1. Encrypt with Go

```golang
package main

import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/x509"
    "encoding/base64"
    "encoding/pem"
    "errors"
    "fmt"
    "io/ioutil"
    "log"
)

// Copied from https://segmentfault.com/q/1010000002505932
func RsaEncrypt(publicKey []byte, origData []byte) ([]byte, error) {
    block, _ := pem.Decode(publicKey)
    if block == nil {
        return nil, errors.New("public key error")
    }
    pubInterface, err := x509.ParsePKIXPublicKey(block.Bytes)
    if err != nil {
        return nil, err
    }
    pub := pubInterface.(*rsa.PublicKey)
    return rsa.EncryptPKCS1v15(rand.Reader, pub, origData)
}

func main() {
    // Here is the text we want to encrypt.
    plaintext := "Hello world!"

    // Special thanks to: https://stackoverflow.com/a/53077471
    publicKey, err := ioutil.ReadFile("public.pem")
    if err != nil {
        log.Fatal(err)
    }

    encrypted, _ := RsaEncrypt(publicKey, []byte(plaintext))
    if err != nil {
        log.Fatal(err)
    }

    // Generate a string we can copy and paste into our JavaScript code.
    encoding := base64.StdEncoding.EncodeToString([]byte(encrypted))

    fmt.Println(encoding)
    // OUTPUT:
    // gfON9I1jF6ZJk6VTOG3iuoDo60FCVAzvJXW1ysYy3JanZYvLDm7zmYuyWVSvo3NO2ZiLGQwSNm0l/L2IzodQMGZH+sBzuN0AQugfX1O9A685qn9/PwWSXg4gRYaBB9bQXiWDNHcQMFNTWtXxYGrNCrzY53lX4pzTGy1mD//Fa2E=
}
```

## 3.2. Decrypt in JS

```javascript
import { JSEncrypt } from "jsencrypt";


function App() {
    // Copied from https://github.com/travist/jsencrypt
    var privateKey = `
    -----BEGIN RSA PRIVATE KEY-----
    MIICXQIBAAKBgQDlOJu6TyygqxfWT7eLtGDwajtNFOb9I5XRb6khyfD1Yt3YiCgQ
    WMNW649887VGJiGr/L5i2osbl8C9+WJTeucF+S76xFxdU6jE0NQ+Z+zEdhUTooNR
    aY5nZiu5PgDB0ED/ZKBUSLKL7eibMxZtMlUDHjm4gwQco1KRMDSmXSMkDwIDAQAB
    AoGAfY9LpnuWK5Bs50UVep5c93SJdUi82u7yMx4iHFMc/Z2hfenfYEzu+57fI4fv
    xTQ//5DbzRR/XKb8ulNv6+CHyPF31xk7YOBfkGI8qjLoq06V+FyBfDSwL8KbLyeH
    m7KUZnLNQbk8yGLzB3iYKkRHlmUanQGaNMIJziWOkN+N9dECQQD0ONYRNZeuM8zd
    8XJTSdcIX4a3gy3GGCJxOzv16XHxD03GW6UNLmfPwenKu+cdrQeaqEixrCejXdAF
    z/7+BSMpAkEA8EaSOeP5Xr3ZrbiKzi6TGMwHMvC7HdJxaBJbVRfApFrE0/mPwmP5
    rN7QwjrMY+0+AbXcm8mRQyQ1+IGEembsdwJBAN6az8Rv7QnD/YBvi52POIlRSSIM
    V7SwWvSK4WSMnGb1ZBbhgdg57DXaspcwHsFV7hByQ5BvMtIduHcT14ECfcECQATe
    aTgjFnqE/lQ22Rk0eGaYO80cc643BXVGafNfd9fcvwBMnk0iGX0XRsOozVt5Azil
    psLBYuApa66NcVHJpCECQQDTjI2AQhFc1yRnCU/YgDnSpJVm1nASoRUnU8Jfm3Oz
    uku7JUXcVpt08DFSceCEX9unCuMcT72rAQlLpdZir876
    -----END RSA PRIVATE KEY-----`

    // Paste the output from the Golang app.
    var encrypted = "gfON9I1jF6ZJk6VTOG3iuoDo60FCVAzvJXW1ysYy3JanZYvLDm7zmYuyWVSvo3NO2ZiLGQwSNm0l/L2IzodQMGZH+sBzuN0AQugfX1O9A685qn9/PwWSXg4gRYaBB9bQXiWDNHcQMFNTWtXxYGrNCrzY53lX4pzTGy1mD//Fa2E=";

    // Decrypt with the private key...
    var decrypt = new JSEncrypt();
    decrypt.setPrivateKey(privateKey);
    var uncrypted = decrypt.decrypt(encrypted);

    return (
        <div>
          <h1>Plaintext</h1>
          <p>{uncrypted}</p>
        </div>
    );
}

export default App;
```

# 4. Generate Private Public Keys
If you are building an application which needs to generate RSA key pairs on the client side using JavaScript or server side in Golang, the follow sections contain code you can use.

## 4.1. Client Side with JavaScript

```javascript
import { JSEncrypt } from "jsencrypt";


function App() {
    // Start our encryptor.
    var encrypt = new JSEncrypt();

    // Generate a RSA key pair using the `JSEncrypt` library.
    var crypt = new JSEncrypt({default_key_size: 2048});
    var PublicPrivateKey = {
        PublicKey: crypt.getPublicKey(),
        PrivateKey:crypt.getPrivateKey()
    };

    console.log(PublicPrivateKey);

    // PUBLIC KEY
    var publicKey = PublicPrivateKey.PublicKey;

    console.log(publicKey);
    /*
    EXAMPLE OUTPUT:

    -----BEGIN PUBLIC KEY-----
    MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAj7+7e8PgCpjF2zaNbnd6
    l903kGIVJqh4VZS2m/BrCctNOM/y/2r6BRBUwbtC+Dtz7J8VD1kK8SOuzKyozwc0
    z50TEFIKpJjX2lPKdpfXWHnbsk7v+5o1fQmmMiQSeZ/+NQ2Nhecj7krdk668cI49
    b9IhVinJxFqa1XG13o/DoOAVseCujxTk/XdztBw1wnWa9zTozvi/utOOhh/Gqcf9
    VbkaD0Eh01HIMNVLl3o15ysREkPHvtu/G+hffW8pacB7rGa1+6fj5RpNw11ShxVN
    01MF+rYz+t2tR3LjYOGJ1CqdfmyJYo6oFyb4ah8OsGnk+bHrIyWoBvpUKBj64JEZ
    nwIDAQAB
    -----END PUBLIC KEY-----
    */

    // PRIVATE KEY
    var privateKey = PublicPrivateKey.PrivateKey;

    console.log(privateKey);
    /*
    EXAMPLE OUTPUT:

    -----BEGIN RSA PRIVATE KEY-----
    MIIEpAIBAAKCAQEAj7+7e8PgCpjF2zaNbnd6l903kGIVJqh4VZS2m/BrCctNOM/y
    /2r6BRBUwbtC+Dtz7J8VD1kK8SOuzKyozwc0z50TEFIKpJjX2lPKdpfXWHnbsk7v
    +5o1fQmmMiQSeZ/+NQ2Nhecj7krdk668cI49b9IhVinJxFqa1XG13o/DoOAVseCu
    jxTk/XdztBw1wnWa9zTozvi/utOOhh/Gqcf9VbkaD0Eh01HIMNVLl3o15ysREkPH
    vtu/G+hffW8pacB7rGa1+6fj5RpNw11ShxVN01MF+rYz+t2tR3LjYOGJ1CqdfmyJ
    Yo6oFyb4ah8OsGnk+bHrIyWoBvpUKBj64JEZnwIDAQABAoIBACd6EjTlEAwY9I1F
    KAYkTciS+gVuyjw5nAJ0usmMdvjTmjt18FfwuwTU/VHO6Y9eVHGxJol2fKjIkeKn
    sBxa8Efr7SZYQY/+YZkV1c5H2N31aT5Iq2M/cF0MX1X5zhEUvS04sZsKZTW13bAH
    Fr0acwjYfks5Yq3H7Cmd9sJOXP064ta6UvtICLdyyHvq3JQPWKnrwfVvsnSmHfLV
    QKrcALmIYZppzeJINiV45QsDlQXLj+Twp7oNDJ1RJ4H51TLBbmeEwygHDXqH7NXV
    0lLxgNnNlt2yvxFQ7G7QVcIntbQNILrq0DQDfJ/2YUlxWt9WbfbOD0goxzUzmzQf
    vzXsGDECgYEA2Inhhbm+lx9WLGGWF9n13PRd10ZJatD7wwzMit6kNy+1p7Z7R+Z5
    MbtPt0pUBNfvfz2SfMojv+Yhwavt5eP87utHb2ZxgV1yCCRN2ujQLHRcW+vveSjD
    fwoRjaXEiGKnikIizZcUj9RE22MHB0CqNLW2xjAY02Gw4ImBTrKU7tUCgYEAqfID
    24SzwSxW7ocE309zstdzlsP5fwLP0iyOcK0Po5dch0XdFrdrir4/XQAh89IXKUAT
    Dsd5253tAk7E5T0g0CIcpiCLvBbB2vmDs7BO61SS6t8ahoh64Kq4zZombH7Bu+3d
    Tcazv716Z3L8i09MzHCHlsPiFTjVRA8W18Qc6KMCgYEAoYoDB1rxNxY2mDdY3IRK
    qcJXe3DA9oHfP7x9nx/HDDB4aRx2TcY/JX2iU4+MrGxXC+poLNYz40YQasYTXMw/
    dhFpok6fYK3QkwhaWHQUUQWhnSWe6hkh9tUREUXYHxLSAA+knREXUtE9aRkwNhXk
    pBvntWROMOuRI4ERSR9qgd0CgYEAqNM8k8mLjP6QSZsl8vWJ+YNhV8fNxigz7hXH
    VxYFMD3AdL2pudRy6DzA05G7KO1vhtIZXJg7bTnA5ob7wMNuInWQwlQYnLx6zh8L
    f+lJLS0yWlNSlY1ljGTs+4sEWsm9igTt0ULw9Cy2OaiYS4h2wa2UdOiZYv23l0nq
    JmSzV0MCgYAsCccSf7Btieny/hJQ+nrgdGtZxOCwAFFtfV7WwYCyGszOG0lL2wrk
    skpr/uvmxZXzVLt4IBJJ0CSDoVHNJ/zrB+tJl+h5iO170WfrGAtnstj6fcRm8ovA
    cXAYLGdhRQYgznFmQwrCZSd7ZhrFXGi0rb55lm/bX8zc5XIN6MfDZg==
    -----END RSA PRIVATE KEY-----
    */

    // Assign our encryptor to utilize the public key.
    encrypt.setPublicKey(publicKey);

    // Perform our encrypt based on our public key - only private key can read it!
    var encrypted = encrypt.encrypt("Hello world!");

    // Decrypt with the private key...
    var decrypt = new JSEncrypt();
    decrypt.setPrivateKey(privateKey);
    var uncrypted = decrypt.decrypt(encrypted);

    return (
        <div>
          <h1>Ciphertext</h1>
          <p>{encrypted}</p>
          <p><b>Copy and paste the above ciphertext into your Golang app</b></p>
          <h1>Plaintext</h1>
          <p>{uncrypted}</p>
        </div>
    );
}

export default App;
```

## 4.2. Server Side in Golang

```golang
package main

import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/x509"
    "encoding/pem"
    "os"
)

// Copied from https://stackoverflow.com/a/53077471
func GenRsaKey(bits int) error {
    privateKey, err := rsa.GenerateKey(rand.Reader, bits)
    if err != nil {
        return err
    }
    derStream := x509.MarshalPKCS1PrivateKey(privateKey)
    block := &pem.Block{
        Type:  "privete key",
        Bytes: derStream,
    }
    file, err := os.Create("private.pem")
    if err != nil {
        return err
    }
    err = pem.Encode(file, block)
    if err != nil {
        return err
    }
    publicKey := &privateKey.PublicKey
    derPkix, err := x509.MarshalPKIXPublicKey(publicKey)
    if err != nil {
        return err
    }
    block = &pem.Block{
        Type:  "public key",
        Bytes: derPkix,
    }
    file, err = os.Create("public.pem")
    if err != nil {
        return err
    }
    err = pem.Encode(file, block)
    if err != nil {
        return err
    }
    return nil
}

func main() {
    // Generate our RSA key pairs into `private.pem` and `private.pem`.
    GenRsaKey(2048)
}
```

## 4.3 Command Line

```
openssl genrsa -out key.pem
openssl rsa -in key.pem -pubout > pub.pem
```


# Notes:
* Special thanks to [**apayan** from HackerNews](https://news.ycombinator.com/user?id=apayan) whom posted the [Show HN: End-to-end encrypted location sharing service like Google Latitude (zood.xyz)](https://news.ycombinator.com/item?id=25347915) post that got me super interested in end-to-end encryption that inspired me to write this article.
* Cover photo by [FLY:D](https://unsplash.com/photos/4Tu-sIOXeA0) on [Unsplash](https://unsplash.com/photos/4Tu-sIOXeA0).
