---
layout: post
author: gsagie
title: NFV Acceleration (Part 2)
date: 2015-01-25 16:25:06 -0700
categories:
- SDN
- NFV
- Virtualization
tags:
- SDN
- NFV
- DPDK
- SR-IOV
- Passthrough
- HW Offloading
- TSO
- STT
---

In a [previous post of this series](http://galsagie.github.io/sdn/nfv/virtualization/2015/01/21/nfv-acceleration/) i wrote about accelarating virtual functions using networking user space drivers, Intel's DPDK is one main example but others exists just as well.

In this post i am going to concentrate on by passing host layers and HW offloading, all meant to improve performance and use "standard" hardware to ease some bottlenecks in virtualized environments and networking functions.

## Passthrough and SR-IOV

It is possible to directly assign a host's PCI network device to a guest. One prerequisite for doing this assignment is that the host must support either the Intel VT-d or AMD IOMMU extensions. 
Doing this bypass the hypervisor and other switching processing done inside the host and hence improve the networking
performance of the guest (the virtual appliance).

There are two methods of setting up assignment of a PCI device to a guest:
 
### Standard Passthrough
 
This assignment allows virtual machines exclusive access to PCI devices for a range of tasks, and allows PCI devices to appear and behave as if they were physically attached to the guest operating system.
Basically exclusively assigning a NIC port to a VM.

### SR-IOV

SRIOV network cards provide multiple "Virtual Functions" (VF) that can each be individually assigned to a guest using PCI device assignment, and each will behave as a full physical network device. This permits many guests to gain the performance advantage of direct PCI device assignment, while only using a single slot on the physical machine.

Following is a picture that explains SR-IOV with Intel's DPDK

<img src="http://www.dpdk.org/doc/guides/_images/fast_pkt_proc.png" />

In the picture you can see that each VM has a VF that is identified by a separate VLAN id, what the NIC does is switch the packets into RX queues depending on the VLAN.
There is also a PF that is attached to the virtual switch demonstrating we can use SR-IOV with conjunction to "standard" virtual switching deployments.

You can read more about SR-IOV in [this detailed post blog.](http://blog.scottlowe.org/2009/12/02/what-is-sr-iov/)

It is clear that out of the two passthrough options, SR-IOV is much more flexible, however still has some disadvantages:

One clear disadvantage is the ability to configure the virtual function in a different way between the VM's (which might be of different tenants).
Think what happens if one of the VM's want to change the interface admin status.
Another disadvantage is the fact that SR-IOV still doesn't support live-migration of VM's (as far as i know) which is a key feature in virtualized environments.
But probably the major problem is still the lack of full support for SR-IOV in management and orchestration solutions for SDN and NFV.

SR-IOV can currently switch packets into VF's using L2 data (MAC address/VLAN) but i believe we are going to see much more flexible options coming soon when NIC's start offloading tunneling data (for example switching according to VXLAN).

Rather or not SR-IOV become a standard in NFV deployments is a good questions, i think it really depends on the performance gap compared to other more flexible and software based solutions.

## HW Offloading

HW offloading is a big terminology which probably deserve at least its own post (if not many more).
What i mean in our context is the ability of standard networking NIC's to offload all sort of functions that are else done in software and hence free more CPU cycles for the actual application.

Virtual appliances can leverage HW offloading to increase their performance, few examples:

* Filtering - NIC's support the ability to define specific/wildcard filtering on header information up to L4 (for example Intel's Flow Director).
This feature can help in implementing firewall and DPI functions.
Future NIC's might be able to achieve more than just L4 filters.

* Receive Side Scaling (RSS) - 
This feature allows a NIC to load balance traffic between different cores by hashing the traffic and splitting it into different RX queues.
Few instances of the same VNF can poll different RX queues and process packets simultaneously.
in DPDK you can retrieve the hash result (Toeplitz hash) on the packet meta data and use it in internal flow table implementation (instead of re-calculating hash again). 

* Tunneling Encapsulation/Decapsulation - 
Using the NIC to parse protocols and tunnels (VXLAN, NVGRE, VLANS) which are extensively used in virtualized environments. 

* TCP Segmentation Offload and Large Receive Offload - 
These two have great benefit for physical standard networking, offloading sequence numbering and packet header creation to the NIC (usually combined with checksum offloading as well) free CPU cycles from the networking stack / network function.

STT (which is an overlay tunnel protocol used in Nicira's SDN solution) leverage TCP Segmentation offload capabilities.

TSO is also useful for virtualization environments; TSO is a subset of the more generic segment offload (GSO) on Linux.
With virtio it is possible to receive GSO packets as well as send them. This feature is negotiated between the guest and host.

The idea is that between guests they can exchange jumbo (64K) packets even with
a smaller MTU. This helps in many ways. One example is only a single
route lookup is needed

Dont worry if you didnt understand all the various features i just described, the intention was to give you a sense of what is currently possible with standard NIC's and how it can be used.
I do plan on dwelling on each of these subjects later in this series and understand specific use cases ,limitations and correct implementation.

In the next post of this series i am going to describe some commercial solutions that combines all the topics we discussed about and more.
We are going to see software offerings from 6Wind and Windriver to accelerate NFV and achieve carrier grade deployments with standard software and devices, and a very interesting hardware offerings that fits into the NFV/SDN world like EZchip and Tilera.

Stay tuned..

