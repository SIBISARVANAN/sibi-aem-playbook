# Oak & JCR Internals — NodeStore, MVCC, TarMK, MongoMK

> This is the deep architecture layer beneath `jcr-oak-fundamentals.md` (node types, property types, ValueMap). Where that file covers the API surface you code against, this covers what's actually happening underneath.

## JCR vs Oak

JCR (JSR-170/283) is a **specification only** — a contract for content-as-a-tree (nodes/properties, versioning, search, ACLs, observation). **Apache Jackrabbit Oak** is the actual engine implementing that contract, and what AEM 6.x/AEMaaCS runs on.

## Oak's Core Architecture — the NodeStore Abstraction

```
JCR API → Oak Core (query engine, security, observation)
              → NodeStore (pluggable storage)
                  ├── SegmentNodeStore (= TarMK)
                  └── DocumentNodeStore (= MongoMK / RDB)
```

This separation is *why* the same AEM application code runs unmodified on completely different storage backends.

## MVCC — Concurrency Without Locking

Oak never locks for reads. Every commit produces a new **immutable NodeState** — like a Git commit: the new state mostly shares structure with the previous one (structural sharing); only changed nodes get new records. Readers always see one consistent point-in-time snapshot, even during concurrent writes elsewhere.

## TarMK (Segment Node Store)

- Default engine for most single-instance/on-prem deployments; conceptually the basis for AEMaaCS storage too.
- Data is written as **segments** (~256KB binary chunks: Node/Property/Template/Blob/List/Map records), packed append-only into `.tar` files.
- A **Journal** file points to the current head revision.
- **Git-like commits:** a save writes only new/changed records; the new root mostly points to unchanged old segments.
- **Compaction** (Online Revision Cleanup) reclaims disk by removing segments no longer referenced by any retained revision — necessary because old superseded data is never deleted at write time.

| Compaction type | Behaviour |
|---|---|
| Tail compaction (default) | Incremental, online, no maintenance window |
| Full compaction | Thorough rewrite; historically needed downtime, modern Oak versions handle more of this online |

- Performance depends heavily on memory-mapped file access plus in-process caches (Segment/String/Node Cache) — undersized caches are a common root cause of slow instances.
- **Clustering limitation:** TarMK is local to one JVM's filesystem. **Cold Standby** provides one master + continuously-synced standby instances for DR/read-scaling — active-passive, not true multi-master.

## MongoMK (Document Node Store)

- Solves concurrent multi-author-instance writes by storing each JCR node as one MongoDB document.
- Mongo's own replication/sharding lets multiple AEM JVMs (each a distinct `clusterId`) read/write the same backing store concurrently.
- **Cluster coordination:** each node writes periodic heartbeats/leases to a `clusterNodes` collection; a missed lease triggers **recovery** to repair half-committed changes from a crashed node.
- Concurrent conflicting writes from different nodes can surface as `OakState0001: Unable to merge changes` — application code must handle retries.
- Every cache miss is a network round-trip to Mongo — Oak layers a `NodeCache`/diff cache plus an optional local **Persistent Cache** (TarMK-format file) specifically to reduce Mongo calls.

> **Interview-critical fact:** AEM as a Cloud Service does **NOT** use MongoMK. AEMaaCS reverted to a Segment Store approach backed by cloud blob storage (Azure Blob/AWS S3) instead of local disk — Adobe judged a separate Mongo cluster too much operational overhead for a cloud-native, auto-scaling architecture. MongoMK is primarily relevant to **on-prem/AMS AEM 6.x author clustering** today.

## Oak Indexing

- **Property indexes:** updated synchronously, inside the same commit — always immediately consistent.
- **Lucene indexes** (full-text, sorting, complex queries): updated asynchronously on a background "async" lane (typically every few seconds) — why newly-created content can briefly not appear in search results.

## Common Interview Questions

