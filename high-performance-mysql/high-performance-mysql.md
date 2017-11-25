# High Performance MySQL

## MySQL Architecture and History

### MySQL’s Logical Architecture

![architecture](DB-architecture.png)

第一层：

> connection handling, authentication, security, and so forth.

第二层：

> Much of MySQL’s brains are here, including the code for query parsing, analysis, optimization, caching, and all the built-in functions (e.g., dates, times, math, and encryption). Any functionality provided across storage engines lives at this level: stored procedures, triggers, and views, for example.

大部分功能都在这一层

第三层：

> They are responsible for storing and retrieving all data stored “in” MySQL.

> The storage engines don’t parse SQL 1 or communicate with each other; they simply respond to requests from the server.

### Concurrency Control

#### Read/Write Locks

> Systems that deal with concurrent read/write access typically implement a locking system that consists of two lock types. These locks are usually known as shared locks and exclusive locks, or read locks and write locks.

> Read locks on a resource are shared, or mutually nonblocking: many clients can read from a resource at the same time and not interfere with each other.Write locks, on the other hand, are exclusive—i.e., they block both read locks and other write locks—because the only safe policy is to have a single client writing to the resource at a given time and to prevent all reads when a client is writing.

Read locks 共享不互斥，而 Write locks 不仅仅自身互斥，而且与 Read locks 也是互斥的，但这样限制了并发能力。

#### Lock Granularity

> Minimizing the amount of data that you lock at any one time lets changes to a given resource occur simultaneously, as long as they don’t conflict with each other.
>
> The problem is locks consume resources.
> 
> A locking strategy is a compromise between lock overhead and data safety, and that compromise affects performance.
> 
> fixing the granularity at a certain level can give better performance for certain uses, yet make that engine less suited for other purposes.

通过控制锁的粒度来获得安全和资源消耗之间的平衡。

##### table locks

> **Table locks have variations for good performance in specific situations.** For example, READ LOCAL table locks allow some types of concurrent write operations. Write locks also have a higher priority than read locks, so a request for a write lock will advance to the front of the lock queue even if readers are already in the queue (write locks can advance past read locks in the queue, but read locks cannot advance past write locks).

锁表时，写锁相对于读锁拥有更高的优先级，因此哪怕读锁已经在队列中，新进来的写锁也会排在读锁前面。

> Although storage engines can manage their own locks, MySQL itself also uses a variety of locks that are effectively table-level for various purposes. For instance, **the server uses a table-level lock for statements such as ALTER TABLE**, regardless of the storage engine.

无论什么引擎，`ALTER TABLE` 都是锁表。

##### Row locks

> Row locks are implemented in the storage engine, not the server (refer back to the logical architecture diagram if you need to).

行锁是在数据库引擎中实现的，并不在 server 层。

### Transactions

> ACID stands for Atomicity, Consistency, Isolation, and Durability.

事务的四个特性：原子性、一致性、隔离性、持久性。

#### Isolation Levels

> Lower isolation levels typically allow higher concurrency and have lower overhead.

更低的隔离级别意味着更高的并发，更低的消耗。

- READ UNCOMMITTED

> This level is rarely used in practice, because its performance isn’t much better than the other levels, which have many advantages.

- READ COMMITED

> This level still allows what’s known as a nonrepeatable read. This means you can run the same statement twice and see different data.

不可重复读的出现在于 `UPDATE` 和 `DELETE`

- REPEATABLE READ

> REPEATABLE READ solves the problems that READ UNCOMMITTED allows. It guarantees
that any rows a transaction reads will “look the same” in subsequent reads within the same transaction, but in theory it still allows another tricky problem: **phantom reads**. Simply put, a phantom read can happen when you select some range of rows, another transaction **inserts a new row into the range**, and then you select the same range again; you will then see the new “phantom” row. InnoDB and XtraDB solve the phantom read problem with multiversion concurrency control, which we explain later in this chapter.
>
> REPEATABLE READ is MySQL’s default transaction isolation level.

