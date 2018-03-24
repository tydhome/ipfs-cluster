# Collaborative pinsets

This document outlines a possible implementation plan for "collaborative pinsets" as described below, within the scope of ipfs cluster.

## Definition

A **collaborative pinset** is a collection of CIDs which is pinned by a number of ipfs-cluster peers which:

* Trust one or several peers to publish and update such pinset, but not others
* May freely participate or stop participating in the cluster, without this affecting
other peers or the pinset

## State of the art in ipfs-cluster

ipfs-cluster currently supports pinsets in a trusted environment where every node, once participating in the cluster has full control of the other peers (via unauthenticated RPC). The cluster secret (pnet key) ensures that only peers with a pre-shared key can request joining the cluster.

Maintenance of the peer-set is performed by the **consensus component**, which provides an interface to 1) The maintenance of the peer-set 2) The maintenance of the pinset (*shared state*).

Other components of cluster are independent from these two tasks, and provide functionality that will be useful in scenarios where the peer-set and pin-set maintenance works in a different manner:

* A state component provides pinset representation and serialization
* An ipfs connector component provides facilities for controlling the ipfs daemon (and the proxy)
* A pintracker component provides functionality to ensure the shared state is followed by ipfs
* A monitoring component provides metric collection.

Thus, just replacing the consensus layer (with some caveats ellaborated below) is a relatively simple approach to support collaborative pinsets.

## New consensus: collaborative pinsets using pubsub

### State of the art of the consensus component

The consensus component is defined by the following interface:

```go
type Consensus interface {
	Component
	Ready() <-chan struct{}
	LogPin(c api.Pin) error
	LogUnpin(c api.Pin) error
	AddPeer(p peer.ID) error
	RmPeer(p peer.ID) error
	State() (state.State, error)
	Leader() (peer.ID, error)
	WaitForSync() error
	Clean() error
	Peers() ([]peer.ID, error)
}
```

This is a wrapper of the go-libp2p-consensus which adds functionality and utility methods which until now applied to raft (provided by go-libp2p-raft)

The purpose of Raft is to maintain a shared state by providing a distributed append-only log. In a Raft cluster, the log is maintained by an elected Leader, compacted and snapshotted convieniently. ipfs-cluster has spent significant efforts in detaching the state representation from raft, and allowing transformations (upgrades) to run indepedently, based solely on the "state component".

### A new consensus component with pubsub

In order to be consistent with how components are meant to interact with each-other, it makes sense to implement the shared state maintenance in a new `go-libp2p-consensus` implementation, even if we don't aim for pubsub to be a real consensus implementation and we will admit certain leeway when it comes to divergent states (see below). `pubsub` should on the other hand, be an escalable and (hopefully), efficient method of broadcasting state updates to a large number of cluster peers.

This work may be accompanied by a further effort to thin out and simplify the current consensus component by moving raft-specific code to `go-libp2p-raft`, if this helps better understanding and improving the role of consensus component in cluster.

A thin consensus component in cluster will allow to more easily swap and provide alternative `go-libp2p-consensus` implementations.

For collaborative pinsets, we propose using `pubsub` to implement `OpLogConsensus` and maintain the shared state in cluster.


### Stablishing trust: authentication and authorization for collaborative pinning

We assume that in collaborative pinsets, there is a special set of **trusted peers** which is granted:

* Access to remote RPC endpoints from all peers in the cluster
* Capacity to modify the pinset

Libp2p provides authentication of peers by default (as long as secio is enabled).

Pnets offers basic-level, all-or-nothing, authorization by only allowing peers with a pre-shared key to communicate.

In order to support **trusted peers**, we propose additionally:

* Remote RPC authentication in go-libp2p-rpc: The Server will be initialized to allow execution of RPC methods to a "trusted" set of peer IDs only.
* Signed pubsub messages: every cluster state operation published to pubsub, will be signed by the issuer (TBD how, in order to avoid replaying, possibly timestamped).

This should allow the "trusted peers" to maintain the pinset (next section), and also to run RPC methods on any peer (for example, when sharding a file and adding blocks directly on the destination).


### Maintaining the shared state with pubsub

In it's simplest form:

* Peers subscribe to a topic. This topic receives log operations which are applied to the local state.
* State updates are performed by publishing operations on that topic.

However, pubsub alone does not provide good "consensus", and states may diverge as operations are applied in different order, or simply, don't reach the receiving end. On the other hand, a non-trusted node only has interest in the pinset which is allocated to it, not being of special relevance whether its view of pins allocated to other is correct or not.

There are a few ways of addressing these, always considering that in a large collaborative pinset cluster we don't expect every peer to have a 100% accurate view on the pinset.

* Republishing and re-allocation of pins (sending pin-messages several times, reguarly, for example, after a peer "checks-in" (see below about peerset maintenance).
* We can let peers publish regularly the full serialized state to ipfs [and point to it via IPNS]. Nodes can then download the "official" version of the state and consolidate their copy.
* If needed for performance reasons, a blockchain of updates (as a ledger of operations to the state) can also be published. This will require finding a balance between full-state checkpoint and blockchain length.

Again, we have to stress that collaborative pinsets are meant to scale to large quantities of peers of oportunistic participants and we do not mind so much whether that every pin is exactly pinned in specific places an exact number of times at an exact moment.


### Maintaining the peerset with pubsub

In "collaborative pinsets":

* The current peerset is only revelant to peers publishing/updating the state, as they perform the allocations.
* Only when replication factor != -1 (pin everywhere)

Thus, in a "replicate everywhere" scenario, the actual peerset does not need to be mantained because it doesn't matter. However this constraint is not useful when using ipfs-cluster to increase ipfs capacities for storage (also when sharding). Thus we need to provide a way for peers to announce themselves, be discovered and be allocated (and eventually de-allocated content).

We can take advantage of pubsub too for this matter:

* Peers may announce themselves by publishing pubsub messages
* Membership may expire unless such messages are received regularly
* Peerset may be extracted from an improved monitoring component (pub-sub based too, which already includes approaches to metrics and expiration).

Since collaborative pinset clusters should be able to grow significantly, with peers departing and joining at will, we make no strict tracking of peers. In such clusters, it is not so important to track every peer, but rather to ensure that the content is relatively well pinned. Thus, we forsee the use of high maximum replication factors and small minimum replication factors (in comparison), that provide the necessary leeways for such scenarios.

## UX in collaborative pinsets

We aim to get that, non-trusted peers of such clusters, will have a super-easy setup:

1. Run the go-ipfs daemon
2. `ipfs-cluster-service -c /ipns/Qmxxx`

a configuration template with pre-filled "trusted" peers and the optimal configuration for the collaborative pinset cluster will be provided via ipfs/ipns, and be upgradeable this way too.
