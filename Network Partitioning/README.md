# Split-Brain Scenario in Redis Cluster

This documentation provides a step-by-step guide for simulating and managing a split-brain scenario in a Redis Cluster. A split-brain occurs when network partitions prevent nodes from communicating, leading to potential data inconsistencies and conflicts. This guide will demonstrate how to simulate this condition, observe the behavior of the cluster, and perform manual recovery.

## What is Split-Brain Situation?

**Scenario: Redis Cluster Split-Brain Issue**

Imagine we have a Redis cluster with three primary shards. Each primary shard has one replica, forming a six-node cluster (three primaries and three replicas). This setup is functioning normally, handling requests and ensuring data consistency across the cluster. However, one day, a network partition occurs, splitting the cluster into two isolated groups.

On the **left side** of the partition, Group A contains some of the primary shards and their replicas. On the **right side**, Group B holds the remaining shards and their replicas. Due to the network partition, neither group can communicate with the other, so each side believes the other is offline.



### Problem: Simultaneous Failover
Both Group A and Group B detect that their primary shards are unreachable, so each group triggers a failover. The replicas in each group are promoted to new primaries to maintain availability. Now, **both sides** of the partition believe they have all the primary shards. Group A has promoted replicas to primary status, and Group B has done the same.

### Split-Brain Situation
With both sides functioning independently, they continue to accept client requests. On the **left side**, Client A sends a command to set the key `"foo"` to `"bar"`. Meanwhile, on the **right side**, Client B sets the same key `"foo"` to `"baz"`. Since the groups are isolated, they have no idea that the same key has been modified with different values.

### Conflict After Partition Resolution
Once the network partition is resolved and the two sides rejoin, a serious conflict arises. The two groups attempt to synchronize, but now there are **two primary shards** with conflicting data for the same key `"foo"`. Group A holds `"foo": "bar"`, while Group B holds `"foo": "baz"`. Both shards claim to be the authoritative source for this key, but they cannot agree on which version of the data is valid.

This situation, where two or more parts of a distributed system think they are the primary and continue operating independently, is called a **split-brain**. It’s a critical issue in distributed systems, especially for databases like Redis, where consistency of data is crucial. Resolving which data is valid after a split-brain scenario is often complex and can lead to data loss or inconsistency.

## Scenario Overview

In this lab, we are going to simulate a Redis Cluster deployment across multiple subnets to understand how Redis behaves under network partitioning. The purpose of this lab is to demonstrate how Redis Cluster handles node failures, communication breakdowns, and automatic failover, while also learning how to recover the cluster after the partition is resolved.

## Prerequisites

- A Redis Cluster set up with at least three master nodes and three slave nodes.
- Access to AWS EC2 to create and manage security groups.
- Basic understanding of Redis commands and cluster management.

## Step 1: Verify Initial Cluster State

Before simulating the split-brain scenario, verify the initial state of your Redis cluster.

### Command:
```bash
redis-cli -h 172.31.0.1 -p 6379 CLUSTER NODES
```

### Expected Output:
```
a1b2c3d4e5f6g7h8 172.31.0.1:6379@16379 myself,master - 0 1591234567000 1 connected 0-5461
i9j0k1l2m3n4o5p6 172.31.0.2:6379@16379 master - 0 1591234568000 2 connected 5462-10922
q7r8s9t0u1v2w3x4 172.31.0.3:6379@16379 master - 0 1591234569000 3 connected 10923-16383
y5z6a7b8c9d0e1f2 172.31.0.4:6379@16379 slave a1b2c3d4e5f6g7h8 0 1591234570000 1 connected
g3h4i5j6k7l8m9n0 172.31.0.5:6379@16379 slave i9j0k1l2m3n4o5p6 0 1591234571000 2 connected
o1p2q3r4s5t6u7v8 172.31.0.6:6379@16379 slave q7r8s9t0u1v2w3x4 0 1591234572000 3 connected
```

### Explanation:
- This output shows three master nodes and three slave nodes.
- All nodes are connected and functioning correctly.

## Step 2: Simulate Network Partition

Simulate a network partition using AWS security groups.

### Detailed Process:

1. **Create Security Groups**:
   - Create two security groups: `Redis-Cluster-Group-A` and `Redis-Cluster-Group-B`.

