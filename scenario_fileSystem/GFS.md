- [Single master](#single-master)
  - [Master as directory service](#master-as-directory-service)
    - [Chunk size of 64MB](#chunk-size-of-64mb)
    - [Three types of metadata](#three-types-of-metadata)
    - [Access pattern](#access-pattern)
  - [High availability](#high-availability)
    - [Restart master](#restart-master)
    - [Switch to master backup](#switch-to-master-backup)
    - [Read from shadow backup](#read-from-shadow-backup)
- [Network bottleneck](#network-bottleneck)
  - [Identify the bottleneck](#identify-the-bottleneck)
  - [GFS write process](#gfs-write-process)
    - [Separation of control and data flow](#separation-of-control-and-data-flow)
    - [Streamlined network transmission](#streamlined-network-transmission)
      - [Process](#process)
      - [Reason](#reason)

# Single master
## Master as directory service
### Chunk size of 64MB
* When compared with chunk server used for dat storage, master is a 
directory service. 
* GFS also uses namespace + filename to identify a file. 
* Each file is divided to chunk of size 64MB. 
* GFS client knows which chunk server to find the file. 

![](../.gitbook/assets/gfs_chunkserver.png)

### Three types of metadata
* File and chunk namespace
* Mapping from file full path name to chunk handle list. 
* Mapping from chunk handle list to chunk server list. 

![](../.gitbook/assets/gfs_structure.png)

### Access pattern
1. GFS client sends file name and chunk offset to GFS master. Since all chunks have the same size of 64MB, chunk index could be easily calculated. 
2. After GFS master gets the request, it will return addresses of chunk servers to clients. 
3. After GFS client gets addresses, it could reach out to any of it to get chunk data. 

![](../.gitbook/assets/gfs_accesspattern.png)

## High availability

![](../.gitbook/assets/gfs_highavailability.png)

### Restart master
* All master metadata is cached inside memory. There will be checkpoints where all memory is dumped to disk. 
* If the master has software failures, then it will first recover from checkpoints. And then operation logs after that timestamp will be replayed. 

### Switch to master backup
* The above procedure could handle software but not hardware failures. 
* If the master has hardware failures, then it could failover to the backups which master synchronously replicates to. 

### Read from shadow backup 
* The switch process could take seconds or minutes to complete. 
* In the meanwhile, When compared with shadow master used for availability for asynchronous read. 
* The data in shadow back might be stale. But the chance that client read stale metadata from shadow backup is quite slim because it only happens when all these three conditions are met: 
  * Master is dead. 
  * The metadata on master has not completely been replicated to shadow backup. 
  * The data clients is trying to read is just these metadata not replicated yet. 

# Network bottleneck
## Identify the bottleneck
* In GFS case
  * It uses 100Mbps network card, it has a maximum throughput of 12.5MB/s. 
  * It uses 5400 rpm disk, its bandwidth is usually 60~90MB/s. And multiple hard disks could be plugged together, and there will be a maximum bandwidth of 500MB/s. 
  * The bottleneck is in network layer. 

## GFS write process
1. Client queries master for locations of chunk servers. 
2. Master replies with secondary replica locations of chunk servers. And master also replies with primary replica locations of chunk servers. 
3. Client sends data to all chunkservers. However, after chunkservers receive the data, they will not immediately write it to disk. Instead, they will cache it in the memory. 
4. After all secondary replicas receive data, clients will send a write request to master. Master will order these write requests. 
5. Master will forward all write requests to secondary replicas. Then all secondary replicas will write data to disk with the same order. 
6. After secondary replicas finish writing, it will reply to master that it has finished. 
7. Master will tell clients that write requests have completed successfully. 

![](../.gitbook/assets/gfs_writeprocess.png)

### Separation of control and data flow
* Master only tells GFS client which chunk servers to read/write data. After that it is out of the business. 

### Streamlined network transmission
#### Process
* Data might not first be transmitted to primary replica. It depends on which replica is closer to the client. 
* Then the closer replica will send the data to the next replica. 

![](../.gitbook/assets/gfs_writeBandwidth.png)

#### Reason
* All servers on the same rack will be plugged to the same access switch. 
* All switches on the rack will connect to a single aggregate switch. 
* Aggregate switches will connect to core switch. 

![](../.gitbook/assets/gfs_network_topology.png)