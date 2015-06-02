---
layout: post
author: gsagie
title: OVN L2 Deep Dive
date: 2015-05-30 16:25:06 -0700
categories:
- SDN
- Openstack
- OVS
tags:
- OVS
- OVN
- Openstack
- Openflow
---

I have written several posts about OVN, feel free to search them under the OVN category, in this post i will not cover the high level overview but rather a deep dive into the current L2 implementation and how it is expressed in Openflow flows in OVS.

OVN is an ongoing project, i will describe some “deficiencies” during this blog as they are currently implemented but bare in mind that work is being done to fix them as we go.

I wanted to mention that there is a great deal of work with the OpenStack integration and many new areas we are interested in implementing and exploring, if you are interested to contribute email me or join our freenode IRC channel.

In my setup i have a multi node OpenStack with OVN plugin.
I have deployed three VM’s one in the controller node (VM1) and two in the other compute node (VM2 and VM3) all connected to the “private” network (created by devstack).

The following diagram depicts the network topology as observed from OpenStack:

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/topology-ovn.jpeg" />

We can look at the OVN Southbound DB and see that it recognize the two nodes as chassis and created a geneve tunnel between them, it's important to note that the tunnel was created only when VM’s from the same logical network were actually deployed in the two nodes.
The Southbound DB also consist of two Encap table entries, one for each compute node.

