layout: post
title: NFV Acceleration (Part 1)
date: 2015-01-21 16:25:06 -0700
categories:
- SDN
- NFV
- Virtualization
tags:
- SDN
- NFV
- DPDK
- PF_RING
- netmap
- OPNFV
---

NFV is certainly not a term needed to be explained in this point of time, feel free to google it and understand what it is if you never heard about it.

There are various efforts that are currently taking place in regards to NFV, important one is by trying to standardize the use cases of deploying and managing virtual network functions, [OPNFV](https://www.opnfv.org) is an important project in that aspect and i will sure write about it as it matures.
Of course that another one is the [ETSI NFV Group](http://www.etsi.org/technologies-clusters/technologies/nfv)

In this post series i want to focus on another important effort, improving the NFV performance and QoS aspects, hopefully reaching to carrier-grade performance/availability (or close to it) that is obtained by the physical devices.
This gets more crucial as virtualizing network functions is happening in the mobile network where specific standards and latencies must be met.

There are a lot of networking experts that doubt virtual appliances will ever reach a level of service good enough, on this post i will leave these arguments and focus on what are the current options to accelerate virtual appliances in production.

### User Space Network Acceleration Drivers (DPDK, PF_RING, Netmap)

DPDK is a project i have been involved in quite closely.
The main goal of DPDK is to provide a simple, complete framework for fast packet processing in data plane applications.
DPDK utilizes standard x86 and Intel NIC's but has support for other vendors (like Mellanox) and is starting to be ported to others as well.

The main idea is to create interrupt free run to completion model for packet processing in user space (by mapping the NIC packet buffers directly to userspace).
It also provide a set of libraries for efficient memory allocations (using hugepages) and memory pools creation , high precision timers  and more importantly multi core processing.
DPDK also includes virtualization support and virtualization network driver support. 

OpenVSwitch support DPDK as a data path option in the upstream code base, this suppose to provide an accelerated OVS functionality, all running in user space.
(These two were a separate projects which got merged recently).

A major effort done in that project is to address the use case of service chaining inside the hypervisor by reducing the data copy of packets between the VM's and the virtual switch (using shared memory between the host and the VM's)

It is important to remember that there is still a big overhead when using DPDK, DPDK has no network stack and actually unbinds the interfaces from the host linux kernel.
This means the interfaces used by DPDK can not be managed and manipulated using standard linux tools and API's.

DPDK offers an "exception-path" solution of injecting traffic to the host network stack but this solution limits the performance of DPDK and multi core processing due to bottlenecks in the network stack.
In later posts in this series i am going to write about other commercial solutions to this from companies like 6wind and windriver.

There are many other features in DPDK and i intend on writing more about building applications using DPDK and about OpenVSwitch DPDK in the near future.

Other frameworks which pretty much focus on the same aspect of DMA'ing packet buffers to user space are PF_RING and netmap, in my opinion they lack the richness/performance of DPDK but are also important to keep in mind.

These user space networking acceleration frameworks are helping virtual appliance creators to improve performance running on COTS servers and building efficient infrastructure.

There is another aspect which appliance vendors like when it comes to creating drivers in user space, and thats licensing.
Linux kernel is GPL, which forces drivers to be open sourced, moving things to user space remove this restriction and is a very charming aspect for vendors trying to hide their secret sauce.

I am not sure how this issue affect the openness needed to build networks in the SDN and NFV era, but i am pretty sure we are going to see more and more code moving to user space.

In the next post i am going to write about accelerating NFV's by bypassing the hypervisors layer with technologies like SR-IOV / Pass Through, and HW offloading solutions.

