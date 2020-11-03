---
layout: article
title: The Big Internet Bois
date: 2020-10-23 09:39:00+0200
coverPhoto: /contents/images/2020/10/Internet_Bois.png
---
![](/contents/images/2020/10/Internet_Bois.png)
# Once Upon a time on the Internet
Back long ago, the internet was quite small and made of HTML pages. A lot has changed since then. If we have a look back at Routeviews IP to Prefix mapping from 2005, we can see who some of the bigger ISPs were back then. AT&T was big and some of the biggest IP Space advertisers are the same as now.

December 2005 IP Space Owners.

| ASN   	| Hosts Sum 	| Unique Prefix Count 	| Name                                                       	|
|-------	|-----------	|---------------------	|------------------------------------------------------------	|
|   721 	|  91318784 	|                1056 	| DoD Network Information Center                             	|
|  3356 	|  43552256 	|                 497 	| Level 3 Parent                                             	|
|  7018 	|  42561793 	|                1455 	| AT&T Services                                              	|
|  4134 	|  39326080 	|                1019 	| CHINANET-BACKBONE                                          	|
|   714 	|  35786496 	|                  32 	| APPLE-ENGINEERING                                          	|
|   701 	|  35059200 	|                5285 	| MCI Communications Services, Inc. d/b/a Verizon Business   	|
| 17676 	|  27970048 	|                 470 	| Softbank BB Corp.                                          	|
|   174 	|  23576320 	|                1051 	| COGENT-174                                                 	|
|  7132 	|  22628352 	|                 468 	| AT&T Corp.                                                 	|
|    71 	|  20471808 	|                  87 	| HP-INTERNET-AS                                             	|
|   237 	|  18594720 	|                 170 	| MERIT-AS-14                                                	|
|  2381 	|  18372608 	|                  35 	| WISCNET1-AS                                                	|
|  2686 	|  17890832 	|                 457 	| AT&T Global Network Services                               	|
|  2647 	|  17559157 	|                 100 	| Societe Internationale de Telecommunications Aeronautiques 	|

November 2020 IP Space Owners.

| ASN   	| Hosts Sum 	| Unique Prefix Count 	| Name                                                     	|
|-------	|-----------	|---------------------	|----------------------------------------------------------	|
|  7018 	| 115894529 	|                1753 	| AT&T Services                                            	|
|  4134 	| 115514368 	|                 818 	| CHINANET-BACKBONE                                        	|
|   721 	|  74855936 	|                 278 	| DoD Network Information Center                           	|
|  7922 	|  71336960 	|                 184 	| COMCAST-7922                                             	|
| 17676 	|  70163968 	|                 781 	| EWORLD-AS-PK-AP                                          	|
|  4766 	|  69298208 	|                2542 	| Korea Telecom                                            	|
|  4837 	|  58777856 	|                 869 	| CHINA169-Backbone                                        	|
|  3356 	|  58003136 	|                1491 	| Level 3 Parent                                           	|
|  9808 	|  57068032 	|                2569 	| China Mobile                                             	|
|   714 	|  47885056 	|                 966 	| APPLE-ENGINEERING                                        	|
|   701 	|  44234752 	|                1081 	| MCI Communications Services, Inc. d/b/a Verizon Business 	|
|  9394 	|  43384576 	|                2150 	| Generation IT                                            	|
|  8075 	|  38174976 	|                 220 	| MICROSOFT-CORP-MSN-AS-BLOCK                              	|
|  4538 	|  36675584 	|                5073 	| China Education and Research Network Center              	|
 
[^1]: Data provided by Caida http://data.caida.org/datasets/routing/routeviews-prefix2as/
This was created using a small snippit of code
```python
import gzip
import ipaddress
import pandas
def count_hosts(prefix):
    if ":" in prefix:
        v6_net = ipaddress.IPv6Network(prefix)
        return v6_net.num_addresses, v6_net.version
    else:
        v4_net = ipaddress.IPv4Network(prefix)
        return v4_net.num_addresses, v4_net.version
if __name__ == "__main__":
    data = []
    with gzip.open("routeviews-rv2-20201101-1200.pfx2as.gz") as inputFile:
        for line in inputFile:
            line = line.decode("UTF-8")
            prefix, cidr, asn = line.split("\t", 3)
            asn = asn.strip()
            prefix = f"{prefix}/{cidr}"
            data.append([prefix, asn])
    df = pandas.DataFrame(data, columns=['Prefix', 'ASN'])
    df['Hosts'], df['Version'] = zip(*df['Prefix'].apply(count_hosts))
    agg = df.groupby('ASN')['Hosts'].agg(['sum','count'])
    agg.sort_values(by='sum', ascending=False).to_csv('agg_2020.csv')
```
The internet has grown exponentially, with the current global routing table standing at 883483 routes for IPv4 and 100521 for IPv6 (26/10/2020). So let's have a look at the main transit forces in the new world.
 
