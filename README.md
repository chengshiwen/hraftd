_For background on this project check out this [blog post](http://www.philipotoole.com/building-a-distributed-key-value-store-using-raft/). Please note that this project has been archived, and is no longer maintained._

hraftd
======

hraftd is a reference example use of the [Hashicorp Raft implementation v1.0](https://github.com/hashicorp/raft). [Raft](https://raft.github.io/) is a _distributed consensus protocol_, meaning its purpose is to ensure that a set of nodes -- a cluster -- agree on the state of some arbitrary state machine, even when nodes are vulnerable to failure and network partitions. Distributed consensus is a fundamental concept when it comes to building fault-tolerant systems.

A simple example system like hraftd makes it easy to study the Raft consensus protocol in general, and Hashicorp's Raft implementation in particular. It can be run on Linux, OSX, and Windows.

## Reading and writing keys

The reference implementation is a very simple in-memory key-value store. You can set a key by sending a request to the HTTP bind address (which defaults to `127.0.0.1:11000`):
```bash
curl -XPOST 127.0.0.1:11000/key -d '{"foo": "bar"}'
```

You can read the value for a key like so:
```bash
curl -XGET 127.0.0.1:11000/key/foo
```

## Running hraftd
*Building hraftd requires Go 1.13 or later. [gvm](https://github.com/moovweb/gvm) is a great tool for installing and managing your versions of Go.*

Starting and running a hraftd cluster is easy. Download hraftd like so:
```bash
git clone https://github.com/chengshiwen/hraftd.git
make
```

Run your first hraftd node like so:
```bash
./bin/hraftd -id node0 ./data/node0
```

You can now set a key and read its value back:
```bash
curl -XPOST 127.0.0.1:11000/key -d '{"user1": "batman"}'
curl -XGET 127.0.0.1:11000/key/user1
```

### Bring up a cluster
_A walkthrough of setting up a more realistic cluster is [here](https://github.com/chengshiwen/hraftd/blob/master/CLUSTERING.md)._

Let's bring up 2 more nodes, so we have a 3-node cluster. That way we can tolerate the failure of 1 node:
```bash
./bin/hraftd -id node1 -haddr 127.0.0.1:11001 -raddr 127.0.0.1:12001 -join 127.0.0.1:11000 ./data/node1
./bin/hraftd -id node2 -haddr 127.0.0.1:11002 -raddr 127.0.0.1:12002 -join 127.0.0.1:11001 ./data/node2
```
_This example shows each hraftd node running on the same host, so each node must listen on different ports. This would not be necessary if each node ran on a different host._

This tells each new node to join the existing node. Once joined, each node now knows about the key:
```bash
curl -XGET 127.0.0.1:11000/key/user1 -L
curl -XGET 127.0.0.1:11001/key/user1 -L
curl -XGET 127.0.0.1:11002/key/user1 -L
```

Furthermore you can add a second key:
```bash
curl -XPOST 127.0.0.1:11001/key -d '{"user2": "robin"}' -L
```

Confirm that the new key has been set like so:
```bash
curl -XGET 127.0.0.1:11000/key/user2 -L
curl -XGET 127.0.0.1:11001/key/user2 -L
curl -XGET 127.0.0.1:11002/key/user2 -L
```

You can now delete a key and its value:
```bash
curl -X DELETE 127.0.0.1:11002/key/user2 -L
```

#### Three consistency level's reads
You can now read the key's value by different read consistency level:
```bash
curl -XGET '127.0.0.1:11000/key/user1?level=stale'
curl -XGET '127.0.0.1:11001/key/user1?level=default' -L
curl -XGET '127.0.0.1:11002/key/user1?level=consistent' -L
```

### Leader and servers API

```bash
curl -XGET 127.0.0.1:11000/leader
curl -XGET 127.0.0.1:11002/servers
```

### Tolerating failure
Kill the leader process and watch one of the other nodes be elected leader. The keys are still available for query on the other nodes, and you can set keys on the new leader. Furthermore, when the first node is restarted, it will rejoin the cluster and learn about any updates that occurred while it was down.

A 3-node cluster can tolerate the failure of a single node, but a 5-node cluster can tolerate the failure of two nodes. But 5-node clusters require that the leader contact a larger number of nodes before any change e.g. setting a key's value, can be considered committed.

### Leader-forwarding
Automatically forwarding requests to set keys to the current leader is not implemented. The client must always send requests to change a key to the leader or an error will be returned.

## Production use of Raft
For a production-grade example of using Hashicorp's Raft implementation, to replicate a SQLite database, check out [rqlite](https://github.com/rqlite/rqlite).
