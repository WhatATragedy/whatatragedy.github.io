---
layout: article
title: Creating a RPi VPN Gateway
date: 2020-04-28 09:39:00+0200
---
## Intro 
With everyone working from home more, I set up a dedicated VPN to use with my work IT. This deosn't give me much really, but in my mind, work traffic going over a VPN I control is more trusted by me.
My work VPN is currently being run on AWS, so my plan was to get a dedicated VPN to AWS, where my work traffic will then be handled inside the AWS Autonomous System (Presumably
). Here is a guide of how to set up your Pi as an access point with an always on VPN solution using wireguard.

## Creating the Wireless Access Point
Most of the configuration I used is in the documentation to set this up.. [Source](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md). 

First things first, plug your Pi in and update it, this can take a bit of time...
```bash
sudo apt-get update
sudo apt-get upgrade
```
We're going to use Hostapd (Wifi Access Point) and Dnsmasq (DHCP) for our access point and DHCP. If you're not sure what these are, give them a google! 
```bash
sudo apt-get -y install hostapd
sudo apt-get -y install dnsmasq
```
When these download, they start by default, because we will be playing around with the configuration files, we should stop these services using "systemctl".
```bash
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
```

Now we're going to use the DHCPd client in order to configure a static IP address for the Wireless Lan interface we will be setting up (wlan0). This static IP will be used as the default gateway for clients connecting and to set this up, we need to tell dhcp not to dish out this IP to client devices. 
To do this we change teh /etc/dhcpd/conf file.
```bash
sudo nano /etc/dhcpcd.conf
```
append the following lines to the end
```bash
interface wlan0
        static ip_address=192.168.4.1/24
        nohook wpa_supplicant
```
Exit out of that file with Ctrl-X, Y, then Enter. So we now have a static IP assoicated with wlan0, but we have no wlan0 interface so lets set that up now using hostpad. The nohook wpa_supplicant stops some pre-canned code running on boot that will mess with our configs.

```bash
sudo nano /etc/hostapd/hostapd.conf
```
This file should be clean, if not then delete the contents and write the following, changing the values NETWORK and PASSWORD.

```bash
interface=wlan0
bridge=br0
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
ssid=<NETWORK>
wpa_passphrase=<PASSWORD>
```

## WireGuard Setup

That should be the config for your wireless access point. We haven't started the hostpad daemon yet so this won't be visible when you search for available WiFi networks, but even if you could connect, we haven't actually set up our DHCP server, so no clients would get IP address, therefore no connectivity.
Next on the list then, let's configure DHCP. We download the dnsmasq software for this. It comes with some preset config that we don't want, so let's move that to keep as a backup and create a new file.
```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.backup
sudo touch /etc/dnsmasq.conf
sudo nano /etc/dnsmasq.conf
```
Let's start editing.. Add these lines to the file
```bash
interface=wlan0
        dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
        dhcp-option=6,10.0.0.1
```

I had a look around to work out the best VPN Software to use for this. It was a toss up between OpenVPN and Wireguard. I had previously used OpenVPN and it felt quite clunky, Wireguard boast an easy setup, you can have a look [here](https://www.wireguard.com/). It's also something I hadn't used before so why not.

First let's set up the wireguard server, this is luckily quite simple. Choose your favourite cloud and spin up a VM (AWS, Azure, GCP). I used EC2 on AWS to spin up an AMI. This is a a RedHat distro and therefore if you want to follow along, it's useful to also spin up a RedHat distro VM. To get wireguard installed issue the following commands:

```bash
sudo yum install epel-release
sudo yum config-manager --set-enabled PowerTools
sudo yum copr enable jdoss/wireguard
sudo yum install wireguard-dkms wireguard-tools
```

While we're on the CLI, let's also enable IP Forwarding, this will let our box forward IP traffic for us. 
```bash
sysctl -w net.ipv4.ip_forward=1
``` 
and to check that worked 
```bash
Sysctl net.ipv4.ip_forward
```
Should be equal to one.
To make this a persistent setting, edit the sysctl conf file - /etc/sysctl.conf and add the line...
net.ipv4.ip_forward = 1

Let's reload the service so it picks up our changes
```bash 
sysctl -p /etc/sysctl.conf
```

Now for the wireguard Setup, if the installation all went okay, these should just work. Make a directory somewhere on your box to store your client and server keys we're about to generate.
```bash
cd /etc/wireguard/
umask 077 
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
```