# Who are the new big boys on the block
I wanted to find out who the biggest Tier 1 networks are, and, if using BGP would be an ample way of deciding this. If you know about BGP, you know how different autonomous systems peer and exchange data. BGP is great because you can see who owns which IP Prefixes, you can see IPv6 uptake, rPKI uptake etc. 
BGP does have limits though. I am looking to find out who the biggest transit providers are in the world, basically, who is handling the most data. BGP tells us about who is peering, but not how much traffic is being exchanged. For example using BGP we have no idea the load that is on the peering links and the bandwidth. 
 
## The Experiment
#### Can we use AS_PATHs in order to find out who is carrying the most traffic globally?
If you have seen BGP TABLE DUMPs, you know there are AS PATHs in BGP advertisements. If you telnet to a routeviews project and do "show bgp all" you should see this.
 
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
```cisco
TABLE_DUMP2|26/10/20 18:36:13|B|206.24.210.80|3561|1.0.0.0/24|3561 209 3356 13335|IGP
```
 
The path attribute (3561 209 3356 13335) is all the autonomous systems this advertisement passed through. The experiment will work out, if analysed in bulk, could we use AS_PATH to find the most common transmitted ASN.
 
## The Code
Routeviews very kindly offer all of their collectors BGP tables every 2 hours. I wrote some code in GoLang to pull down some* of the routeviews collectors from 26/10/2020 at 00:00. Parse them from the compressed MRT data format of BGPDUMP (using Isolario BGPScanner) and push them to a PostgreSQL database. It's viewable on my github page here: https://github.com/WhatATragedy/KaleBlazer. 
I played around with some GoLang concepts as it was my first time using GoLang and I was pleasantly surprised. I toyed around with worker and master logic which uses channels and such (very GoLang) and managed to get that working! But in the end reverted to a goroutine per routeview collector to make it easier to manage for now while developing. So it looks something like this.

```go
ribHandler.l.Println("[Main] Welcome To Kale Blazer Reborn...")
	ribHandler.GetCollectors()
	ribHandler.l.Println("[Main] Finished Getting Collectors")
	postgresConnector := ribHandler.connectPostgres(ribHandler.l)
	err := ribHandler.createPostgresTable(postgresConnector)
	if err != nil {
		panic(err)
	}
	sem := make(chan struct{}, 10)
	var wg sync.WaitGroup
	taskNum := 0 
	for i, collectorName := range ribHandler.collectors {
		wg.Add(1)
		latestCollection := ribHandler.LatestCollection(collectorName)
		go ribHandler.getFile(&wg, collectorName, latestCollection, postgresConnector, sem)
		taskNum++		
	}
	wg.Wait()
	ribHandler.l.Printf("[Main] Completed %v Tasks..\n", taskNum)
	ribHandler.l.Println("[Main] Done Collecting Files...")
```

Worker logic looks a bit different, an example is here
```go
func (ribHandler *RibHandler) createWorkerPool(noOfWorkers int, jobChannel chan string, postgresChan chan string) {
	var wg sync.WaitGroup
	for i := 0; i < noOfWorkers; i++ {
		go parseBGPFile(i, &wg, jobChannel, postgresChan)
	}
	wg.Add(len(jobChannel))
     wg.Wait()
     
//create worker pools and parse it the channel so they can read from it when needed (5 jobs created.)
ribHandler.createWorkerPool(5, parseJobChan, postgresChan)
var wg sync.WaitGroup
for i, collectorName := range ribHandler.collectors {
     latestCollection := ribHandler.LatestCollection(collectorName)
     ribHandler.l.Printf("%v Latest Collection %v\n", collectorName, latestCollection)
     go ribHandler.getFile(&wg, collectorName, latestCollection, parseJobChan)
     if i > 5 {
          break
     }
}
wg.Wait()
ribHandler.l.Printf("Wrote %v Tasks..\n", i)
//close the channel when all files have been collected as filepaths should have been pushed
close(parseJobChan)
```

