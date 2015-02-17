---
layout: post
author: gsagie
title: NFV Acceleration (Part 4)
date: 2015-02-17 16:25:06 -0700
categories:
- SDN
- NFV
- Virtualization
tags:
- SDN
- NFV
- Tilera
- EZchip
- Mellanox
- NSX
- OVS
- OpenStack
---

If you are new to this series, make sure to check out the previous parts first, 
[part 1](http://galsagie.github.io/sdn/nfv/virtualization/2015/01/21/nfv-acceleration/), [part 2](http://galsagie.github.io/sdn/nfv/virtualization/2015/01/25/nfv-acceleration-2/) and [part 3](http://galsagie.github.io/sdn/nfv/virtualization/2015/02/08/nfv-acceleration-3/).
I am writing about techniques and solutions to increase performance for virtual network functions.
In the previous post i wrote about commercial software based solutions from Windriver and 6Wind.

In this post i am going to describe solutions that combine commercial hardware but are still trying to address “standard” NFV use cases, and an approach to offer the functions as services by distributing them in the infrastructure.

## Tilera

Tilera Corporation is a fabless semiconductor company focusing on scalable multicore embedded processor design.
Tilera's primary product family is the Tile CPU. Tile is a multicore design, with the cores communicating via a new mesh architecture, called iMesh, intended to scale to hundreds of cores on a single chip.

It is important to mention that Tilera is now owned by EZchip (our next topic), which shows how a hardware company like EZchip adjusts to the new era of SDN and NFV.

The interesting project from Tilera that is aimed at improving NFV performance is the TILE-OVS.
An optimized Open vSwitch (OVS) offload solution for network functions virtualization (NFV) deployments in data center and telco networks. Powered by Tilera's TILE-Gx(TM) manycore processors and deployed in a PCI Express form factor, the solution delivers up to 80Gbps of OVS processing with additional headroom to spare for many other sophisticated networking applications such as deep packet inspection (DPI), network analytics and cyber security processing.

Very interesting approach, however one of the greatest advantages of OVS is the fact that its written in software and is very dynamic for changes, new overlays protocols (like GENEVE and Generic UDP Encapsulation) and features are added regularly to OVS (or planned to be added).
I am not sure what it takes to update the hardware solution with a new OVS software, i have a feeling its currently not an easy/possible process.

## Mellanox eSwitch

Mellanox has a rich portfolio when its comes to SDN and NFV solutions, from open source packages like Accelio/VMA/DPDK driver to SDN Controller (Unified Fabric Manager) which i plan on writing about in next posts.

The reason why i mention them here is the embedded switch in their NIC, this provide a similar solution to the above OVS offloading, this time to the NIC.

The eSwitch is a great solution which provide virtual ports / virtual NICs definitions for HW based VM traffic switching in addition to QoS and DCB.
The supported match fields include VLAN, MAC Address, Ether type, Src/Dst IP address and TCP/UDP ports (not sure about tunneling, but i am sure it is planned if not already there)

The eSwitch support few actions based on these match fields which include: Allow, Drop, Count, Mirror and Priority setting.  

## EZchip NP-S

EZchip main business is its network processors family, which are (or at least were) extensively used by Cisco.
The reason why i am writing about them here is because of a very interesting product which is not out yet (planned later this year) but is promising to be a base architecture for SDN and NFV use cases.

“The NPS – Network Processors for Smart networks – will bring maximum flexibility through C‑based programming, Linux OS and full 7-layer processing, as well as integrated traffic management, security and DPI hardware acceleration.”
(Taken from EZchip official website)

In short, NPS tries to provide a standard x86 server look (Linux / C programming) with the acceleration engines of a network processor.
You should visit [NPS-400 specification webpage for more details](http://www.tilera.com/products/?ezchip=598&spage=603)

There are many questions yet to be answered in regards to NPS, From how really compatible is its Linux OS to standard Linux (could you take binary applications and run them without modification) To how its concurrency and threading model really works.
Of course another interesting point is the cost/performance ratio between the NPS and a standard accelerated server.

## Distributed Network Functions

This is a trend that i believe we are going to see more and more, and in my eyes capture some of the advantages of SDN.

The usual deployment of a virtualized network function is to take the function and put it in a virtual machine, then traffic can be steered from/to it just like a physical appliance.
This approach gives the orchestrator great flexbility, however introduce performance problems as these VM’s start to become bottlenecks in our traffic path.
(In addition to the fact that we now must design load balancing and HA for these virtual appliances)

A new approach is to distribute the function between the infrastructure nodes (which are currently our hypervisors/host machines) and orchestrate them with the centralized SDN controller.
This eliminate the function point in the traffic path and allows us to handle traffic loads in a distributed ways.

Notable projects in this area are:

* VMware NSX distributed firewall (which is working on integrations with other security vendors like Palo Alto Networks and F5).

* Distributed Routing - DVR project in OpenStack (also exists in VMware NSX)

* DPI engine from Qosmos which can be integrated with OVS or other infrastructure to create a DPI as a Service in the infrastructure.

* Midokura Midonet is a complete distributed SDN solution (as far as i know and read about, i do plan to write another post specifically about MidoNet)

The important points to keep in mind, at least in my eyes, are the overhead of the function sitting in each and every node of the infrastructure and the management/orchestration complexity of such solutions which require specific integrations (which are not standard as opposed to running the function as a VM).
The benefits i believe are quite obvious.

This series is getting pretty long and there are still many topics i want to cover, i have decided to split it to specific sub-series or posts, topics like Performance in OpenStack or DPDK pitfalls and NFV interesting solutions (OPNFV projects, ConteXtream) and so on..

Stay tuned..
