---
title: DGA - Domain Turtle For Malwares
date: 2021-02-25 02:55 +03:00
tags: [domain generation algorithm, dga, malware dga, dga implementation, what is the dga, pseudrandom generation algorithm, pseudorandom dga, dictionary based dga]
description: Domain generation algorithm is a most popular techniques last times by used common malwares. I explained this technique from technical persective.
image: "/assets/img/dga_domain_turtle/img/cover.jpg"
toc: true
---

Domain generation algorithm is a most popular techniques last times by used common malwares. This technique was used by Conficker, Murofet, BankPatch and more malwares. I will explain you how domain generation algorithm work from technical perspective in this blog post.

## What is the Domain Generation Algorithm (DGA)

Domain generation algorithm is liberation way for the malware authors. Before to DGA, malware required connect to command and control server for receive and send own commands. But there was one problem, command and control server of the malware is a was one. If security researcher or malware protection product catch this domain of the command and control server, would immediately block it. And malicious operation ended for malware and malware author. Because he/she will lost connection to malware and never send any command. This problem is cause point of the DGA emerge.

## Purposes of Use 

Malicious actor, not want provide only communicate between server (C&C) and client (malware). He/She wants to use it another few way in addition. 

- Confusing the perception of network based protector by genereate many DNS request.
- Hiding their activities.
- Increasing malware lifetime through dynamically worked Command Control mechanism. 

## What is underlying fundemantal logic of the DGA? 

I describe in the previous section why malware authors needed improve communication skills of the malware. Now, I explain you how it work DGA.

DGA is a cryptographic algorithm unified string crafter. Mainly, it's need to one thing, meeting date. The meeting date means, communication point of the C&C server with malware. 

Usually meeting date consist of three or six component. The three component is: 
- Year 
- Month
- Day

And three of the six components is same with above, but in addition:
- Hour
- Minute
- Second

The malware author if want per day one connection attempt, will choose the first three component techniques. But security products very strong nowadays, hence resulted domain of this technique, will be very quickly blocked. We can easily see which one is more reliable by looking at the result. Of course, has six components model more reliable, at the same time very harder for security product. The attacker be able to generate **millions of domain** in just a day with six component model like a turtle which transport own house. Therefore; security products, security researchers or security companies can't block all of them. Now that we're talking about meeting point models, we can talk about attacker how generate domains and what is the type of domains. 

## What is the create domain generate types? 

Domain generation algorithm have a few types. I will mention pseudorandom algorithm and dictionary based algorithm types. You may will find other few algorithm in the web.

### Pseudorandom Algorithm 

The main logic behind the pseudorandom algorithm, generate domains randomly selected by characters in alphabet. Here is a example domain of character based random algorithm: **vkfjsnfkclsmoj[.]com**

#### Let's Implement Pseudo-Random Algorithm 
Pseudo-random is a cryptographic algorithm which also implement in DGA. Main concept is based on bit shifting and goal of randomness. However, we're not saying that actually provides truly randomness because algorithm needs to initializer value, and this value generally time of system.

<script src="https://gist.github.com/fatihsnsy/5c3b591250c820aef5d2dbba958b584a.js"></script>

I customize known pseudorandom algorithm little bit. My goal is to time-sensitive domain generation. When I say time, I am talking about the hour, minute and second. Thus, I added this values to day, month and year values. However, my math is not good for real randomness so, I targeted randomness time based sensitive and I think success it but I don't allegate ensure to the pseudorandom algorithm conditions. Decision is yours. 

In my example, I determine the meeting point as **25/02/2021 17:54:23** and set domain length 15 characters. And result is: **syqbyrgonyhdyqn[.]org**

### Dictionary Based DG Algorithm 

In this case, we have the string array defined earlier. The main purpose of this technique is to make it difficult for AI or ML based security products to detect DGA by creating different domains other than those created with random characters. Domains generated with dictionary based algorithm will appear legal and the security products and security engineer observing network inspect tools will not be suspicious. Because this domains looks like this: catinthefridgefloor[.]xyz or worldcarsrealdonut[.]com.

#### Let's Implement Dictionary Based DG Algorithm 

Firstly, should I say this technique mixed with pseudo-random algorithm. We have need randomly created integer number as use dictionary array index. So we supply random index number with pseudo-random algorithm. In other hand, we need to determine how many string combined as domain. 

<script src="https://gist.github.com/fatihsnsy/8074b79538ca7dbac5479dbdc4934858.js"></script>

If we talk about this example, I determine the meeting point as **25/02/2021 11:07:34 **and set domain length is 4 string. Result is: **timeatgameworld[.]io**
Dictionary based algorithm have sometimes collisions in this instance. Because our dictionary is very limited. If you would test this algorithm with large dictionary, you're probably will getting better result then me.

Today, we talked about how DGA's work and what is this types from technical perspective. I'm waiting your thoughts in comment section. I wish you healthy lifes :) 

## References 

[1] https://en.wikipedia.org/wiki/Domain_generation_algorithm
[2] https://zvelo.com/domain-generation-algorithms-dgas/