If you're interested, browse the code, worker logic is commit [cae152efeac9fcc0333dda908e9831407c8a88a2](https://github.com/WhatATragedy/KaleBlazer/tree/cae152efeac9fcc0333dda908e9831407c8a88a2) and the current commits are a single Goroutine.

## Cool, what does the table look like?
 
```sql
CREATE UNLOGGED TABLE ribs (
          Prefix INET,
          ASPath bigint[],
          OriginatingIP INET,
          OriginatingASN bigint,
          SourceRIB TEXT,
          OriginatingDatetime TIMESTAMP
     ) PARTITION BY RANGE ( originatingasn );
     CREATE TABLE records_0 PARTITION OF ribs
               FOR VALUES FROM (0) TO (1000);
     CREATE TABLE records_1000 PARTITION OF ribs
               FOR VALUES FROM (1000) TO (2000);
```
 
I created the table initially, and realised I had a ton of rows and therefore my queries were taking ages. This led to me having to partition the table. This means the postgres can parallelize my queries a lot better over all the tables which speeds up queries massively. I haven't included all the partitions here but there are 18 partitions now.
Here is an example of the collectors that I had ingester
```sql
routing_information=# SELECT DISTINCT source_rib FROM ribs;
        source_rib         
---------------------------
 route-views.eqix-rib
 route-views.flix-rib
 route-views3-rib
 route-views.fortaleza-rib
 route-views6-rib
 route-views.chicago-rib
 route-views4-rib
 route-views.amsix-rib
 route-views.chile-rib
```
 
