## Capacity Planning Single Index Cluster
while sizing the ES cluster, we need to consider two factors - a) minimum storage requirement b) computation power required based on data ingestion throughput. Each rule will come up with its own node requirement and we need to take the max of the two.

**Key Points to remember**
>1) Primary Shards - while creating index, we need to define a setting 'number_of_shards'. It is highly recommended not to change this settings once created. 
>2) Active Shards - Ideally, an index rolls over a period such as daily/hourly. For a daily index, the active index is current day's index where the writes/updates are happening [log analytics usecase]. For a active index, we have primary shard and replica. If we have 10 primary shard and replica factor of 1, then we have total 20 active shards for a daily index.

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
> we have 400K concurrent user sending heartbeats, where each heartbeat is of size 1200  bytes. We want to
>do proper capacity planning for our ElasticSearch cluster. Lets consider, we are using one index only for now.

**Storage Requirement**
```
        Heartbeat Arrival       = 20s
        Heartbeat Size          = 1200 bytes
        Concurrent Users        = 400000
        Concurrent Request/Sec  = Concurrent Users/Heartbeat Arrival = 400000/20 = 20000
        Daily Total Request     = Concurrent Request/Sec *60*60*24 = 20000*60*60*24 = 1728*10^6
        Daily Source Data       = Heartbeat Size * Daily Total Request = 1200 * 1728 * 10^6 / 10^9 GB = 2073.6 GB
        Replica                 = 1    
        [Considering r5.4xlarge]
        r5.4xlarge 
                EBS Volume = 6000 GB   ......(VI)
                VCPU       = 16        ......(VII)
                MEMORY     = 128 GB    ......(VIII)
        UltraWarm Large
                Volume     = 20TB
                VCPU       = 16
```


**Capacity Planning without ultrawarm single index**

```
        Retention Period        = 90
        Primary Shard Count     = Math.Ceiling((Daily Source Data * 1.25)/50 GB) [**(III)** ] = 52
        Active Shards           = Primary Shard Count * (1+ Replica) [**(V)**] = 52 ( 1+1) = 104
        Min. Storage Required   = Daily Source Data * (1 + Replica) * Retention Period [**(I)**] = 2073.6*(1+1)*90 = 373248 GB 
        [Considering r5.4xlarge]
        No. Instances based on Storage      = Math.CEILING(Min. Storage Required / EBS Volume **[VI]**) = Math.CEILING(373248/6000) = 63 ..(IX)
        No. Instances based on CPU Required = Math.CEILING(Active Shards * Shard vs CPU [**(IV)**]/VCPU Per Instance [**(VII)**)) = 10   ..(X)

        Considering (IX) && (X), we should have 63 instances of R5.4xlarge instances. 
        Now, number of active shards should be divisible by number of instances to make sure no data skew.
        So, 63*2 = 126, so ideally we should have 126 active shards, ie. 63 primary shards instead of 52. 
        So, we define, number_of_shards in index setting as 63. 
       
        As recommended to use 3 availability zone, each zone will have 63/3 = 21 instances.

        **Cost Calculation...Monthly**
        Cost for R5.4xlarge instances = 63*HourlyRate*24*30 = 63*1.487*24*30 = **67450.32** ....(XI)
        
        
```

**Capacity Planning with ultrawarm single index**

```
        Hot Retention Period        = 7
        Warm Retention Period       = 83
        Primary Shard Count     = Math.Ceiling((Daily Source Data * 1.25)/50 GB) [**(III)** ] = 52
        Active Shards           = Primary Shard Count * (1+ Replica) [**(V)**] = 52 ( 1+1) = 104
        Min. Storage Required   = Daily Source Data * (1 + Replica) * Hot Retention Period [**(I)**] = 2073.6*(1+1)*7 = 29030.4 GB 
        **[Considering r5.4xlarge]**
        No. Instances based on Storage      = Math.CEILING(Min. Storage Required / EBS Volume **[VI]**) = Math.CEILING(29030.4/6000) = 5 ..(XII)
        No. Instances based on CPU Required = Math.CEILING(Active Shards * Shard vs CPU [**(IV)**]/VCPU Per Instance [**(VII)**)) = 10   ..(XIII)
        **[Considering UltraWarm large]**
        Storage Requirement     = Daily Source Data * Warm Retention Period = 2073.6*83 = 172108.8 GB
        UltraWarm Instances Req.= Math.Ceiling(Storage Requirement/ UltraWarm Volume Per Instance) = Math.Ceiling(172108.8/20000) = 9

        Considering (IX) && (X), we should have 10 instances of R5.4xlarge instances. 
        Now, number of active shards should be divisible by number of instances to make sure no data skew.
        So, 104+6 = 110 [divisible by 10], so ideally we should have 110 active shards, ie. 55 primary shards instead of 52. 
        So, we define, number_of_shards in index setting as 55. 
       
        As recommended to use 3 availability zone, each zone will have 10/3 = 3 instances [one zone will have 4]

        **Cost Calculation...Monthly**
        Cost for R5.4xlarge instances = 10*HourlyRate*24*30 = 10*1.487*24*30 = **10706.4**
        Cost for UltraWarm large instances = 9 * HourlyRate * 24  * 30 = 9* 2.68 * 24 *30 = **17366.4**
        Total Monthly Cost Using UltraWarm = 28072.8 ... (XIV)
       
        
```

## Cost Comparison based on XIII & XIV, UltraWarm is almost 60% cheaper for this use case particular
