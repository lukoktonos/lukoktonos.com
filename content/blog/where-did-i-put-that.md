+++
title = "Where Did I Put That?"
date = 2026-03-03
+++

*Note: This post was written with Claude as an experiment in AI-assisted writing.*

*A survey of file systems and object stores — from spinning disks to distributed blobs — and what actually matters when you're choosing one.*

## "The Lie That Cost Us the Weekend"

It started as a non-incident.

A service wrote "checkpoint" files to a shared NFS mount. It wasn't exotic: write bytes, `fsync`, keep the file descriptor open, and let other processes read the latest checkpoint whenever they needed it. In staging, everything looked perfect. In production, half the fleet began serving stale checkpoints—quietly, confidently, and indefinitely.

Nothing was "wrong" in the logs. No errors. No timeouts. No corruption. Just clients reading yesterday's reality because they never closed and reopened the file, and NFS never promised they had to see the update. The team spent the weekend hunting ghosts in application code before realizing the bug was a contract no one had read. It was right there in RFC 1813. The behavior was documented, correct, and completely invisible until it wasn't.

That's the trap: storage APIs look simple, and they feel like physics. You write a thing, you close a thing, the thing exists. For a single machine, the illusion mostly holds.

```c
int fd = open("data.txt", O_RDONLY);
ssize_t n = read(fd, buf, sizeof(buf));
close(fd);
// Elegant. Simple. And full of implicit promises
// that 30 years of distributed systems would slowly,
// painfully reveal as contracts no one actually signed.
```

On a local file system, this sequence carries a pile of implied guarantees: that readers will see your writes "soon enough," that `rename()` is atomic, that a closed file is durable, that operations happen in the order you issued them. Move one hop away—into a network file system, a distributed file system, or an object store—and those guarantees don't disappear all at once. They fail selectively. They fail differently. And the failures ripple outward into application design, operational playbooks, and incident response.

Every storage system makes promises about three things:

- **Visibility:** when do *other* clients see my write?
- **Failure atomicity:** what state exists after a crash mid-operation?
- **Ordering:** can someone observe B without having observed A?

Most of the time the promises hold well enough that you don't notice them. The interesting question—the design question—is which promises break first, under what conditions, and what happens to everything around them when they do.

This post is a guided tour of those promises across the storage spectrum: local file systems, network file systems, distributed file systems, and object stores. Not a catalog of features. A survey of contracts—especially the ones you'll only discover at 2 AM.


## Principles

_Before we compare systems, we need a shared lens. Skip this section and you'll end up with a list of features rather than a framework for thinking. The features change. The questions don't. The six clauses below — visibility, failure atomicity, ordering, metadata, durability, and impedance — are the template for every system in the survey._


In 2018, Craig Ringer posted to the PostgreSQL hackers mailing list with a subject line that made a lot of engineers stop what they were doing: _"PostgreSQL's handling of fsync() errors is unsafe and risks data loss at least on XFS."_

The scenario: a PostgreSQL instance running on XFS encountered a storage error during async writeback. The kernel marked the affected pages as failed and set an error flag. When PostgreSQL called `fsync()` at the next checkpoint, `fsync()` returned `EIO`. PostgreSQL treated the checkpoint as failed and didn't advance the redo position. Correct behavior so far.

Then PostgreSQL retried the checkpoint. It called `fsync()` again on the same file descriptor. This time `fsync()` returned success — because the first `fsync()` call had already cleared the error flag on the dirty pages. As Ringer put it in his post: _"When fsync() returns success it means 'all writes since the last fsync have hit disk' but we assume it means 'all writes since the last SUCCESSFUL fsync have hit disk'."_

A subsequent `fsync()` could report success even though earlier writeback had failed — because the error state had already been consumed and cleared — so the retry moved the system forward on a false premise. The write never made it to disk. PostgreSQL completed the checkpoint and carried on. The data was gone. The database had no idea.

This wasn't a PostgreSQL bug in the conventional sense. PostgreSQL was calling `fsync()` and checking its return code — exactly what every piece of advice told it to do. The problem was that the contract between the application and the kernel, on Linux before kernel 4.13, did not guarantee what PostgreSQL (and most other database engineers) had assumed it guaranteed.

A 2020 paper from the University of Wisconsin-Madison, "Can Applications Recover from fsync Failures?" (Rebello et al., USENIX ATC), studied the problem systematically across five widely-used applications: PostgreSQL, Redis, LMDB, LevelDB, and SQLite. The finding was unambiguous: although each application used different failure-handling strategies, none were sufficient. Fsync failures could cause data loss and corruption in all of them. Redis didn't even check `fsync()`'s return code before reporting success to the application layer.