## The querying 
So, the process I was going to use to find out the biggest transit provider was just to find out the ASN that's seen most in the AS PATH. SO I wrote some SQL to do that. Postgres has an EXPLAIN ANALYSE function that tells you how your query will run and the time it's estimated to take, so I did that because I need to expand out the AS_PATH column.
```sql
routing_information=# EXPLAIN ANALYSE  SELECT COUNT(*) as count, arrayaspath from (SELECT unnest(as_path, E' ') as arrayaspath FROM ribs) pathasns GROUP BY arrayaspath ORDER BY count DESC;
                                                                                      QUERY PLAN                                                                                      
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=22250677.97..22250678.47 rows=200 width=40) (actual time=1569677.392..1569736.099 rows=71672 loops=1)
   Sort Key: (count(*)) DESC
   Sort Method: external merge  Disk: 1720kB
   ->  Finalize GroupAggregate  (cost=22250619.66..22250670.33 rows=200 width=40) (actual time=1569055.483..1569602.122 rows=71672 loops=1)
         Group Key: (unnest(records_17000.as_path, ' '::text)))
         ->  Gather Merge  (cost=22250619.66..22250666.33 rows=400 width=40) (actual time=1569055.454..1569375.670 rows=195853 loops=1)
               Workers Planned: 2
               Workers Launched: 2
               ->  Sort  (cost=22249619.64..22249620.14 rows=200 width=40) (actual time=1568985.821..1569078.174 rows=65284 loops=3)
                     Sort Key: (unnest(records_17000.as_path, ' '::text)))
                     Sort Method: external merge  Disk: 1712kB
                     Worker 0:  Sort Method: external merge  Disk: 1664kB
                     Worker 1:  Sort Method: external merge  Disk: 1640kB
                     ->  Partial HashAggregate  (cost=22249609.99..22249611.99 rows=200 width=40) (actual time=1568590.388..1568703.635 rows=65284 loops=3)
                           Group Key: (unnest(records_17000.as_path, ' '::text)))
                           ->  Parallel Append  (cost=0.00..6642110.14 rows=433541680 width=32) (actual time=884.180..1202736.851 rows=155415697 loops=3)
                                 ->  ProjectSet  (cost=0.00..2853786.36 rows=275293480 width=32) (actual time=292.969..471602.358 rows=102549262 loops=3)
                                       ->  Parallel Seq Scan on records_17000  (cost=0.00..1202025.48 rows=27529348 width=24) (actual time=292.956..109120.714 rows=22023479 loops=3)
                                 ->  ProjectSet  (cost=0.00..189481.82 rows=18515260 width=32) (actual time=1009.358..42297.717 rows=18855970 loops=1)
                                       ->  Parallel Seq Scan on records_9000  (cost=0.00..78390.26 rows=1851526 width=24) (actual time=1009.334..8492.732 rows=4443663 loops=1)
                                 ->  ProjectSet  (cost=0.00..175628.51 rows=17097930 width=32) (actual time=0.810..41716.606 rows=18008790 loops=1)
                                       ->  Parallel Seq Scan on records_7000  (cost=0.00..73040.93 rows=1709793 width=24) (actual time=0.802..8059.992 rows=4103504 loops=1)
                                 ->  ProjectSet  (cost=0.00..142076.62 rows=13977660 width=32) (actual time=4.620..29441.197 rows=12850264 loops=1)
                                       ->  Parallel Seq Scan on records_8000  (cost=0.00..58210.66 rows=1397766 width=24) (actual time=4.612..5455.891 rows=3354638 loops=1)
                                 ->  ProjectSet  (cost=0.00..128744.29 rows=12639470 width=32) (actual time=9.411..27068.944 rows=11955764 loops=1)
                                       ->  Parallel Seq Scan on records_4000  (cost=0.00..52907.47 rows=1263947 width=24) (actual time=9.404..5009.633 rows=3033472 loops=1)
                                 ->  ProjectSet  (cost=0.00..120444.05 rows=11755580 width=32) (actual time=1.387..25620.538 rows=11172175 loops=1)
                                       ->  Parallel Seq Scan on records_12000  (cost=0.00..49910.57 rows=1175558 width=24) (actual time=1.378..4491.649 rows=2821338 loops=1)
                                 ->  ProjectSet  (cost=0.00..108665.58 rows=10647940 width=32) (actual time=4.832..13829.047 rows=5277422 loops=2)
                                       ->  Parallel Seq Scan on records_6000  (cost=0.00..44777.94 rows=1064794 width=24) (actual time=4.825..3363.893 rows=1277752 loops=2)
                                 ->  ProjectSet  (cost=0.00..106602.90 rows=10391700 width=32) (actual time=7.701..22653.906 rows=10044397 loops=1)
                                       ->  Parallel Seq Scan on records_11000  (cost=0.00..44252.70 rows=1039170 width=24) (actual time=7.693..3999.854 rows=2494008 loops=1)
                                 ->  ProjectSet  (cost=0.00..96375.73 rows=9464390 width=32) (actual time=7.918..21294.107 rows=9808779 loops=1)
                                       ->  Parallel Seq Scan on records_0  (cost=0.00..39589.39 rows=946439 width=24) (actual time=7.908..3643.169 rows=2271453 loops=1)
                                 ->  ProjectSet  (cost=0.00..87106.29 rows=8545470 width=32) (actual time=0.919..17525.944 rows=7960023 loops=1)
                                       ->  Parallel Seq Scan on records_3000  (cost=0.00..35833.47 rows=854547 width=24) (actual time=0.900..3080.596 rows=2050912 loops=1)
                                 ->  ProjectSet  (cost=0.00..82100.33 rows=7986190 width=32) (actual time=7.804..17636.951 rows=7877196 loops=1)
                                       ->  Parallel Seq Scan on records_16000  (cost=0.00..34183.19 rows=798619 width=24) (actual time=7.795..3117.915 rows=1916686 loops=1)
                                 ->  ProjectSet  (cost=0.00..75040.05 rows=7221150 width=32) (actual time=3.571..17305.334 rows=8563676 loops=1)
                                       ->  Parallel Seq Scan on records_10000  (cost=0.00..31713.15 rows=722115 width=24) (actual time=3.564..2809.957 rows=1733076 loops=1)
                                 ->  ProjectSet  (cost=0.00..70977.63 rows=6897090 width=32) (actual time=9.100..15827.897 rows=6933096 loops=1)
                                       ->  Parallel Seq Scan on records_13000  (cost=0.00..29595.09 rows=689709 width=24) (actual time=9.087..2629.161 rows=1655301 loops=1)
                                 ->  ProjectSet  (cost=0.00..53958.77 rows=5263110 width=32) (actual time=8.982..11515.075 rows=4990087 loops=1)
                                       ->  Parallel Seq Scan on records_15000  (cost=0.00..22380.11 rows=526311 width=24) (actual time=8.972..2096.086 rows=1263146 loops=1)
                                 ->  ProjectSet  (cost=0.00..52583.65 rows=5115950 width=32) (actual time=6.820..11855.855 rows=5632167 loops=1)
                                       ->  Parallel Seq Scan on records_5000  (cost=0.00..21887.95 rows=511595 width=24) (actual time=6.811..2049.817 rows=1227827 loops=1)
                                 ->  ProjectSet  (cost=0.00..49580.88 rows=4818840 width=32) (actual time=9.972..10868.487 rows=4908774 loops=1)
                                       ->  Parallel Seq Scan on records_14000  (cost=0.00..20667.84 rows=481884 width=24) (actual time=9.961..1813.635 rows=1156521 loops=1)
                                 ->  ProjectSet  (cost=0.00..45096.59 rows=4409370 width=32) (actual time=0.040..10007.041 rows=4402638 loops=1)
                                       ->  Parallel Seq Scan on records_2000  (cost=0.00..18640.37 rows=440937 width=24) (actual time=0.030..1652.666 rows=1058249 loops=1)
                                 ->  ProjectSet  (cost=0.00..36151.70 rows=3501100 width=32) (actual time=764.441..9605.530 rows=4080666 loops=1)
                                       ->  Parallel Seq Scan on records_1000  (cost=0.00..15145.10 rows=350110 width=24) (actual time=764.413..2245.715 rows=840264 loops=1)
 Planning Time: 12.526 ms
 JIT:
   Functions: 180
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 13.783 ms, Inlining 247.202 ms, Optimization 1459.844 ms, Emission 942.922 ms, Total 2663.752 ms
 Execution Time: 1569794.354 ms
```
This query gives me a nice list of something like this 
| index   | Count        | ASN     |
|-------  |----------    |------   |
| 0       | 19309411     | 6939    |
| 1       | 18351608     | 3356    |
| 2       | 16863414     | 1299    |
| 3       | 11309279     | 174     |
 
