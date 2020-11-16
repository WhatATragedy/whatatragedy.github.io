---
layout: article
title: Who Has Your Advertising Data
date: 2020-07-08 09:39:00+0200
coverPhoto: /contents/images/2020/07/DatabutBig.PNG
---
![](/contents/images/2020/07/DatabutBig.PNG)
## Introduction
During my time working from home, I was browsing [Hacker News](https://news.ycombinator.com/), it's a great site that has some great information on it if you can get over the hideous UI. I stumbled on something called 'Malvertising', the post I came across is a few months old and I have only just got around to doing a writeup, so I've since lost it. You can read about it here though [https://blog.eccouncil.org/malvertising-what-it-is-and-how-to-avoid-it/](https://blog.eccouncil.org/malvertising-what-it-is-and-how-to-avoid-it/). 

Reading this sparked my interest, who was serving the biggest Ad Exchanges on the internet? We know about Google and Facebook, but who are the other players?

## Real Time Bidding Data
So I began to dig into the world of Real Time Bidding and how it works. If you just google the phrase 'Real Time Bidding' you'll come across a lot of posts, there is a nice wiki page though [Wiki](https://en.wikipedia.org/wiki/Real-time_bidding). 

The 101 to Advertising data is this. There are things called Advertising Exchanges, websites that put advertising on their page, will use an advertising exchange to serve these adverts. When you sign up to google advertising on your website, you embeed a bit of googles code that decides what advert to serve to the user when they visit the site. When a user visits the site and this little bit of google code executes, it asks all of its 'buyers' who wants to serve the advert, a buyer can be any company that want's to serve ads to people, Spotify, Levis, Amazon etc. 
Here is a digram of that happening: 
![](../../../contents/images/2020/07/rtb_diagram.PNG)

In this example, I visited 'newsnow.co.uk', this is just a news aggregator. Newsnow have signed up to serve adverts via two Advertising Exchanges, Rubicon and AppNexus. When I go to that site, AppNexus and Rubicon javascript executs to tell their buyers, someone is ready to serve to be served an advert. In this case, Rubicon tell Spotify and Amazon. Spotify and Amazon then decide if they want to serve this user an advert (based on cookies, location or other user data) or, in Amazons case here, it could not decide to serve and advert but ask some other buyers if they would like too. *This probably wouldn't happen with these companies because of size but the point here is it's hierarchical.*

So let's see what that looks like visually - here is me opening the webpage and seeing the element that is serving the advert.
![](../../../contents/images/2020/07/newsnow_rtb.png)

On the left, the highlighted part is the element I'm currently hovering over. On the right is the iFrame (Javascript Code) used to speak to Adnxs (AppNexus) in this instance.

## The Code...
Okay Great, now we know that, how do we map that to millions of websites. Well, there is a thing called domain ranks, some companies will publish the top domains based on a couple of metrics. I decided to go with cisco umbrella's top 1 million domains for this scrape. [Here](https://umbrella.cisco.com/blog/cisco-umbrella-1-million). My next action was to visit all of these pages, and see which ad exchanges were present there. Usually adverts will be embeeded in a page using a tags, iFrame tags and so on... Because I was only looking for the top ones, I just hoover all of these up and add them to a python list, the assumption here is if a link is specific to the page I visited, it should only show up with one count in the list anyway so will be insignificant. To do the scraping I used python3, requests and beautiful soup. It looked something like this.

```python
domain_and_method = 'https://' + 'newsnow.co.uk/'
user_agent = 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/35.0.1916.47 Safari/537.36'
headers = {'User-Agent': user_agent}
r = requests.get(domain_and_method, headers=headers)
soup = BeautifulSoup(r.text, 'html.parser')
a_tags_with_href = soup.find_all('a', href=True)
all_iframes = soup.find_all('iframe')
iframe_tags = soup.find_all('iframe', src=True)
script_tags = soup.find_all('script', src=True)
link_tags_with_href = soup.find_all('link', href=True)
```
### The Javascript Devil
This worked fine, but I realised I was getting very little results compared to when actually visiting the page on Chrome. For example newsnow.co.uk would only give me these results
```
['https://medium.com/newsnow/welcome-to-the-newsnow-redesign-925ccdf008f8']
['/ico/fav/apple-touch-icon.png?v=201612141100', '/ico/fav/favicon-32x32.png?v=201612141100', '/ico/fav/favicon-16x16.png?v=201612141100', '/ico/fav/manifest.json?v=201905031600', '/ico/fav/safari-pinned-tab.svg?v=201612141100', '/ico/nn.ico?v=201612141100', 'https://plus.google.com/104980600374081608265', '/scache/1_0_967c7c36bf090c2ff22de906e4107d15.css', 'https://use.typekit.net/mss5xmd.css', '/searchplugin.xml']
['https://www.googletagmanager.com/ns.html?id=GTM-WR53CPM']
['/scache/8_0_9138a5b5ff37383283274bacd8b02ac0.js', '/scache/33_0_da00391ce930836c4c3f8c435acc3848.js', 'https://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js', '/scache/2_0_2648055b1326fe5fecf1323a6e4f718b.js']
```

but this is nowhere near the amount of a, iFrame etc tags I would see visiting the page. I gave it some thought and some googling and here is why. All of the webpages are now dynamically loaded, meaning when you view it, it loads content a few seconds after (after the code has JS code has run). Request will provide you a snapshot and doesn't wait for all the content to load. So I needed to actually use something to load the page for a few seconds before scraping it. From here I switched out the requests library, for the selenium library.
Selenium would let me use the chromium driver to request the page, load all of the javascript, and then return the content for me to scrape. Great. My new results were now this.

```
['https://medium.com/newsnow/welcome-to-the-newsnow-redesign-925ccdf008f8']
['/ico/fav/apple-touch-icon.png?v=201612141100', '/ico/fav/favicon-32x32.png?v=201612141100', '/ico/fav/favicon-16x16.png?v=201612141100', '/ico/fav/manifest.json?v=201905031600', '/ico/fav/safari-pinned-tab.svg?v=201612141100', '/ico/nn.ico?v=201612141100', 'https://plus.google.com/104980600374081608265', '/scache/1_0_967c7c36bf090c2ff22de906e4107d15.css', 'https://use.typekit.net/mss5xmd.css', '/searchplugin.xml']
['about:blank', 'https://www.googletagmanager.com/ns.html?', 'https://eu-u.openx.net/w/1.0/pd?plm=6&ph=', '//acdn.adnxs.com/ib/static/usersync/v3/async_usersync.html', 'https://vars.hotjar.com/box-469cf41adb11dc78be68c1ae7f9457a4.html']
['https://www.google-analytics.com/plugins/ua/linkid.js', 'https://www.google-analytics.com/analytics.js', 'https://static.criteo.net/js/ld/publishertag.prebid.js', '//ib.adnxs.com/', '//ib.adnxs.com/jpt?callback=pbjs.handleAnCB&referrer=https%3A%2F%2Fwww.newsnow.co.uk', 'https://as-sec.casalemedia.com/cygnus?v=7&fn=cygnus_index_parse_res&s=345916&', 'https://static.hotjar.com/c/hotjar-1692966.js?sv=7', '//c.amazon-adsystem.com/aax2/apstag.js', 'https://www.googletagservices.com/tag/js/gpt.js', 'https://www.googletagmanager.com/gtm.js?id=GTM-WR53CPM', '/scache/8_0_9138a5b5ff37383283274bacd8b02ac0.js', '/scache/33_0_da00391ce930836c4c3f8c435acc3848.js', 'https://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js', '/scache/2_0_2648055b1326fe5fecf1323a6e4f718b.js', 'https://adservice.google.co.uk/adsid/integrator.js?domain=www.newsnow.co.uk', 'https://adservice.google.com/adsid/integrator.js?domain=www.newsnow.co.uk', 'https://securepubads.g.doubleclick.net/gpt/', 'https://script.hotjar.com/modules.cf522d0ae101e277829e.js']
```
That's exactly what I wanted to see. we can see google-analytics, adnxs.com, securepubads and so on.
Now I needed a way to classify if a link was an advert or not. I didn't want to build a website classifier or use a pre-existing API as it would be too slow. Luckily for me, I've previously set up pihole for my home WiFi. PiHole is a blackhole to advertising domains, basically it blacklists DNS requests for them. I thought if a domain was blacklisted on the PiHole domain list, it was probably an advertising domain so I just had to see if the domain was in that list! The lists I used are located here : [https://github.com/hectorm/hmirror](https://github.com/hectorm/hmirror)

## Results

So now I had a way of querying the top million domains, adding all of their external links to a list, and checking it against known advertising domains. So that's exactly what I did for 100,000 of the top websites according to Cisco Umbrella.
So what was the answer... Well here are the top 20.


**Advertising Domain**|**Count**
:-----:|:-----:
www.google-analytics.com|652
www.googletagmanager.com|559
www.googleadservices.com|214
googleads.g.doubleclick.net|185
bat.bing.com|105
ssl.google-analytics.com|93
analytics.twitter.com|85
static.ads-twitter.com|79
munchkin.marketo.net|77
adservice.google.com|68
js-agent.newrelic.com|56
securepubads.g.doubleclick.net|54
www.googletagservices.com|52
hm.baidu.com|45
cdn.optimizely.com|39
cdn.bizible.com|38
script.crazyegg.com|36
tags.tiqcdn.com|34
js.adsrvr.org|34
sb.scorecardresearch.com|34

There is still more work that needs to be done, if the domain doesn't start with http or https I drop it, but as seen in the output from news now that elimates some results '//ib.adnxs.com/' and there is no facebook here, I don't know where their ad exchanges are hiding.

Anyway I thought it would be a good read. When I makes some changes to the code and make it a little more readable, I will make it useable by everyone. I always need to add async support so I don't have to wait for a domain to finish before moving on, that will speed up this process a lot, I could then probably do a million instead of capping out at 100,000.

Thanks for reading. Alex







