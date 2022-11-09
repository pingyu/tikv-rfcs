# RawKV Cross Cluster Replication

- RFC PR: https://github.com/tikv/rfcs/pull/86
- Tracking Issue: https://github.com/tikv/repo/issues/0000

## Summary

This proposal introduces the technical design of RawKV Cross Cluster Replication.

## Motivation

Customers are deploying TiKV clusters as non-transaction Key-Value (RawKV) storage. But the lack of *Cross Cluster Replication* is a obstacle to deploy disaster recovery clusters for providing more highly available services.

The ability of cross cluster replication will help TiKV to be more adoptable to industries such as Banking and Finance.

## Key Challenges & Solutions

#### Capturing Change

As there is no timestamp in RawKV entries, it is not possible to capture which and when data is changed. So in this proposal, we add timestamp to data. See "[Add Timestamp to Data](#2-add-timestamp-to-data)" chapter.

#### Consistency

Data in recovery cluster is required to be consistent with main cluster. While RawKV do not provide transaction feature, for the replication we provide [Eventual Consistency](https://en.wikipedia.org/wiki/Eventual_consistency), with [At-Least-Once](https://www.cloudcomputingpatterns.org/at_least_once_delivery/) delivery pattern. Write sequence of different keys in recovery cluster is not guaranteed to be the same as in main cluster.

#### High Latency between Clusters

Customers require that [RPO](https://en.wikipedia.org/wiki/Disaster_recovery#Recovery_Point_Objective) should be no more than tens of seconds. *RPO* is consist of duration in TiKV, TiKV-CDC *(see below)*, and network latency. As the first two can be improved by code optimization and scale-out, we should be careful to deal with the last.

A typical ping latency of cross city network will be more than 30 milliseconds, which is much higher than IDC internal network. We should use appropriate concurrency and batch size to share latency to each data entry, at the same time limit transmission rate to avoid package loss.

In this proposal, we provide a TiKV sink with configurable concurrency, batch size, and backoff parameters, to be adapted to different network environments.

## Detailed Design

### 1. Utilize TiKV-BR & TiKV-CDC as Data Replication Component

**TiKV-BR**, a fork of [BR](https://github.com/pingcap/tidb/tree/master/br), is the tool for backup and restoration of TiKV. We utilize **TIKV-BR** to accomplish data initialization of recovery cluster.

**TiKV-CDC**, a fork of [TiCDC](https://github.com/pingcap/tiflow), is the tool for replicating incremental data of TiKV, which gets change data from TiKV's observer component, with high performance, high available, and scalable.

Besides, **TiKV-CDC** can also provide the ability to connect to other components, e.g. Message Queues, which will help TiKV integrate with customers' data systems, and further extend the applicable scenarios of TiKV.

![rawkv-cdc](../media/rawkv-cdc.png)

### 2. Add Timestamp to Data

As a kind of [Change Data Capture](https://en.wikipedia.org/wiki/Change_data_capture), timestamp (or version) is necessary to indicate which and when data is changed.

#### 2.1 Requirement

Among requests of a key, the timestamp must be monotonic with the sequence of data flushed to disk in TiKV.

In general, if request `a` ["happened before"](https://en.wikipedia.org/wiki/Happened-before) `b`, then `Timestamp(a) < Timestamp(b)`. As to cross cluster replication, we provide [Causal Consistency](https://en.wikipedia.org/wiki/Causal_consistency) by keeping order of the timestamp the same as sequence of data flush to disk in TiKV. Downstream systems apply data according to the timestamp order, will always lead to the consistent result.

At the same time, as RawKV doesn't provide cross-rows transaction and snapshot isolation, we treat requests of different keys as concurrent, which means that the sequence of two different keys would be inconsistent in replication, to improve efficiency.

#### 2.2 Timestamp Generation

Timestamp is generated by TiKV internally, to get a better overall performance and client compatibility.

We use [HLC](https://cse.buffalo.edu/tech-reports/2014-04.pdf) to generate timestamp, which not only is easy to use (by being closed to real world time), but also can capture causality.

Timestamp is 64 bits, composed of physical part with 46 bits, and logical part with 18 bits. 18 bits are large enough for logical part, and it is just the same as [TSO](https://github.com/tikv/pd/blob/master/server/tso/tso.go) of [PD](https://github.com/tikv/pd), as well as other timestamps (e.g. `start_ts` and `commit_ts`) in TiKV, which will help to reuse logics on different kinds of timestamps. 

##### 2.2.1 Physical Part of Timestamp

Physical part of HLC is acquired from TSO of [PD](https://github.com/tikv/pd).

In other HLC systems, the physical part is extracted from local system clock, and depends on the stabilization of the local system clock. So in most cases, the [NTP](https://en.wikipedia.org/wiki/Network_Time_Protocol) is necessary, along with some other procedures to ensure correctness (e.g.  [strict monotonicity](https://github.com/cockroachdb/cockroach/blob/13c5a25238ce75cfb7ff151d620e82aa44c72e27/pkg/util/hlc/doc.go#L150) in CockroachDB). As we tend to be free from NTP, we use TSO as the source of physical time.

TSO is a global monotonically increasing timestamp, with high performance and high available, fully meets the requirements for HLC.

Physical part is refreshed by repeatedly acquiring TSO in a period defaults to `500ms`, to keep it being closed the real world time. The period is configurable, shorter period will get more accurate metrics of CDC (e.g. *RPO*), but more pressures on PD.

On startup, physical part must be initialized by a successful TSO, to ensure that HLC is larger than last running. RawKV write operations will fail if the initialization has not been completed. (TiKV also can't startup when PD is out of service).

After successful initialization, HLC can tolerate fault of TSO. Even if some or all TiKV cannot connect to PD, the read and write operations are normal. But meanwhile, time-related metrics such as *RPO* is not reasonable, because the physical part of timestamp is at a standstill. It is also worth noting that as TiKV-CDC being highly dependent to PD (for cluster and task management, checkpoint, etc.), the replication will pause.

##### 2.2.2 Logical Part of Timestamp

According to [principle](https://cse.buffalo.edu/tech-reports/2014-04.pdf) of HLC, logical part (i.e. `logical clock` in paper) is a counter, increased when a write operation happens after another, to indicate the causality.

TiKV is a distributed system, as a result in every store we maintain an instance of the counter. As it is only required to keep causality among requests of every key (see [2.1 Requirement](#21-requirement)), every key exists in only one region, and only one peer (the leader) in the region can write, so it's safe to generate timestamps locally in each store, in most cases.

The exception happens when leader transfer. The diagram below is an example:

![rawkv-cdc-example](../media/rawkv-cdc-example.png)

*(**LC**: Logical Clock of store, **lc**: logical clock with PUT operation)*

Suppose the leader of a region was on store 1 at the beginning. When the first PUT happened, it got a **lc** as 10, and **LC** of store 1 was advanced to 11. When the leader was transfered to store 2, we should also take the **LC** of store 1 to store 2 (as **LC** of store 2 is smaller), otherwise the second PUT would get a **lc** as 8, which is smaller than previous PUT operation, and violate the causality.

This logic will be implemented in TiKV observer component to avoid interfering with main procedure of leader transfer: every peer observes the raft message applied and keeps the maximum timestamp of the region (called `Region-Max-TS`). On becoming leader, the peer advances the HLC of its store to be no less than `Region-Max-TS` . 

Besides, other region changes will be handled as following:

* On region merge: The `Region-Max-TS` of the region is the max of the two original regions.

* On region split: The `Region-Max-TS` of the new region is copied from the original one.

#### 2.3 Resolved Timestamp

*Resolved Timestamp* (or `resolved-ts`) is generated by TiKV, to indicate the largest timestamp of a replication task that all events before the timestamp have been replicated, to help downstream avoid getting partial data. Downstream streaming system can also use `resolved-ts` as [*watermark*](https://www.oreilly.com/radar/the-world-beyond-batch-streaming-102/).

In implementation, we notify `resolved-ts` event generation that the minimum timestamp of data that on the way from *proposed* to *applied* (by [`resolver.track_lock`](https://github.com/tikv/tikv/blob/v5.0.4-20211201/components/resolved_ts/src/resolver.rs#L47-L57)), to synchronize the write and `resolved-ts` routines.

![rawkv-cdc-track-lock](../media/rawkv-cdc-track-lock.png)

### 3. Deletion

To make deletions be captured as change data, we turn physical deletion to logical deletion, i.e., set a *deleted* flag, other than physically delete the entry.

The *garbage collection* of deleted data is implemented by setting TTL to a *lifetime* defaults to 24 hours. After the *lifetime* expired, the deleted data will be physically deleted by [compaction filter](https://github.com/facebook/rocksdb/wiki/Compaction-Filter). The *lifetime*, meanwhile, is the earlier start timestamp of **TiKV-CDC** replication task, which will ensure that the replication will not miss some deletion events. Shorter *lifetime* will save some storage, but shorten the duration **TiKV-CDC** replication task can pause. Replication before *lifetime* can only be archived by a full backup & restore using **TiKV-BR**.

### 4. Encoding

Timestamp & deleted flag are encoded as following:

```
r{user key}{MCE Padding}{^timestamp:u64}: {user value}{expire ts}{meta flags}
```

*(Deleted flag is a bit of meta flags in value)*

* Encoding based on TiKV API version [V2](https://github.com/tikv/rfcs/blob/master/text/0069-api-v2.md). *(As there is no flag or meta fields, it is not possible to encode the two necessary fields for CDC in older API version.)*
* Timestamp encoded into key will be beneficial to the following, comparing with encoded into value:
  * More effective catch up, as we can get scope of change data without read & decode all values.
  * [PiTR](https://en.wikipedia.org/wiki/Point-in-time_recovery) is available.
  * Less code complexity, as data structure of RawKV & TxnKV are the same.
* And with the following acceptable loss:
  * More read latency in tens of microseconds *(to be further verified)*.
  * A little more write overhead caused by [MCE Padding](https://github.com/facebook/mysql-5.6/wiki/MyRocks-record-format#memcomparable-format).
  * An extra Garbage Collection is required (used with TxnKV).
  * Possible more space occupied (related to update pattern).

### 5. Incremental Scan

On replication task resumed after paused, we scan *KVDB* to get the change data during pause by extracting timestamp encoded in key. The scan will be heavy as we actually scan all the keys, so we should avoid scanning as possible, and limit scan rate.

*Incremental Scan in SQL / TxnKV scenario scans write_cf to get change data scope, which is much cheaper. So an alternative to reduce cost for RawKV is writing timestamp to write_cf other than encoded in key, but will introduce an additional write for each put request. We tend to the lower delay of writing now.*

### 6. TiKV-CDC

**TiKV-CDC** is a fork of TiCDC. The following modifications will be applied:

* Owner:
  
  * A new type of [*changefeed*](https://docs.pingcap.com/tidb/stable/manage-ticdc#manage-replication-tasks-changefeed) which is specified by key range.
  
  * A new type of **task** which is specified as a **sub-range** of the whole key range, and is handled by a *KV Processor* on a [*Capture*](https://docs.pingcap.com/tidb/stable/ticdc-overview#ticdc-architecture).

* Scheduler: A new scheduler which splits the whole key range of *changefeed* into *sub-ranges*, and distributes to *captures*.
  
  * For simplicity, we split whole key range into fixed number of parts, which can be specified by *changefeed* argument, and defaults to **2**. As we sink data to downstream TiKV by **batch put** and bottleneck of replication should not be on TiKV-CDC, a parallel of **2** will be enough.
  
  * Key range splitting is align with boundary of Regions. For simplicity, each *sub-range* contains almost equal number of Regions.
  
  * *Rebalance* can only be performed by *tikv-cdc cli*. As *rebalance* will trigger *incremental scan* and impact performance of TiKV, it is not running automatically. We need more information to determine the best moment for *rebalance* by TiKV-CDC itself in the future.

* Capture: Accepts key-range tasks, and creates *KV Processors* to handle them.

* Processor:
  
  * A new *KV Processor* which accepts key-range tasks, and processes key-value data

* Sink:
  
  * A new *TiKV sink* to batch put data into downstream TiKV cluster, with configurable concurrency, batch size, and backoff parameters.

## Prototype

We have developed a [prototype](https://github.com/pingyu/tikv/issues/1) to verify feasibility of this proposal. All of the correctness validation cases are passed. And the benchmark results are as follows:

| Case                          | gPRC P99 Duration (ms) | gPRC Avg Duration (ms) |
|:-----------------------------:|:----------------------:|:----------------------:|
| Rawkv-cdc 600 threads raw_put | 15.3                   | 5.9                    |
| Baseline 600 threads raw_put  | 14.9                   | 5.2                    |
| Diff                          | -2.7%                  | -13.5%                 |
| Rawkv-cdc 600 threads raw_get | 0.6778                 | 0.0996                 |
| Baseline 600 threads raw_get  | 0.7007                 | 0.108                  |
| Diff                          | 3.3%                   | 7.8%                   |

*(Environment: Kingsoft Cloud, 32C 64GB, 500GB local SSD, Main: 1 KV + 1 PD + 1 TiCDC, Recovery: 1 KV + 1 PD, YCSB)*

We also developed a prototype to verify the performance impact of encoding timestamp in key. And the benchmark results are as following:

| Case                          | gRPC p99 duration (ms) | gRPC Avg Duration (ms) |
|:-----------------------------:|:----------------------:|:----------------------:|
| Rawkv-cdc 600 threads raw_put | 31                     | 8.8                    |
| Baseline 600 threads raw_put  | 31                     | 8.7                    |
| Diff                          | 0%                     | -0.1%                  |
| Rawkv-cdc 600 threads raw_get | 1.3                    | 0.31                   |
| Baseline 600 threads raw_get  | 1.2                    | 0.31                   |
| Diff                          | -8.3%                  | 0%                     |

*(Environment: Kingsoft Cloud, 32C 64GB, 500GB local SSD, Main: 1 KV + 1 PD YCSB, 10kw key-value pairs, block cache hit 80% with 100% index and 34.8% data)*

Regarding the gains mentioned in [4. encoding](#4-encoding), the performance impact is acceptable, so the proposal of encoding timestamp in key is choosen.

## Drawbacks

- Longer duration of `raw_put` *(about 2.7% for P99 in prototype so far)*

- Bigger storage *(more 9 bytes for every entry)*

## Alternatives

- Replication by Raft, i.e. deploying recovery cluster as *Raft Learner* of main cluster. This solution doesn't provide enough isolation between main and recovery cluster, and is not acceptable by some customers.
- A new component replicated from main TiKV as *Raft Learner*. *TBD*.

## Unresolved questions

*TBD*.