2. **Configure Security Groups**:
   - Allow inbound traffic on port `6379` from within the same security group.
   - Allow SSH access (port `22`) from your IP address.

3. **Assign EC2 Instances**:
   - Assign instances for nodes 1-3 to `Redis-Cluster-Group-A` (172.31.0.1, 172.31.0.2, 172.31.0.3).
   - Assign instances for nodes 4-6 to `Redis-Cluster-Group-B` (172.31.0.4, 172.31.0.5, 172.31.0.6).

4. **Block Traffic**:
   - Remove any rules allowing inbound traffic between the two security groups.

### Important Notes:
- This creates isolated network segments without physically disconnecting nodes.
- Nodes within each group can still communicate.

## Step 3: Observe Initial Behavior

After creating the network partition, observe the behavior of the cluster.

### Commands:
```bash
# On a node in Group A
redis-cli -h 172.31.0.1 -p 6379 CLUSTER NODES

# On a node in Group B
redis-cli -h 172.31.0.4 -p 6379 CLUSTER NODES
```

### Expected Behavior:
- **In Group A**:
  - Nodes 1-3 will show as active masters.
  - Nodes 4-6 will be marked as "disconnected" or "fail".

- **In Group B**:
  - Nodes 4-6 will be slaves but disconnected from their masters.
  - Nodes 1-3 will be marked as "disconnected" or "fail".

### Explanation:
- Redis uses a gossip protocol for failure detection.
- Nodes unable to communicate mark each other as failed, resulting in a partially unavailable state.

## Step 4: Write Data to Both Partitions

Test how the cluster handles write operations in this partitioned state.

### Commands:
```bash
# In Group A
SET key1 "value1"

# In Group B
SET key2 "value2"
```

### Expected Results:
- **Group A**: The SET operation succeeds.
- **Group B**: The SET operation fails with a `CLUSTERDOWN` error.

### Explanation:
- Redis maintains write availability only in the partition with a majority of master nodes.

## Step 5: Prolong the Partition

Keep the network partition in place for longer than the `cluster-node-timeout` (default 15000ms or 15 seconds).

### Importance:
- Allows the cluster to recognize the partition as a sustained issue.
- Triggers internal management processes.

## Step 6: Manual Intervention (Creating Split-Brain)

Create a split-brain situation by promoting slaves to masters in the minority partition.

### Command (run on each node in Group B):
```bash
CLUSTER FAILOVER FORCE
```

### Explanation:
- This command forces a slave to promote itself to a master, potentially leading to conflicting master assignments.

## Step 7: Write Data in Both Partitions

Test write operations again after the manual promotion.

### Commands:
```bash
# In Group A
SET key3 "value3"

# In Group B
SET key4 "value4"
```

### Expected Results:
- Both operations succeed, leading to divergent datasets in each partition.

## Step 8: Resolve Network Partition

To resolve the network partition:

1. **Go to the AWS EC2 console.**
2. **Edit security group rules** to allow inbound traffic on port `6379` from the other security group.

### Result:
Restores network communication but does not automatically resolve the split-brain condition.

## Step 9: Observe Cluster Behavior

After restoring communication, observe the cluster state.

### Command:
```bash
redis-cli -h 172.31.0.1 -p 6379 CLUSTER NODES
```

### Expected Behavior:
- Conflicting information about node roles and data distribution.
- The cluster may report inconsistencies.

## Step 10: Attempt Automatic Recovery

Redis will attempt to reconcile the conflicting states, but full automatic recovery is unlikely due to conflicting master assignments.

## Step 11: Manual Resolution

Manual intervention is typically necessary.

### Steps for Manual Resolution:
1. **Decide which partition's data to keep**.
2. On nodes in the other partition, run:
   ```bash
   CLUSTER RESET SOFT
   ```
3. Rejoin these nodes to the cluster:
   ```bash
   CLUSTER MEET <ip> <port>
   ```
4. Manually reassign slots and resynchronize data.

### Important Note:
This process may result in data loss and requires careful consideration.

## Conclusion

This documentation illustrates the complexities of managing distributed systems like Redis Cluster, especially during network failures. Proper configuration, monitoring, and robust recovery processes are essential for maintaining data integrity and availability in the face of partitioning scenarios.