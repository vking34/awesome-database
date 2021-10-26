# Mysql


## Table of Contents

- [Transaction](#transaction)

- [ACID](#acid)

- [Isolation Levels](#isolation-levels) 

- Race conditions

- [Locking](#locking)

- Deadlocks

- [References](#references)

## Transaction

Transactions are atomic units of work that can be __committed or rolled back__. When a transaction makes multiple changes to the database, either all the changes succeed when the transaction is committed, or all the changes are undone when the transaction is rolled back.


## ACID

- A: Atomicity

- C: Consistency

- I: Isolation

- D: Durability

### Atomicity

- If one part of transaction does not work like it's supposed to, the other will fail as a result, and vice versa

- This property states that a transaction must be treated as an atomic unit, that is, either all of its operations are executed or none. There must be no state in a database where a transaction is left partially completed

- The database remains in a consistent state at all times — after each commit or rollback, and while transactions are in progress.

- Rollback:
    - __UNDO LOG__ contains undo log record which contains information about how to undo the latest change made by a transaction. If another transaction needs to see the original data as part of a consistent read operation, the unmodified data is retrieved from undo log records. Each transaction is allotted undo log slot based on the operations it has to perform.

    - __REDO LOG__ — This is another term used in recovery of data. By defination “The redo log is a disk-based data structure used during crash recovery to correct data written by incomplete transactions.” The changes which could not make it upto the data files because of system crash or any other reason are replayed automatically during initialization/restart of the server after the crash, and before the connections are accepted. To write data from REDO LOG to Disk InnoDB uses group commit functionality to group multiple flush requests from REDO LOG to avoid one flush for each commit taking place. With group commit, InnoDB makes a single write to the log file to perform the commit action for multiple user transactions that commit at about the same time, significantly improving throughput.

### Consistency

- The DB state must be valid after the transaction. All constraints must be satisfied.

The consistency aspect of the ACID model mainly involves internal InnoDB processing to protect data from crashes. Related MySQL features include:

- __Doublewrite Buffer__:

    - __Page__: a unit that specifies how much data can be transferred between disk and memory. A page can contain one or more rows, depending on how much data is in each row. If a row does not fit entirely into a single page, InnoDB sets up additional pointer-style data structures so that the information about the row can be stored in one page.

    - __Flush__: Whenever we write something to the database it isn’t written instantly to the disk, for performance reasons MySQL stores that either in the memory(Buffer pool — explained below) or in temporary disk storage. InnoDB storage structures that are periodically flushed include the redo log, the undo log, and the buffer pool. Flushing can happen because a memory area becomes full and the system needs to free some space, because a commit operation means the changes from a transaction can be finalized, or because a slow shutdown operation means that all outstanding work should be finalized. When it is not critical to flush all the buffered data at once, InnoDB can use a technique called fuzzy checkpointing to flush small batches of pages to spread out the I/O overhead.

    - The doublewrite buffer is __a storage area where InnoDB writes pages flushed from the buffer pool before writing the pages to their proper positions in the InnoDB data files(tables on the disk)__. If the system crashes in the middle of a page write, InnoDB can find a good copy of the page from the doublewrite buffer during crash recovery.

- __Crash recovery__:

    - On a server crash, InnoDB automatically recovers after the server is restarted. InnoDB automatically checks the logs and performs a roll-forward of the database to the present. InnoDB automatically rolls back uncommitted transactions that were present at the time of the crash.

    - All the data from the REDO LOG is restored before any connections are setup.


### Isolation

- Concurrent transactions must no effect each others

- The isolation mainly involves transactions. In a database system where more than one transactions are being executed simultaneously and in parallel, the property of isolation states that all the transactions will be carried out and executed as if it is the only transaction in the system. No transaction will affect the existence of any other transaction. MySQL achieves with help of locking strategies.


### Durability

- Data written by a successful transaction must be recorded in persistent state.

- The durability aspect of the ACID model involves MySQL software features interacting with your particular hardware configuration. The database should be durable enough to __hold all its latest updates even if the system fails or restarts__. If a transaction updates a chunk of data in a database and commits, then the database will hold the modified data. If a transaction commits but the system fails before the data could be written on to the disk then the data should be written back to the disk on restart. In short the data written should be persistent.


## Isolation Levels

- Consider a transaction:
```
BEGIN TRANSACTION;
SELECT * FROM T;
WAITFOR DELAY '00:01:00'
SELECT * FROM T;
COMMIT;
```

- 4 types of isolation levels:

    - 1. __READ UNCOMMITTED__: There isn’t much isolation in this level as it reads the latest uncommitted value at every step that can be updated from other transactions that haven’t committed yet. It can be a possibility that the row read which was recently updated by another transaction rollbacked because of failure of the transaction. So this leads to the problem of __dirty read__. That means transactions could be reading data that may not even exist eventually because the other transaction that was updating the data rolled-back the changes and didn’t commit.

    - 2. __READ COMMITED__:
        - In READ COMMITTED isolation level, the phenomenon of dirty read is avoided, because __any uncommitted changes are not visible to any other transaction until the change is committed__.

        - In this level, __each SELECT statement will have its own snapshot of the data__ which can be problem if we are using the same statement again in the transaction and before using the statement again, another transaction updates the row on which SELECT statement was dependent. This is known as non-repeatable read.

        - the second SELECT may return any data. A concurrent transaction may update the record, delete it, insert new records. The second select will always see the new data.

    - 3. __REPEATABLE READ__:
        - Repeatable read is a higher isolation level, that in addition to the guarantees of the read committed level, it also guarantees that __any data read cannot change, if the transaction reads the same data again__, it will find the previously read data in place, unchanged, and available to read.

        - the second SELECT is guaranteed to display at least the rows that were returned from the first SELECT unchanged. New rows may be added by a concurrent transaction in that one minute, but the existing rows cannot be deleted nor changed.

    - 4. __SERIALIZABLE__:
    
        - makes an even stronger guarantee: in addition to everything repeatable read guarantees, it also guarantees that no new data can be seen by a subsequent read.

        - the second select is guaranteed to see exactly the same rows as the first. No row can change, nor deleted, nor new rows could be inserted by a concurrent transaction.

## Locking

### Shared and Exclusive Locks
    
- A __shared (S) lock__ permits the transaction that holds the lock to read a row.
- An __exclusive (X) lock__ permits the transaction that holds the lock to update or delete a row.


### Intention Locks

- Intention Locks are table-level locks, used to indicate the kind of lock the transaction intends to acquire on rows in the table

- Intention Shared Lock (__IS__): a transaction intends to set a shared lock on individual rows in a table

- Intention Exclusive Lock (__IX__): a transaction intends to set an exclusive lock on individual rows on a table

- Example:
    - Different transaction can acquire different kinds of intention locks on the same table.
    - However, the first transaction to acquire an intention exclusive (IX) lock on a table prevents other transactinos from acquiring any S or X lock on the table.
    - Conversely, the first transaction to acquire an IS lock on a table prevents other transactions from acquiring any X locks on the table.
    - The two-phase process allows the lock requests to be resolved in order, without blocking locks and corresponding operations that are compatible.

- The rules of intention lock:

|  | X | IX | S | IS |
|-|-|-|-|-|
| X | Conflict | Conflict | Conflict | Conflict |
| IX | Conflict | Compatible | Conflict | Compatible |
| S | Conflict | Conflict | Compatible | Compatible |
| S | Conflict | Compatible | Compatible | Compatible |

## Deadlocks

- A deadlock is a situation where __different transactions are unable to proceed__ because __each holds a lock that the other needs__. Because both transactions are waiting for a resource to become available, neither ever release the locks it holds.

- When deadlock detection is enabled (the default) and a deadlock does occur, InnoDB detects the condition and rolls back one of the transactions (the victim). If deadlock detection is disabled using the innodb_deadlock_detect variable, InnoDB relies on the innodb_lock_wait_timeout setting to roll back transactions in case of a deadlock.

- Deadlocks are a classic problem in transactional databases, but they are not dangerous unless they are so frequent that you cannot run certain transactions at all. Normally, you must write your applications so that they are always prepared to re-issue a transaction if it gets rolled back because of a deadlock.

### Example

- First, client A creates a table containing one row, and then begins a transaction. Within the transaction, A obtains an S lock on the row by selecting it in share mode:

```
mysql> CREATE TABLE t (i INT) ENGINE = InnoDB;
Query OK, 0 rows affected (1.07 sec)

mysql> INSERT INTO t (i) VALUES(1);
Query OK, 1 row affected (0.09 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM t WHERE i = 1 FOR SHARE;
+------+
| i    |
+------+
|    1 |
+------+
```

- Next, client B begins a transaction and attempts to delete the row from the table. The delete operation requires an X lock. The lock cannot be granted because it is incompatible with the S lock that client A holds, so the request goes on the queue of lock requests for the row and client B blocks.:

```
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> DELETE FROM t WHERE i = 1;
```

- Finally, client A also attempts to delete the row from the table:
```
mysql> DELETE FROM t WHERE i = 1;
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
```

- Deadlock occurs here because client A needs an X lock to delete the row. However, that lock request cannot be granted because client B already has a request for an X lock and is waiting for client A to release its S lock. Nor can the S lock held by A be upgraded to an X lock because of the prior request by B for an X lock. As a result, InnoDB generates an error for one of the clients and releases its locks


### Minimize and handle deadlocks

- At any time, issue SHOW ENGINE INNODB STATUS to determine the cause of the most recent deadlock. That can help you to tune your application to avoid deadlocks.

- Always be prepared to re-issue a transaction if it fails due to deadlock. Deadlocks are not dangerous. Just try again.

- Keep transactions small and short in duration to make them less prone to collision.

- If you use locking reads (SELECT ... FOR UPDATE or SELECT ... FOR SHARE), try using a lower isolation level such as READ COMMITTED.

- When modifying multiple tables within a transaction, or different sets of rows in the same table, do those operations in a consistent order each time. Then transactions form well-defined queues and do not deadlock. For example, organize database operations into functions within your application, or call stored routines, rather than coding multiple similar sequences of INSERT, UPDATE, and DELETE statements in different places.

- Add well-chosen indexes to your tables so that your queries scan fewer index records and set fewer locks. Use EXPLAIN SELECT to determine which indexes the MySQL server regards as the most appropriate for your queries.

- Use less locking. If you can afford to permit a SELECT to return data from an old snapshot, do not add a FOR UPDATE or FOR SHARE clause to it. Using the READ COMMITTED isolation level is good here, because each consistent read within the same transaction reads from its own fresh snapshot.


## Buffer Pool

- The buffer pool is an area in main memory(__RAM__) where InnoDB __caches table and index data__ as it is accessed.

- For efficiency of cache management, the buffer pool is implemented as a linked list of pages; data that is rarely used is aged out of the cache using a variation of the LRU algorithm.

- The Buffer pool list maintains two sub lists
    - At the head, a sublist of new (“young”) pages that were accessed recently.

    - At the tail, a sublist of old pages that were accessed less recently.

- So, whenever new entry has to be added to the list, it is added in the middle of the list which is also the head of old sublist. Whenever a query is executed which uses data from old sublist, that page is moved to the head of young/new sublist.


## References

- https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html

- https://medium.com/iamdeepinder/mysql-acid-properties-and-buffer-pool-1a7645ea5ac

- https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html