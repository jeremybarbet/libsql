# The wal interface
 ## append frames to the log
 ## locate page in the log

 [page 1] -> [page 5] -> [page 1] -> ...
 [frame1] -> [frame2] -> [frame3] -> ...

 ## read transactions:
 current max frame_no: 2

                         |       uncommitted ... |
 [page 1] -> [page 5] -> [page 1] -> ...
 [frame1] -> [frame2] -> [frame3] -> ...
                 ^- read mark

query:  page1 => frame1
        page5 => frame2

if the uncommitted frames are committed; the reader don't see the changes,
because it's only allowed to read pages anterior to the read mark.

 ## checkpointing:

any frame that is less than the smallest read mark can be checkpointed: no snpashot previous 
to that point will be read, so only the most recent version up to that mark needs to be kept.

                                  v readmark 2
[f1] -> [f2] -> [f3] -> [f4] -> [f5] -> [f6] -> ...
|           |    ^readmark 1
chekpointable

on checkpoint, the most recent version of each page is copied to the main database file.

 ## write transactions

a read transaction can be upgraded to a write transaction if:
- no other write transaction exist
- the readmark points to the most recent snapshot (if not, a new readmark need to be acquired if possible.)

# Sqlite wal
Sqlite wal uses a single log file. It can only checkpoint perform a full checkpoint when the last reader exits is a the very end of the wal.
*issues*: checkpoint starvation: if the last reader cannot catch up with the writer, the file cannot be checkpointed
*issues*: checkpointing blocks writes for the duration of the checkpoint process

wal locking: if a lock cannot be acquired by the wal, it returns `SQLITE_BUSY` to the pager. The pager calls the busy handler (basically exponential backoff). Poor performance under contention, unfair.
interprocess synchronization: sqlite wal uses a memory mapped file to handle cross-process synchronization, and share an index among processes

# Libsql wal

Libsql wal aims to address the specific needs of sqld:
- single process use
- replication (frame injection)
- many conccurent connections
- durability
- PITR

+ fix the current shortcomings of sqld:
  - long snashot creation
  - wal duplication (3x)
  - hacks (injection, lock stealing...)