Just eyeballing this, it looks reasonable, 6939 is HE, 3356 level 3, 1299 Telia and 174 Cogent, which does nicely look similar to [Caida](https://asrank.caida.org/).
 
I wrote a quick python script to query this and used my [HailBlazer](https://github.com/WhatATragedy/HailBlazer) endpoint to enrich with ASN names. Which produced this.
 
| Index   | Count        | arrayaspath  | name                                                 |
|-------  |----------    |------------- |-------------------------------------------------     |
|     0   | 19309411     |        6939  | HURRICANE                                            |
|     1   | 18351608     |        3356  | LEVEL3                                               |
|     2   | 16863414     |        1299  | TELIANET Telia Company AB                            |
|     3   | 11309279     |         174  | COGENT-174                                           |
|     4   | 10541619     |        2914  | NTT-COMMUNICATIONS-2914                              |
|     5   |  9007172     |        3257  | GTT-BACKBONE GTT Communications Inc.                 |
|     6   |  6461046     |       58511  | ANYCAST-GLOBAL-BACKBONE Anycast Global Backbone      |
|     7   |  5910483     |        6453  | AS6453                                               |
|     8   |  5310383     |         209  | CENTURYLINK-US-LEGACY-QWEST                          |
|     9   |  4244612     |        7545  | TPG-INTERNET-AP TPG Telecom Limited                  |
|    10   |  4209766     |       39120  | CONVERGENZE-AS Convergenze S.p.A.                    |
|    11   |  4141897     |        6461  | ZAYO-6461                                            |
|    12   |  3980431     |        3491  | BTN-ASN                                              |
|    13   |  3589007     |        6762  | SEABONE-NET TELECOM ITALIA SPARKLE S.p.A.            |
|    14   |  3257376     |       16552  | TIGGEE                                               |
|    15   |  3248613     |        6830  | LibertyGlobal Liberty Global B.V.                    |
|    16   |  3089704     |        4637  | ASN-TELSTRA-GLOBAL Telstra Global                    |
|    17   |  2834779     |         293  | ESNET                                                |
|    18   |  2562372     |       52320  | GlobeNet Cabos Submarinos Colombia, S.A.S.           |
|    19   |  2515635     |      199524  | GCORE G-Core Labs S.A.                               |
|    20   |  2502649     |       42541  | FIBERBY Fiberby ApS                                  |
 
 
Nice, the table looks okay and comparing it to Caida, the top ASNs are definitely there. I've got a few issues here, like Hurricane Electric being so high on the graph but the reason for this is probably because they have a lot of BGP peers, but not a lot of throughput. There is also ASN 58511 (Anycast-Global) but this is probably because CDN and anycast providers advertise to a lot of upstreams in order to ensure availability of service. This means they will be in a lot of ASPATHs.
 
 I would much rather use traceroute data to create a table like this as it will show path actuality for various prefixes instead of reachability. But that is for another blog!
 
 

