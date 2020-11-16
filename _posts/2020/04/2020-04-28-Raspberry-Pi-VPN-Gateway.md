---
layout: article
title: Creating a RPi VPN Gateway
date: 2020-04-28 09:39:00+0200
coverPhoto: /contents/images/2020/04/Ra_Pi_Compressed.jpg
---
![](/contents/images/2020/04/Ra_Pi_Compressed.jpg)
## Intro 
With everyone working from home more, I set up a dedicated VPN to use with my work IT. This doesn't give me much really, but in my mind, work traffic going over a VPN I control is more trusted by me.
My work VPN is currently being run on AWS, so my plan was to get a dedicated VPN to AWS, where my work traffic will then be handled inside the AWS Autonomous System (Presumably
). Here is a guide of how to set up your Pi as an access point with an always on VPN solution using wireguard.
 
## Creating the Wireless Access Point
Most of the configuration I used is in the documentation to set this up.. [Source](https://thepi.io/how-to-use-your-raspberry-pi-as-a-wireless-access-point/). There is a difference here that we're not making a bridge, we will instead double NAT!
 
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
 
Installing dnsmasq gave us a DHCP daemon. We are going to set that up as our DHCP server. We are going to be using the interface wlan0 for our WiFi Access Point, and it will need a static address. This static IP will be used as the default gateway for clients connecting and to set this up, we need to tell dhcp not to dish out this IP to client devices. 
To do this we change the /etc/dhcpd/conf file, write the following command to open a text editor. Then we need to add the interface to the end of the file and give it a static IP.
```bash
sudo nano /etc/dhcpcd.conf
```
```bash
interface wlan0
        static ip_address=192.168.4.1/24
        nohook wpa_supplicant
```
Exit out of that file with Ctrl-X, Y, then Enter. So we now have a static IP associated with wlan0, but we have no wlan0 interface so let's set that up now using hostpad. The nohook wpa_supplicant stops some pre-canned code running on boot that will mess with our configs.
That has set up our DHCP. Now we are going to use the HostAp daemon to actually turn on our access point so people can connect to it. We do this by editing the hostapd.conf file. Much the same as last time but the file should be empty (If not, delete the contents)
 
```bash
sudo nano /etc/hostapd/hostapd.conf
```
 
```bash
interface=wlan0
driver=nl80211
ssid=<NETWORK> # Change this value to the WiFi name you want
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=<PASSWORD> # Change this value to the password e.g "Thisisthewifipassword"
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
 
```
We haven't started the hostap or dnsmasq daemon services yet, so nothing should be running. If we did turn these on you would have an access point you can connect to via your phone, tablet etc. There are a few things we are missing, but we will work on. We have a static IP for wlan0 but haven't specified an IP range for clients, therefore they will get no ip on joining the network. Our Pi has no idea how to forward or even if it should forward client packets, and we have no encrypted pipe anyway.
 
### DHCP Setup
 
Next on the list, then, let's configure DHCP. We download the dnsmasq software for this. It comes with some preset config that we don't want, so let's move that to keep as a backup and create a new file.
```bash
## If the file doesn't exist then do cd /etc/ && touch dnsmasq.conf
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.backup
sudo touch /etc/dnsmasq.conf
sudo nano /etc/dnsmasq.conf
```
Add these lines to your file.
```bash
interface=wlan0
        dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
        dhcp-option=6,10.0.0.1
```
dhcp-range will define the IP addresses that we want to use for clients, and the lease time. For example here we are allowing 19 devices, giving them a /24 (255 Addresses) and 24 hours lease, so we could expand the amount of hosts if we wanted. 
dhcp-option defines what options we want to pass to clients, options can be found in [RFC 2132](https://tools.ietf.org/html/rfc2132). The one we are using here is option 6, for Domain Name Server. I've set this to 10.0.0.1 as later down the line, we will be setting up PiHole. You can set this as 8.8.8.8 or 1.1.1.1 if you wanted.
 
Let's start our services and make sure that if our Pi loses power, these services come back up by default. To do that use the following commands
```bash
DAEMON_CONF="/etc/hostapd/hostapd.conf"
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq
```
 
## Wireguard Server Setup
 
I had a look around to work out the best VPN Software to use for this. It was a toss up between OpenVPN and Wireguard. I had previously used OpenVPN and it felt quite clunky. Wireguard boasts an easy setup, you can have a look [here](https://www.wireguard.com/). It's also something I hadn't used before so why not.
 
First let's set up the wireguard server, this is luckily quite simple. Choose your favourite cloud and spin up a VM (AWS, Azure, GCP). I used EC2 on AWS to spin up an Ubuntu server. This is a Debian based distro and therefore if you want to follow along, it's useful to also spin up a Debian distro VM. To get wireguard installed, there is a guide online. [Here](https://www.wireguard.com/install/).
 
```bash
sudo apt install wireguard
```
 
While we're on the CLI, let's also enable IP Forwarding, this will let our box forward IP traffic for us. This will be our wireguard server, so when clients send traffic to it over the pipe, it needs to know it can forward traffic from the pipe to the internet. 
```bash
sysctl -w net.ipv4.ip_forward=1
``` 
We can issue a command to make sure that worked below. If after running the command, it prints a 1, you're all set.
```bash
Sysctl net.ipv4.ip_forward
```
 
To make this a persistent setting, edit the sysctl conf file - /etc/sysctl.conf and add the line...
net.ipv4.ip_forward = 1
 
 
Now for the wireguard Setup, if the installation all went okay, these should work. Make a directory somewhere on your box to store your client and server keys we're about to generate.
```bash
cd /etc/wireguard/
umask 077 
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
```
 
We've just generated our server and client keys, which we will need to create the tunnel. Setting up the server isn't too bad. You have to make a wg0.conf file and add some configuration that I'll run through.
```bash
touch wg0.conf
nano wg0.conf
```
When in the file, the following lines are used to setup our client and server
 
```bash
[Interface]
Address = 10.0.0.1/24 # This gives our server an IP in the pipe
Address = fd86:ea04:1115::1/64 #same as above but IPv6
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
#The post Ups and Downs just adds the NAT rules for if the wireguard services comes up or down, basically if WG is up it will NAT, otherwise don't NAT
ListenPort = 51820 # Assign a port that isn't used for anything else.
PrivateKey = <SERVER PRIVATE KEY> # We will change this in a moment
 
[Peer]
PublicKey = <CLIENT PUBLIC KEY> #We will change this in a moment
AllowedIps = 10.0.0.2/32 # this will be the IP address assigned to the client with this public key.
```
 
I've got some helpful sed commands that will replace our placeholders with the actual key contents.
```bash
sed -i "s/<SERVER PRIVATE KEY>/$(sed 's:/:\\/:g' server_private.key)/" wg0.conf
sed -i "s/<CLIENT PUBLIC KEY>/$(sed 's:/:\\/:g' client_public.key)/" wg0.conf
```
Once we have the config setup, we need to make sure that IPTables is going to let our connections occur, I've used UFW (User Friendly Firewall) as it's nicer to interact with IPTables this way, I'll allow SSH and 51820 (Port assigned to wireguard) then enable the firewall.
 
```bash
sudo ufw allow 22/tcp
sudo ufw allow 51820/tcp
sudo ufw enable
```
 
Following all this, you should now be able to bring up your wireguard service. Wireguard gives us some nice CLI tools to bring it up and down, so let's use them here
 
```bash
wg-quick up wg0
systemctl enable wg-quick@wg0.service
```
 
Boom, your server is all setup, now let's get back to our Pi
 
## Wireguard Pi Setup
So you have your wireguard server and your access point. Now we have to mesh the two.
 
In the previous part, we generated 4 keys, 2 server ones (Public, Private), two client ones (Public Private) make sure to grab your client private key as this will be used on the Pi.
Firstly, install wireguard again as before (sudo apt install wireguard) and enable IP forwarding on the Pi.
 
edit the sysctl conf file - /etc/sysctl.conf and add the line...
net.ipv4.ip_forward = 1
 
Now, we need to do some NAT rules to let traffic coming from the wlan0 interface to be proxied out of the wg0 interface.
 
```bash
sudo iptables -A FORWARD -i wlan0 -o wg0 -j ACCEPT
sudo iptables -A FORWARD -i wg0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -t nat -A  POSTROUTING -o wg0 -j MASQUERADE
```
 
Now those changes to iptables is made, let's make sure they are persistent over boots
```bash
sudo apt install iptables-persistent
```
Answer yes to the prompt and this should handle it all for you!
We're almost there, just need to set up the wireguard client now. For this, make sure you have access to your wireguard server as we will use the keys you generated on there.
 
Let's create boilerplate config and add our keys
 
```bash
nano wg0.conf
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/32, fd86:ea04:1115::5/128
DNS = 10.0.0.1
 
[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <Server IP>:51820
AllowedIps = 0.0.0.0/0
PersistentKeepalive = 21
```
 
Here, the interface block assigns your Pi an IP address in the tunnel, which we defined on the server. The Peer block defines where our Pi goes to establish the tunnel. This is why we need to provide it with the servers public IP address and the servers public key.
 
```bash
sudo su -
cd /etc/wireguard
umask 077
echo "<CLIENT_PRIVATE_KEY_ON_SERVER>" > client_private.key
echo "<SERVER_PUBLIC_KEY_ON_SERVER>" > server_public.key
sed -i "s/<SERVER_PUBLIC_KEY>/$(sed 's:/:\\/:g' server_public.key)/" wg0.conf
sed -i "s/<CLIENT_PRIVATE_KEY>/$(sed 's:/:\\/:g' client_private)/" wg0.conf
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```
 
You should now have a fully functioning Raspberry Pi VPN Access Gateway. Just to ensure it worked, add a device to the WiFi access point and navigate to https://ifconfig.co/ or curl ifconfig.co and you should see your server's IP Address.
 
## Adding PiHole and Cloudflare's DoH service
This is an optional extra, but if you defined your DNS server as your Wireguard server, you will need this.
[PiHole](https://pi-hole.net/)
[Cloudflared](https://developers.cloudflare.com/1.1.1.1/dns-over-https/cloudflared-proxy)
How to do this, is laid out [here...](https://docs.pi-hole.net/guides/dns-over-https/)
 
Firstly, install the cloudflare daemon. This Cloudflare server will enable us to use DoH (DNS Over HTTPS), which means that our previously plaintext DNS queries, will now be encrypted using HTTPS. You can read more about it here [RFC8484](https://tools.ietf.org/html/rfc8484).
PiHole is a blackhole for advertising domains, all that means is that PiHole returns the IP Address of 0.0.0.0 for domains that are advertising domains. 
 
Let's get going - First thing we are going to download is the cloudflared service.
```bash
cd /tmp
wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-amd64.deb
sudo apt install ./cloudflared-stable-linux-amd64.deb
##Verify that the installation worked
cloudflared --version
#should look something like this
root@ip-172-26-12-159:/etc/wireguard# cloudflared --version
cloudflared version 2020.11.6 (built 2020-11-15-0231 UTC)
```
 
Now let's set up our configs
 
```bash
sudo mkdir /etc/cloudflared/
sudo nano /etc/cloudflared/config.yml
```
```
proxy-dns: true
proxy-dns-port: 5053
proxy-dns-upstream:
  - https://1.1.1.1/dns-query
  - https://1.0.0.1/dns-query
```
```bash
sudo cloudflared service install --legacy
sudo systemctl start cloudflared
sudo systemctl status cloudflared
 
#check it worked using dig
pi@raspberrypi:~ $ dig @127.0.0.1 -p 5053 google.com
 
; <<>> DiG 9.11.5-P4-5.1-Raspbian <<>> @127.0.0.1 -p 5053 google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12157
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
 
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 22179adb227cd67b (echoed)
;; QUESTION SECTION:
;google.com.                    IN      A
 
;; ANSWER SECTION:
google.com.             191     IN      A       172.217.22.14
 
;; Query time: 0 msec
;; SERVER: 127.0.0.1#5053(127.0.0.1)
;; WHEN: Wed Dec 04 09:29:50 EET 2019
;; MSG SIZE  rcvd: 77
```
 
Installing PiHole is nice and easy. Use this one line 
```
curl -sSL https://install.pi-hole.net | bash
```
There is a few prompts that come up, the one you need to change is the "Choose an Interface". To stop us from our pihole being exposed to the world, make sure this is set to wg0 interface! This step is important!
 
When it asks for an Upstream DNS Provider, select Custom.
Then enter "127.0.0.1#5053"
Continue clicking enter for a bit until it asks about a static IP, it will give you some defaults, from there click "No"
Instead enter what you set in your DHCP config, so for me "10.0.0.1/24" and I set the default gateway to the IP address of my servers external interface, so for this 
```bash
ubuntu@ip-172-26-12-159:~$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9001
        inet 172.26.12.159  netmask 255.255.240.0  broadcast 172.26.15.255
        inet6 fe80::462:eff:fe41:9974  prefixlen 64  scopeid 0x20<link>
        ether 06:62:0e:41:99:74  txqueuelen 1000  (Ethernet)
        RX packets 52553854  bytes 39749346669 (39.7 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 47132475  bytes 38029706037 (38.0 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
10.0.0.1/24 and a default gateway of 172.26.12.159
The next few bits are parts you can decide. How much logging etc. Once you've set it up. If you decide to go with the interface, when you connect to the WiFi you can view the logs at http://10.0.0.1/admin
 
Let's double check our PiHole and Cloudflared services can't be used externally. Get your servers public IP and run 
 
```bash
root@raspberrypi:/# dig @3.10.199.72 google.com
 
; <<>> DiG 9.11.5-P4-5.1-Raspbian <<>> @3.10.199.72 google.com
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
 
root@raspberrypi:/etc/wireguard# dig @3.10.199.72 -p 5053 google.com
 
; <<>> DiG 9.11.5-P4-5.1-Raspbian <<>> @3.10.199.72 -p 5053 google.com
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
 
## But we should be able to access them in the pipe
 
 
root@raspberrypi:/etc/wireguard# dig @10.0.0.1 google.com
 
; <<>> DiG 9.11.5-P4-5.1-Raspbian <<>> @10.0.0.1 google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16558
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
 
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;google.com.                    IN      A
 
;; ANSWER SECTION:
google.com.             204     IN      A       216.58.210.46
 
;; Query time: 21 msec
;; SERVER: 10.0.0.1#53(10.0.0.1)
;; WHEN: Mon Nov 16 14:05:43 GMT 2020
;; MSG SIZE  rcvd: 65
```
 
You're all done, your very own end to end VPN/Adblocking/DoH Gateway. We have ended up with something like this.
What we have made is something like this
![](/contents/images/2020/04/Diagram.png)
 
This shows we are encrypted when leaving our ISP, so they cannot sniff traffic. We also are not directly connected to the internet and therefore, nobody knows where we have come from. It instead looks like you have come from an Amazon EC2 in this instance.
 
 
 
 
 
 
 
 
 
 
 
 
 

