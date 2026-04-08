# Write-ahead logging: the backbone of database durability

Write-Ahead Logging (WAL) is the dominant crash-recovery mechanism in modern databases. Its core principle is deceptively simple: **before any change is applied to a data page on disk, a log record describing that change must first be written to stable storage**. This single invariant — log before data — provides the foundation for atomicity and durability across virtually every production database system in existence, from SQLite to PostgreSQL to Oracle. WAL displaced shadow paging in the 1980s because it converts expensive random I/O into fast sequential appends, enables the highest-performance buffer management policy (steal/no-force), and produces a change stream that powers replication, point-in-time recovery, and change data capture. The ARIES algorithm, published by C. Mohan et al. in 1992, codified the most sophisticated version of WAL and remains the theoretical basis for recovery in nearly every major DBMS.

This report provides a mechanically precise walkthrough of WAL — its theory, its alternatives, and its concrete implementations in SQLite and PostgreSQL — grounded in primary sources, database internals documentation, and the landmark academic papers that defined the field.

---

## The fundamental mechanism behind crash recovery

A database faces a core tension: it must keep data in volatile memory (DRAM) for performance but guarantee that committed transactions survive power failures, crashes, and OS faults. WAL resolves this tension through a two-phase approach. First, every modification is recorded as a log record appended sequentially to a persistent log file. Second, the actual data pages are written to disk lazily, at a time chosen by the buffer manager — not forced at commit time. A transaction commits by flushing its log records (culminating in a commit record) to stable storage via `fsync()`. The data pages may still be dirty in the buffer pool. If the system crashes before those dirty pages reach disk, recovery replays the log from the last checkpoint, reapplying committed changes (redo) and rolling back uncommitted ones (undo).

The WAL protocol enforces two rules. The **undo rule** requires that before a dirty page is flushed to disk, all log records for that page must be flushed first — this enables rollback of uncommitted changes that were written prematurely. The **redo rule** requires that all log records for a transaction are flushed before the commit record is acknowledged to the client — this ensures committed transactions can be reconstructed after a crash. Together, these rules enable the **steal/no-force** buffer management policy: the buffer manager may evict ("steal") uncommitted dirty pages at any time, and committed transactions need not force their data pages to disk. This policy yields optimal runtime performance at the cost of requiring both redo and undo capabilities during recovery.

WAL directly provides two of the four ACID properties. **Durability** is guaranteed because the log, flushed to stable storage before commit acknowledgment, contains all information needed to reconstruct committed transactions. **Atomicity** is guaranteed because uncommitted transactions can be rolled back using the undo information in the log, and partially-completed writes are detected and corrected during recovery. Consistency and isolation are handled by concurrency control mechanisms (locking, MVCC) that operate alongside WAL.

## Why sequential I/O made WAL the universal choice

WAL became dominant for one overwhelming mechanical reason: **on spinning disks, sequential writes are roughly 100× faster than random writes**, and WAL converts all transaction modifications into sequential log appends. A transaction modifying 1,000 scattered pages writes 1,000 sequential log records to a single file rather than performing 1,000 random seeks across the data files. Even on modern SSDs, sequential I/O delivers significantly higher throughput due to reduced flash translation layer overhead and better wear leveling.

The steal/no-force policy enabled by WAL is the highest-performance buffer management strategy available. The "no-force" property means commit latency is bounded by a single sequential log flush, typically microseconds rather than the milliseconds required to force scattered data pages. The "steal" property means the buffer pool can evict any page at any time, preventing memory exhaustion during large transactions that modify thousands of pages. Shadow paging, the approach WAL replaced, requires the opposite policy — no-steal/force — creating both memory pressure (cannot evict dirty pages) and high commit latency (must force all modified pages).

WAL also produces a natural change stream. The sequential log file serves as the basis for streaming replication, point-in-time recovery, WAL archiving, and change data capture. Shadow paging produces no such change stream. This secondary benefit has become increasingly important as distributed systems and real-time analytics demand continuous access to database change events.

## Alternatives to WAL and their mechanical trade-offs

### Shadow paging trades I/O efficiency for recovery simplicity

