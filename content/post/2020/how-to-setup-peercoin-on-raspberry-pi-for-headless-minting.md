---
title: "How to Setup Peercoin on Raspberry Pi for Headless Minting"
date: 2020-07-26T12:15:15-04:00
draft: False
author: "Bartlomiej Mika"
categories:
- "Developer Operations"
tags:
- "Cryptocurrency"
- "Peercoin"
- "Raspberry Pi"
- "HOWTO"
- "Tutorial"
---

Would you believe you can participate in cryptocurrency production using a simple Raspberry Pi computer? No need for powerful ASIC miners! Peercoin is an interesting altcoin that has **minting** capability built-in, and minting is the ability to create new coins from the ones you already have. In this tutorial, you'll learn how to set up a dedicated Raspberry Pi computer to mint the Peercoins in your wallet.

<!--more-->

## Assumptions

This guide assumes the following:
* You've downloaded [Raspbian Lite ISO image](https://www.raspberrypi.org/downloads/raspbian/).
* You've burned the image on the SDCard and you have some experience working with the Raspberry Pi computer
* You've installed ``Peercoin`` on your computer and you want to transfer your ``wallet.dat`` file to your Raspberry Pi computer.
* You already have some Peercoins in your wallet.

This guide has been confirmed working on:
* Pi 2 Model B v1.1	1GB	a21041 (Embest, China) [*](https://www.raspberrypi-spy.co.uk/2012/09/checking-your-raspberry-pi-board-version/)
* [Peercoin Daemon version v0.8.1.0](https://github.com/peercoin/peercoin/releases)

## Instructions

### (Optional) Changing the Host on your Pi

The following section was taken from [this link](https://www.howtogeek.com/167195/how-to-change-your-raspberry-pi-or-other-linux-devices-hostname/).

1. Open up this file.

    ```
    sudo nano /etc/hosts
    ```

2. Leave all of the entries alone except for the very last entry labeled 127.0.1.1 with the hostname ``raspberrypi``. This is the only line you want to edit. Replace “raspberrypi” with whatever hostname you desire. I named my computer ``peercoin-pi``.

3. Back at the terminal, type the following command to open the hostname file:

    ```
    sudo nano /etc/hostname
    ```

4. Replace the default “raspberrypi” with the same hostname you put in the previous step (e.g. ``peercoin-pi``).

5. Commit our changes.

    ```
    sudo /etc/init.d/hostname.sh
    ```

6. Reboot the computer.

    ```
    sudo reboot
    ```


### Improve security

1. Install firewall.

    ```
    sudo apt-get install git ufw
    ```

2. Enable firewall and allow only ssh access

    ```
    sudo ufw allow 22
    sudo ufw allow 9901
    sudo ufw --force enable
    sudo ufw status
    ```

3. To add more security, read [this link](https://www.raspberrypi.org/documentation/configuration/security.md).


### Swap

The following section was inspired by [this article](http://raspberrypimaker.com/adding-swap-to-the-raspberrypi/). Please be warned that running the following code can shorten the life-span of your SDCard.

1. Configure using the following.

    ```
    sudo dd if=/dev/zero of=/swapfile bs=64M count=16
    sudo mkswap /swapfile
    sudo swapon /swapfile
    ```

2. (Optional) Stop our ``swap`` memory and remove it.

    ```
    sudo swapoff /swapfile
    sudo rm -f /swapfile
    ```

3. (Optional) Make the swap memory permanent on every boot.

    ```
    sudo vi /etc/fstab
    ```

4. (Optional) Append the contents:

    ```
    /swapfile none swap defaults 0 0
    ```


### Peercoin
#### Build Executable
Please note these instructions where taken from [this section](https://github.com/peercoin/peercoin/blob/master/doc/build-unix.md). The following instructions are to be run on your raspberry pi.

1. Update the libraries.

    ```
    sudo apt-get update
    ```

2. Install build requirements:

    ```
    sudo apt-get install build-essential libtool autotools-dev automake pkg-config libssl-dev libevent-dev bsdmainutils python3
    ```

3. Install more requirements

    ```
    sudo apt-get install libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev
    ```

4. (Optional) If above command doesn't work, you can install all boost development packages with:

    ```
    sudo apt-get install libboost-all-dev
    ```

5. According to [this link](https://github.com/peercoin/peercoin/blob/master/doc/build-unix.md#arm-cross-compilation), we are compiling on ``arm`` processor so we need to run:

    ```
    sudo apt-get install g++-arm-linux-gnueabihf curl
    ```

6. Install ``git`` because it’s not installed.

    ```
    sudo apt-get install git
    ```

7. Clone [Peercoin](https://github.com/peercoin/peercoin) to our home directory.

    ```
    cd /home/pi
    git clone https://github.com/ppcoin/ppcoin.git
    ``

6. To build our dependencies for ``arm``, according to [this link](https://github.com/peercoin/peercoin/blob/master/doc/build-unix.md#arm-cross-compilation), then run the following:

    ```
    cd depends
    make HOST=arm-linux-gnueabihf NO_QT=1
    cd ..
    ./configure --prefix=$PWD/depends/arm-linux-gnueabihf --enable-glibc-back-compat --enable-reduce-exports LDFLAGS=-static-libstdc++
    make
    ```

6. To build our executable for ``arm``, according to [this link](https://github.com/peercoin/peercoin/blob/master/doc/build-unix.md#arm-cross-compilation), then run the following:

    ```
    ./autogen.sh
    ./configure
    make
    make install # optional
    ```

#### Setup and Run Executable

1. Upload the ``wallet.dat`` file on your computer to your raspberry pi.


    ```
    rsync -avz ~/Desktop/wallet.dat pi@192.168.1.10:/home/pi/.peercoin/wallets/wallet.dat
    ```

2. Run the following code to confirm the ``Peercoin`` daemon starts running in the background.

    ```
    cd /home/pi/ppcoin/src
    ./peercoind -listen=0 -daemon -server
    ```

3. If the daemon loads up then you have successfully built the executable.

#### (Optional) Run ```peercoind``` in background
This section had assistance from [this link](https://raspi.tv/2012/using-screen-with-raspberry-pi-to-avoid-leaving-ssh-sessions-open).

1. Install ``screen``.

    ```
    sudo apt-get install screen
    ```

2. Open up our screen session.

    ```
    screen bash
    ```

3. Start our session.

    ```
     cd /home/pi/ppcoin/src
    ./peercoind -listen=0 -daemon -server
    ```

4. Detach the screen session so it runs in the background. Enter ``CTRL`` plus ``A`` then ``D``.

5. Confirm our background process is running.

    ```
    screen -list
    ```

6. (Optional) If you would like to resume your background session then run the following, else skip this step.

    ```
    screen -r
    ```

7. Confirm our daemon is running in background by running the following command. (Don't forget to run ``CTRL`` plus ``X`` when you finish)

    ```
    top | grep peercoin
    ```

8. (Optional) If you get any error, please investigate the ``debug.log`` file by running the following.

    ```
    ~/.peercoin/debug.log
    ```

#### Start ```peercoind``` on system startup

We are going to create a service in ``systemd`` to have our ``peercoind`` startup on boot time.

**DEVELOPERS NOTE: For some reason the following code does not want to work. If someone can comment on how to fix this, that would be great!**

1. While being logged in as ``pi`` run the following:

    ```
    cd ~/
    touch peercoind_startup.sh
    vi peercoind_startup.sh
    ```

2. Populate the contents of our new file with the following

    ```
    #!/bin/sh
    cd /home/pi/ppcoin/src
    ./peercoind -listen=0 -daemon -server
    ```

3. Permit to run our script.

    ```
    chmod u+x peercoind_startup.sh
    ```

4. Create our ``systemd`` service to handle loading our startup script.

    ```
    sudo vi /etc/systemd/system/peercoind.service
    ```

2. Copy and paste the following contents.

    ```
    [Unit]
    Description=Peercoin Daemon
    After=multi-user.target

    [Service]
    Type=idle
    ExecStart=/home/pi/peercoind_startup.sh
    Restart=on-failure
    KillSignal=SIGTERM

    [Install]
    WantedBy=multi-user.target
    ```

3. We can now start the Gunicorn service we created and enable it so that it starts at boot:

    ```
    sudo systemctl start peercoind
    sudo systemctl enable peercoind
    ```

4. Confirm our service is running.

    ```
    sudo systemctl status peercoind.service
    ```

5. If the service is working correctly you should see something like this at the bottom:

    ```
    raspberrypi systemd[1]: Started Peercoin Daemon.
    ```

6. Congratulations, you have set up a Peercoin headless minter service.

7. If you see any problems, run the following service to see what is wrong. More information can be found in [this article](https://unix.stackexchange.com/a/225407).

    ```
    sudo journalctl -u peercoind
    ```

8. To reload the latest modifications to ``systemctl`` file.

    ```
    sudo systemctl daemon-reload
    ```

# Donate

If you found this article useful, please consider donating:

* Peercoin ```PXTyiBqraYCn95cvEP2jcoCfYEBscNHxBW```
