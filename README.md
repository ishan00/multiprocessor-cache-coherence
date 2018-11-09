## Introduction
Shared memory is a powerful abstraction for interprocess communication. Starting in the late 1980s, the emergence of multilevel caches reduced the memory bandwidth requirement of a single processor. This advancement led to the idea of developing multiprocessors where several processors shared a single physical memory. Such multiprocessor systems proved to be extremely cost-effective, provided that sufficient amount of memory bandwidth existed.

Most modern multiprocessors use seperate L1 and L2 caches rather than a shared cache for each processor. This arrangement leads to reduced contention that may exist for shared data items being read  by multiple processors simultaneously. However, these processors have a shared L3 cache which serves as another bridge to park information like processor commands and frequently used data in order to prevent bottlenecks resulting from the fetching of these data from the main memory. 

Unfortunately, use of separate caches introduces a new problem because the view of memory held by two different processors is through their individual caches, which, without any additional precautions, could end up seeing two different values. This leads to inconsistencies among the processors which is generally referred to as the cache coherence problem. There may be several reasons for this-

* Sharing of writable data
* Process migration
* I/O activity

There are two different aspects of memory system behavior, both of which are critical to writing correct shared-memory programs. The first aspect, called coherence, defines what values can be returned by a read. The second aspect, called consistency, determines when a written value will be returned by a read. This issue is defined by a memory consistency model, however we will be only focussing on coherence in our analysis. A memory system is coherent if : 

1. A read by a processor P to a location X that follows a write by P to X, with no writes of X by another processor between the write and read by P, always returns the value written by P.
2. A read by a processor to location X that follows a write by another processor to X returns the written value if the read and write are sufficiently separated in time and no other writes to X occur between the two accesses.
3. Writes to the same location are serialized; that is, two writes to the same location by any two processors are seen in the same order by all processors. For example, if the values 1 and then 2 are written to a location, processors can never read the value of the location as 2 and then later read it as 1.

## Snooping

### 1. Write-Once

#### States { INVALID, VALID, RESERVED, DIRTY }
#### Protocol

* **Read miss** - If another copy of the block exists that is in state DIRTY, the cache with that copy inhibits the memory from supplying the data and supplies the block itself, as well as writing the block back to main memory. If no cache has a DIRTY copy, the block comes from memory. All caches with a copy of the block set their state to VALID. 

* **Write hit** - If the block is already DIRTY, the write can proceed locally without delay. If the block is in state RESERVED, the write can also proceed without delay, and the state is changed to DIRTY. If the block is in state VALID, the word being written is written through to main memory (i.e., the bus is obtained, and a one-word write to the backing store takes place) and the local state is set to RESERVED. Other caches with a copy of that black (if any) observe the bus write and change the state of their block copies to INVALID. If the block is replaced in state RESERVED, it need not be written back, since the copy in main memory is current. 
* **Write miss** - Like a read miss, the block is loaded from memory, or, if the block is DIRTY, from the cache that has the DIRTY copy, which then invalidates its copy. Upon seeing the write miss on the bus, all other caches with the block invalidate their copies. Once the block is loaded, the write takes place and the state is set to DIRTY. 


### 2. Synapse

#### States { INVALID, VALID, DIRTY }
#### Protocol

* **Read miss** -  If another cache has a DIRTY copy, the cache submitting the read miss receives a negative acknowledgement. The owner then writes the block back to main memory, simultaneously resetting the bit tag and changing the local state to INVALID. The requesting cache must then send ,an additional miss request to get the block from main memory. In all other cases the block comes directly from main memory. Note that the block is always supplied by its owner, whether memory or a cache. The loaded block is always in state VALID. 
* **Write hit** - If the block is DIRTY, the write can proceed without delay. If the block is VALID, the procedure is identical to a write miss (including a full data transfer) since there is no invalidation signal. 
* **Write miss** - Like a read miss, the block always comes from memory-if the block was DIRTY in another cache, it must first be written to memory by the owner. Any caches with a VALID block copy set their state to INVALID, and the block is loaded in state DIRTY. The block’s tag in main memory is set so that the memory ignores subsequent requests for the block.

### 3. Berkeley

#### States { INVALID, VALID, SHARED-DIRTY, DIRTY}
#### Protocol
* **Read Miss** - The cache with state DIRTY or SHARED-DIRTY must provide the block contents to the requesting cache and its state is changed to SHARED-DIRTY. If there is no cache with states DIRTY or SHARED-DIRTY, then the block is fetched from the main memory. The state of the requesting cache is always set to VALID.
* **Write Hit** - If the block is DIRTY, then the write proceeds with no delay. Otherwise, if it is VALID or SHARED-DIRTY, then an invalidation signal is send on the bus before the write proceeds. The state is always changed to DIRTY in the requesting cache.
* **Write miss** - The block comes directly from the cache (if any) having the block in DIRTY or SHARED-DIRTY state or from the memory if there is no such cache. All other caches invalidates their copies and the requesting cache sets the state to DIRTY. 	

