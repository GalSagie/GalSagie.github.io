---
layout: post
title: OVN and Distributed Controller
date: 2015-01-16 16:25:06 -0700
categories:
- OVS
- Virtualization
tags:
- SDN
- OVS
- NFV
- OVN
---
 

If you havent heard about it until now, make sure to read the following blog post:
[OVN, Bringing Native Virtual Networking to OVS.](http://networkheresy.com/2015/01/13/ovn-bringing-native-virtual-networking-to-ovs/)
And the [more detailed architecture design of OVN in OVS offical documentation.](http://openvswitch.org/pipermail/dev/2015-January/050380.html)

The more relevant and interesting phrase, in my opinion, from that entire blog is this:

“Open vSwitch is the most popular choice of virtual switch in OpenStack deployments. To make OVS more effective in these environments, we believe the logical next step is to augment the low-level switching capabilities with a lightweight control plane that provides native support for common virtual networking abstractions.”


I have seen a lot of posts regarding this, and it certainly a very interesting project to look for, but i want to tackle this from few interesting views:

## Policy

Reading this architecture reminded me a lot of something else, it reminded me the way [Cisco OpFlex](http://www.cisco.com/c/en/us/solutions/collateral/data-center-virtualization/application-centric-infrastructure/white-paper-c11-731302.html) is communicated between the controller and the agent running on the end nodes.

Similar or not, in my view SDN implementors are starting to realize that we tried to abstract the control from the data plane using only protocols, and thats just not enough.

By having an agent / controller / lightweight controller / you name it running on the infrastructure at every end point (hypervisors and physical devices) we can concentrate on two things:

* Building a  common policy language/protocol/API between all IT teams, which is application aware and can be distributed to all the network infrastructure nodes. 
Without having to worry how every infrastructure piece is going to implement it
(Few interesting projects in that area are OpFlex and Congress)

* Building protocols/standards (many) for locally applying the above policy to software/hardware
(For example OpenFlow)

Having a "control" entity in the end nodes can show value in other areas (hint OAM, wait for my next post).

There are some solutions already fully distributed, running on the end nodes, [Midokura MidoNet](http://bradhedlund.com/2012/10/06/mind-blowing-l2-l4-network-virtualization-by-midokura-midonet/) is one of them.

## Virtual Network Function Distribution

The trend of NFV is already here, virtual appliances for network functions are already
deployed and used in the data center, especially for east-west traffic.

The way they are used today is usually as a VM / dedicated node that is used as a logical gateway edge appliance where all traffic goes through.

This approach proves to cause bottlenecks and we are seeing more and more solutions that try to distribute these network functions to the infrastructure end nodes (more specifically the hypervisors).

Example projects are OpenStack DVR, and NSX micro segmentation approach (distributed firewall).

In my opinion this new approach will be used more and more, having a control entity in the hosts can certainly help manage these functions.


## Context
This relates to the last subject but i feel is important enough to stand on its own.

The hypervisor is the only place where application meets the network, we can not extract this value or correlation context anywhere else in the network.

physical devices usually try to apply sophisticated (performance expensive) DPI methods to extract this context.

There are many use cases where a smart control entity (sitting in the hypervisor) can use this information to make better decisions.


