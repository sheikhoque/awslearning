#Capacity Planning of ES Cluster

> while sizing the ES cluster, we need to consider two factors - a) minimum storage requirement b) computation power required based on data ingestion throughput. Each rule will come up with its own node requirement and we need to take the max of the two.

---
**Key Points to remember**
1) Primary Shards - while creating index, we need to define a setting 'number_of_shards'. It is highly recommended not to change this settings once created. 
2) Active Shards - Ideally, an index rolls over a period such as daily/hourly. For a daily index, the active index is current day's index where the writes/updates are happening [log analytics usecase]. For a active index, we have primary shard and replica. If we have 10 primary shard and replica factor of 1, then we have total 20 active shards for a daily index.

**Formulas**
```
       Storage Required        = Daily Source Data * (1 + Replica) * Days of Retention .... (I)
       Primary Shard Size      = 50 GB [log Analytics] ***[40GB for search usecase]    .... (II)
       Primary Shard Count     = Daily Source Data * 1.25 / Primary Shard Size *** 0.25 is the overhead .... (III)  
```

---
> ##Use Case
> Consider a use case, where we have heartbeat getting ingested every 20 second to elastic search. Lets say,
> we have 400K concurrent user sending heartbeats, where each heartbeat is of size 1200  bytes. We want to
>do proper capacity planning for our ElasticSearch cluster.

**Capacity Planning [minimum size req]**

```
        Concurrent Users        = 400000
        Heartbeat Arrival       = 20s
        Concurrent Request/Sec  = Concurrent Users/Heartbeat Arrival = 400000/20 = 20000
        Daily Total Request     = Concurrent Request/Sec *60*60*24 = 20000*60*60*24 = 1728*10<sup>6</sup>
```