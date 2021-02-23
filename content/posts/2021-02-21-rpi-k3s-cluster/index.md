---
title: "Raspberry Pi Cluster Part 1 - Saibamen Attack!"
author: "Matt Rathbun"
draft: false
slug: "rpi-k3s-cluster"
toc: true
tags:
  - raspberrypi
  - linux
  - kubernetes
  - k3s
---

Over the holidays I made a [k3s](https://k3s.io/) (that's Kubernetes minus 5) cluster out of Raspberry Pis I had lying around. And while it's _extremely_ cool, it's also _pretty_ unstable.

I'll save my complaints for part 2. In this post, I'll outline what this new cluster looks like and how I set it up.

<!--more-->

# Overview

![saibamen diagram](saibamen_diagram.svg)

To keep it simple, this cluster has just two nodes. In keeping with my tradition of naming computers after Dragon Ball Z villains, it's collectively named the `saibamen` cluster (after those [nasty little green aliens](https://dragonball.fandom.com/wiki/Saibamen) Nappa and Vegeta unleashed on our unsuspecting Z Fighters).

* `saibaman1` -> master node
    * Also runs a DHCP server, assigning IP addresses on the cluster network
    * Connected to the outside network over WiFi and shares that connection with the rest of the cluster
* `saibaman2` -> worker node
    * Only connected to the cluster network

## The Hardware

* 2 x Raspberry Pi 3
    * Note that [k3s doesn't support ARMv6](https://github.com/kubernetes/kubeadm/issues/253#issuecomment-296738890), so you need at least a Pi 2. Older Pis like the Pi 1 and Pi Zero aren't supported.
* 2 x MicroSD cards
* 1 x Ethernet switch
* 1 x USB power hub (I got [this one](https://www.amazon.com/gp/product/B00P933OJC), though I wonder if this is the culprit for some of the issues I've had)

## The Software

* Raspberry Pi OS (formerly known as raspbian)
    * I tried NixOS first (I'm a bit of a fanboy) but had some trouble getting it working. Maybe someday!
* k3s
    * _Why k3s?_ Because it's easy to set up and getting it working on Raspberry Pi is well documented.
* dnsmasq - the DHCP server running on the master node
* iptables - Used to share wifi connection with the rest of the cluster

# Setup

## Installing and Configuring Linux

Do this on both Pis.

1. _Use your favorite method to install Raspberry Pi OS on the microSD card._ Personally, I like good ol' `dd`:

    ```shell
    sudo dd bs=4M if=path/to/image.img of=/dev/sdX conv=fsync
    ```

2. You'll likely want to run `sudo raspi-config` and change a few things:
    ```plain
    In `System Options`,
      * Set a password
      * Set a hostname
    In `Interface Options`,
      * Enable SSH access
    In `Localisation Options`,
      * Set locale
      * Set timezone
      * Enable WiFi (on master only)
    ```

3. Install your favorite editor (Vim, obvs)
    ```shell
    sudo apt install vim
    ```

4. **Important!** Add this line to the end of `/boot/cmdline.txt` so k3s can run containers

    ```txt
    cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
    ```

5. Reboot!

    ```shell
    sudo reboot
    ```

## Networking

To keep cluster traffic isolated from my home network, the master node serves as a DHCP server and shares its WiFi connection over ethernet. It ends up looking something like this:

```
Wifi (192.168.1.x) -> saibaman1 -> Ethernet (192.168.2.x) -> saibaman2
```

### Sharing a WiFi Connection

The following is copied with a few small changes (mostly that I changed it to use `192.168.2.x`) from [this excellent forum post](https://www.raspberrypi.org/forums/viewtopic.php?t=223295).

1. Configure the Pi's WiFi connection. Check out [this guide](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md) for how to do that. Once you've set up WiFi, you'll probably want to disconnect it from ethernet if it's connected. Running multiple DHCP servers can get weird pretty fast.

2. Install dnsmasq

    ```shell
    sudo apt install dnsmasq
    ```

3. Set a static IP address for `eth0`. Open `/etc/dhcpcd.conf` and add this to the end:

    ```conf
    interface eth0
    static ip_address=192.168.2.1/24
    ```

4. Save or remove the original `dnsmasq.conf`

    ```shell
    sudo mv /etc/dnsmasq.conf /etc/dnsmaq.conf.orig
    ```

5. Open `/etc/dnsmasq.conf` and add this:

    ```conf
    interface=eth0
    dhcp-range=192.168.2.10,192.168.2.250,255.255.255.0,12h
    ```

6. Open `/etc/sysctl.conf` and uncomment

    ```conf
    net.ipv4.ip_forward=1
    ```

7. Open `/etc/rc.local` and add this line above `exit 0`

    ```shell
    iptables -t nat -A  POSTROUTING -o wlan0 -j MASQUERADE
    ```

9. Reboot!

    ```shell
    sudo reboot
    ```

## Setting up k3s

### Install k3s on the master node

1. `ssh` into the master pi

2. Install `k3s`

    ```shell
    curl -sfL https://get.k3s.io | sh -
    ```

3. Check to see if it's working. Run the following to see what nodes have been added to the cluster, you should see one node:

    ```shell
    sudo kubectl get nodes
    ```

4. Copy the join token, you'll need this to add nodes to the cluster

    ```shell
    sudo cat /var/lib/rancher/k3s/server/node-token
    ```

### Install k3s on the worker node

1. `ssh` into the worker pi

2. Install `k3s` and be sure to pass the token you copied earlier and the IP address of the master node:

    ```shell
    curl -sfL http://get.k3s.io | K3S_URL=https://192.168.2.1:6443 \
    K3S_TOKEN=<join-token> sh -
    ```

3. Once that's done, check to see if it's working. `ssh` back into the master node and run the following command. You should see two nodes:

    ```shell
    sudo kubectl get nodes
    ```

### Grab your kubeconfig

1. `ssh` into the master node

2. Copy its kubeconfig

    ```shell
    sudo cat /etc/rancher/k3s/k3s.yaml
    ```

3. Return to the computer you'd like to put the kubeconfig onto

4. Make a `.kube` folder if one doesn't exist already

    ```shell
    mkdir .kube
    ```

5. Open `.kube/config` and paste the contents of the file you copied earlier

6. Replace `localhost` with the IP address of the master node relative to the computer you're setting this up on:

    ```yaml
    server: https://192.168.1.x:6443
    ```

7. Change the permissions of the new file so kubectl won't complain about it (oh _and_ because it's the right thing to do):

    ```shell
    chmod 600 ~/.kube/config
    ```

8. Try it out!

    ```shell
    kubectl get nodes
    ```

# So How Is It?

Well, let's just say I wouldn't put anything too important on it.

k3s crashes when I push the cluster to its limits, like when installing a particularly complex Helm chart. To make matters worse, that crash can put it in a state where it can't start up again. I've had to reinstall it several times now.

I have a couple theories for why this happens.

## Power Issues?

Raspberry Pis are notoriously sensitive to power supply issues. Remember that xkcd about how clicking links in a Wikipedia article will eventually lead you to "Philosophy"? I've found that debugging any Raspberry Pi issue will eventually lead you to a storefront selling a better power supply. It's weird how that always happens.

On one hand, it could be that my USB hub isn't putting out enough power. I didn't really look into it too much to be honest. I didn't feel like buying new usb chargers and making the wiring more obnoxious for something that _might_ be the issue.

I'm wondering if one of my Micro USB cables is to blame. Running `vcgencmd measure_volts` repeatedly under high CPU usage showed that one of my cables had a lower maximum voltage. I swapped it out and I feel like it's been a bit better but it might just be wishful thinking.

## Are My Raspberry Pis Too Slow?

[This article](https://medium.com/@ikarus/run-kubernetes-on-your-raspberry-pi-cluster-with-k3s-ac3687d6eb1a) makes the case that the Raspberry Pis I'm using are too slow to reliably run etcd (which is used by k3s to keep track of cluster state). Before the Raspberry Pi 4; Ethernet, USB, and the SD card are all on one USB 2.0 bus. I'm likely making this worse by having the same Pi handle both the control plane _and_ the cluster's network connection.

## Whatever, It's Good Enough

Despite all that, this has been a fun project. I built a kubernetes cluster on the cheap, mostly using parts I already had. As long as I don't push it too hard, it works just fine!
