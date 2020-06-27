## Capacity Planning Multiple Index Cluster

**Formulas**
```
       Storage Required        = Daily Source Data * (1 + Replica) * Days of Retention                  .... (I)
       Primary Shard Size      = 50 GB [log Analytics] ***[40GB for search usecase]                     .... (II)
       Primary Shard Count     = Daily Source Data * 1.25 / Primary Shard Size *** 0.25 is the overhead .... (III) 
       SHARD VS CPU            = 1 : 1.5                                                                .... (IV)
       ACTIVE SHARDS           = Primary Shard Count * (1+Replica)                                      .... (V)
```

##Use Case
> Consider a use case, where we have heartbeat getting ingested every 20 second to elastic search. Lets say,
> we have 400K concurrent user sending heartbeats, where each heartbeat is of size 1200  bytes. We are 
>ingesting some quality metrics to ElasticSearch in two indices - 1) short_interval*, 2)long_interval*. We want to
>do proper capacity planning for our ElasticSearch cluster. 