可重复读虽然解决了读已提交的不可能重复读的问题，但是还会出现幻读，幻读在于 `INSERT`。

如果使用锁机制来实现这种隔离级别，在可重复读中，该 sql 第一次读取到数据后，就将这些数据加锁，其它事务无法修改这些数据，就可以实现可重复读了。但这种方法却无法锁住 `INSERT` 的数据，所以当事务 A 先前读取了数据，或者修改了全部数据，事务B还是可以 `INSERT` 数据提交，这时事务 A 就会发现莫名其妙多了一条之前没有的数据，这就是幻读，不能通过行锁来避免。需要 Serializable 隔离级别 ，读用读锁，写用写锁，读锁和写锁互斥，这么做可以有效的避免幻读、不可重复读、脏读等问题，但会极大的降低数据库的并发能力。

因此实际上来讲，是使用的 Multiversion Concurrency Control (MVCC) 来实现的，MySQL 避免了幻读，Oracle 避免了不可重复读。

- SERIALIZABLE

#### Deadlocks

> To combat this problem, database systems implement various forms of deadlock detection and timeouts. The more sophisticated systems, such as the InnoDB storage engine, will notice circular dependencies and return an error instantly.

### Multiversion Concurrency Control

> Most of MySQL’s transactional storage engines don’t use a simple row-locking mechanism. Instead, they use row-level locking in conjunction with a technique for increasing concurrency known as multiversion concurrency control (MVCC).

事务的核心是锁和并发

> You can think of MVCC as a twist on row-level locking; it avoids the need for locking at all in many cases and can have much lower overhead.

MVCC 可以看作是 row-level locking 变相的实现方式。

优化事务其实就是在锁和并发之间寻找平衡。

> MVCC works by keeping a snapshot of the data as it existed at some point in time. This means transactions can see a consistent view of the data, no matter how long they run. It also means different transactions can see different data in the same tables at the same time!

数据对应多个版本，同个事务内对应同一个版本，不同事务中对应可以是不同的版本。

> **InnoDB implements MVCC by storing with each row two additional, hidden values that record when the row was created and when it was expired (or deleted). Rather than storing the actual times at which these events occurred, the row stores the system version number at the time each event occurred. This is a number that increments each time a transaction begins. Each transaction keeps its own record of the current system version, as of the time it began. Each query has to check each row’s version numbers against the transaction’s version.** Let’s see how this applies to particular operations when the transaction isolation level is set to REPEATABLE READ:

>`SELECT` InnoDB must examine each row to ensure that it meets two criteria:

> - a. InnoDB must find a version of the row that is at least as old as the transaction (i.e., its version must be less than or equal to the transaction’s version). This ensures that either the row existed before the transaction began, or the transaction created or altered the row.

> - b. The row’s deletion version must be undefined or greater than the transaction’s version. This ensures that the row wasn’t deleted before the transaction began. Rows that pass both tests may be returned as the query’s result.

> `INSERT` InnoDB records the current system version number with the new row. 

> `DELETE` InnoDB records the current system version number as the row’s deletion ID.

> `UPDATE` InnoDB writes a new copy of the row, using the system version number for the new row’s version. It also writes the system version number as the old row’s deletion version.

上述的规则保证了可重复读的隔离级别。

> MVCC works only with the REPEATABLE READ and READ COMMITTED isolation levels.

![mvcc](mvcc.gif)

如上图中，当前查询对应的 SCN(System Change Number) 是 10023，只会查询 SCN 小于等于 10023 的记录。

## Profiling Server Performance

### Introduction to Performance Optimization

