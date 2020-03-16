## InnoDB fulltext indexing problems

InnoDB Fulltext indexing has two separate set of problems, though.

First, the design has several oddities. This is due to the fact that, rather than being independently and optimally designed, then integrated, it has recycled the existing functionalities in a rather forceful way.

Functionally speaking, an oddity is that updates are not reflected in searchs until a transaction is completed. This is due to how fulltext indexes works in general (although I'm not aware of this problem in PostgreSQL), and it may come across as a surprise. See [the manpage](https://dev.mysql.com/doc/refman/8.0/en/innodb-fulltext-index.html).

Most importantly though, InnoDB fulltext indexes suffer from very serious administration problems.

First, the requirement to use global settings for maintenance is a significant complexity for AWS (RDS) systems.

The most serious problem though, is that maintenance must be performed periodically (on a system that performs updates on indexed rows), and it [requires `OPTIMIZE TABLE` invocations](https://dev.mysql.com/doc/refman/8.0/en/fulltext-fine-tuning.html#fulltext-optimize), which can cause concurrent access to fail with an error, even if it doesn't block.

In the near future, I'll publish a detailed article on this subject.