Shadow paging, used in IBM's original System R prototype (1970s), maintains two page tables: a shadow (committed) and a current (in-progress). When a page is modified, a new physical copy is written at a fresh disk location, and the current page table is updated. Committing atomically swaps the root pointer from shadow to current. Recovery is trivial — discard the current page table and revert to the shadow. No log replay is needed.

Shadow paging lost to WAL because it scatters related data across the disk, destroying locality and requiring random I/O for every modification. It cannot support fine-granularity (record-level) locking — it operates at page granularity, causing false conflicts. It does not support partial rollbacks within a transaction. And it provides no replication or incremental backup mechanism. LMDB is the most prominent modern system using a shadow-paging variant (copy-on-write B+ trees), but it accepts the single-writer constraint and works best for read-dominated workloads where its zero-copy reads via `mmap` compensate for write amplification.

### ARIES is WAL perfected, not an alternative to it

A common misconception conflates ARIES with a separate recovery method. **ARIES is the most sophisticated refinement of WAL**, not an alternative. Developed by C. Mohan and colleagues at IBM Research and published in 1992, ARIES defined the canonical WAL protocol that virtually every modern database implements in some variant. Its key innovations beyond basic WAL include: **Compensation Log Records (CLRs)** that log undo operations themselves, preventing unbounded logging during repeated crash-restart cycles; **fuzzy checkpoints** that capture dirty-page and active-transaction tables without halting the system; **physiological logging** (physical page identification with logical intra-page description) that permits page reorganization after logging; and the **Repeat History** paradigm, where redo replays all logged operations — including those of uncommitted transactions — to reach the exact crash-time state before performing undo.