The Linux kernel improved `fsync()` error reporting in version 4.13. PostgreSQL was patched to `PANIC` on `fsync()` failure. The PostgreSQL wiki documents the full history under ["Fsync Errors / fsyncgate 2018"](https://wiki.postgresql.org/wiki/Fsync_Errors); Dan Luu's [fsyncgate writeup](https://danluu.com/fsyncgate/) and [LWN coverage](https://lwn.net/Articles/752063/) go deeper.

The fix is not the interesting part. The interesting part is what the incident revealed: a contract clause that virtually every database engineer had assumed, that had been written down nowhere, and that turned out to be false on a major file system under a specific failure mode. Data was silently lost on production systems, by software doing exactly what it was supposed to do.


Every storage system is a contract. Most of the contract is implicit. You agree to the terms by writing the code. The fine print only becomes visible when something breaks. The six principles below make the contract explicit before the incident rather than after it.

### 2.1 Consistency is Three Questions

Engineers say "consistency" as though it's one thing. It isn't. There are three distinct questions, and conflating them is the source of more storage bugs than almost anything else.

**Visibility**: when does another client see my write? After your write completes, does a process on another machine see it immediately? Eventually? Only after it closes and reopens the file?

**Failure atomicity**: what does my own client see if I crash mid-write? The old version? A partial file? A torn inode? Local journaled file systems give you strong guarantees here. Object stores give you all-or-nothing — a partial PUT is never visible.

**Ordering**: if I do A then B, can another client observe B without A? Write a config file, then rename a sentinel to signal readiness. On a local file system, is the rename guaranteed to be visible after the config write? Not without `fsync` and barrier semantics. On an object store, per-object consistency doesn't mean multi-object ordering — which is why table formats like Iceberg exist.

The key question: _what is the smallest unit the system can make atomic?_ That unit is the primitive your application builds on — a journaled rename on ext4, a lease-protected rename on HDFS, a conditional PUT on S3.


### 2.2 Metadata is the Hidden Bottleneck

Here is the pattern that surprises nearly every team: they architect for throughput, benchmark for throughput, provision for throughput — and then hit a wall that has nothing to do with throughput.

The wall is metadata.

Metadata is a database. Directory listings, inode allocation, small-file creation, renames — these operations live in a fundamentally different performance regime than reading or writing large files. A file system delivering 10 GB/s of sequential reads can grind to a halt creating 10,000 small files per second.

Ask honestly: _is my workload metadata-dominant?_ Many small files? Frequent renames? Wide directory listings? If yes, your bottleneck is metadata, not throughput — and the "cheap and fast" storage choice can become neither at scale. The HDFS small-files problem is a NameNode heap problem. The CephFS MDS is a metadata bottleneck. S3 LIST is billed per request and tells you exactly what it is through its pricing model.


### 2.3 The Durability Stack is a Chain of Contracts

Every layer in the durability stack makes a promise: "if you get an acknowledgment from me, your data survives this failure mode." The problem is that layers lie. Disk firmware acknowledges flushes before writing to stable media. RAID controllers report success with data in volatile cache. NFS clients return from `write()` with data still in a buffer that hasn't hit the server.

**Durability is a chain, and if any layer acknowledges before the data is stable, your mental model is wrong** — not theoretically, but in the sense that you will eventually lose data and be surprised.

For local file systems, two contracts matter and cannot be conflated:

**Data durability**: call `fdatasync()` on the file descriptor. **Metadata durability**: call `fsync()` on the *parent directory* after creating or renaming a file.

Many engineers do the first and skip the second. The result: the file's content survives power loss but the directory entry doesn't. On the next boot, `data.final` simply doesn't exist. The inode is intact, the blocks are on disk, but the path is gone — the file ends up in `lost+found`. Ted Ts'o's writing on ext4 and the [LWN rename/write ordering discussion](https://lwn.net/Articles/457667/) cover exactly why this surprises people.

fsyncgate showed the chain can break below the application layer too: PostgreSQL was doing everything right, and the contract was being broken two levels down.


### 2.4 The API is the Data Model

The API you use to access storage is not neutral. It shapes what operations are cheap, what operations are expensive, and what invariants you can express. Choosing storage is partly choosing a data model.

A **directory tree** makes hierarchical organization and atomic rename cheap. Flat global enumeration is expensive.

An **object store** makes geographic distribution and large sequential objects cheap. Rename, append, and locks don't exist — the operations you relied on in a file system are either absent or expensive copy-then-delete simulations.

The impedance mismatch between how you want to think about your data and how the storage system actually works is a tax you pay every day. Choose the API that matches your access patterns.


### 2.5 Failure Domains: Blast Radius for Storage

Durability is about surviving a failure at a given layer. Failure domains are about *which* layer fails and how much it takes down with it — a distinct question that belongs at design time, not incident response.

When a node fails, what breaks? Most teams make this choice implicitly and discover the blast radius during an incident.

The hierarchy: **Disk → Node → Rack → Availability Zone → Region**. Each level requires a different redundancy mechanism and carries a different cost. ext4 on a single disk: failure domain is the disk. HDFS with rack-aware 3x replication: failure domain is a rack. S3 with cross-region replication: failure domain is a region.

Two questions that surface the decision before the hard way:

_Do you require atomic rename as a correctness primitive?_ If yes, pure object stores won't provide it. You need a file system, or a table format that reconstructs it via conditional writes.

_Will you ever enumerate the namespace at scale?_ Object store LIST is billed per request with no efficient "give me everything" operation. If you need this, you'll build a catalog — which is why Iceberg and Delta Lake exist.

Know your failure domain before you commit.


### 2.6 The Impedance Mismatch Tax

Every abstraction layer costs something. The bill always comes due — the question is whether you pay it during development or in production.

The costs come in four forms: **latency** (network hops, SDK retries, request signing); **semantic loss** (operations you give up when crossing a layer boundary — rename, append, locks); **debug surface** (more layers means more invisible state and more places for reality to diverge from expectation); and **blast radius** (a failure in a distant layer can break you in ways that are hard to attribute).

The right response isn't to minimize layers — sometimes more layers is correct. It's to know what each layer costs and decide explicitly that you're willing to pay it. Idempotent operations, application-level checksums, structured retry, and an external source of truth for ordering are how you build correctness on top of storage abstractions that don't provide it natively.


## The Contract Ledger

These six clauses are the template for every system comparison that follows. Here is the ledger in compact form:

| Clause                | The question to ask                                                 | Common failure when you assume wrong                                                 |
| --------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Visibility**        | When do other clients see my write?                                 | NFS cache serving stale data after a write from another client                       |
| **Failure atomicity** | What state exists after my crash?                                   | Partial file visible after incomplete write; torn inode after power loss             |
| **Ordering**          | Can B be observed without A when I wrote A then B?                  | Sentinel rename visible before config write on NFS or object store                   |
| **Metadata**          | Is my workload small-files, frequent-rename, or wide-listing heavy? | NameNode heap exhaustion; MDS overload; excessive LIST request costs                 |
| **Durability**        | Which layer in the stack am I trusting to honor flush semantics?    | `fsync()` returns success but data is in a write cache that won't survive power loss |
| **Impedance**         | What operations am I losing by crossing this layer boundary?        | Application depends on atomic rename; object store provides only copy-then-delete    |

_With this ledger in hand, every system in the survey becomes an exercise in filling it in — not a ranking, but a reading of the contract. What does this system buy you? What does it cost? Under what failure conditions does a clause break?_

_Let's find out._


## The Survey

### 3.1 Local File Systems

*The baseline. The systems where the contracts are clearest, the failure modes are best understood, and the tooling has had forty years to mature. Start here — not because local file systems are simple, but because everything that follows is a variation on what they get right and what they give up.*


Local file systems are the baseline — what most engineers picture when they think about storage. The POSIX interface is so familiar it feels like physics. It isn't. It's a contract, and understanding where it's strong, where it's silent, and where it depends on layers you don't control is the foundation for every system that follows.

### ext4

ext4 has been one of the most common default Linux file systems since around 2008. It descends from ext2 via ext3, inheriting a design philosophy of reliability and broad compatibility over novelty. If you've run Linux in production, you've run ext4.

The core mechanism that separates ext4 from its predecessors is journaling: before committing changes to the file system proper, ext4 writes a record of the intended changes to a journal (a reserved area of the disk). If the system crashes mid-operation, the journal lets the file system replay or discard the incomplete transaction on the next mount rather than leaving the structure in an inconsistent state. `fsck` on an ext2 volume after an unclean shutdown could take a very long time on large disks. On ext4, recovery is far faster.

On a local file system, visibility is immediate to processes on the same machine — there's no replication lag, no cache coherency protocol between nodes. The sharp edges are failure atomicity and ordering across crashes and caches, which is where the journaling mode matters.

But journaling has three modes, and which mode you're running determines exactly what the journal protects:

- **`data=journal`** — data and metadata are both journaled. Strongest guarantee; slowest; uncommon in practice.
- **`data=ordered`** — the default. Dirty data blocks are written before the corresponding metadata is committed to the journal, preventing metadata from pointing at uninitialized garbage after a crash. Durability still requires an explicit flush — `data=ordered` is not a substitute for `fsync()`.
- **`data=writeback`** — metadata is journaled but data writes are not ordered relative to metadata commits. Fastest mode; weakest guarantee. After a crash, a file can reference blocks containing old data from a previous allocation.

Most production systems run `data=ordered`. It's a reasonable default: you're protected against file system corruption, and you get consistent metadata. What it does *not* give you — what no journaling mode gives you without explicit flushing — is durability. A journaled write is not a durable write. The journal write can sit in volatile caches until the full stack flushes to stable media. And `close()` is not a durability boundary — it releases the file descriptor, nothing more.

This is where the contract gets specific. If your application needs data to survive a power failure, call `fdatasync()` or `fsync()`. If you need the directory entry to survive — so the file is actually reachable after a crash — `fsync()` the parent directory too:

```c
int fd = open("data.tmp", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, 0644);
write(fd, data, len);
fdatasync(fd);         /* flush file content */
close(fd);
rename("data.tmp", "data.final");

int dir_fd = open(".", O_RDONLY | O_DIRECTORY | O_CLOEXEC);
fsync(dir_fd);         /* flush the directory entry */
close(dir_fd);
```

Note: `data.tmp` and `data.final` must be on the same file system — `rename()` across file systems silently becomes copy-then-delete, losing atomicity.

**Failure domain**: a single disk, or whatever RAID/LVM layer sits beneath.


### ZFS

ZFS (Sun Microsystems, early 2000s) was designed around one question: what if the disk lies? Firmware returns success before data is stable. Sectors corrupt silently. The file system layer traditionally had no way to detect it.

ZFS's answer is **end-to-end checksumming**: every block carries a checksum stored in its parent block pointer. On reads, ZFS verifies the checksum; on a mirrored or RAIDZ pool, it can detect which copy is wrong, serve the correct one, and repair the damaged block automatically.

The other foundational decision is **copy-on-write (CoW)**: ZFS never overwrites data in place. Every modification writes to a new location, updates block pointers, then releases old blocks. A crash always leaves the disk at the state of the last committed transaction group (TXG) — there is no in-progress overwrite to interrupt. This is also what makes ZFS snapshots instantaneous: a snapshot is a record of which blocks belonged to the file system at a point in time, not a data copy.

One gotcha for database engineers: the **`recordsize` property** defaults to 128 KB. Databases with smaller page sizes (PostgreSQL defaults to 8 KB) pay up to 16x write amplification. Set `recordsize` to match the database page size before creating the dataset.

```bash
zfs create -o recordsize=8k tank/postgres
```

**Failure domain**: single node (mirrors/RAIDZ expand within-node; ZFS has no built-in network replication).


### XFS

XFS (Silicon Graphics, early 1990s; long-time RHEL default) was designed for large files, high throughput, and concurrent I/O. Its key structural difference from ext4 is **allocation groups**: the disk is divided into independent regions, each with its own free space and inode management. Heavy parallel writes that would contend on ext4's shared metadata structures spread across groups instead.

What XFS trades away: no built-in checksums (unlike ZFS), no native snapshots, more trust placed in the storage layer.

**Failure domain**: single disk or underlying storage — same as ext4.


### So What?

Local file systems give you the strongest consistency guarantees in this survey and the smallest failure domain. Choose ext4 for a reliable, well-supported default. Choose ZFS if silent corruption detection, snapshots, and CoW atomicity justify the operational overhead. Choose XFS for parallel large-file workloads on enterprise storage. All three expose the same principle: the durability contract runs from your application code down through every layer to the storage medium, and every layer has to honor its part.


### 3.2 Network File Systems

*Local file systems give you the most predictable contract and the smallest failure domain. The moment you add a network — so that multiple machines can share the same namespace — you give up something. The question is which something, and whether you designed around it.*


The Network File System was designed at Sun in the mid-1980s to make remote files look like local ones. Mount a remote directory, get a directory tree. The POSIX interface, preserved across the network.

This was a useful lie. It held well enough that NFS became infrastructure — woven into data centers, HPC clusters, and shared home directories across the industry. The teams that notice the contract is different are the ones whose applications relied on a local-FS guarantee that NFS doesn't provide.

### NFS v3

NFS v3 (RFC 1813, 1995) is stateless: the server keeps no record of which clients have files open. Every operation is self-contained — the client sends full context with each request. This makes the server simple and crash-recoverable.

**Visibility**: NFS v3 implements *close-to-open cache consistency*. When a client opens a file, it validates its cached copy by checking `mtime` and size against the server. If those match, it reads from cache without going to the server. A write from client A becomes visible to client B when client B *closes and reopens* the file — not when A writes it, not when B reads again on an open descriptor.

By default, clients cache attributes for seconds to tens of seconds; exact behavior depends on mount options (`acregmin`, `acregmax`, `acdirmin`, `acdirmax`, `actimeo`).

**Failure atomicity**: the server has no knowledge of what the client was doing before a crash. Durability depends on the server's storage configuration.

**Ordering**: NFS v3 provides no cross-client ordering contract beyond server-side RPC atomicity; cache revalidation is not synchronized. `rename()` is atomic on the server, but other clients may observe the renamed file before they observe the data written to it — the data write and the rename are two separate RPCs.

### NFS v4 and v4.1

NFS v3's stateless model showed cracks over time: NLM was fragile across failures, WAN round-trips were expensive, and CIFS was gaining ground. The IETF took over NFS stewardship and in 2003 published RFC 3530: NFS v4, a deliberate break from v3.

v4 became stateful: the server tracks open files, byte-range locks, and *delegations* — grants that let clients cache aggressively without round-tripping on every operation. Locking moved into the core protocol (no separate NLM daemon). RENAME, CREATE, and REMOVE operations return a `change_info` structure — a per-client signal for cache invalidation, not a cross-client push.

The integrated locking uses *leases*. The lease model introduces a new failure mode: a partition longer than the lease period causes the server to expire client state — locks, delegations, and open handles are revoked.

NFS v4.1 added pNFS, allowing clients to read and write directly to storage devices. A storage device failure during a pNFS write may not surface through the same error path as a server failure — monitoring designed for traditional NFS may miss it.

**Amazon EFS** is managed NFSv4.1: same consistency model, same close-to-open behavior, same lease semantics — with no server to provision and automatic multi-AZ durability. General Purpose throughput mode is the default and the right choice for most workloads.

```c
/* Writer process */
int fd = open("config.tmp", O_WRONLY | O_CREAT | O_TRUNC, 0644);
write(fd, new_config, len);
fsync(fd);       /* flush to server via COMMIT — see stable-write semantics above */
close(fd);
rename("config.tmp", "config.json");   /* atomic on the server */

/* Reader process (different machine, same NFS mount) */
int fd = open("config.json", O_RDONLY);
read(fd, buf, sizeof(buf));
/* Is this the new config or the old one? */
close(fd);
```

On NFS v3, there are two problems. First: if the reader already has `config.json` open, it will keep reading from its cache — close-to-open consistency only triggers on open. Second: even if the reader reopens, the directory attribute cache on the reader's NFS client may not have expired yet. Because the cached directory `mtime` still reflects the pre-rename state, the client has no reason to re-issue a LOOKUP for the file name — it uses its cached name-to-filehandle mapping, which still points to the old file.

On NFS v4, delegations can mitigate the second problem: if the reader holds a delegation that the server considers conflicted by the rename, the server will recall it before proceeding — but whether a rename of a different file onto the delegated name triggers a recall is implementation-dependent. The first problem — readers holding open file descriptors — is an application design issue that no protocol version fixes.

### So What?

NFS is the clearest example of the impedance mismatch tax: the API looks identical to a local file system, so the contract feels identical — and that's where the damage happens. For NFS v3: treat it as eventually consistent for cross-client reads, and avoid using `rename()` as a cross-client commit primitive unless readers reopen on each read. NFS v4 tightens this substantially; the new surface area is the lease model under partitions.

Read the mount options. Read the RFC. Test the failure mode.


### 3.3 Distributed File Systems

*Network file systems share a namespace across machines but still depend on a server — a single point through which all operations flow. Distributed file systems remove that assumption. Data and metadata are spread across many nodes, failures are expected rather than exceptional, and the system is designed to survive them. The cost is a contract that diverges further from POSIX than NFS ever did.*


### HDFS

The Hadoop Distributed File System was designed at Yahoo in the mid-2000s to run on commodity hardware at the scale Google's GFS paper described. The design philosophy was explicit: hardware failure is the norm, not the exception. The result is a file system optimized for a specific workload: large files, written once, read many times, in long sequential streams. Anything outside that envelope — small files, random writes, frequent metadata operations — is outside the design envelope, and the system's failure modes cluster there.

A single NameNode maintains the entire file system namespace in memory. Clients talk to the NameNode to resolve paths and locate blocks, then stream data directly to and from DataNodes, bypassing the NameNode for data transfer.

**Visibility**: HDFS is built around a lease-held writer and readers that typically treat close as the visibility boundary. Streaming readers exist, but the write-once-read-many contract assumes data is consumed after close. There is no concurrent multi-writer support — one writer per file at a time, enforced by the lease.

**Failure atomicity**: For application-level commits, the common atomicity primitive is write-to-temp + close + rename, where rename is the namespace-atomic step. The rename is a NameNode metadata operation, atomic in the namespace: it either appears to all subsequent lookups or it doesn't.

**Ordering**: within a single writer, ordering is guaranteed by the write pipeline. The rename-as-commit pattern works because the NameNode serializes namespace metadata updates — rename is atomic in the namespace.

The NameNode is the architectural load-bearing wall of HDFS, and it is also the most common source of production pain at scale. Because it holds the entire namespace in JVM heap, the NameNode's memory consumption grows linearly with the number of files and blocks — not with the amount of data, but with the number of distinct objects.

```python
# Naive pattern: one file per event
# At 10,000 events/second → 864 million files/day
for event in event_stream:
    with hdfs.open(f"/events/{event.timestamp}/{event.id}.json", "w") as f:
        f.write(event.to_json())
# Each file+block consumes ~hundreds of bytes of NameNode heap.
# At ~1B files, metadata pushes into hundreds of GB — NameNode GC pauses,
# falls behind, stops responding. The cluster isn't out of disk. It's out of namespace.

# Fix: aggregate into fewer, larger files
with hdfs.open(f"/events/{date}/batch_{batch_id}.parquet", "w") as f:
    write_parquet(f, batch_of_events)
```

**Failure domain**: DataNode failures are transparent — the block replication factor absorbs them. NameNode failure, without HA, brings down the entire cluster. HDFS HA runs a standby NameNode with a shared edit log (via JournalNodes), allowing failover.


### CephFS

Ceph was designed at UC Santa Cruz in the mid-2000s by Sage Weil, aiming to be a general-purpose distributed storage system — one that could simultaneously present a POSIX file system (CephFS), an object store (RADOS Gateway), and a block device (RBD) on top of a common storage backend called RADOS.

One key architectural insight in Ceph is the separation of the metadata layer from the data layer. CephFS adds a Metadata Server (MDS) tier on top of RADOS — a separate layer of processes responsible for the file system namespace, directory structure, and inode management. Clients talk to the MDS for metadata operations and go directly to OSDs for data.

**Visibility**: CephFS provides strong consistency by default via a capability system — when a client holds a capability on a file, it can read or write without going to the MDS on every operation. When another client needs conflicting access, the MDS revokes the capability, forcing the first client to flush before proceeding.

**Failure atomicity**: operations acknowledged by the MDS and OSDs are intended to survive recovery — "completed" here means the client received an ACK and the operation was durably recorded in the relevant journals; session failures can still complicate edge cases.

**Ordering**: within a single client, CephFS provides ordered writes. Cross-directory atomic operations (rename across directories managed by different MDS instances) depend on MDS coordination — which is why hot directories and cross-subtree renames can cause surprising latency spikes under concurrent load.

```python
# The MDS bottleneck under metadata-heavy load

# Problematic pattern: millions of small files in a flat directory
output_dir = "/checkpoints/run_42/"
for step in range(1_000_000):
    path = f"{output_dir}/step_{step}.pt"
    with open(path, "wb") as f:
        f.write(checkpoint_bytes)
# At scale, the MDS for this subtree becomes the bottleneck.
# Writes slow down not because OSDs are saturated, but because
# every CREATE serializes through one MDS process.

# Better: shard across subtrees so multiple MDS instances share the load
for step in range(1_000_000):
    shard = step % 256
    path = f"{output_dir}/shard_{shard:03d}/step_{step}.pt"
    with open(path, "wb") as f:
        f.write(checkpoint_bytes)
```

**Failure domain**: OSD failures are absorbed by replication or erasure coding within RADOS. MDS failures trigger a standby MDS taking over the journal. The MDS tier has its own failure domain, separate from the data tier.


### GlusterFS

GlusterFS (Red Hat, mid-2000s) distributes the namespace via a hash-based Distributed Hash Table (DHT) with no dedicated metadata server — clients compute file locations algorithmically from the file path, so there is no NameNode equivalent to fail. Ceph's maturation has substantially reduced GlusterFS's mindshare; most new deployments now evaluate CephFS instead. It remains in production at many existing installations.

The key failure mode is **split-brain**: when a replica set partitions, both sides accept writes, and when the partition heals, self-heal cannot automatically determine which copy is correct. The dangerous property is that reads return a valid-looking file with no obvious error signal — the application sees plausible data, just not necessarily the right data. Resolution requires manual intervention (`gluster volume heal`). Replica count of 3 (majority quorum) mitigates this more reliably than 2.

**Failure domain**: the replica set. No centralized metadata service to fail — GlusterFS's key architectural advantage over HDFS.

### So What?

HDFS draws the line at the file: single-writer leases and atomic rename are real guarantees. What breaks is the metadata layer when the workload violates the large-file assumption. CephFS draws the line at the capability: strong consistency is real but runs through the MDS, so throughput dashboards can look fine while the application starves on metadata. GlusterFS draws the line at the replica set: consistency is intended within a healthy set, but split-brain is silent and requires manual resolution. Know where the line is before you write the code that depends on it.


### 3.4 Object Stores

*Distributed file systems stretched the POSIX contract but kept most of the vocabulary. Object stores discard it. No directory tree. No rename. No append. No locks. What you get instead is a flat namespace, an HTTP API, and a set of guarantees that are in some ways stronger than NFS and in other ways absent entirely.*


The object store model emerged from a simple observation: most of the complexity in a POSIX file system exists to support workloads that web-scale storage doesn't have. Strip them out, and what remains is something that can scale horizontally across thousands of machines, survive regional failures, and serve millions of concurrent readers without coordination overhead.

### S3

Amazon S3 stores objects in *buckets*. Each object has a key — a string up to 1,024 bytes, often written with `/` separators to simulate a directory hierarchy, but treated by S3 as a flat opaque identifier. There is no directory tree. There are no inodes. There is no rename.

**Visibility**: S3 has provided strong read-after-write consistency for all operations — GET, PUT, LIST, HEAD, and metadata — since December 1, 2020. Strong consistency doesn't imply multi-object atomicity — a workflow that writes object A then object B has no way to make those two writes atomic. This is the foundational reason table formats like Apache Iceberg and Delta Lake exist.

**Failure atomicity**: a PUT is all-or-nothing. Multipart uploads introduce a nuance: an incomplete multipart upload leaves orphaned parts that consume storage and are not visible as objects. Lifecycle rules to abort incomplete multipart uploads are standard operational hygiene.

**Ordering**: S3 provides no cross-object ordering guarantees. Write A then B, and a concurrent reader may observe B without A. The application is responsible for constructing ordering — typically by writing a manifest or pointer object last.

In 2024, S3 added first-class conditional writes — first `If-None-Match: *` (succeed only if the object does not exist), then `If-Match` / ETag (succeed only if the object matches a known ETag). Together these give you a real single-object CAS primitive.

```python
import boto3
from botocore.exceptions import ClientError

s3 = boto3.client("s3")

# If-None-Match: only succeed if the object doesn't exist yet.
try:
    s3.put_object(
        Bucket="my-bucket",
        Key="locks/leader",
        Body=b"worker-7",
        IfNoneMatch="*",
    )
    print("Won the election — proceeding as leader")
except ClientError as e:
    code = e.response["Error"]["Code"]
    if code == "PreconditionFailed":
        print("Lost the election — another worker got here first")
    elif code == "ConditionalRequestConflict":
        # Another conditional write is in-flight; backoff + retry
        print("Conflict with concurrent writer — retry with backoff")
    else:
        raise

# If-Match / ETag: only succeed if the object hasn't changed since you read it.
head = s3.head_object(Bucket="my-bucket", Key="config/settings.json")
current_etag = head["ETag"]
try:
    s3.put_object(
        Bucket="my-bucket",
        Key="config/settings.json",
        Body=updated_config,
        IfMatch=current_etag,
    )
except ClientError as e:
    code = e.response["Error"]["Code"]
    if code in ("PreconditionFailed", "ConditionalRequestConflict"):
        raise ConcurrentCommitException("Retry with updated state")
    else:
        raise
```

**Enumeration**: LIST operations return keys in lexicographic order, paginated via continuation tokens — you still pay per page and there is no atomic snapshot of a large prefix. LIST is billed per 1,000 objects returned.

**Failure domain**: S3 stores objects across at least three Availability Zones within a region by default. The data is AZ-redundant; availability is still a regional service property.


### GCS

Google Cloud Storage follows the same object model as S3 with a few meaningful differences.

**Visibility**: strong global consistency for all object operations.

**Failure atomicity**: GCS adds one meaningful multi-object primitive that S3 lacks: `compose()`. A compose operation concatenates up to 32 source objects into a single destination object as a single atomic operation — the destination is created or updated in one step. (Larger assemblies require chaining compose calls.)

```python
from google.cloud import storage

client = storage.Client()
bucket = client.bucket("my-bucket")

parts = []
for i, chunk in enumerate(data_chunks):
    blob = bucket.blob(f"uploads/object/part-{i:04d}")
    blob.upload_from_string(chunk)
    parts.append(blob)

# Atomically compose into the final object.
destination = bucket.blob("uploads/object/final")
destination.compose(parts)

for part in parts:
    part.delete()
```

**Ordering**: same as S3. GCS generation-match preconditions (`if_generation_match`) for conditional writes have been available since launch, predating S3's 2024 addition.

**Failure domain**: GCS multi-region storage replicates across geographically separated regions with strong consistency maintained across the replication.


### ADLS Gen2

Azure Data Lake Storage Gen2 adds *hierarchical namespace* (HNS). With HNS enabled, directory and file renames are single atomic metadata operations rather than enumerate-copy-delete sequences.

```python
# With HNS enabled, this rename is a single atomic metadata operation:
datalake_client.get_file_client("mycontainer", "_temp/data.parquet") \
    .rename_file("mycontainer/data/data.parquet")
# Without HNS: copy + delete — not atomic, not safe as a commit primitive.
```

Some tools and API paths don't take full advantage of HNS semantics — validate your tooling path before using rename as a commit primitive.


### MinIO

MinIO is an open-source S3-compatible object store for on-premises or private-cloud deployments where S3 is unavailable or data residency requirements apply. The API contract is the S3 contract; differences are operational. Verify conditional write support against your specific version before building correctness-critical code on top of it.


### Ceph RGW

Ceph RADOS Gateway provides an S3-compatible object store on top of the same RADOS backend as CephFS — useful if you're already running Ceph. API compatibility is high but not complete; `If-None-Match` and `If-Match` conditional write support varies by Ceph version and should be verified before building commit protocols that depend on them. Failure domain is configurable via the CRUSH map.


### So What?

The four assumptions that cause the most damage — visibility, atomicity, and ordering in disguise, expressed in file-system vocabulary: **rename is free and atomic** (it isn't; there's no rename); **LIST reflects current state** (it's paginated, not transactional, and billed per request); **a successful write makes data visible together with other writes** (per-object only, not across objects); **the API is the same everywhere** (MinIO and RGW implement the S3 API, but feature parity with current AWS S3 is not guaranteed). Design for the model you have.


### 3.5 From Primitives to Table Formats

*Every system in the survey so far is a product — something you deploy, mount, and hand to an application. This section is different. CAS, LSM trees, and table formats like Iceberg are patterns and layers built on top of the systems you've already chosen, reconstructing guarantees those systems don't natively provide.*


The survey has moved in one direction: away from POSIX and toward primitives that give up more guarantees in exchange for scale, durability, or cost. The systems in this section are about building it back — not by adding it to the storage layer, but by constructing it in the layer above.

### Content-Addressable Storage

Content-addressable storage names data by what it is — a cryptographic hash of its content — rather than where it lives.

The implications follow directly. Two identical objects have identical names, so deduplication is automatic. A stored object is immutable by definition: changing the content changes the hash, which changes the name. Verification is trivial: recompute the hash, compare to the name.

Git is the most widely deployed CAS system most engineers have used without thinking of it that way. Every commit, tree, and blob in a Git repository is a hash of its content, stored as an object in `.git/objects`. The entire history is a Merkle DAG.

```python
import hashlib, os, zlib

def cas_write(store_dir: str, data: bytes) -> str:
    digest = hashlib.sha256(data).hexdigest()
    prefix, suffix = digest[:2], digest[2:]
    obj_dir = os.path.join(store_dir, prefix)
    obj_path = os.path.join(obj_dir, suffix)
    if not os.path.exists(obj_path):
        os.makedirs(obj_dir, exist_ok=True)
        # Write compressed, then rename into place — idempotent and atomic
        # on a local file system. On an object store, you'd use a conditional
        # write instead; rename doesn't exist as a primitive there.
        tmp = obj_path + ".tmp"
        with open(tmp, "wb") as f:
            f.write(zlib.compress(data))
        os.rename(tmp, obj_path)
    return digest

def cas_read(store_dir: str, digest: str) -> bytes:
    prefix, suffix = digest[:2], digest[2:]
    obj_path = os.path.join(store_dir, prefix, suffix)
    with open(obj_path, "rb") as f:
        data = zlib.decompress(f.read())
    assert hashlib.sha256(data).hexdigest() == digest, "Corruption detected"
    return data
```

Visibility is immediate once published — once the object exists at its hash address, any reader with the hash can retrieve it. Failure atomicity is trivially satisfied by immutability. Ordering must be constructed above the CAS layer — which is exactly what Git's commit graph and Iceberg's snapshot chain both do.


### Log-Structured Merge Trees

CAS gives you immutable, efficiently verifiable storage. What it doesn't give you is efficient mutable state. LSM trees solve this by leaning into immutability rather than fighting it.

The core insight: don't update records in place. Write every mutation as a new record appended to a write-ahead log, then periodically sort and compact those records into increasingly large sorted files. Reads merge the results from multiple levels; the most recent write for any key wins. The structure trades write amplification against read amplification, with compaction as the knob that adjusts the tradeoff.

```python
class SimpleLSM:
    def __init__(self):
        self.memtable = {}       # in-memory, mutable; keys kept sorted on flush
        self.sstables = []       # list of sorted (key, value) lists, oldest-first
        self.wal = []            # write-ahead log — every mutation recorded here first

    def put(self, key, value):
        self.wal.append(("put", key, value))   # recorded first (in a real system, fsynced to disk)
        self.memtable[key] = value
        if len(self.memtable) > 1000:
            self._flush()

    def delete(self, key):
        self.wal.append(("del", key, None))    # recorded first
        self.memtable[key] = None              # tombstone

    def get(self, key):
        # Check memtable first, then SSTables newest-to-oldest.
        # This linear scan is intentionally naive; real LSMs use per-SSTable indexes
        # and Bloom filters to avoid scanning each SSTable on every point read.
        if key in self.memtable:
            val = self.memtable[key]
            return None if val is None else val
        for sstable in reversed(self.sstables):
            for k, v in sstable:
                if k == key:
                    return None if v is None else v
        return None

    def _flush(self):
        self.sstables.append(sorted(self.memtable.items()))
        self.memtable.clear()
        # Without compaction: read amplification grows with the number of SSTables,
        # and deleted keys (tombstones) continue to consume space indefinitely.
```

The failure scenario: deletes in an LSM tree don't remove data immediately — they write a tombstone. The actual data is removed during compaction. If compaction doesn't run — because the write rate is too high, the compaction budget is too low, or the operator disabled it — tombstones accumulate, deleted data continues to consume disk space, and read latency grows. This is the RocksDB "compaction debt" problem.


### Apache Iceberg

Iceberg gives you ACID table semantics on object storage — atomic multi-file commits, snapshot isolation, and time travel — by combining immutable data files with an atomic "current snapshot" publish step, typically mediated by a catalog or commit mechanism.

The data model is a layered tree of metadata. Data lives in immutable Parquet (or ORC or Avro) files. Manifest files list which data files belong to a snapshot. A manifest list points to a set of manifest files. A table metadata file points to the current snapshot. The current snapshot is identified by the metadata file that the table's current metadata pointer (often managed by a catalog; in file-based deployments, sometimes a locator file like version-hint.text) points to.

In most production deployments, the catalog is the linearizable "current metadata" register — it replaces the conditional object write with its own commit serialization, whether that's a database transaction (Hive Metastore), a distributed version store (Nessie), or a managed service (AWS Glue).

```python
# Iceberg commit protocol — simplified object-store-only illustration.
# Real deployments typically use a catalog for commit serialization.

import json, uuid, boto3
from botocore.exceptions import ClientError

s3 = boto3.client("s3")
BUCKET = "my-warehouse"
TABLE  = "db/events"

def iceberg_commit(new_data_files, base_snapshot_id, base_metadata_etag):
    new_snapshot_id = str(uuid.uuid4())

    # 1. Write manifest file (data files already written — immutable)
    manifest_key = f"{TABLE}/metadata/manifests/{new_snapshot_id}.avro"
    s3.put_object(Bucket=BUCKET, Key=manifest_key,
                  Body=json.dumps({"snapshot-id": new_snapshot_id,
                                   "data-files": new_data_files}).encode())

    # 2. Write new table metadata
    metadata_key = f"{TABLE}/metadata/{new_snapshot_id}.json"
    s3.put_object(Bucket=BUCKET, Key=metadata_key,
                  Body=json.dumps({"current-snapshot-id": new_snapshot_id,
                                   "parent-snapshot-id": base_snapshot_id,
                                   "manifest-list": manifest_key}).encode())

    # 3. Atomically advance the current-pointer.
    #    If-Match/ETag: S3 added If-Match conditional writes in November 2024.
    #    412 PreconditionFailed = another writer won; re-read and retry.
    #    409 ConditionalRequestConflict = concurrent write in-flight; backoff + retry.
    try:
        s3.put_object(Bucket=BUCKET, Key=f"{TABLE}/metadata/current-pointer",
                      Body=new_snapshot_id.encode(),
                      IfMatch=base_metadata_etag)
    except ClientError as e:
        code = e.response["Error"]["Code"]
        if code == "PreconditionFailed":
            raise ConcurrentCommitException("Stale snapshot; re-read and retry")
        elif code == "ConditionalRequestConflict":
            raise ConcurrentCommitException("Concurrent writer; backoff and retry")
        raise

    return new_snapshot_id
```

The correctness invariant: the current-pointer always points to a committed, consistent snapshot. Only step 3, the current-pointer swap, is conditional. Steps 1–2 are unconditional — orphaned files from failed commits are cleaned up by expiry policy. Orphan cleanup is both a correctness and a cost concern.


### So What?

The three systems form a progression. CAS makes immutability exact and verification trivial. LSM builds efficient mutable semantics on top of immutable writes via sequential append and compaction. Iceberg applies both to the table layer: immutable data files, an append-only metadata tree, and a single conditional pointer swap that makes the whole structure transactional.

Table formats are the antidote to the "rename is atomic" and "LIST is a catalog" assumptions from Section 3.4 — they reconstruct those primitives explicitly. When the storage contract is explicit, you can read it. When you can read it, you can design around it. Section 5 turns this into a decision tree.


## System Contract Snapshot

_The six clauses from Section 2 — visibility, failure atomicity, POSIX compliance, atomic replace, enumeration, and failure domain — applied to every system in the survey. No system is best across all columns. Every cell is a tradeoff, not a score._

|System|Visibility consistency|Failure atomicity|POSIX?|Atomic single-name replace|Atomic multi-object|Enumeration model + cost|Failure domain|
|---|---|---|---|---|---|---|---|
|**ext4**|Immediate (single node)|Journaled metadata; `data=ordered` enforces data-before-metadata on journal commit (durability still requires explicit flush)|Full|`rename()` is atomic; durability across power loss requires `fsync`/`fdatasync` + parent-dir `fsync`|—|`readdir()`; cheap; consistent|Single disk (RAID/LVM expands this)|
|**ZFS**|Immediate (single node)|Transactional copy-on-write; crash leaves you at the last committed TXG — no torn on-disk state|Full|`rename()` atomic; CoW avoids torn metadata|Snapshots are pool-wide consistent points-in-time|`readdir()`; cheap; consistent|Single node (mirrors/RAIDZ expand this)|
|**NFS v3**|Close-to-open cache consistency; stale reads possible without close/reopen|Client-cache + server-recovery dependent|Partial|Server `rename()` is atomic; cross-client visibility can lag due to attribute caching|—|`readdir()`; can be stale; moderate cost|NFS server (plus network as a failure injector)|
|**NFS v4 / v4.1**|Close-to-open remains the model; v4 adds integrated locking + delegations|Stateful client/server recovery; lease/state-reclaim dependent|Closer to POSIX than v3 in practice|Server `rename()` atomic; cross-client visibility is cache-mediated|—|`readdir()`; stale less likely than v3 under correct locking/delegation use|NFS server; stateful lease model means partitions longer than lease period can expire state|
|**HDFS**|Strong per-file under lease; readers observe last closed state|Lease recovery on crash; append model|No|Single-file primitives only; no cross-file atomicity|—|NameNode-mediated namespace ops; consistent; metadata bottleneck|Node / rack (rack-aware 3× replication typical)|
|**CephFS**|Strong under MDS coordination; configurable client caching|RADOS-backed storage; metadata journaling via MDS|Full (via MDS)|`rename()` via MDS — atomic within the namespace|—|MDS-consistent; metadata bottleneck at scale|Configurable (node → rack → DC)|
|**GlusterFS**|Strong within healthy replica sets; partitions can produce split-brain|Replica-based; split-brain can leave ambiguous state|Partial|`rename()` is atomic when replicas agree; split-brain makes it unsafe as a global commit primitive|—|DHT distribution; enumeration cost depends on layout|Node / replica set|
|**S3**|Strong for GET/PUT/LIST since Dec 1, 2020|PUT is atomic: you get the whole object or nothing|No|— (no rename); single-object conditional write (`If-None-Match`, `If-Match`) is your CAS primitive|— (copy+delete only; not atomic)|Strong consistency; lexicographic; paginated via continuation tokens; billed per request|AZ + Region; 11 nines durability across ≥3 AZs by default|
|**GCS**|Strong everywhere|PUT is atomic; `compose()` atomic for up to 32 source objects|No|— (no rename); generation-match conditional write|`compose()` atomic for up to 32 source objects|Strong; paginated; billed per operation|AZ + Region|
|**ADLS Gen2 (HNS)**|Strong|No partial write exposed at the file op level|No|Atomic rename is a metadata op with HNS enabled|—|Strong; paginated; HNS enables efficient directory ops|AZ + Region|
|**MinIO**|Strong (read-after-write in the S3 model)|PUT atomic|No|— (no rename); S3-style conditional ops (verify version support)|—|S3-like listing + pagination|Node / site (EC within site)|
|**Ceph RGW**|Read-after-write consistency for object operations|RADOS-backed; no partial object visible|No|— (no rename); conditional write support varies by Ceph version|—|S3-compatible listing + pagination|Configurable (node → rack → DC)|


## How to Choose

_The survey is done. This section is the distillation: six clauses, each phrased as a question, that map your workload to the right storage system. Answer them in order. The first question that produces a clear answer is usually the one that matters most._


The systems in this survey differ in ways that matter and ways that don't. A fast system with the wrong contract is worse than a slow system with the right one, because fast systems fail silently and confidently.

Work through the six clauses. Most workloads will have a decisive answer by clause three.


**Clause 1: Visibility — how quickly do other clients need to see your writes?**

If readers consume closed files — batch jobs, log archival, model checkpoints after training — visibility latency doesn't matter. Any system works. Default to object store plus a table format or catalog: cheap, durable, and the close-to-open model fits exactly.

If readers need to see updates on open file descriptors or across concurrent writers in near-real-time — shared config, leader election, coordination — you need strong visibility. Local file systems give you this for free. NFSv4 can provide predictable coherence when applications use integrated locking and delegations correctly; without that, it reduces to close-to-open. Object stores give strong per-object consistency but no cross-object atomicity. HDFS and CephFS give strong consistency under their respective coordination models. NFS's close-to-open model will not serve memory-mapped or persistent-tail readers.


**Clause 2: Failure atomicity — what must survive a crash mid-operation?**

If you need individual file writes to survive a crash in a consistent state, any journaled local file system (ext4, XFS, ZFS) gives you consistent on-disk file system state with correct `fsync` usage — application-level durability still requires correct flush ordering for both the file and its parent directory.

If you need multi-file or multi-object commits to survive a crash — either all files are visible or none are — your options narrow. Local file systems give you atomic rename within a volume. HDFS gives you atomic rename at the NameNode. Object stores do not give you this natively; you need a table format (Iceberg, Delta Lake) or an external catalog that mediates the commit.

If you need full ACID transactions across many files or objects, you are above the storage layer. You need a database or a table format with a catalog.


**Clause 3: Ordering — does correctness depend on observing operations in the order they were issued?**

If your application uses write-then-rename as a commit signal — write config, rename sentinel, readers check for sentinel — you need atomic rename as a real primitive. Local file systems provide it. HDFS provides it. Object stores do not; ADLS Gen2 with HNS provides it as a metadata operation; table formats reconstruct it via conditional writes.

If your application needs strict cross-client ordering across machines — client A writes A1 then A2, client B on a different node must never observe A2 without A1 — you need a system with explicit ordering guarantees. This typically means a database, a message queue, or a table format with snapshot isolation. No raw file system or object store provides this across concurrent writers without application-level coordination.

If ordering only matters within a single writer, almost every system in this survey satisfies you.


**Clause 4: Metadata — is your workload metadata-dominant?**

If your workload creates, renames, or lists many small files — build systems, CI pipelines, per-step checkpoints, per-event ingestion — metadata throughput is your bottleneck. NFS saturates the server before the data path. HDFS's NameNode is a JVM heap; small files exhaust it before exhausting disks. CephFS's MDS is the bottleneck for metadata-heavy jobs; shard across subtrees. Object store LIST is billed per page and not a transactional view — don't use it as a catalog. Default away from HDFS small-files patterns and object-store LIST-as-catalog.

If your workload is large-file sequential — video, backups, analytics over Parquet — metadata is not the bottleneck. Optimize for throughput and cost.


**Clause 5: Durability — which layer in the stack are you trusting to honor flush semantics?**

If you're on local storage: identify your journaling mode, use `fdatasync` for data and `fsync` on the parent directory for new files, and verify that the hardware beneath you honors flush commands.

If you're on NFS: the durability chain runs through the server's storage stack. `FILE_SYNC` writes are safer than `UNSTABLE` writes; `COMMIT` RPCs are the flush boundary.

If you're on an object store: PUT atomicity and durability are the provider's responsibility once the upload completes. Your job is to complete the upload — handle retries, verify ETags, and abort incomplete multipart uploads.

If you're on a distributed file system: three replicas with quorum writes typically tolerates a single node failure without data loss; tolerating two depends on replica placement and quorum assumptions. Two replicas with asynchronous replication does not guarantee survival of a simultaneous failure of both nodes.


**Clause 6: Impedance — what operations are you losing by crossing this layer boundary?**

Before committing to a storage system, list the operations your application depends on and check whether the target system provides them natively, as expensive simulations, or not at all.

**Atomic rename**: native on local file systems and HDFS; atomic metadata operation on ADLS Gen2 with HNS; copy-then-delete simulation on flat object stores; reconstructed via conditional writes by table formats.

**Append**: native on local file systems; append-only model on HDFS (single writer); not available on object stores (rewrite the full object); reconstructed via GCS `compose()` for small assemblies (bounded: 32 sources per call; chaining needed for larger).

**Byte-range locks**: native on local file systems; via NLM/integrated locking on NFS; not available on object stores or HDFS.

**Efficient namespace enumeration**: `readdir()` on local file systems and NFS; NameNode-mediated on HDFS; MDS-mediated on CephFS; paginated and billed on object stores; catalog-dependent on table formats.

If your application depends on an operation in the "not available" column for your chosen system, you have three options: redesign the application to work without it, add a layer above the storage system that reconstructs it, or choose a different storage system. Discovering which option applies at design time costs an afternoon. Discovering it in production costs a weekend.


_One final heuristic: if you're unsure which clause is decisive for your workload, it's usually Clause 3 (ordering) or Clause 6 (impedance). Those are the two where the gap between what the API implies and what the system actually provides is widest — and where the failure is most likely to be silent. If it's either of those, re-read Sections 2.1 and 2.6 before committing to a system._


## The Abstraction Always Leaks

Every abstraction in this survey was built to hide something. POSIX hides the difference between a local disk and a network server. NFS hides the difference between a local file system and a remote one. S3 hides the difference between a single machine and a globally distributed object store. Table formats hide the difference between a flat namespace with no transactions and a database with snapshot isolation.

The hiding is useful. Nobody wants to reason about disk geometry when they call `write()`. Nobody wants to think about RADOS object placement when they query a Parquet table. The abstraction is the point: it lets you think at a higher level, build faster, and compose systems that would otherwise require deep expertise at every layer.

But every abstraction leaks. Not randomly, and not unpredictably — they leak at the seams, at the edges of what they were designed to hide, under conditions the original designers either didn't anticipate or explicitly chose not to handle. The leak is always there. The only question is whether you've read the contract well enough to know where it is.

The survey showed you where each abstraction leaks. Local file systems leak at `fsync`: the interface implies that `write()` + `close()` is durable, and it isn't. NFS leaks at visibility: the interface implies that other clients see your writes when you make them, and they don't. HDFS leaks at scale: teams assume the NameNode is infinite, and it isn't — it's a JVM heap with a capacity limit that has nothing to do with your data. Object stores leak at atomicity: file-system muscle memory implies you can rename, and you can't — not natively, not atomically, not without a layer above.

These leaks are not bugs. They are engineering decisions made explicitly by people who understood the tradeoffs better than most. ext4's `data=ordered` mode does not promise durability; it promises consistency, and the difference is documented. NFS's close-to-open consistency is not a mistake; it's the tradeoff that makes the protocol stateless and crash-recoverable. S3's lack of rename is not an oversight; it's what makes the namespace scale to billions of objects without coordination. The designers knew. The contract was written. The fine print was there.

What wasn't there was a clear statement at the API surface that said: *this is where the abstraction ends.*

The storage API is not the storage contract. The API is the interface; the contract is the set of guarantees the system actually provides under failure, under load, under concurrent access. The gap between them is where incidents live. The fsync behavior, the close-to-open cache, the NameNode heap, the object store's lack of rename — none of these are surprising once you've read the relevant document. They are all surprising the first time you encounter them at 2 AM, when the system is behaving exactly as documented and nothing makes sense. Start with Clause 3 and Clause 6 from Section 5 if you don't know where to look.

The abstraction always leaks. The leak is usually documented somewhere. And it's almost always discoverable before the incident — if you go looking.

The only variable is when you read it.
