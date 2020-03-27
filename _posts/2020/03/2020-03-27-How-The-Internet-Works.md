---
layout: article
title: How The Internet Works
date: 2020-03-27 09:39:00+0200
---

If you've previously worked or researched networking, you will have heard of the OSI 7 layer model and how traffic is encapsulated and thrown around globally. How does that traffic know where to go? 
I'll asumme you know about IPv4 Addressing and subnets and dive straight into, if one IP wants to talk to another IP, across the globe, how does my home router know how to get there? 
To deal with the IP protocol and ensuring all connected devices knew where to send their packets, we have routing protocols. There are different types of routing protocols for different pruproses. Interior Gateway Protocols (IGPs) such as (OSPF, ISIS, RIP) are used for when you are routing packets in a domain that you own, for example an office would run an IGP for their network. Exterior Gateway Protocols (EGPs) such as (BGP) are used for when networking domains, owned by different organisations, want to speak to each other.

What that looks like on a router is, something like this...

'''cisco
R2#show ip route
Codes:
C – connected, S – static, I – IGRP, R – RIP, M – mobile, B – BGP
D – EIGRP, EX – EIGRP external, O – OSPF, IA – OSPF inter area
N1 – OSPF NSSA external type 1, N2 – OSPF NSSA external type 2
E1 – OSPF external type 1, E2 – OSPF external type 2, E – EGP
i – IS-IS, L1 – IS-IS level-1, L2 – IS-IS level-2, ia – IS-IS inter area
* – candidate default, U – per-user static route, o – ODR   P – periodic downloaded static route Gateway of last resort is not set.
C 192.168.1.0/24 is directly connected, Serial0/0/0
C 192.168.2.0/24 is directly connected, Serial0/1/0
O 192.168.3.0/24 [120/1] via 192.168.1.1, 00:00:18, Serial0/0/0
O 192.168.4.0/24 [120/2] via 192.168.1.1, 00:00:18, Serial0/0/0
'''
[Sourced from networkstraining.com](https://www.networkstraining.com/cisco-show-ip-route-command/)

take the top line for example, this cisco router has a route for "192.168.3.0/24" via 192.168.1.1 which is out of this routers Serial0/0/0 port. This router has got this route via OSPF (Look at the codes and the start of the line). We can infer from this that there is anoher router on the other end of the Serial0/0/0 port, with an IP address of 192.168.1.1 that is running OSPF, and that router advertised to our router, it knows how to get to 192.168.3.0/24.

British Telecoms (BT) is a big Internet Service Provider (ISP) in the UK, they have a lot of customers connecting to their network, if those customers want to communicate, BT will use an IGP to facilitate this. Take this diagram as an example.

![](../../../contents/images/2020/03/BT_IGP.png)

2 Customers link into RouterA, RouterA know about them customers and the routers directly connected, but not anything directly connected to them (ie customer 10.0.0.236, 192.168.0.1 etc.). RoutersA-E will share information about customers connected to them via an IGP, for example RouterA would tell RouterB and RouterE it knows how to get to 17.16.29.91 and 192.168.71.1. RouterE and RouterB would then tell their adjacent routers (RouterD and RouterC).

So we've now got a working ISP, and the customers can speak to each other. What if a BT customer wants to speak to a Virgin Media customer? BT wouldn't run an IGP to Virgin because they don't own the Virgin Routing devices. This is where BGP comes in.

Let's say BT has registered he IP range 192.168.0.0/24 with RIPE (Internet Registry Authority) and they now own that range. They will start letting other big carriers know, if they want to send traffic to anyone in 192.168.0.0/24, send the traffic to BT. In a digram, it would look something like this.

![](../../../contents/images/2020/03/BIG_BGp_Map.png)