#### Important Points

* A block in either state SHARED-DIRTY or DIRTY must be written back to main memory if it is selected for replacement. A block in state DIRTY can be in only one cache. A block can be in state SHARED-DIRTY in only one cache, but it might also be present in state VALID in other caches.
* This protocol has two major differences from the Synapse approach
It uses direct cache-tocache transfers in the case of shared blocks.
DIRTY blocks are not written back to memory when they become SHARED requiring one additional state. 

### 4. Illinois

#### States { INVALID, VALID-EXCLUSIVE, SHARED, DIRTY}
#### Protocol
* **Read Miss** - Any other cache having a copy of the block puts it on the bus. If the block is DIRTY, then it is also written to the main memory. If the block is shared, then the cache with the highest priority provides the block on the bus. All caches having the copy of the block will observe the bus and set their states to SHARED, and the requesting cache sets the state of the loaded block to SHARED. If the block comes from memory, no other caches have the block, and the block is loaded in state VALID-EXCLUSIVE.
* **Write Hit** - If the block is DIRTY or VALID-EXCLUSIVE, it can be written immediately. If the block is SHARED, then the write is delayed until an invalidation signal can be sent on the bus, which causes all other caches with a copy to set their state to INVALID. The state of the block is always changed to DIRTY.
* **Write miss** - Like a read miss, the block comes from a cache, if any cache has a copy of the block. All other caches invalidate their copies, and the block is loaded in state DIRTY.

#### Important Points
It is assumed that the requesting cache will be able to determine the source of the block. Each time that a block is loaded it can therefore be determined whether or not it is shared. Blocks are written back at replacement only if they are in state DIRTY. 	

### 5. Firefly

#### States { VALID-EXCLUSIVE, SHARED, DIRTY }
#### Protocol

* **Read Miss** - If another cache has the block, SharedLine is raised. All the caches with the block respond by putting a copy on the bus. All caches set the state to SHARED. If the SharedLine was not raised, the block is supplied by the memory and the state set to VALID-EXCLUSIVE. If the owning cache has the block in DIRTY, it is written to memory
* **Write Hit** - If the block is DIRTY or VALID-EXCLUSIVE the write can be performed immediately, with the final state DIRTY. If the block was in state SHARED, the write is delayed until the bus is acquired and write to main memory is initiated. Other caches observe the write on the bus and update their copy of the block.
* **Write Miss** - If any other cache has the block, it supplies the copy. The requesting block determines from the SharedLine whether the block was provided by other cache or main memory. If the block came from memory, the state is set to DIRTY, If the block came from other cache the state is set to SHARED and requesting cache must write the block to memory. Other caches with the copy will observe this write and update the old block with new one.


### 6. Dragon

#### States { VALID-EXCLUSIVE, SHARED-DIRTY, SHARED-CLEAN, DIRTY }
#### Protocol
* **Read Miss** - If other cache has a DIRTY or SHARED-DIRTY copy, it provides the cache block, raises the SharedLine and the block state is set to SHARED-DIRTY in all caches. If other cache has the block in state VALID-EXCLUSIVE or SHARED-CLEAN it will provide the block, raise the SharedLine and the block state will be set to SHARED-CLEAN in all caches. If the SharedLine was not raised, the block is received from memory with state VALID-EXCLUSIVE.
* **Write Hit** - If the block is DIRTY or VALID-EXCLUSIVE the write can be performed immediately, with the final state DIRTY. If the block was in state SHARED-CLEAN or SHARED-DIRTY, the write is delayed until the bus is acquired and write to main memory is initiated. Other caches observe the write on the bus and update their copy of the block.
* **Write Miss** - As with a read miss, the block comes from a cache if it is DIRTY or SHARED-DIRTY and from memory otherwise. Other caches with copies set their local state to SHARED-CLEAN. Upon loading the block, the requesting cache sets the local state to DIRTY if the SharedLine is not raised. If the SharedLine is high, the requesting cache sets the state to SHARED-DIRTY and performs a bus write to broadcast the new content.


## Directory Based Protocols
Just as with a snooping protocol, there are two primary operations that a directory protocol must implement: handling a read miss and handling a write to a shared, clean cache block. (Handling a write miss to a block that is currently shared is a simple combination of these two.) To implement these operations, a directory must track the state of each cache block. In a simple protocol, these states could be the following : 

* **Shared (S)** - One or more processors have the block cached, and the value in memory is up to date (as well as in all the caches).
* **Uncached (U)** - No processor has a copy of the cache block.
* **Modified (M)** - Exactly one processor has a copy of the cache block, and it has written the block, so the memory copy is out of date. The processor is called the owner of the block.