> Our definition is that performance is measured by the time required to complete a task. In other words, **performance is response time**.
> 
> A database server’s performance is measured by query response time, and the unit of measurement is time per query.
> 
> But this is a trap. Resources are there to be consumed. Sometimes making things faster requires that you increase resource consumption. We’ve upgraded many times from an old version of MySQL with an ancient version of InnoDB, and witnessed a dramatic increase in CPU utilization as a result. This is usually nothing to be concerned about. It usually means that the newer version of InnoDB is spending more time doing useful work and less time fighting with itself.
> 
> So if the goal is to reduce response time, we need to understand why the server requires a certain amount of time to respond to a query, and reduce or eliminate whatever unnecessary work it’s doing to achieve the result. In other words, we need to measure where the time goes. This leads to our second important principle of optimization: **you cannot reliably optimize what you cannot measure**. 
> 
> First, some tasks aren’t worth optimizing because they contribute such a small portion of response time overall. Because of Amdahl’s Law, a query that consumes only 5% of total response time can contribute only 5% to overall speedup, no matter how much faster you make it. Second, if it costs you a thousand dollars to optimize a task and the business ends up making no additional money as a result, you just deoptimized the business by a thousand dollars. Thus, optimization should halt when the cost of improvement outweighs the benefit.

## Optimizing Schema and Data Types

### Choosing Optimal Data Types

> Avoid NULL if possible.
> 
> **A lot of tables include nullable columns even when the application does not need to store NULL (the absence of a value), merely because it’s the default. It’s usually best to specify columns as NOT NULL unless you intend to store NULL in them.**
>
> It’s harder for MySQL to optimize queries that refer to nullable columns, because they make indexes, index statistics, and value comparisons more complicated. **A nullable column uses more storage space and requires special processing inside MySQL. When a nullable column is indexed, it requires an extra byte per entry and can even cause a fixed-size index (such as an index on a single integer column) to be converted to a variable-sized one in MyISAM.**
>
> **The performance improvement from changing NULL columns to NOT NULL is usually small, so don’t make it a priority to find and change them on an existing schema unless you know they are causing problems.** However, **if you’re planning to index columns, avoid making them nullable if possible.**
>
> There are exceptions, of course. For example, it’s worth mentioning that **InnoDB stores NULL with a single bit, so it can be pretty space-efficient for sparsely populated data.** This doesn’t apply to MyISAM, though.

#### Real Numbers

> The DECIMAL type is for storing exact fractional numbers. In MySQL 5.0 and newer, the DECIMAL type supports exact math. MySQL 4.1 and earlier used floating-point math to perform computations on DECIMAL values, which could give strange results because of loss of precision. In these versions of MySQL, **DECIMAL was only a “storage type**.”
>
> The server itself performs DECIMAL math in MySQL 5.0 and newer, **because CPUs don’t support the computations directly. Floating-point math is significantly faster, because the CPU performs the computations natively.**

对于低版本的 MySQL，Decimal 只是一个存储类型，实际运算的时候仍是浮点数运算，因此会丢失精度。CPU 并不支持 Decimal 的直接运算，所以高版本的 MySQL 会在自身的 Server 中进行 Decimal 的运算。

> DECIMAL numbers were converted to DOUBLEs for computational purposes
>
> Floating-point types typically use less space than DECIMAL to store the same range of values. A FLOAT column uses four bytes of storage. DOUBLE consumes eight bytes and has greater precision and a larger range of values than FLOAT. **As with integers, you’re choosing only the storage type; MySQL uses DOUBLE for its internal calculations on floating-point types.**
> 
> **Because of the additional space requirements and computational cost, you should use DECIMAL only when you need exact results for fractional numbers**—for example, when storing financial data. **But in some high-volume cases it actually makes sense to use a BIGINT instead, and store the data as some multiple of the smallest fraction of currency you need to handle.** Suppose you are required to store financial data to the tenthousandth of a cent. **You can multiply all dollar amounts by a million and store the result in a BIGINT, avoiding both the imprecision of floating-point storage and the cost of the precise DECIMAL math.**

#### String Types

##### VARCHAR and CHAR types

> VARCHAR stores variable-length character strings and is the most common string data type. It can require less storage space than fixed-length types, because it uses only as much space as it needs (i.e., less space is used to store shorter values).
> 
> VARCHAR uses 1 or 2 extra bytes to record the value’s length: 1 byte if the column’s maximum length is 255 bytes or less, and 2 bytes if it’s more.
> 
> VARCHAR helps performance because it saves space. However, because the rows are variable-length, they can grow when you update them, which can cause extra work.

