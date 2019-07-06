[toc]

------
概念解释

A Region in HBase is defined as the Rows between two row key's. If you have more than one ColumnFamily in your Table, you will get one Store per ColumnFamily per Region. Every Store will have a MemStore and 0 or more StoreFiles

StoreFiles are created when the MemStore is flushed. Every so often, a background thread will trigger a compaction to keep the number of files in check. There are two types of compactions: major and minor. When a Store is targeted for a minor compaction, it will also pick up some adjacent StoreFiles and rewrites them as one. A minor compaction will not remove deleted/expired data. If a minor compaction picks up all StoreFiles in a Store, it's promoted to a major compaction. In a major compaction, all StoreFiles of a Store are rewritten as one StoreFile.

Ok... so what is a Compaction Queue? It is the number of Stores in a RegionServer that have been targeted for compaction. Similarly a Flush Queue is the number of MemStores that are awaiting flush.

As to the question of why there is a queue when you can do it asynchronously, I have no idea. This would be a great question to ask on the HBase mailing list. It tends to have faster response times.


```  sh
Table       (HBase table)
  Region      (Regions for the table)
    Store       (Store per ColumnFamily for each Region for the table)
      MemStore    (MemStore for each Store for each Region for the table)
      StoreFile   (StoreFiles for each Store for each Region for the table)
        Block       (Blocks within a StoreFile within a Store for each Region for the table)
```