| Initial State | Request | Response/ Action(by Directory Controller) | New State |
|:-----:|:-----:|:--------:|:------:|
|U    | Read Miss or Write Miss | <ul><li>Fetch block from the memory directly</li><li>Send the memory block to requesting cache</li></ul>| M |
| M | Read Miss | Send request to the cache holding the modified block to provide the data to requesting cache | S |
| | Write Miss | Send request to the cache containing the modified block to invalidate it | - |
| S | Read Miss | Reply to the requesting cache with the memory block | - |
| | Write Miss | <ul><li>Reply to the requesting cache with the memory block</li><li>Send request to all the caches which are sharing the block to invalidate it</li></ul>| M |
| | Write Hit | <ul><li>Send request to all the caches which are sharing the block to invalidate it</li><li>Reply to the requesting cache that the block can now be modified by it</li></ul> | M |

In addition to cache state, a directory must track which processors have data when in the shared state. This is required to for sending invalidation and intervention requests to the individual processor caches which have the cache block in shared state. Few of the popular implementation approaches are :

* Full bit-vector - In this approach, a bit field for each processor at the directory node are maintained. The storage overhead scales with the number of processors.
* Limited pointer - In this approach, directory information of limited number of blocks is kept at the directory to reduce storage overhead.

Scalability is one of the strongest motivations for going to directory based designs. What we mean by scalability, in short, is how good a specific system is in handling the growing amount of work that it is responsible to do . For this criteria, Snoopy protocols cannot do well due to the limitation caused when having a shared bus that all nodes are using in the same time. For a relatively small number of nodes, snoopy systems can do well. However, while the number of nodes is growing, some problems may occur in this regard. Especially since only one node is allowed to use the bus at a time, which will significantly harm the performance of the overall system. Directory-based systems on the other hand face no such bottleneck to constrain the scalability of the system. 

Hence we can say that Snoopy schemes are used on small scale multiprocessors which can live with the bandwidth constraints of the shared bus while Directory based schemes are better suited for building large scale, cache coherent multiprocessors where single bus is unsuitable as a communication mechanism.

### Analyzing performance of snooping protocols
For the comparison of the above mentioned protocols, we read another paper which used a simulation based approach for benchmarking the protocols. We briefly explain the method used and the important insights gained.

**Simulation Model** - The basic model consists of a Simula process for each processor, a process for each cache, and a single process for the system bus. Each processor, after performing useful work for some w cycles (picked from some distribution),
generates a memory request, puts that request into the service queue of its cache, and waits for a response, during which time no work is done. Processor utilization is measured by the ratio of time spent doing useful work to the total run time. System performance is measured by the total sum of processor utilization in the system.

Each cache services memory requests from its processor by determining whether the requested block is present or absent, or, more precisely, whether the request can be serviced without a bus transaction. If so, after one cycle the cache sends the processor a command to continue. If a bus transaction is required, a bus request is generated and inserted into the service queue of the bus. The cache sends the processor a command to continue only upon completion of the bus transaction.

The cache can also receive commands from the bus process relating to actions that must be performed on blocks of which it has copies. Such commands have higher priority for service by the cache than processor memory requests. In a multiprocessor, this is equivalent to matching a block address on a bus transaction and halting the service of processor requests to take action as specified by tbe protocol. After that action is completed, the cache is free to respond to processor requests. 

**System & Simulation Parameters** - Number of cycles to fetch an item from main memory was taken to be four times of cycles to fetch from cache. The block size is four words, where a single word is the smallest unit of data transferred. The number of processors are varied from 1 to 15. The cache size is varied from 2K to 16K words. Invalidation signals require one cycle of bus time. The probability of memory reference to shared block ranges from 0.1% to 5%. The hit ratio on private block is varied from 95% to 98%. There are other statistics like the probability of a private block being modified incase of reference, the percent of memory requests which are read, probability of shared block being referenced at two or more caches in small interval etc which are mentioned in the paper but we would not go in the exact details.

**Performance - Private blocks** - When the vast majority of all references are to the private blocks, the differences between the performance of these protocols is entirely due to private block overhead. For the cache-coherence protocols that we modeled there are only two differences in the handling of private blocks: the actions that must be taken on a write hit on a unmodified block, and the actions that must be taken when a block is replaced in the cache.

In case of write hits on unmodified private blocks - 
* Theoretically, any overhead is logically unnecessary since private blocks are never in other caches, but only the Dragon, Firefly, and Illinois schemes are able to detect this information dynamically. In these schemes the state can be changed from VALID-EXCLUSIVE to DIRTY without any bus transaction since it is known that the unmodified block is not present elsewhere. 
* The Berkeley scheme requires a single bus cycle for an invalidation signal. 
* Writeonce (like write-through) requires a single word write to main memory.
* The Synapse scheme performs a complete block load, as if a write miss had occurred.

The difference that arises in the replacement of blocks in the cache is that, for
the write-once scheme, the probability that a block needs to be written back is
reduced. In this scheme those blocks written exactly once are up-to-date in
memory and therefore do not require a write-back. 

**Performance - Shared blocks**
// TODO Graph


## Hybrid Protocols
## Conclusion
