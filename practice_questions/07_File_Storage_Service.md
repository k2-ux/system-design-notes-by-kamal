# Design a File Storage Service like Google Drive
> Complete interview preparation guide — Q&A format, beginner to advanced.

---

## Table of Contents
1. [What Is It?](#what-is-it)
2. [Step 1 — Requirements](#step-1--requirements)
3. [Step 2 — Capacity Estimation](#step-2--capacity-estimation)
4. [Step 3 — The Hard Problems](#step-3--the-hard-problems)
5. [Step 4 — Database Design](#step-4--database-design)
6. [Step 5 — Sync Architecture](#step-5--sync-architecture)
7. [Full System Diagram](#full-system-diagram)
8. [Quick Revision](#quick-revision)

---

## What Is It?

**Q: What makes Google Drive architecturally interesting?**

> Three hard problems:
> 1. **Deduplication** — millions of users uploading the same files. How do you avoid storing duplicates and wasting petabytes?
> 2. **Delta sync** — a user edits one sentence in a 1 GB file. Do you re-upload 1 GB? No.
> 3. **Conflict resolution** — two devices edit the same file simultaneously while offline. Which version wins?

---

## Step 1 — Requirements

**Q: What are the functional requirements?**

> 1. Users can upload and download files of any type
> 2. Files sync automatically across all of a user's devices
> 3. Users can share files/folders with others (view / comment / edit permissions)
> 4. Version history — users can restore previous versions of files

**Q: What are the non-functional requirements?**

> - **Durability** — 99.999999999% (11 nines) — files must NEVER be lost under any circumstance
> - **Availability** — 99.9% uptime
> - **Consistency** — edits on one device must appear on all other devices
> - **Security** — files accessible only by authorized users
> - **Low latency** — small files upload/download fast; large files handled via chunking

**Q: Which non-functional requirement matters most for file storage, and why?**

> **Durability.** If Google Drive lost your files permanently, no amount of speed, privacy, or availability would fix that. Data loss is catastrophic and unrecoverable — it's the one failure mode that cannot be retried or worked around.

---

## Step 2 — Capacity Estimation

**Q: Walk through storage capacity for Google Drive.**

> **Assumptions:**
> - 1 billion users
> - Each user stores an average of 15 GB (free tier limit)
> - 10% of users are active daily
>
> **Storage:**
> ```
> Total storage: 1B users × 15 GB = 15 exabytes (15,000 petabytes)
> Daily uploads: 100M active users × ~10 MB average upload = 1 PB/day new data
> ```
>
> **Conclusion:** Only object storage (S3 / Google Cloud Storage) at massive scale handles this. Files cannot go in a database.

---

## Step 3 — The Hard Problems

### Problem 1: Deduplication

**Q: Two users upload the exact same 5 GB video with different filenames. Does Google store it twice?**

> **No — Content Hashing + Deduplication:**
>
> ```
> User A uploads: "vacation_video.mp4" (5 GB)
> User B uploads: "movie_final_v2.mp4" (same exact file, different name)
>
> Step 1: Split file into 4 MB chunks
> Step 2: Compute SHA-256 hash of each chunk
>
> User A's file:     User B's file:
> chunk_a3f9...  ←──── chunk_a3f9... (already exists!)
> chunk_b72c...  ←──── chunk_b72c... (already exists!)
> chunk_d891...  ←──── chunk_d891... (already exists!)
>
> → Google stores ZERO new bytes for User B's file
> → Just adds a metadata pointer: "User B's file = [a3f9, b72c, d891]"
> ```
>
> Google saves petabytes this way across billions of users.

**Q: Both users see their own filename — how?**

> The actual file data is stored once. But metadata is stored per user:
> ```
> S3 Storage (one physical copy):
>   chunk_a3f9 → [actual bytes]
>   chunk_b72c → [actual bytes]
>
> Metadata DB (per user, per file):
>   user_A | "vacation_video.mp4" | chunks: [a3f9, b72c, d891]
>   user_B | "movie_final_v2.mp4" | chunks: [a3f9, b72c, d891]
> ```
> Each user sees their own filename. The storage is shared invisibly. Like two shortcuts pointing to the same file — the shortcut names are different, the underlying file is one.

### Problem 2: Delta Sync

**Q: You change one sentence in a 1 GB document. Does Google Drive re-upload the entire 1 GB?**

> **No — Delta Sync using chunk-level diffing:**
>
> ```
> Original file (1 GB = 256 chunks of 4 MB):
>   Chunk 1:   hash "a3f9..."
>   Chunk 2:   hash "b72c..."
>   ...
>   Chunk 47:  hash "cc21..."  ← you edited this sentence
>   ...
>   Chunk 256: hash "ff34..."
>
> After your edit:
>   Chunk 47:  hash "NEW_HASH" ← only this changed
>
> Google Drive client compares old hashes vs new hashes
>   → Only Chunk 47 changed
>   → Upload ONLY Chunk 47 (4 MB instead of 1 GB)
>   → Update metadata: position 47 = NEW_HASH
>   → Other devices download only Chunk 47
> ```
>
> Result: 250x less data transferred. Critical for slow connections and large files.

### Problem 3: Conflict Resolution

**Q: You edit a file on your laptop and phone simultaneously (both offline). Both come online. Two different versions exist. What happens?**

> **Google Drive keeps BOTH versions — never silently overwrites:**
>
> ```
> Laptop version (comes online first):
>   → Saved as: "report.docx"  (original filename, first write wins)
>
> Phone version (comes online second):
>   → Saved as: "report (conflicted copy - Kamal's phone - 2024-01-15).docx"
> ```
>
> Both files appear in Drive. Google notifies you: "conflict detected, here are both versions — you decide."
>
> **Why not Last Write Wins?** Last Write Wins would silently delete one version of your work. Annoying the user with a conflict notification is infinitely better than permanently destroying their data.
>
> **Rule:** When in doubt, preserve data. Never silently overwrite.

---

## Step 4 — Database Design

**Q: What databases does Google Drive use and for what?**

> ```
> Data Type          Database          Reason
> ────────────────────────────────────────────────────────────
> File chunks        S3/GCS            Object storage — only option for petabytes of files
> File metadata      PostgreSQL        Structured: filename, size, owner, permissions, chunk list
> User sessions      Redis             Ephemeral, fast auth token lookups
> Version history    Cassandra         Time-ordered versions per file, high write volume
> Sharing/perms      PostgreSQL        Relational: who has what access to which file
> ```

**Q: Why are file chunks in S3 and not PostgreSQL?**

> Average file = 10 MB. 1 billion users × 15 GB = 15 exabytes. Databases are for structured data, not binary blobs.
>
> S3 provides:
> - Unlimited storage (scales to exabytes)
> - Automatic geo-replication (11 nines durability built-in)
> - Cheap per-GB pricing for cold data
> - Presigned URLs for direct browser upload/download (bypasses app servers)

---

## Step 5 — Sync Architecture

**Q: How does Google Drive sync a file change to all your devices?**

> ```
> You edit file on laptop
>         ↓
> Drive client detects change (file watcher)
>         ↓
> Client chunks file, computes hashes, diffs vs last known hashes
>         ↓
> Upload only changed chunks to S3
>         ↓
> Update metadata in PostgreSQL
>         ↓
> Publish "FILE_UPDATED" event to Kafka
>         ↓
> Sync Service notifies all your other connected devices
>         ↓
> Phone/tablet downloads only the changed chunks from S3
>         ↓
> Local Drive folder updates ⚡
> ```

**Q: How does sharing with permissions work?**

> ```
> Permissions table (PostgreSQL):
> ┌──────────┬──────────────┬────────────┬───────────┐
> │ file_id  │ user_id      │ permission │ granted_by│
> ├──────────┼──────────────┼────────────┼───────────┤
> │ file_123 │ user_kamal   │ OWNER      │ -         │
> │ file_123 │ user_alice   │ EDIT       │ user_kamal│
> │ file_123 │ user_bob     │ VIEW       │ user_kamal│
> └──────────┴──────────────┴────────────┴───────────┘
>
> When user accesses file:
>   1. Auth service checks permissions table
>   2. OWNER/EDIT → can read and write
>   3. VIEW → can only read, download blocked for edit
>   4. No entry → access denied
> ```

---

## Full System Diagram

```
[User Device - Laptop/Phone]
  │ File watcher detects changes
  │ Chunks + hashes computed locally
  ↓
[Load Balancer]
  ↓
[Upload Service]          [Download Service]
  │                              │
  ↓                              ↓
[S3 / GCS]  ←──────────────────────
(chunked file storage,
 geo-replicated,
 11 nines durability)
  │
[Metadata DB - PostgreSQL]
(filenames, chunk lists,
 permissions, version history)
  │
[Kafka]
(FILE_UPDATED events)
  ↓
[Sync Service]
(notifies other devices)
  ↓
[Other Devices - download changed chunks]
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| File storage | S3 + geo-replication | Petabyte scale; 11 nines durability built-in |
| Deduplication | Content hashing (SHA-256 per chunk) | Same file stored once regardless of filename or owner |
| Efficient sync | Delta sync (chunk-level diff) | Upload only changed chunks — 250x less data for small edits |
| Conflict resolution | Keep both versions | Never silently overwrite — data loss is worse than user annoyance |
| File metadata | PostgreSQL | Structured relational data — filenames, permissions, chunk lists |
| Version history | Cassandra | Time-ordered, high write volume, simple queries |
| Cross-device sync | Kafka events | Notify other devices asynchronously after each change |

### Three things to mention proactively in an interview
> 1. **Content hashing for deduplication** — shows you understand storage efficiency at petabyte scale
> 2. **Delta sync** — shows you know NOT to re-upload entire files on small edits
> 3. **Conflict resolution keeps both versions** — shows you know data preservation > convenience
