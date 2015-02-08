---
layout: post
author: gsagie
title: NFV Acceleration (Part 3)
date: 2015-02-08 16:25:06 -0700
categories:
- SDN
- NFV
- Virtualization
tags:
- SDN
- NFV
- DPDK
- WindRiver
- 6Wind
- 6WindGate
- OVP
- OVS
---

In [part 1](http://galsagie.github.io/sdn/nfv/virtualization/2015/01/21/nfv-acceleration/) of this series i wrote about user space drivers and in particular about DPDK, in [part 2](http://galsagie.github.io/sdn/nfv/virtualization/2015/01/25/nfv-acceleration-2/) i wrote about pass throught technologies (standard/SR-IOV) and HW offloading to the NIC.
In this post i am going to describe commercial offerings that combine all the above techniques and more.

Disclaimer !!! i used to work for WindRiver on the networking part of the below described product, i do however have a strong respect to 6Wind which are a direct competitor.
I will therefore keep this post objective with information that can be found online, open to everyone.
If i got anything wrong here, or anyone would like to add, feel free to comment or send me an email and i will fix it.
The intention of this post is to familiarize the readers with whats out there for use.

## 6Wind - 6WindGate and Virtual Accelerator

In the 6WINDGate software architecture, the control plane and data plane are separate,
Within the data plane, the 6WINDGate fast path runs isolated from the Linux operating system on a dedicated set of processor cores.

The fast path protocols process the majority of network packets without incurring any of the Linux overheads that degrade overall performance. The fast path implements a run-to-completion model whereby all cores run the same software and can be allocated as required according to the necessary level of packet processing or Linux application performance.

Packet processing information that is configured or learned (through control plane protocols) in Linux is automatically and continuously synchronized with the fast path so that the presence of the fast path is completely transparent to Linux and its applications.

This synchronization allows the user a transparent experience when using the fast-path, which means it looks just like a normal linux machine with all the same  management  applications. (this is a crucial problem when using DPDK stand alone).

6Wind has created its own distributed/multiprocessor networking stack in order to accommodate with performance needs and that stack is used in the fast-path.
Usually network stacks in common Unix/Linux distribution are aimed at being RFC compliant and handle all the possible small standard corner cases of every possible protocol.
This behaviour limits their design and prevent them to achieve the target performance results, that is the reason why 6Wind decided to redesign the stack.

Of course that such an approach also comes with a big overhead, the linux networking stack is constantly worked on by many many developers around the world, it is tested and verified by many users and has a big community.
Writing a proprietary stack instead means that 6Wind will have to update and maintain it, patch and add any new features to stay compliant as much as possible to "standard Linux".
I am not sure how exactly is this implemented inside 6WindGate, but being able to easily leverage/add the upstream community work and changes is a an important point. 

6Wind uses hardware acceleration drivers/libraries to implement its fast-path, in particular Intel DPDK which we talked about in earlier posts, but the fast-path API's are generic and can use other hardware libraries like Cavium/Broadcom and Tilera.

6Wind offers an accelerated virtual switching platform that enable use cases for NFV. the platform runs within the hypervisor layer and also in the VM (if your virtual appliance uses DPDK or is build on top of 6Wind SDK).
This provides accelerated virtual switching using openvswitch, which offload some of its features to the fast-path described above.
This platform is synced seamlessly with linux control plane and openvswitch control plane and provides special virtual interfaces for accelerating forwarding and service chaining use cases. (mainly using shared memory regions to introduce zero copy between the host, virtual switch and the VM)

I believe that the target customers of 6Wind were companies building products for NFV and virtualized environments that wanted to run on COTS hardware and leverage 6Wind SDK, but some of their recent activities and products like the Turbo Router/IPSec and the Virtual Accelerator makes me think they are starting to target infrastructure customers (data centers).

This is of course my opinion only, but it sounds to me there is a lot of value for service providers by improving a given infrastructure in a transparent way.

<img src="http://www.6wind.com/blog/wp-content/uploads/2014/01/6WINDGate-10x-performance.png"/>


## WindRiver - OVP / Titanium Server

Windriver OVP (Open Virtualization Platform) has a networking part that is called INP (Intelligent Networking Platform) which also offers a proprietary distributed/multi-core network stack and synchronization integration with the Linux control plane.
The running model is similar to the previous solution.

OVP also leverage Intel's DPDK and address NFV and service chaining acceleration use cases.
Since i dont want to get into features comparisons and benchmarking i will state some of the different aspects that are provided by WindRiver as opposed to the previous solution.

OVP is based on Wind River RT Linux distribution, OVP can work on other distributions as well but a complete solution can benefit from RT linux distribution as we will soon see (of course that 6Wind solution can also works on this Linux distribution, but one company that offers both combined has its many advantages from support to better integrations).

The use of RT Linux in NFV is important at certain use cases, especially when trying to achieve carrier-grade performance and characteristics.
For example interrupt latency is an important factor when virtualizing mobile network components, it has a huge gap in normal linux from the needed standards, Windriver RT Linux address this issue among others.

WindRiver also offers an accelerated OpenVSwitch which is based on a proprietary implementation (based on DPDK vSwitch open source project), this as opposed to 6Wind which offers a more transparent experience with their solution. 
The important factor here  in my opinion, what is the performance gap, if it exists, between the two solutions. keeping the experience transparent and not re-inventing the wheel has many advantages as long as it doesnt compromise too much of needed performance (same problems with the networking stack we talked earlier).
Implementing the network functions in software provides flexibility and rapid way for innovation with dynamic fast changes combined with an open community.
All of this can be one of the major stimulator for NFV, we dont want to block this by software infrastructure which is not standard.

In addition OVP offers carrier grade OpenStack distribution and accelerated KVM, it also has SDK integrations with DPI and pattern matching frameworks by 3rd-party vendors.
Windriver combined all the above and is creating a full ecosystem of partner integrations in a project called ["Titanium Server".](http://www.windriver.com/products/titanium-server/).

INP (Networking part inside OVP):
<img src="http://www.windriver.com/products/platforms/intelligent-network/inp-diagram.jpg"/>


