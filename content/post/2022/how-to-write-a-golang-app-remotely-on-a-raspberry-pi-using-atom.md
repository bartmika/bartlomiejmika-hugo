---
title: "How to Write a Golang App Remotely on a Raspberry Pi Using Atom"
date: 2022-03-13T00:39:16-05:00
draft: true
author: "Bartlomiej Mika"
categories:
- "Internet of Things"
tags:
- "Golang"
- "Linux"
- "HOWTO"
- "Raspberry Pi"
---

![Raspberry Pi](/img/2022/03/13/harrison-broadbent-c3YpscwJb04-unsplash.jpg)
Photo by [Harrison Broadbent](https://unsplash.com/photos/c3YpscwJb04) on [Unsplash](https://unsplash.com/s/photos/raspberry-pi).

Wouldn't it be cool to remotely program a Golang app on a Raspberry Pi? In this post, I write how to do that if you are using Atom.

## **TL;DR;**
> 1. Setup a Raspberry Pi Device
> 2. Setup Golang on that Device
> 3. Install `remote-ftp` on your Machine
> 4. Connect from your Atom
> 5. Write a Golang Application

<!--more-->

*Please note, before we begin, I am using a Mac so this tutorial is assuming you have either a Mac or Linux computer. If you are using Windows, I recommend using a Linux emulator like Cygwin*.

## 1. Setup a Raspberry Pi Device
### Setup OS
Let's start by downloading the [Raspberry Pi OS](https://www.raspberrypi.com/software/) onto your computer. Next follow this 45 second video explaining how to setup the OS onto a flash drive:

{{< youtube ntaXWS8Lk34 >}}

### Headless Configuration
Once you have the card created, please follow this article [How to Set Up a Headless Raspberry Pi, Without Ever Attaching a Monitor](https://www.tomshardware.com/reviews/raspberry-pi-headless-setup-how-to,6028.html#:~:text=Write%20an%20empty%20text%20file,command%20line%20from%20your%20PC.) to get everything configured so you won't have to manually SSH into system and manually configure.

### How to Find your IP Address
According to the [Raspberry Pi docs](https://www.raspberrypi.com/documentation/computers/remote-access.html#remote-access), we can use the existing `multicast DNS` to get the IP address on our machine by running the following command:

```
ping raspberrypi.local
```

This command will return the IP address of the device! In our example, we will use **192.168.1.138**.

### SSH Into your Device

1. On your machine, please run:

    ```shell
    ssh -l pi 192.168.1.138
    ```

2. When asked about authenticity of the device, please select yes.

3. When asked about password, please enter `raspberry`. You should now be in!

### Passwordless SSH Access

This is an essential step, please read the [Raspberry Pi](https://www.raspberrypi.com/documentation/computers/remote-access.html#passwordless-ssh-access) documentation and follow along with their instructions.

## 2. Setup Golang on that Device

1. Let's SSH into your Raspberry Pi. In a new terminal windows, please run the following command:

    ```shell
    ssh -l pi 192.168.1.138
    ```

2. Next check to see the latest version of Golang. To get the latest version, browse the [Go download page](https://go.dev/dl/) and look for the latest version. Search for **"ARM v6 version"** for the Raspberry Pi.

    Whichever archive you choose, use wget to download it:

    ```shell
    mkdir ~/src
    cd ~/src
    wget https://dl.google.com/go/go1.17.8.linux-arm64.tar.gz
    ```

    *Please note that if you are using a 32bit OS then download the `armv6l`, else if you are using 64bit OS then download `armv64`.*

3. Now you’ll want to extract the package into your local folder:

    ```shell
    sudo tar -C /usr/local -xzf ~/src/go1.17.8.linux-arm64.tar.gz
    ```

4. Clean up.

    ```shell
    rm go1.17.8.linux-arm64.tar.gz
    ```

5. Open the following:

    ```
    vi ~/.profile
    ```

6. And append the following:

    ```
    PATH=$PATH:/usr/local/go/bin
    GOPATH=$HOME/go
    ```

7. Update your shell with your changes:

    ```
    source ~/.profile
    ```

8. Finally run the following to confirm that `go` works:

    ```
    go version
    ```

9. And the output should be:

    ```
    go version go1.17.8 linux/arm64
    ```

## 3. Install `remote-ftp` on your Machine
We are going to get your Atom to connect with the Raspberry Pi by using `remote-ftp`.

1. To start go to the following: **Atom > Preferences > Install**

2. Search for `remote-ftp` and click install. When finished you should see:

    ![remote-ftp](/img/2022/03/13/a1.png)

3. Next click the **Packages** and select the **remote-ftp**.

    ![remote-ftp](/img/2022/03/13/a2.png)

## 4. Connect from your Atom

1. Click on **Remote** and you should see:

    ![remote-ftp](/img/2022/03/13/a3.png)

2. Afterwords click **Edit Configuration** and you will see a blank file. You need to fill out the document. Check out the [developers docs](https://github.com/icetee/remote-ftp) to read and learn more. I've modified the following SFTP configuration for my purposes:

    ```json
    {
        "protocol": "sftp",
        "host": "192.168.1.138", // string - Hostname or IP address of the server. Default: 'localhost'
        "port": 22, // integer - Port number of the server. Default: 22
        "user": "pi", // string - Username for authentication. Default: (none)
        "pass": "rasbperry", // string - Password for password-based user authentication. Default: (none)
        "promptForPass": false, // boolean - Set to true for enable password/passphrase dialog. This will prevent from using cleartext password/passphrase in this config. Default: false
        "remote": "/", // try to use absolute paths starting with /
        "agent": "", // string - Path to ssh-agent's UNIX socket for ssh-agent-based user authentication. Linux/Mac users can set "env" as a value to use env SSH_AUTH_SOCK variable. Windows users: set to 'pageant' for authenticating with Pageant or (actual) path to a cygwin "UNIX socket." Default: (none)
        "privatekey": "/Users/bmika/.ssh/id_rsa", // string - Absolute path to the private key file (in OpenSSH format). Default: (none)
        "passphrase": "", // string - For an encrypted private key, this is the passphrase used to decrypt it. Default: (none)
        "hosthash": "", // string - 'md5' or 'sha1'. The host's key is hashed using this method and passed to the hostVerifier function. Default: (none)
        "ignorehost": true,
        "connTimeout": 10000, // integer - How long (in milliseconds) to wait for the SSH handshake to complete. Default: 10000
        "keepalive": 10000, // integer - How often (in milliseconds) to send SSH-level keepalive packets to the server (in a similar way as OpenSSH's ServerAliveInterval config option). Set to 0 to disable. Default: 10000
        "keyboardInteractive": false, // boolean - Set to true for enable verifyCode dialog. Keyboard interaction authentication mechanism. For example using Google Authentication (Multi factor)
        "keyboardInteractiveForPass": false, // boolean - Set to true for enable keyboard interaction and use pass options for password. No open dialog.
        "watch":[ // array - Paths to files, directories, or glob patterns that are watched and when edited outside of the atom editor are uploaded. Default : []
            "dist/stylesheets/main.css", // reference from the root of the project.
            "dist/stylesheets/",
            "dist/stylesheets/*.css"
        ],
        "watchTimeout":500, // integer - The duration ( in milliseconds ) from when the file was last changed for the upload to begin.
        "filePermissions":"0644" // string - Permissions for uploaded files. WARNING: if this option is set, previously set permissions on the remote are overwritten!
    }
    ```

3. The following changes you'll need to do:

    * `host` - This is the IP address of your Raspberry Pi, we replaced with our value `192.168.1.138`.
    * `user` - Change to the username of admin on the Raspberry Pi, we change to `pi`.
    * `pass` - Change to the default password `raspberry`.
    * `privatekey` - Set the location of your private key. For me, I am using a Mac and this is the location of my private key via the value of `/Users/bmika/.ssh/id_rsa`.

4. Finally when you finish, save the file.

5. When you are ready, click **Connect** and if everything works you should see the following:

    ![remote-ftp](/img/2022/03/13/a4.png)

## 5. Write a Golang Application

1. While connected, open up to the `/home/pi` folder:

    ![remote-go-app](/img/2022/03/13/b1.png)

2. Then right click and click **Add File**:

    ![remote-go-app](/img/2022/03/13/b2.png)

3. When you click it will ask you to input the file name.

    ![remote-go-app](/img/2022/03/13/b3.png)

4. Go ahead and let's create our **hello.go** file. Afterwards enter a simple program. You should see this.

    ![remote-go-app](/img/2022/03/13/b4.png)

5. Finally in your open terminal (outside of Atom), run the following code and you should see the code running!

    ![remote-go-app](/img/2022/03/13/b5.png)

## Bonus Points: Run Terminal inside Atom
1. Go to **Atom > Preferences > Install** and search for **platformio-ide-terminal**.

2. We are going to get your Atom to connect with the Raspberry Pi by using `remote-ftp`. To start go to the following:

    ![remote-go-app-on-pi](/img/2022/03/13/c1.png)

3. Next click the **Packages** and select the **platformio-ide-terminal**.

    ![remote-go-app-on-pi](/img/2022/03/13/c2.png)

4. A terminal will load up at the bottom of the screen, enter the command to SSH into your Raspberry Pi:

    ```
    ssh -l pi 192.168.1.138
    ```

5. This is it! You should see:

    ![remote-go-app-on-pi](/img/2022/03/13/c3.png)

6. While logged in, run the Golang app:

    ```
    go run hello.go
    ```

7. And you should see something like this:

    ![remote-go-app-on-pi](/img/2022/03/13/c4.png)

8. Finally go ahead and run

## Conclusion

Atom, and its plugins are all you need to start remote development with Golang.

Happy coding!
