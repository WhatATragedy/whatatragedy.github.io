---
layout: article
title: The Big Internet Bois
date: 2020-10-23 09:39:00+0200
coverPhoto: /contents/images/2020/10/Internet_Bois.png
---

# Once Upon a time in the Internet
Back long ago, the internet was quite small and made of HTML pages. A lot has changed since then. If we have a look back at Routeviews IP to Prefix mapping from 2005, we can see who some of the bigger ISPs were back then.

ADD Some DATA Here from http://data.caida.org/datasets/routing/routeviews-prefix2as/2005/12/

[^1]: Data provided by Caida http://data.caida.org/datasets/routing/routeviews-prefix2as/

The internet has grown exponentionally, with the current global routing table standing at 883483 routes for IPv4 and 100521 for IPv6 (26/10/2020).

# Who are the new big boys on the block
I wanted to find out, who the biggest Tier 1 networks are, and, if using BGP would be an ample way of deciding this. If you know about BGP, you know it's how different autonomous systems peer and exchange data. BGP is great becuase you can see who owns which IP Prefixes, you can see IPv6 uptake, rPKI uptake etc. 
BGP does have limits though. I am looking to find out who the biggest transit providers are in the world, basically, who is handling the most data. BGP tells us about who is peering, but not how much traffic is being exchanged. For example using BGP we have no idea the load that is on the peering links and the bandwidth. 

## The Experiment
#### Can we use AS_PATHs in order to find out who is carrying the most traffic globally?
If you have seen BGP TABLE DUMPs, you know there are AS PATHs in BGP advertisments. If you telnet to a routeviews project and do "show bgp all" you should see this.

```cisco
route-views>show bgp all
For address family: IPv4 Unicast
     Network          Next Hop            Metric LocPrf Weight Path
Nr>  0.0.0.0          162.251.163.2                          0 53767 3257 i
V*   1.0.0.0/24       206.24.210.80                          0 3561 209 3356 13335 i
V*                    162.250.137.254                        0 4901 6079 13335 i
V*                    202.93.8.242                           0 24441 13335 i
V*                    137.39.3.55                            0 701 2914 13335 i
```
This is converted to a BGP DUMP format that looks like this - 
TABLE_DUMP2|26/10/20 18:36:13|B|206.24.210.80|3561|1.0.0.0/24|3561 209 3356 13335|IGP

The path attribute (3561 209 3356 13335) is all the autonmous systems this advertisment passed through. The experiment will work out, if analysed in bulk, could we use AS_PATH to find the most common transitted ASN.

## The Code
Routeviews very kindly offer all of their collectors BGP tables every 2 hours. I wrote some code in GoLang to pull down all of the routeviews collectors from 26/10/2020 at 00:00. Parse them from the compressed MRT data format of BGPDUMP and push them to an PostgreSQL database.

