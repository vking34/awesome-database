# Mysql


## Table of Contents

- [Transaction](#transaction)

- [ACID](#acid)

- [Isolation Levels](#isolation-levels) 

- Race conditions

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

- This property states that a transaction must be treated as an atomic unit, that is, either all of its operations are executed or none. There must be no state in a database where a transaction is left partially completed

- The database remains in a consistent state at all times — after each commit or rollback, and while transactions are in progress.

- Rollback:
    - __UNDO LOG__ contains undo log record which contains information about how to undo the latest change made by a transaction. If another transaction needs to see the original data as part of a consistent read operation, the unmodified data is retrieved from undo log records. Each transaction is allotted undo log slot based on the operations it has to perform.

    - __REDO LOG__ — This is another term used in recovery of data. By defination “The redo log is a disk-based data structure used during crash recovery to correct data written by incomplete transactions.” The changes which could not make it upto the data files because of system crash or any other reason are replayed automatically during initialization/restart of the server after the crash, and before the connections are accepted. To write data from REDO LOG to Disk InnoDB uses group commit functionality to group multiple flush requests from REDO LOG to avoid one flush for each commit taking place. With group commit, InnoDB makes a single write to the log file to perform the commit action for multiple user transactions that commit at about the same time, significantly improving throughput.

### Consistency

The consistency aspect of the ACID model mainly involves internal InnoDB processing to protect data from crashes. Related MySQL features include:

- __Doublewrite Buffer__:

    - __Page__: a unit that specifies how much data can be transferred between disk and memory. A page can contain one or more rows, depending on how much data is in each row. If a row does not fit entirely into a single page, InnoDB sets up additional pointer-style data structures so that the information about the row can be stored in one page.

    - __Flush__: Whenever we write something to the database it isn’t written instantly to the disk, for performance reasons MySQL stores that either in the memory(Buffer pool — explained below) or in temporary disk storage. InnoDB storage structures that are periodically flushed include the redo log, the undo log, and the buffer pool. Flushing can happen because a memory area becomes full and the system needs to free some space, because a commit operation means the changes from a transaction can be finalized, or because a slow shutdown operation means that all outstanding work should be finalized. When it is not critical to flush all the buffered data at once, InnoDB can use a technique called fuzzy checkpointing to flush small batches of pages to spread out the I/O overhead.

    - The doublewrite buffer is __a storage area where InnoDB writes pages flushed from the buffer pool before writing the pages to their proper positions in the InnoDB data files(tables on the disk)__. If the system crashes in the middle of a page write, InnoDB can find a good copy of the page from the doublewrite buffer during crash recovery.

- __Crash recovery__:

    - On a server crash, InnoDB automatically recovers after the server is restarted. InnoDB automatically checks the logs and performs a roll-forward of the database to the present. InnoDB automatically rolls back uncommitted transactions that were present at the time of the crash.

    - All the data from the REDO LOG is restored before any connections are setup.


### Isolation Levels

- The isolation mainly involves transactions. In a database system where more than one transactions are being executed simultaneously and in parallel, the property of isolation states that all the transactions will be carried out and executed as if it is the only transaction in the system. No transaction will affect the existence of any other transaction. MySQL achieves with help of locking strategies.

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

### Durability

- The durability aspect of the ACID model involves MySQL software features interacting with your particular hardware configuration. The database should be durable enough to __hold all its latest updates even if the system fails or restarts__. If a transaction updates a chunk of data in a database and commits, then the database will hold the modified data. If a transaction commits but the system fails before the data could be written on to the disk then the data should be written back to the disk on restart. In short the data written should be persistent.


## References

- https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html

- https://medium.com/iamdeepinder/mysql-acid-properties-and-buffer-pool-1a7645ea5ac