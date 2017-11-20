# High Performance MySQL

## MySQL’s Logical Architecture

![architecture](DB-architecture.png)

第一层：

> connection handling, authentication, security, and so forth.

第二层：

> Much of MySQL’s brains are here, including the code for query parsing, analysis, optimization, caching, and all the built-in functions (e.g., dates, times, math, and encryption). Any functionality provided across storage engines lives at this level: stored procedures, triggers, and views, for example.

大部分功能都在这一层

第三层：

> They are responsible for storing and retrieving all data stored “in” MySQL.

> The storage engines don’t parse SQL 1 or communicate with each other; they simply respond to requests from the server.


