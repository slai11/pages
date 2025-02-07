---
title: "Redis Cluster Cron"
date: 2024-08-16T22:43:51+08:00
draft: false
---

### Why am I writing this?

Recently GitLab had a incident where the VM running a Redis Cluster node was somewhat borked after a kernel livepatch. I took a trip down to the cluster.c file in Redis 7.0.15.

The smoking gun was metrics showing a drop in pings received and pongs sent by all other nodes (roughly proportional to the ratio of affected nodes to total nodes). This suggests that the affected node's outgoing clusterbus messages were affected. Since the VM's logs and metrics got knocked out, there is no way to be sure but it still worth a dig/dive to understand how the cluster state might have been affected.

Since I'm still fresh from all that investigating, lets pen down my understanding of Redis Cluster then. I'll refer to valkey instead of Redis 7.0.15 since things did not change too much between the two.

Here's what I've learnt about cluster gossip:

### General overview

The `redis-server` process runs a `serverCron` which includes a bunch of housekeeping operations.

Specifically for a `cluster-enabled` process,  `clusterCron()` runs to:
- Ping the oldest node out of a random sample on every 10 iterations.
- Check all node health looking at ping/data delays.
- Mark nodes with slow pong responses as `PFAIL`.
- If the node is a replica, it handles manual failovers and replica migrations.
- Updates cluster state at the end.

### Anatomy of a cluster message

From `src/cluster_legacy.h`, we can look at the `clusterMsg` struct which gets sent to other nodes during gossip:

```c
typedef struct {
    char sig[4];                  /* Signature "RCmb" (Cluster message bus). */
    uint32_t totlen;              /* Total length of this message */
    uint16_t ver;                 /* Protocol version, currently set to CLUSTER_PROTO_VER. */
    uint16_t port;                /* Primary port number (TCP or TLS). */
    uint16_t type;                /* Message type */
    uint16_t count;               /* Number of gossip sections. */
    uint64_t currentEpoch;        /* The epoch accordingly to the sending node. */
    uint64_t configEpoch;         /* The config epoch if it's a primary, or the last
                                     epoch advertised by its primary if it is a
                                     replica. */
    uint64_t offset;              /* Primary replication offset if node is a primary or
                                     processed replication offset if node is a replica. */
    char sender[CLUSTER_NAMELEN]; /* Name of the sender node */
    unsigned char myslots[CLUSTER_SLOTS / 8];
    char replicaof[CLUSTER_NAMELEN];
    char myip[NET_IP_STR_LEN]; /* Sender IP, if not all zeroed. */
    uint16_t extensions;       /* Number of extensions sent along with this packet. */
    char notused1[30];         /* 30 bytes reserved for future usage. */
    uint16_t pport;            /* Secondary port number: if primary port is TCP port, this is
                                  TLS port, and if primary port is TLS port, this is TCP port.*/
    uint16_t cport;            /* Sender TCP cluster bus port */
    uint16_t flags;            /* Sender node flags */
    unsigned char state;       /* Cluster state from the POV of the sender */
    unsigned char mflags[3];   /* Message flags: CLUSTERMSG_FLAG[012]_... */
    union clusterMsgData data;
} clusterMsg;
```

`clusterMsgData` would contain various types of information, gossip (ping/pong/meet), failure, updates etc.

### How failures are propagated

In broad strokes, when a node discovers a possible failure during cluster cron, where ping responses are taking longer than the `cluster-node-timeout`, it marks a node with the pfail flag.

Looking at `void markNodeAsFailingIfNeeded(clusterNode *node)`,  when a node receives a ping with details of a pfail node, it keep track of the pfail node information in a failure report. The node with pfail flag gets marked as failing if it is already marked as pfail by the receiving node and sufficient number of reports have been received. The node receiving the ping will also broadcast the fail message to all nodes using the `clusterBroadcastMessage`.

In summary, every node independently pings other nodes and to decide if any node is possibly failing. The nodes then gossip their "suspicions" through pings and pongs. Upon receiving sufficient confirmation, the suspected failing node can be marked as failing.

### What happens when a node has unstable network connectivity

After a node is marked as possibly failing, `void clusterUpdateState(void)` method is called and
- checks slot coverage.
- check if the current node is in a minority partition.
- sets cluster state to `CLUSTER_FAIL` if needed.

In the incident, all nodes (even the affected one was running fine) which led me to believe that the affected node had flaky connectivity to the other nodes. i.e. the `node->flag` would oscillate between `pfail` and a non-`pfail` state. This would have allowed the affected node to continue serving Redis commands.

If the state were to be `CLUSTER_FAIL`, the `getNodeByQuery` method would have set a `CLUSTER_REDIR_DOWN_STATE` for the error code and failed all incoming Redis commands.

### How recovery happens

If a pong is received from a node that is marked `pfail`, the `pfail` flag gets unset.

---

### Self-note

I wanted to write more about failover handling but got busy so I'll publish this first.

TODO

- [ ] Add links to code
- [ ] Add more details to recovery