VARCHAR 不设定长度虽然可以节省空间，但是在进行更新增加长度的时候会有额外的资源消耗，所以在创建表的时候都会设置一个长度。

> CHAR is fixed-length: MySQL always allocates enough space for the specified number of characters. When storing a CHAR value, MySQL removes any trailing spaces.
> 
> CHAR is useful if you want to store very short strings, or if all the values are nearly the same length. For example, CHAR is a good choice for MD5 values for user passwords, which are always the same length. CHAR is also better than VARCHAR for data that’s changed frequently, because a fixed-length row is not prone to fragmentation. For very short columns, CHAR is also more efficient than VARCHAR; a CHAR(1) designed to hold only Y and N values will use only one byte in a single-byte character set, 1 but a VARCHAR(1) would use two bytes because of the length byte.

#### Date and Time Types

> The finest granularity of time MySQL can store is one second.

##### DATETIME

> This uses eight bytes of storage space.

##### TIMESTAMP

> TIMESTAMP uses only four bytes of storage

DATETIME 会随着时区的改变而改变，而 TIMESTAMP 不会，所以一般都是使用 DATETIME。

#### Choosing Identifiers

> As we demonstrated earlier in this chapter, it’s a good idea to use the same data types in related tables, because you’re likely to use them for joins.

> **MySQL does index NULLs, unlike Oracle, which doesn’t include non-values in indexes**.

#### Normalization and Denormalization

##### Pros and Cons of a Normalized Schema

> People who ask for help with performance issues are frequently advised to normalize their schemas, especially if the workload is write-heavy. This is often good advice. It works well for the following reasons:
>> - Normalized updates are usually faster than denormalized updates.
>> - When the data is well normalized, there’s little or no duplicated data, so there’s less data to change.
>> - Normalized tables are usually smaller, so they fit better in memory and perform better.
>> - The lack of redundant data means there’s less need for DISTINCT or GROUP BY queries when retrieving lists of values. Consider the preceding example: it’s impossible to get a distinct list of departments from the denormalized schema without DIS TINCT or GROUP BY, but if DEPARTMENT is a separate table, it’s a trivial query.
>
> **The drawbacks of a normalized schema usually have to do with retrieval. Any nontrivial query on a well-normalized schema will probably require at least one join, and perhaps several.** This is not only expensive, but it can make some indexing strategies impossible. For example, normalizing may place columns in different tables that would benefit from belonging to the same index.

##### A Mixture of Normalized and Denormalized

关键就是在这两者间寻找平衡点

#### Cache and Summary Tables

> **When you rebuild summary and cache tables, you’ll often need their data to remain available during the operation. You can achieve this by using a “shadow table,” which is a table you build “behind” the real table.** When you’re done building it, you can swap the tables with an atomic rename. For example, if you need to rebuild my_summary, you can create my_summary_new, fill it with data, and swap it with the real table:

>
```
mysql> DROP TABLE IF EXISTS my_summary_new, my_summary_old; 
mysql> CREATE TABLE my_summary_new LIKE my_summary; 
-- populate my_summary_new as desired 
mysql> RENAME TABLE my_summary TO my_summary_old, my_summary_new TO my_summary;
```
> If you rename the original my_summary table my_summary_old before assigning the name my_summary to the newly rebuilt table, as we’ve done here, you can keep the old version until you’re ready to overwrite it at the next rebuild. It’s handy to have it for a quick rollback if the new table has a problem.

这里之所以要将原表更名为 old ，新表替换原表工作，是因为可以检测新表是否有问题，若有问题则可以迅速回滚。

##### Materialized Views

- Materialized views are disk based and are updated periodically based upon the query definition.
- Views are virtual only and run the query definition each time they are accessed.

##### Counter Tables

