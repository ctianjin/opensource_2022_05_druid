# Druid Metadata

## Metadata storage
store various metadata about the system, but not to store the actual data.
The Metadata Storage stores the entire metadata which is essential for a Druid cluster to work. For production clusters, consider using MySQL or PostgreSQL instead of Derby. Also, it's highly recommended to set up a high availability environment because there is no way to restore if you lose any metadata.

## Derby
```
druid.metadata.storage.type=derby
druid.metadata.storage.connector.connectURI=jdbc:derby://localhost:1527//opt/var/druid_state/derby;create=true
```
## Adding custom dbcp properties
```
druid.metadata.storage.connector.dbcp.maxConnLifetimeMillis=1200000
druid.metadata.storage.connector.dbcp.defaultQueryTimeout=30000
```
## Metadata storage tables
stores metadata about the segments that should be available in the system.
```
{
 "dataSource":"wikipedia",
 "interval":"2012-05-23T00:00:00.000Z/2012-05-24T00:00:00.000Z",
 "version":"2012-05-24T00:10:00.046Z",
 "loadSpec":{
    "type":"s3_zip",
    "bucket":"bucket_for_segment",
    "key":"path/to/segment/on/s3"
 },
 "dimensions":"comma-delimited-list-of-dimension-names",
 "metrics":"comma-delimited-list-of-metric-names",
 "shardSpec":{"type":"none"},
 "binaryVersion":9,
 "size":size_of_segment,
 "identifier":"wikipedia_2012-05-23T00:00:00.000Z_2012-05-24T00:00:00.000Z_2012-05-23T00:10:00.046Z"
}
```

## Accessed by
The Metadata Storage is accessed only by:

Indexing Service Processes (if any)
Realtime Processes (if any)
Coordinator Processes

# Deep storage
Deep storage is where segments are stored. It is a storage mechanism that Apache Druid does not provide. This deep storage infrastructure defines the level of durability of your data, as long as Druid processes can see this storage infrastructure and get at the segments stored on it, you will not lose data no matter how many Druid nodes you lose. If segments disappear from this storage layer, then you will lose whatever data those segments represented.

## Local Mount
A local mount can be used for storage of segments as well. This allows you to use just your local file system or anything else that can be mount locally like NFS, Ceph, etc. This is the default deep storage implementation.

In order to use a local mount for deep storage, you need to set the following configuration in your common configs.

Property	Possible Values	Description	Default
druid.storage.type	local		Must be set.
druid.storage.storageDirectory		Directory for storing segments.	Must be set.

Note that you should generally set druid.storage.storageDirectory to something different from druid.segmentCache.locations and druid.segmentCache.infoDir.

If you are using the Hadoop indexer in local mode, then just give it a local file as your output directory and it will work.

## S3-compatible

## HDFS