**Q: Why can't TarMK support multiple AEM author instances writing concurrently, but MongoMK can?**
TarMK's segment store is local to one JVM's filesystem with a single Journal head — there's no mechanism for two JVMs to coordinate writes to the same files. MongoMK delegates that coordination to MongoDB itself, which is designed for concurrent distributed access, plus Oak's own cluster lease/recovery mechanism.

**Q: Why doesn't a newly published page show up immediately in search?**
Lucene indexes update asynchronously on a background lane, not within the triggering commit — there's a small, normal lag between content being saved and it being searchable.

**Q: What replaced MongoMK in AEM as a Cloud Service?**
A Segment Store (TarMK-style) approach backed by cloud blob storage (Azure Blob/AWS S3), not local disk — chosen for lower operational overhead in a cloud-native, auto-scaling architecture.

**Q: What is structural sharing in the context of Oak's MVCC model?**
When a new NodeState is created after a commit, it reuses references to all unchanged child nodes from the previous state and only creates new records for what actually changed — directly analogous to how a Git commit mostly points to unchanged blobs/trees rather than copying the whole repository.

## Layman's Explanation

Think of it like a citywide records-keeping system.

JCR is just the rulebook — a standard saying "all content must be organized as a tree of folders and files, with version history, search, and permissions." It doesn't say which actual building stores anything.

Oak is the real records office that follows that rulebook — the staff and machinery that actually file things, look things up, and enforce the rules.

Behind Oak's front desk sits a swappable storage room — that's the NodeStore abstraction. The person at the front desk doesn't care whether the storage room is a back-office filing cabinet (TarMK) or a remote shared warehouse (MongoMK) — the experience at the counter is identical either way. This swappability is why the same AEM code runs unmodified on totally different storage backends.

MVCC (immutable NodeState) works like a smart photocopier: every time someone edits a page in a book, it doesn't scribble on the original — it photocopies a "new version" of the book, but cleverly reuses every unchanged page from before and only freshly prints the one page that actually changed. Anyone reading the book while someone else is mid-edit always sees a complete, consistent old copy — never a half-changed mess.

TarMK is that back-office filing cabinet: papers (segments) get stapled into folders (tar files) and slotted into the back of the drawer — nothing already filed ever gets edited in place, only added to. Over time, old superseded paper versions pile up uselessly — that's exactly why compaction exists: it's the cleanup crew that periodically clears out old, no-longer-needed copies so the cabinet doesn't grow forever. Because that cabinet physically lives in one room, only one office can write into it at a time. Cold Standby is like a second back-office across town that gets a photocopy of every new folder the moment it's filed — ready to take over instantly if the first room floods — but it's not simultaneously taking its own walk-in clients.

MongoMK solves the "only one office" problem differently: instead of one back-room, imagine many branch offices across the city, all plugged into the same shared, already-distributed central filing service (MongoDB) that knows how to handle multiple branches reading and writing into it at once. Each branch occasionally calls in (heartbeat/lease) to say "still open, still working" — and if a branch goes dark mid-task without warning, headquarters sends a crew to tidy up whatever paperwork it left half-finished (recovery). If two branches grab for the same file at the same instant, the system just tells one "someone beat you to it, try again" (conflict retry). Because every single lookup means a phone call to the central service, branches keep a personal photocopy of frequently-needed files on a side desk (persistent cache) so they aren't calling headquarters for everything.

AEM as a Cloud Service went a different way: rather than running a whole city of Mongo branch offices (too much overhead to operate), it went back to the single-filing-cabinet style (TarMK), just relocated that cabinet into a shared off-site cloud storage unit (Azure Blob/S3) instead of a local back room — simpler to run, while still getting cloud scale.

Finally — a brand-new book on the shelf gets an instant index card filed the second it arrives (property index), but the big master search catalog (Lucene index) is only rebuilt every few seconds in the background — which is exactly why something you just published can take a few seconds to actually show up in search.
