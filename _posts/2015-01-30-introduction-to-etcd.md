---
layout: post
author: gsagie
title: Introduction to ETCD - Cluster Configuration Sync
date: 2015-01-30 16:25:06 -0700
categories:
- Virtualization
- Cloud
tags:
- ETCD
- Cluster
---

I have recently started being more and more interested and involved in distributed systems implementation, and in particular services cluster.

I think the way services clusters are being build has a direct impact on SDN and NFV requirements and future, but not only that, i think that networking function vendors, has a very different notion of clustering, high availability and scaling requirements than what web application creators have in mind.
I do believe that these two visions will merge when network functions start to be virtualized.

In this post i am going to introduce ETCD, which is a distributed, consistent key value store for shared configuration and service discovery.

etcd is written in Go and uses the Raft consensus algorithm to manage a highly-available replicated log.

There are other similar options that etcd is usually compared against, like ZooKeeper and Doozerd, another new project called Consul is also an interesting project i intend to write about (uses gossip protocol agent to exchange data between the cluster nodes),
I will keep the comparisons to future posts.

Etcd is relatively a new project, but its gaining momentum and hyped community since it is being used in CoreOS and Google Kubernetes.
CoreOS is a new Linux distribution that has been re-architected to provide features needed to run modern infrastructure stacks, it is optimized for containers and is using etcd internally.
Kubernetes is an open source container cluster manager. it also uses etcd internally and being developed by Google for making it simple to run Docker containers on Google Cloud Platform.

The reason i like Etcd is because of its simplicity and the fact it tries to be exactly what it is planned to be and not more, abstraction is always good and hope the maintainers will keep it like that.

Etcd uses HTTP+JSON API, it support SSL client certificate authentication and reliable by using Raft protocol.

You can easily start a single member cluster running the etcd server.
then using a curlable API you can write and read keys and values:


<div class="message">
curl http://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello world"

and read the value using:

curl http://127.0.0.1:2379/v2/keys/message

</div>

Reading a key’s value also gives you the previous value of the key.
The distributed data store is modeled very similar to a file system, you have directories and keys (which we can model as files, with the values being the file content).
(There is even a nice 3rd-party project which integrate FUSE and etcd together to create an actual distributed file system)

You can create and delete directories and create and delete keys inside directories.
Example:
<div class="message">
Creating Directory: curl http://127.0.0.1:2379/v2/keys/dir -XPUT -d dir=true
</div>

The nice thing about modeling it like this, is that we can read all keys inside a certain directory with one API call, this help us model bulk reads which is usually very usefull performance wise.

<div class="message">
curl http://127.0.0.1:2379/v2/keys/dir/ (List all keys under directory dir)
</div>

etcd support setting TTL for directories and keys which are deleted when time expires, This can help to create aging mechanism and as we will soon see also distributed timers.

Setting a TTL of 5 seconds to a key named “foo” , this key is deleted after 5 seconds:
 <div class="message">
curl http://127.0.0.1:2379/v2/keys/foo -XPUT -d value=bar -d ttl=5
</div>
etcd has few primitives for synchronization like Atomic Compare and Swap operation and Atomically creating in order keys.

Compare and swap is an operation to change a specific key’s value, the node that tries to change the value must specify the current value (previous value) and the new value to set.
The compare and swap operation is atomic and only successful if the current value is the correct, this means that if another node changed the value after we read the current value and before our operation ended then our change fails.

CAS (Compare and Swap) is the most basic operation used to build a distributed lock service.

Example of CAS:
<div class="message">
curl http://127.0.0.1:2379/v2/keys/foo -XPUT -d value=one

curl http://127.0.0.1:2379/v2/keys/foo?prevValue=two -XPUT -d value=three

Since the preValue is “one” this will fail with the following error returned:

{
    "cause": "[two != one]",
    "errorCode": 101,
    "index": 8,
    "message": "Compare failed"
}

But this works:

curl http://127.0.0.1:2379/v2/keys/foo?prevValue=one -XPUT -d value=two
</div>

Atomic creation of in-order keys is another synchronization primitive, Using POST on a directory, you can create keys with key names that are created in-order. This can be used in a variety of useful patterns, like implementing queues of keys which need to be processed in strict order. An example use case would be ensuring clients get fair access to a mutex.

Another important feature is the support of “Waiting for a change” using long polling, this means that a reader or another member of the cluster can wait and be waken when a specific key changes.

This mechanism can be used to notify cluster members or readers of changes in the data store or any other needed notification.
Combined with the TTL feature this can also be used as a “distributed timer”.

A cluster node sets a timer for 10 seconds:

<div class="message">
curl http://127.0.0.1:2379/v2/keys/timer -XPUT -d value=10seconds -d ttl=10
</div>

Another cluster members waits on that timer/key:

<div class="message">
curl http://127.0.0.1:2379/v2/keys/timer?wait=true
</div>

After 10 seconds the waiting node is waken with this message:

<div class="message">
{"action":"expire","node":{"key":"/timer","modifiedIndex":5,"createdIndex":4},"prevNode":{"key":"/timer","value":"10seconds","expiration":"2015-01-30T15:59:44.025924358Z","modifiedIndex":4,"createdIndex":4}}
</div>

The “Wait for change” key has a very nice and important feature to prevent nodes and readers from losing notifications/events.
When a node waits on a key and this key is changed, the waiting node get this example information as a response:

<div class="message">
{
    "action": "set",
    "node": {
        "createdIndex": 7,
        "key": "/foo",
        "modifiedIndex": 7,
        "value": "bar"
    },
    "prevNode": {
        "createdIndex": 6,
        "key": "/foo",
        "modifiedIndex": 6,
        "value": "bar"
    }
}
</div>

Using the modified index, we can watch for commands that have happened in the past. This is useful for ensuring you don't miss events between watch commands. Typically, we watch again from the (modifiedIndex + 1) of the node we got.

So if we take the above example response, and issue this command on the same key:

<div class="message">
curl 'http://127.0.0.1:2379/v2/keys/foo?wait=true&waitIndex=7'
</div>

The watch command returns immediately with the same response as previously.
This helps our readers to receive notifications and events, process them and not lose any other notifications/events that could have been triggered by other nodes while the process took place.

There are three ways to bind etcd server nodes to an etcd cluster:

* Statically configuring all the members of the cluster and their IP’s

* Using a discovery service which runs at one of the cluster nodes (or hosted on another machine), the nodes use the discovery service URL to join the cluster

* DNS Discovery

Each cluster has a leader election process, and a failure detection mechanism is embedded inside etcd to find cluster nodes failures.
The failure detection works by using a heartbeat check every given interval, the interval time is configurable and failure detection times can converge to ~200 milliseconds (depending of course on number of cluster nodes and network latency)

I have yet to do a throughout benchmarking of etcd, however from looking online i found numbers talking about 50K writes in ~20 seconds for a cluster of 3 nodes.
Performance is important aspect in frameworks like these and from what i see the maintainers work hard to improve performance and support writing batching to increase it, so stay tuned for more updated results.

This is a very brief/high-level presentation of etcd and its main features, you can find a more detailed explanation and API documents in the [etcd official documentation repository.](https://github.com/coreos/etcd/tree/master/Documentation)