OVN by default uses Geneve as the tunneling protocol, you can read more about it in [this link.](https://tools.ietf.org/pdf/draft-gross-geneve-02.pdf)

We can see the tunnel created by calling ‘ovs-vsctl show’ in each of the nodes:

```
0862065e-ad11-40b6-9fbf-0fcc679dcf43
     Bridge br-int
        fail_mode: secure
        Port "ovn-89f939-0"
            Interface "ovn-89f939-0"
                type: geneve
                options: {key=flow, remote_ip="10.100.100.15"}
        Port br-int
            Interface br-int
                type: internal
        Port "tap3517d591-10"
            Interface "tap3517d591-10"
        Port "qr-8b88cddb-cb"
            Interface "qr-8b88cddb-cb"
                type: internal
        Port "tapd57f6f68-53"
            Interface "tapd57f6f68-53"
                type: internal
        Port "qr-4232963c-0e"
            Interface "qr-4232963c-0e"
                type: internal
     Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port "qg-7c4f6205-93"
            Interface "qg-7c4f6205-93"
                type: internal

```

The tunnel port is created on br-int (as opposed to the default OVS L2 Agent where its created on br-tun), we can see that in current implementation br-tun is not even created.

We can also see the router namespace which connects between the private and public networks and the external port which all created by default from devstack.

The OVN Southbound DB Binding table has entries that link between the logical elements configured in the Northbound DB and their location in the physical infrastructure 

<pre-wrap>
[gal@galstack neutron]$ sudo ovsdb-client dump OVN_Southbound
Binding table
_uuid                                chassis                              logical_datapath                     logical_port                           mac                   parent_port tag tunnel_key
------------------------------------ ------------------------------------ ------------------------------------ -------------------------------------- --------------------- ----------- --- ----------
0fdcc477-5414-4770-bcd4-9d31d32ca58a 36cf1ffc-78fc-437d-8e6b-b2bf51caa3cf ff80b0bb-341f-45ab-b906-8c69566939b2 "3517d591-101d-43be-a390-bc7406c13251" ["fa:16:3e:dd:b9:26"] []          []  5         
27b5a02d-ceb5-4065-b5a7-52323337ee5c 36cf1ffc-78fc-437d-8e6b-b2bf51caa3cf ff80b0bb-341f-45ab-b906-8c69566939b2 "4232963c-0e8b-4851-b767-5cc6eac06078" ["fa:16:3e:b6:bb:4d"] []          []  2         
a82c6785-30d8-4b1c-a6b6-b835e656c558 []                                   19ddea15-1fa4-48ea-b917-98e362970060 "7c4f6205-9391-46fc-8cff-5b2abcca4efa" ["fa:16:3e:d6:87:a7"] []          []  3         
95d6be23-125b-429c-978f-a7be486732b8 36cf1ffc-78fc-437d-8e6b-b2bf51caa3cf ff80b0bb-341f-45ab-b906-8c69566939b2 "8b88cddb-cbf6-40e6-8a9f-6221a23e9e8f" ["fa:16:3e:e2:77:37"] []          []  4         
58faa5c5-c52c-4fb3-87a9-f54fb9ba24ee 36cf1ffc-78fc-437d-8e6b-b2bf51caa3cf ff80b0bb-341f-45ab-b906-8c69566939b2 "d57f6f68-53e5-4a7c-b0e6-770d15b34da2" ["fa:16:3e:26:57:ff"] []          []  1         
45b9ec63-2505-4bb4-a9a2-94c5ef020a36 08d56534-806d-4f49-a890-66368069bb3a ff80b0bb-341f-45ab-b906-8c69566939b2 "d5d580b9-a5ee-49d4-93e5-aa80279f7447" ["fa:16:3e:03:15:b0"] []          []  7         
2c9d67f4-7f7a-4e37-ae2a-0eb6454bcba1 08d56534-806d-4f49-a890-66368069bb3a ff80b0bb-341f-45ab-b906-8c69566939b2 "f905f46c-f1a2-4f99-8561-8dbc6558e5d6" ["fa:16:3e:91:24:4c"] []          []  6  
</pre-wrap>


In this table we can see all of our virtual ports with the ID of the chassis they are deployed in (we can see two unique ID’s for the two nodes) , the logical datapath or Neutron network that these ports are attached too (again two unique ID’s for the public and private network).

An important field to notice is a unique “tunnel_key” for each of the ports (1..7), we will soon see that this key is used in the configured Openflow flows and is actually meant to identify a specific port over a tunnel (for traffic across hypervisors), we can already see that ports from the same network have different tunnel keys.

Lets start observing the flows at each of the nodes, the OVN-controller installs an OpenFlow pipeline of tables that implements the virtual network connectivity according to the Southbound DB, for clarity we will go table by table and see how traffic flows.

I print the flows from the compute nodes where VM2 and VM3 are located for simplicity, but the same corresponding pipeline can be seen in the controller node

### Table 0 - Network classification and incoming tunnel traffic dispatching

```
 cookie=0x0, duration=75385.021s, table=0, n_packets=24, n_bytes=2503, priority=100,in_port=2 actions=set_field:0x1->metadata,set_field:0x6->reg6,resubmit(,16)
 cookie=0x0, duration=74748.204s, table=0, n_packets=20, n_bytes=2223, priority=100,in_port=3 actions=set_field:0x1->metadata,set_field:0x7->reg6,resubmit(,16)
 cookie=0x0, duration=75385.021s, table=0, n_packets=11630, n_bytes=1279320, priority=50,tun_id=0x6 actions=output:2
 cookie=0x0, duration=74748.204s, table=0, n_packets=11525, n_bytes=1267930, priority=50,tun_id=0x7 actions=output:3

```

We can see that the first flows are matched by the in_port field, these are flows for the local ports (VM’s or containers) on this host, they are marked with a metadata field and reg6 is set with the port unique identifier (tunnel_key) and resubmited to the next table in the pipeline (16).

The metadata indicate the logical datapath or network of the ports, the controller keeps an internal mapping of integers (as we can see only 1, as the two VM’s on this host from the same network) to the networks id’s.

The rest of the flows are matching traffic by the tun_id, this as i stated above used to match incoming traffic from another compute node and send it to the relevant destination port, the sending node marks the tunnel id to indicate exactly who is the destination.

(We can see that the id’s 6 and 7 represent the local ports in this compute node)

### Table 16 - Ingress Port Security

```
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=100,metadata=0x2,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=100,metadata=0x1,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=100,metadata=0x2,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=100,metadata=0x1,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=50,reg6=0x3,metadata=0x2,dl_src=fa:16:3e:d6:87:a7 actions=resubmit(,17)
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=50,reg6=0x1,metadata=0x1,dl_src=fa:16:3e:26:57:ff actions=resubmit(,17)
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=50,reg6=0x4,metadata=0x1,dl_src=fa:16:3e:e2:77:37 actions=resubmit(,17)
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=50,reg6=0x2,metadata=0x1,dl_src=fa:16:3e:b6:bb:4d actions=resubmit(,17)
 cookie=0x0, duration=75425.027s, table=16, n_packets=0, n_bytes=0, priority=50,reg6=0x5,metadata=0x1,dl_src=fa:16:3e:dd:b9:26 actions=resubmit(,17)
 cookie=0x0, duration=75388.913s, table=16, n_packets=24, n_bytes=2503, priority=50,reg6=0x6,metadata=0x1,dl_src=fa:16:3e:91:24:4c actions=resubmit(,17)
 cookie=0x0, duration=74749.247s, table=16, n_packets=20, n_bytes=2223, priority=50,reg6=0x7,metadata=0x1,dl_src=fa:16:3e:03:15:b0 actions=resubmit(,17)
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=0,metadata=0x1 actions=drop
 cookie=0x0, duration=75596.009s, table=16, n_packets=0, n_bytes=0, priority=0,metadata=0x2 actions=drop

```

This table blocks broadcast/multicast src addresses and also logical VLANs as they are not yet supported. (these are the first 4 flows)

The rest of the flows matches on reg6 (the port id), metadata (network id) and src mac address, these flows are ensuring that the MAC for that specific port is as configured (prevent MAC spoofing).
We can see that currently this table is populated with entries for all the ports in the system, a future enhancement will only configure the relevant flows depending on the ports located on this compute node. 


### Table 17 - Destination lookup, broadcast, multicast and unicast handling (and unknown MACs)

```
 cookie=0x0, duration=75596.009s, table=17, n_packets=20, n_bytes=2404, priority=100,metadata=0x1,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=set_field:0x6->reg7,resubmit(,18),set_field:0x5->reg7,resubmit(,18),set_field:0x4->reg7,resubmit(,18),set_field:0x2->reg7,resubmit(,18),set_field:0x7->reg7,resubmit(,18),set_field:0x1->reg7,resubmit(,18)
 cookie=0x0, duration=75596.009s, table=17, n_packets=0, n_bytes=0, priority=100,metadata=0x2,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=set_field:0x3->reg7,resubmit(,18)
 cookie=0x0, duration=75596.009s, table=17, n_packets=4, n_bytes=280, priority=50,metadata=0x1,dl_dst=fa:16:3e:b6:bb:4d actions=set_field:0x2->reg7,resubmit(,18)
 cookie=0x0, duration=75596.009s, table=17, n_packets=16, n_bytes=1762, priority=50,metadata=0x1,dl_dst=fa:16:3e:26:57:ff actions=set_field:0x1->reg7,resubmit(,18)
 cookie=0x0, duration=75596.009s, table=17, n_packets=0, n_bytes=0, priority=50,metadata=0x1,dl_dst=fa:16:3e:e2:77:37 actions=set_field:0x4->reg7,resubmit(,18)
 cookie=0x0, duration=75596.009s, table=17, n_packets=0, n_bytes=0, priority=50,metadata=0x2,dl_dst=fa:16:3e:d6:87:a7 actions=set_field:0x3->reg7,resubmit(,18)
 cookie=0x0, duration=75425.027s, table=17, n_packets=4, n_bytes=280, priority=50,metadata=0x1,dl_dst=fa:16:3e:dd:b9:26 actions=set_field:0x5->reg7,resubmit(,18)
 cookie=0x0, duration=75388.913s, table=17, n_packets=0, n_bytes=0, priority=50,metadata=0x1,dl_dst=fa:16:3e:91:24:4c actions=set_field:0x6->reg7,resubmit(,18)
 cookie=0x0, duration=74749.247s, table=17, n_packets=0, n_bytes=0, priority=50,metadata=0x1,dl_dst=fa:16:3e:03:15:b0 actions=set_field:0x7->reg7,resubmit(,18)

```

In this table we can see broadcast destination handling, if we look at the first 2 flows they are matching on the network id (by the metadata field) and by the dst MAC (which is broadcast) and duplicate the packet to all the possible ports.
This is done by setting a different destination port id to reg7 for every port in the same network as the originator port and resubmitting the packet back to the pipeline.

This behavior is not optimal as we are duplicating packets for each port, a possible solution could be to also send the network id (or metadata field in our case) to the relevant compute nodes.
In our case that will save duplications as we have ports from the same network located in the same compute node.

The rest of the flows are matching solely on the destination MAC addresses for all the possible destination ports, we can see that reg7 is filled with the tunnel_key of the destination port.

### Table 18 - ACL

```
 cookie=0x0, duration=75596.009s, table=18, n_packets=0, n_bytes=0, priority=0,metadata=0x2 actions=resubmit(,19)
 cookie=0x0, duration=75596.009s, table=18, n_packets=135, n_bytes=15586, priority=0,metadata=0x1 actions=resubmit(,19)

```
ACL’s are not yet implemented in the Neutron-OVN Integration and that’s why this table is currently empty and just forward the traffic to the next table (19) in the pipeline.
In the future Security Group rules will be implemented as flows here (and maybe FWaaS as well)

### Table 19 - Egress Port Security

```
 cookie=0x0, duration=75596.009s, table=19, n_packets=111, n_bytes=13264, priority=100,metadata=0x1,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,64)
 cookie=0x0, duration=75596.009s, table=19, n_packets=0, n_bytes=0, priority=100,metadata=0x2,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,64)
 cookie=0x0, duration=75596.009s, table=19, n_packets=4, n_bytes=280, priority=50,reg7=0x2,metadata=0x1,dl_dst=fa:16:3e:b6:bb:4d actions=resubmit(,64)
 cookie=0x0, duration=75596.009s, table=19, n_packets=0, n_bytes=0, priority=50,reg7=0x3,metadata=0x2,dl_dst=fa:16:3e:d6:87:a7 actions=resubmit(,64)
 cookie=0x0, duration=75596.009s, table=19, n_packets=16, n_bytes=1762, priority=50,reg7=0x1,metadata=0x1,dl_dst=fa:16:3e:26:57:ff actions=resubmit(,64)
 cookie=0x0, duration=75596.009s, table=19, n_packets=0, n_bytes=0, priority=50,reg7=0x4,metadata=0x1,dl_dst=fa:16:3e:e2:77:37 actions=resubmit(,64)
 cookie=0x0, duration=75425.027s, table=19, n_packets=4, n_bytes=280, priority=50,reg7=0x5,metadata=0x1,dl_dst=fa:16:3e:dd:b9:26 actions=resubmit(,64)
 cookie=0x0, duration=75388.913s, table=19, n_packets=0, n_bytes=0, priority=50,reg7=0x6,metadata=0x1,dl_dst=fa:16:3e:91:24:4c actions=resubmit(,64)
 cookie=0x0, duration=74749.247s, table=19, n_packets=0, n_bytes=0, priority=50,reg7=0x7,metadata=0x1,dl_dst=fa:16:3e:03:15:b0 actions=resubmit(,64)

```

This table is very similar to the ingress port security, since we already know in this host the destination port, rather its local or in another machine (remember its located in reg7), we can validate here that the destination MAC is the actual MAC configured for that port.
This table also handle broadcast traffic.
Every matched traffic continue to the next table in the pipeline (64)

### Table 64 - Output table (Logical to Physical or Local)

```
 cookie=0x0, duration=75596.009s, table=64, n_packets=0, n_bytes=0, priority=100,reg6=0x4,reg7=0x4 actions=drop
 cookie=0x0, duration=75596.009s, table=64, n_packets=0, n_bytes=0, priority=100,reg6=0x1,reg7=0x1 actions=drop
 cookie=0x0, duration=75596.009s, table=64, n_packets=0, n_bytes=0, priority=100,reg6=0x2,reg7=0x2 actions=drop
 cookie=0x0, duration=75422.692s, table=64, n_packets=0, n_bytes=0, priority=100,reg6=0x5,reg7=0x5 actions=drop
 cookie=0x0, duration=75385.021s, table=64, n_packets=10, n_bytes=1202, priority=100,reg6=0x6,reg7=0x6 actions=drop
 cookie=0x0, duration=74748.204s, table=64, n_packets=10, n_bytes=1202, priority=100,reg6=0x7,reg7=0x7 actions=drop
 cookie=0x0, duration=75596.009s, table=64, n_packets=24, n_bytes=2684, priority=50,reg7=0x2 actions=set_field:0x2->tun_id,output:1
 cookie=0x0, duration=75596.009s, table=64, n_packets=36, n_bytes=4166, priority=50,reg7=0x1 actions=set_field:0x1->tun_id,output:1
 cookie=0x0, duration=75596.009s, table=64, n_packets=20, n_bytes=2404, priority=50,reg7=0x4 actions=set_field:0x4->tun_id,output:1
 cookie=0x0, duration=75422.692s, table=64, n_packets=24, n_bytes=2684, priority=50,reg7=0x5 actions=set_field:0x5->tun_id,output:1
 cookie=0x0, duration=75385.021s, table=64, n_packets=10, n_bytes=1202, priority=50,reg7=0x6 actions=output:2
 cookie=0x0, duration=74748.204s, table=64, n_packets=1, n_bytes=42, priority=50,reg7=0x7 actions=output:3

```

This is the last step in the pipeline which now need to send the packet to the correct port (local or over a tunnel to other compute node).

Remember that we have reg6 with the unique key of the source port and reg7 with the unique key of the destination port.
The first flows drop packets which the source and destination are the same for all the ports (this of course can be improved in future implementations to only consist of local ports)

The next flows match remote ports and sends them to the relevant tunnel port in br-int (in our case port 1) and set the tunnel_id to be the key for the destination port.
If you remember in table 0, there is a dispatch to the correct VM port for each of the tunnel_id’s , these flows will match on the remote machine and send the traffic to the relevant port.

The last two flows match for local traffic for ports that reside on this compute node, the action is just to forward the traffic to the correct port number


# Summary

This was a detailed explanation of the current pipeline in OVN, things are very dynamic and changing and i will try to keep everyone updated with any new update.
In the next posts i will describe some of the future work and projects we are going to do in the Openstack-OVN integration and would love to see more people contribute and actively involved with this.