Pre-ARIES WAL implementations existed (notably Franco Putzolu's work at Tandem Computers in the 1980s), but they lacked ARIES's formal rigor and completeness. ARIES's three-phase recovery (analysis → redo → undo) with LSN-based comparison, CLR-bounded undo, and fine-granularity locking support became the template that DB2, SQL Server, MySQL/InnoDB, and others followed.

### Command logging, write-behind logging, and other modern departures

**Command logging**, used by VoltDB and derived from the H-Store research prototype, takes a radically different approach: it logs the stored procedure name and input parameters instead of individual data modifications. A single log record replaces potentially thousands of tuple-level entries, reducing log volume dramatically. However, this requires **deterministic execution** — the same inputs must produce identical outputs on replay — which prohibits non-deterministic SQL, timestamps, or external calls. Recovery is slower because transactions must be re-executed rather than having their effects replayed directly.

**Write-Behind Logging (WBL)**, proposed by Arulraj and Pavlo at CMU (VLDB 2016), inverts WAL for non-volatile memory (NVM). On NVM, WAL's sequential-I/O advantage disappears because random byte-addressable writes are fast. WBL applies changes directly to the database on NVM first, then logs only timestamps and metadata afterward. Their Peloton prototype showed **1.3× higher throughput** than NVM-WAL with recovery times under 100 milliseconds. However, WBL remains a research prototype, and Intel's discontinuation of Optane persistent memory in 2022 has dampened commercial prospects.

**LSM-tree databases** (RocksDB, LevelDB, Cassandra) are sometimes confused with WAL alternatives, but they **still require WAL**. Writes go first to a WAL (sequential append) and then to an in-memory MemTable. When the MemTable fills, it flushes to an immutable SSTable on disk. The WAL protects the volatile MemTable — without it, a crash loses all unflushed data. WAL and LSM trees are complementary, not competing.

## How WAL works in SQLite: a single-writer design with shared-memory coordination

### The WAL file is a sequence of checksummed frames

SQLite's WAL file (the `-wal` file alongside the database) begins with a **32-byte header** containing a magic number (`0x377f0682` for little-endian or `0x377f0683` for big-endian checksums), the file format version (`3007000`), the database page size, a checkpoint sequence number, two salt values, and a checksum over the header. Following the header are zero or more **frames**, each consisting of a 24-byte frame header and exactly `page_size` bytes of page data.

The frame header stores the database page number, a "commit size" field (non-zero only for commit frames, where it records the database size in pages after the transaction), two salt values that must match the WAL header, and a cumulative checksum. This checksum design is critical: the checksum for each frame is computed over the WAL header's first 24 bytes plus all preceding frame headers (first 8 bytes each) and page data, accumulating sequentially. **A single corrupted frame invalidates itself and everything after it**, providing a clean truncation point for crash recovery.

The salt values serve a clever purpose. When a checkpoint completes and the WAL is reset, Salt-1 is incremented and Salt-2 is randomized. Any old frames still physically present in the WAL file (from before the checkpoint) will have stale salt values and fail validation, effectively marking them as invalid without requiring the file to be zeroed or truncated.

### The WAL-index enables lock-free page lookups in shared memory

The WAL-index (the `-shm` file) is a memory-mapped shared-memory structure — never `fsync`'d, fully reconstructible from the WAL after a crash. It is organized in **32,768-byte blocks**, each containing a page-number array (`aPgno`) and a hash table (`aHash`). The first block also contains a **136-byte header** with the WAL-index version, a change counter, the maximum valid frame number (`mxFrame`), the number of backfilled frames (`nBackfill`), and five **read marks** used for concurrency control.

The hash table answers the query: "Given page number P and a maximum frame index M, what is the highest frame index ≤ M that contains page P?" The hash function is `h = (P × 383) % 8192` with linear-probing collision resolution. Since the hash table has 8,192 slots for at most 4,062 entries (in the first block) or 4,096 entries (subsequent blocks), empty slots are guaranteed, bounding probe length. Under default auto-checkpointing at 1,000 pages, the WAL-index rarely exceeds a single 32 KiB block.

The header stores two identical copies of its first 48 bytes (bytes 0–47 and bytes 48–95), enabling readers to detect torn writes without locking: if the two copies disagree, the reader retries.

### Readers use snapshot isolation through read marks

When a read transaction begins, the reader examines `mxFrame` in the WAL-index header and claims one of five read-mark slots (`read-mark[0]` through `read-mark[4]`). The reader sets its chosen read mark to the current `mxFrame` value, establishing an **end mark** — the highest frame this reader will consider. `read-mark[0]` is special: it is always zero, meaning the reader will ignore the WAL entirely and read directly from the database file.

This end mark provides snapshot isolation. Even as new transactions commit frames to the WAL, existing readers continue seeing the state as of their end mark. To read page P, the reader calls `FindFrame(P, end_mark)` on the WAL-index hash tables. If a matching frame exists in the WAL, the page data is read from that frame. Otherwise, the page is read from the main database file. **Readers never block writers, and writers never block readers** — the WAL append and the frozen snapshot are independent.

### A single writer appends frames under an exclusive lock

SQLite permits exactly **one concurrent writer**. The writer acquires an exclusive `WAL_WRITE_LOCK` and appends frames to the WAL for each modified page. Non-commit frames have their commit-size field set to zero; the final frame of a transaction is a **commit frame** with a non-zero commit-size recording the total database page count. After writing the commit frame, the writer updates `mxFrame` in the WAL-index header, making the new frames visible to subsequent readers. Under `PRAGMA synchronous=FULL`, the WAL file is `fsync`'d before updating `mxFrame`; under `PRAGMA synchronous=NORMAL`, the sync is deferred to the next checkpoint.

If `mxFrame` equals `nBackfill` (all WAL content has been checkpointed) and no readers hold `WAL_READ_LOCK(N)` for N > 0, the writer **resets** the WAL: it increments Salt-1, randomizes Salt-2, writes a new header, and begins appending frames from the start of the file, overwriting stale data without truncation.

### Four checkpoint modes control the durability-concurrency trade-off

Checkpointing transfers WAL content back to the main database file. SQLite defines four modes with increasing aggressiveness:

- **PASSIVE** (mode 0): Copies as many frames as possible without waiting for readers or writers. Used by auto-checkpointing (triggered at **1,000 pages**, approximately 4 MB at 4 KiB page size). If an active reader's read mark blocks progress, the checkpoint simply stops and reports partial completion. The busy-handler is never invoked.

- **FULL** (mode 1): Acquires the exclusive writer lock, then waits (invoking the busy-handler) until all readers are reading from a snapshot that has already been fully backfilled. Then checkpoints all remaining frames and syncs the database file.

- **RESTART** (mode 2): Performs a FULL checkpoint, then additionally waits until all readers have released their WAL read locks (reading only from the database file). This guarantees the next writer can reset the WAL from the beginning.

- **TRUNCATE** (mode 3): Performs a RESTART checkpoint, then truncates the WAL file to zero bytes.

The critical concurrency constraint: a checkpoint scanning `WAL_READ_LOCK(0..4)` cannot backfill frames past any active reader's read-mark value. If a reader holds `WAL_READ_LOCK(2)` with `read-mark[2] = 500`, the checkpoint cannot backfill frames beyond 500, because that reader expects the database file to reflect the state at frame 500 — not a later state that includes frames 501+.

### Crash recovery is a single forward scan with checksum validation

When the first connection opens a WAL-mode database with an existing WAL file, it performs recovery by acquiring exclusive locks on all lock slots (preventing concurrent access), then scanning the WAL sequentially from the first frame. For each frame, it verifies that the salt values match the WAL header and that the cumulative checksum is valid. The scan stops at the first frame with a mismatched salt or invalid checksum. The `mxFrame` value is set to the last valid **commit frame** (non-zero commit-size field) — any valid frames after the last commit frame are discarded as belonging to an incomplete transaction.

The WAL-index is then reconstructed from scratch, with `nBackfill` set to zero. Importantly, **SQLite does not immediately replay WAL frames into the database during recovery**. Instead, it sets up the WAL-index so that subsequent reads find the correct page versions from either the WAL or the database file. The actual transfer occurs during the next checkpoint.

A partially-written frame — caused by a crash during a write — will fail checksum validation. Because the checksum is cumulative, this cleanly truncates the valid WAL at the corruption point. If the corrupted frame belonged to an uncommitted transaction (no commit frame was written), the transaction is effectively rolled back by being ignored.

## How WAL works in PostgreSQL: ARIES-influenced redo-only recovery with resource managers

### WAL records are typed by resource manager and carry block references

Every PostgreSQL WAL record begins with a **24-byte `XLogRecord` header** containing: `xl_tot_len` (total record length), `xl_xid` (transaction ID), `xl_prev` (pointer to previous record for backward traversal), `xl_info` (operation-specific flags), `xl_rmid` (resource manager ID), and `xl_crc` (CRC-32C over the entire record). The `xl_rmid` field dispatches the record to one of approximately **21 built-in resource managers**, each responsible for a database subsystem. `RM_HEAP_ID` (ID 10) handles row-level INSERT, UPDATE, and DELETE operations; `RM_BTREE_ID` (ID 11) handles B-tree index operations; `RM_XACT_ID` (ID 1) handles COMMIT and ABORT records; `RM_XLOG_ID` (ID 0) handles checkpoint records and WAL switches.

Following the fixed header, each record contains zero or more `XLogRecordBlockHeader` structures identifying the data blocks referenced (each with a relation file locator, block number, fork flags, and optional full-page image), then the resource-manager-specific payload data. Since PostgreSQL 15, extensions can register **custom resource managers** with IDs in the 128–255 range via `RegisterCustomRmgr()`.

### LSN is a byte offset into an infinite WAL stream

The **Log Sequence Number (LSN)** is a 64-bit unsigned integer representing a byte offset into the conceptual infinite WAL stream. Displayed in `segment/offset` format (e.g., `0/16B3790`), it serves as the total ordering mechanism for all WAL records. Every data page stores a `pd_lsn` field — the LSN of the last WAL record that modified it. During recovery, a non-backup-block WAL record is replayed **only if its LSN exceeds the page's `pd_lsn`**, ensuring idempotent redo. PostgreSQL exposes several LSN positions: `pg_current_wal_insert_lsn()` (where the next record will be inserted), `pg_current_wal_flush_lsn()` (how far has been flushed to disk), and on standbys, `pg_last_wal_replay_lsn()` (how far recovery has replayed).

WAL data is organized into **segment files** of 16 MB by default (configurable at `initdb` time), stored in `$PGDATA/pg_wal/`. Each segment file is named with a 24-hex-character string encoding the timeline ID, logical segment number, and physical file number. Within segments, data is organized into **8 KiB pages**, each starting with an `XLogPageHeaderData` containing a magic number (version-specific), timeline ID, and page address. After a checkpoint, segments no longer needed are recycled (renamed) rather than deleted, bounded by `min_wal_size` (default 80 MB) and `max_wal_size` (default 1 GB).

### WAL buffers, the WAL writer, and the flush path

WAL records are first written to **WAL buffers** in shared memory — a circular buffer sized by the `wal_buffers` parameter (default: auto-tuned to 1/32 of `shared_buffers`, typically several megabytes). Backend processes insert records via `XLogInsert()`, which acquires partitioned WAL insert locks (supporting concurrent insertion from multiple backends since PostgreSQL 9.4), reserves space by advancing the insert LSN, copies the record, and releases the lock.

WAL buffers are flushed to disk under four conditions: when a transaction commits with `synchronous_commit = on` (the default), when the buffers fill, when the **WAL writer** background process wakes up (every `wal_writer_delay`, default **200 ms**), or when a checkpoint occurs. The WAL writer exists to batch-flush WAL buffers, reducing the number of individual `fsync()` calls that committing backends would otherwise need. The `commit_delay` parameter (default 0) can introduce a brief pause before flushing to enable **group commit** — batching multiple transactions' flushes into a single I/O operation.

### Full page writes solve the torn page problem

A critical subtlety: when PostgreSQL writes an 8 KiB data page, the operating system may write it in smaller blocks (512 bytes or 4 KiB). If a crash occurs mid-write, the on-disk page becomes a **torn page** — a mix of old and new data. Replaying a WAL record on top of a torn page produces corruption, because the WAL record assumes the page is in either the old or new state, not a mixture.

**Full page writes** (`full_page_writes = on`, the default) solve this. The first time any data page is modified after a checkpoint, the **entire 8 KiB page image** is included in the WAL record as a backup block (full-page image, or FPI). Subsequent modifications within the same checkpoint cycle log only the row-level changes. During recovery, backup blocks are written directly to the data page unconditionally — no LSN comparison needed — restoring the page to a known-good state. Non-backup records are then applied on top using LSN comparison for idempotence.

This mechanism has a significant performance cost: FPIs roughly double WAL volume after each checkpoint. The `wal_compression` parameter (supporting pglz, lz4, and zstd since PostgreSQL 9.5 and later) compresses FPIs to mitigate this. The `wal_log_hints` parameter extends FPI generation to hint-bit-only modifications, which is required for `pg_rewind` and automatically enabled when data checksums are active.

### Checkpoints establish recovery boundaries by flushing all dirty buffers

A PostgreSQL checkpoint performs four operations: identifies all dirty buffers in the shared buffer pool, flushes them to disk with `fsync()`, writes a CHECKPOINT WAL record containing the **redo point** (the LSN from which recovery must begin), and updates the `pg_control` file with the checkpoint location. Checkpoints are triggered by elapsed time (`checkpoint_timeout`, default **5 minutes**), by WAL volume approaching `max_wal_size` (default **1 GB**), or by manual `CHECKPOINT` commands.

The `checkpoint_completion_target` parameter (default **0.9**) spreads dirty-buffer writes across 90% of the checkpoint interval to prevent I/O spikes — a technique called **spread checkpointing**, introduced in PostgreSQL 8.3. Without spreading, a checkpoint would flush all dirty buffers in a burst, potentially saturating the I/O subsystem and causing latency spikes for concurrent queries.

After a checkpoint completes, WAL segments prior to the redo point become eligible for recycling. This bounds WAL disk consumption and limits recovery time: recovery only needs to replay WAL from the last checkpoint's redo point, not from the beginning of time.

### Crash recovery replays WAL forward from the last checkpoint

When PostgreSQL starts after an unclean shutdown, the startup process reads `pg_control` to find the last checkpoint location, retrieves the checkpoint's redo point from the corresponding WAL record, and begins **sequential forward replay** from that LSN. For each WAL record, the appropriate resource manager's `rm_redo` function is invoked. Backup blocks (FPIs) overwrite the target page unconditionally. Non-backup records are applied only if the record's LSN exceeds the target page's `pd_lsn`.

PostgreSQL's recovery is **redo-only** — there is no explicit undo phase. This is a significant departure from classical ARIES. Instead of undoing uncommitted transactions, PostgreSQL relies on **MVCC**: uncommitted row versions remain on data pages but are invisible to other transactions through the visibility rules (checking `xmin`/`xmax` transaction IDs against the commit log). The MVCC mechanism makes uncommitted writes harmless without requiring explicit undo, eliminating the need for Compensation Log Records (CLRs) and the ARIES undo pass.

This design choice has important implications. PostgreSQL's checkpoints flush **all** dirty buffers rather than recording a dirty page table (as ARIES does). PostgreSQL uses full-page images for torn-page protection rather than relying solely on LSN-based redo as ARIES specifies. And PostgreSQL's recovery is simpler — a single forward pass rather than ARIES's three passes — at the cost of storing MVCC metadata in data pages.

### WAL archiving and streaming replication extend recovery across time and space

PostgreSQL's WAL serves triple duty: crash recovery, archival backup, and replication. **WAL archiving** (`archive_mode = on`) copies completed WAL segments to an archive location via `archive_command` or `archive_library`. Combined with a base backup, archived WAL enables **Point-in-Time Recovery (PITR)**: restoring the database to any moment by replaying archived segments up to a target time, LSN, transaction ID, or named restore point. The `recovery.signal` file triggers archive recovery; `standby.signal` triggers continuous standby mode.

**Streaming replication** transmits WAL records over the network in near-real-time. The primary spawns **WAL sender** processes that read WAL and stream it to standbys, where **WAL receiver** processes write incoming records to `pg_wal/`. The standby's startup process continuously replays received WAL, maintaining a near-current replica. The `synchronous_commit` parameter controls durability guarantees across the spectrum: `off` (asynchronous, risk of data loss), `local` (wait for local flush only), `remote_write` (wait for standby OS receipt), `on` (wait for standby flush), and `remote_apply` (wait for standby replay — the strongest guarantee, enabling read-your-writes consistency on the standby).

**Replication slots** (introduced in PostgreSQL 9.4) prevent the primary from recycling WAL segments until all connected standbys have received them. Physical slots track replay position; logical slots track decoding position for logical replication. Stale replication slots are a common operational hazard — an inactive slot prevents WAL recycling indefinitely, eventually consuming all disk space.

The `wal_level` parameter controls how much information is written to WAL. The `minimal` level generates only enough for crash recovery. The `replica` level (default since PostgreSQL 10) adds information for archiving and physical replication. The `logical` level adds tuple-level change information for logical decoding and logical replication, enabling cross-version and heterogeneous replication.

## The papers and people who defined database recovery

### ARIES formalized WAL into its definitive form

The foundational paper is **C. Mohan, Don Haderle, Bruce G. Lindsay, Hamid Pirahesh, and Peter M. Schwarz, "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging," *ACM Transactions on Database Systems*, Volume 17, Number 1, March 1992, pp. 94–162 (DOI: 10.1145/128765.128770)**. Published by IBM Almaden Research Center, ARIES introduced the Repeat History paradigm, CLRs, fuzzy checkpoints, physiological logging, and LSN-based recovery. The ARIES family expanded through companion papers: ARIES/IM for index management (Mohan and Levine, SIGMOD 1992), ARIES/KVL for key-value locking (Mohan, VLDB 1990), and ARIES/NT for nested transactions (Rothermel and Mohan, VLDB 1989).

### Jim Gray defined the transaction concept and wrote the WAL protocol

**Jim Gray** (1944–2012) received the **1998 ACM Turing Award** "for seminal contributions to database and transaction processing research and technical leadership in system implementation." At IBM San Jose Research Laboratory, Gray defined the Write-Ahead Log protocol and the DO-UNDO-REDO framework for System R's recovery manager, documented in **Gray et al., "The Recovery Manager of the System R Database Manager," *ACM Computing Surveys*, Vol. 13, No. 2, June 1981, pp. 223–242 (DOI: 10.1145/356842.356847)**. His paper **"The Transaction Concept: Virtues and Limitations" (VLDB 1981)** defined the transaction abstraction. His textbook with Andreas Reuter, ***Transaction Processing: Concepts and Techniques* (Morgan Kaufmann, 1993, ISBN: 1-55860-190-2)**, remains the definitive reference on the subject.

### Härder and Reuter named ACID and classified recovery techniques

**Theo Härder and Andreas Reuter, "Principles of Transaction-Oriented Database Recovery," *ACM Computing Surveys*, Vol. 15, No. 4, December 1983, pp. 287–317 (DOI: 10.1145/289.291)** formalized the **ACID acronym** (Atomicity, Consistency, Isolation, Durability) and established the steal/no-steal, force/no-force taxonomy that classifies every recovery scheme by its buffer management policy. This paper — with over 1,375 citations — provided the conceptual vocabulary that the entire field still uses.

### Shadow paging originated in Lorie's System R work

**Raymond A. Lorie, "Physical Integrity in a Large Segmented Database," *ACM Transactions on Database Systems*, Vol. 2, No. 1, March 1977, pp. 91–104 (DOI: 10.1145/320521.320540)** described the shadow paging mechanism used in System R. The broader System R project was documented in **Astrahan et al., "System R: Relational Approach to Database Management," *ACM TODS*, Vol. 1, No. 2, 1976** and evaluated in **Chamberlin et al., "A History and Evaluation of System R," *Communications of the ACM*, Vol. 24, No. 10, 1981 (DOI: 10.1145/358769.358784)**.

### Implementation-specific references anchor the mechanical details

SQLite's WAL implementation is documented at sqlite.org/wal.html (overview), sqlite.org/walformat.html (file format, WAL-index structure, locking protocol), and sqlite.org/fileformat.html (database file format including WAL checksum algorithm). The academic treatment is **Gaffney, Prammer, Brasfield, Hipp, Kennedy, and Patel, "SQLite: Past, Present, and Future," *Proceedings of the VLDB Endowment*, Vol. 15, No. 12, 2022, pp. 3535–3547**. SQLite was created by **D. Richard Hipp** in 2000; WAL mode was added in version 3.7.0 (July 2010).

PostgreSQL's WAL is documented across postgresql.org/docs/current/wal-intro.html (concepts), wal-internals.html (record format and LSN), wal-configuration.html (tuning), wal-reliability.html (fsync and full-page writes), and continuous-archiving.html (PITR). The internal structures are defined in the source headers `xlogrecord.h`, `xlog_internal.h`, and `rmgrlist.h`. **Hironobu Suzuki's *The Internals of PostgreSQL*** (interdb.jp) provides the most detailed third-party treatment of PostgreSQL's WAL architecture.

## Conclusion

WAL's dominance rests on a mechanical advantage that has proven remarkably durable across four decades of hardware evolution: converting random writes to sequential appends. ARIES codified this advantage into a formal recovery protocol that supports fine-granularity locking, steal/no-force buffering, and bounded-time recovery through fuzzy checkpoints. SQLite and PostgreSQL implement strikingly different variants — SQLite uses a single-writer, checksummed frame-based design with no explicit redo replay (deferring recovery to checkpointing), while PostgreSQL implements a resource-manager-dispatched, ARIES-influenced redo-only system with full-page images for torn-page protection and MVCC eliminating the need for undo. Both systems extend WAL well beyond crash recovery: SQLite achieves concurrent readers through WAL-index read marks and snapshot isolation; PostgreSQL uses WAL as the foundation for streaming replication, PITR, and logical decoding. The emergence of persistent memory may eventually shift the calculus — write-behind logging eliminates WAL's sequential-I/O rationale when random writes are equally fast — but for the foreseeable future, WAL remains the indispensable mechanism that makes database durability tractable.