> An application that keeps counts in a table can run into concurrency problems when updating the counters. Such tables are very common in web applications. You can use them to cache the number of friends a user has, the number of downloads of a file, and so on. It’s often a good idea to build a separate table for the counters, to keep it small and fast. Using a separate table can help you avoid query cache invalidations and lets you use some of the more advanced techniques we show in this section.
> 
> To keep things as simple as possible, suppose you have a counter table with a single row that just counts hits on your website:
>
```
mysql> CREATE TABLE hit_counter (
-> cnt int unsigned not null 
-> ) ENGINE=InnoDB;
``` 
>
> Each hit on the website updates the counter:
> 
```
mysql> UPDATE hit_counter SET cnt = cnt + 1;
```
> 
> The problem is that this single row is effectively a global “mutex” for any transaction that updates the counter. It will serialize those transactions. You can get higher concurrency by keeping more than one row and updating a random row. This requires the following change to the table:
> 
```
mysql> CREATE TABLE hit_counter (
-> slot tinyint unsigned not null primary key, 
-> cnt int unsigned not null 
-> ) ENGINE=InnoDB;
```
>
> Prepopulate the table by adding 100 rows to it. Now the query can just choose a random slot and update it:
>
```
mysql> UPDATE hit_counter SET cnt = cnt + 1 WHERE slot = RAND() * 100;
```
>
> To retrieve statistics, just use aggregate queries:
> 
```
mysql> SELECT SUM(cnt) FROM hit_counter;
```
>

通过预置的记录，随机增加 cnt，避免在同一条记录上更新时的互斥。（以前都没有过这样的思路）

> A common requirement is to start new counters every so often (for example, once a day). If you need to do this, you can change the schema slightly:
> 
```
mysql> CREATE TABLE daily_hit_counter (
-> day date not null, 
-> slot tinyint unsigned not null, 
-> cnt int unsigned not null, 
-> primary key(day, slot) 
-> ) ENGINE=InnoDB;
```
>
> You don’t want to pregenerate rows for this scenario. Instead, you can use ON DUPLICATE KEY UPDATE:
> 
```
mysql> INSERT INTO daily_hit_counter(day, slot, cnt) 
-> VALUES(CURRENT_DATE, RAND() * 100, 1) 
-> ON DUPLICATE KEY UPDATE cnt = cnt + 1;
```
>
> If you want to reduce the number of rows to keep the table smaller, you can write a periodic job that merges all the results into slot 0 and deletes every other slot:
> 
```
mysql> UPDATE daily_hit_counter as c 
    -> INNER JOIN ( 
    -> SELECT day, SUM(cnt) AS cnt, MIN(slot) AS mslot 
    -> FROM daily_hit_counter 
    -> GROUP BY day 
    -> ) AS x USING(day) 
    -> SET c.cnt = IF(c.slot = x.mslot, x.cnt, 0), 
    -> c.slot = IF(c.slot = x.mslot, 0, c.slot); 
mysql> DELETE FROM daily_hit_counter WHERE slot <> 0 AND cnt = 0;
```

将所有的计数记录都合并到一条记录，将其他所有记录删除来减少表内的记录数。

## Indexing for High Performance

### Indexing Basics

> An index contains values from one or more columns in a table. If you index more than one column, the column order is very important, because MySQL can only search efficiently on a leftmost prefix of the index. Creating an index on two columns is not the same as creating two separate single-column indexes, as you’ll see.

举例说明下：

```
SELECT * FROM table WHERE first_name="john" AND last_name="doe"
SELECT * FROM table WHERE first_name="john"
SELECT * FROM table WHERE last_name="doe"
```

如果索引是 (`first_time`, `last_name`) 则 1 和 2 语句会用到索引，而 3 不会。如果索引是 (`last_name`, `first_time`) 则 1 和 3 语句可以用到索引，而 2 不会。

因为 MySQL 的索引是 B+Tree 数据结构，表中的数据本身就是按照 B+Tree 的数据结构组成。最左匹配原则 (leftmost prefix of the index) 的原因也是来自于 B+Tree 的数据结构。

索引建立的时候 root page 是地址是不变的，其他的 page 都是根据 root page 衍生，因此才有了最左匹配原则。具体可以参见 [B+Tree index structures in InnoDB](B+Tree-index-structures-in-InnoDB.md)


