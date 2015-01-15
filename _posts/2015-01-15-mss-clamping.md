---
layout: post
title: MSS Clamping As A Tale Of Network Visibility 
date: 2015-01-15 16:25:06 -0700
categories:
- OAM
- Virtualization
tags:
- SDN
- OAM
---

Path MTU Discovery is a standardized method of finding the maximum transmission unit (MTU) between two IP hosts. 
(IP fragmentation usually introduces latency and is a performance bottleneck which we want to avoid even in IPv4)

For IPv4, Path MTU Discovery works by setting the "Dont Fragment" (DF) flag in the IP header and sending the packet to the other host. 

Any device along the path whose MTU is smaller than the packet will drop it, 
and send back an ICMP Fragmentation Needed message containing its MTU, lowering the MTU and 
continuing this process until reaching our end host lets us adjust the MTU of the path.

This approach is nice in theory but still has a lot of issues, 
mainly the fact that security devices along the path tend to block ICMP messages.

In order to overcome this, TCP Clamping is introduced.

The way TCP Clamping solves this is by leveraging the MSS option in the TCP header, 
for each SYN packet, each device along the path, with TCP Clamping enabled, sets the packet MSS size to be adjusted according to its MTU. 

By the time the packet is reached at the end host, the MSS can represent the MTU of the path.
(There are some consideration when applying MSS Clamping on a PE Router, for example
DDoS attacks, You can get more information [here.](http://media.blubrry.com/ipspace/www.ipSpace.net/nuggets/podcast/X1%20TCP%20MSS%20Clamping.mp4)

Why is this interesting you ask or even relevant to the topic?

First, because MTU issues are something that operators fail with when deploying networking virtualization solutions that uses overlay tunneling.

But more importantly, this remind me that even in today's SDN time we still don't have 
full visibility of the path and the devices along the path between two end points in our network, even inside our data center.

Of course some solutions which integrate SDN software and hardware claim they do, 
but when you start integrating network devices and functions from various vendors, 
you find out we still have a long way to go until everything is integrated with the controller. 

Solutions like MSS Clamping solves this in a distributed way, in the next posts I am going
to describe some of the challenges of understanding the logical and more importantly, 
physical path between two VM's in the data center and different approaches to handle these challenges.

When virtual and physical networking start to mix, root cause analysis of the problem is getting a lot more important, 
OAM aspects needs to be thought of, and in my opinion starting now.

Stay tuned for the next post...



