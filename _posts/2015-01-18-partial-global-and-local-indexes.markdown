---
layout: post
title:  "Partial GLOBAL And LOCAL Indexes"
date:   2015-01-18
categories: oracle
---

Partial indexes came with 12c, representing nice option which enables partial indexing of the data. This allows for DBAs to index only active data and save on disk space (by excluding non-active data from indexing) and also gain on performance. For GLOBAL indexes only part of the data is indexed and for LOCAL indexes, non-indexed (excluded) partitions are set to UNUSABLE (*but still defined as metadata*).
Starting with 12c we can manually specify which partitions data we would like to be indexed, so that the optimizer has more information for the data for which it should generate an execution plan. Partial indexes work only for partitioned tables, we can’t use them for non-partitioned objects.

***Pre-12c***:

This previously existed from 11gR2 where we could set some of the local index partitions UNUSABLE and the CBO would perform [Table Expansion Transformation](https://blogs.oracle.com/optimizer/entry/optimizer_transformations_table_expansion){:target="_blank"} in order to use indexes for only USABLE partitions. The key point with table expansion is that it splits the query in multiple query blocks where one part is using indexes to access the data and the other part is using full table access.
I.e. For local partitioned indexes we could get the same effect by setting some of the index partitions to UNUSABLE. But, with global indexes there was not a way to exclude portion of the data from indexing.

{% highlight sql %}
SQL> select x from t where x < 20;

Execution Plan
----------------------------------------------------------
Plan hash value: 3529155236

-------------------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name      | Rows  | Bytes | Cost (%CPU)   | Time      | Pstart| Pstop |
-------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |           |     1 |    13 |   274   (0)   | 00:00:01  |       |       |
|   1 |  VIEW                       | VW_TE_2   |     2 |    26 |   274   (0)   | 00:00:01  |       |       |
|   2 |   UNION-ALL                 |           |       |       |               |           |       |       |
|   3 |    PARTITION RANGE SINGLE   |           |     1 |    25 |     0   (0)   | 00:00:01  |     1 |     1 |
|*  4 |     INDEX RANGE SCAN        | T_IX      |     1 |    25 |     0   (0)   | 00:00:01  |     1 |     1 |
|   5 |    PARTITION RANGE SINGLE   |           |     1 |    25 |   274   (0)   | 00:00:01  |     2 |     2 |
|   6 |     TABLE ACCESS FULL       | T         |     1 |    25 |   274   (0)   | 00:00:01  |     2 |     2 |
-------------------------------------------------------------------------------------------------------------
{% endhighlight %}

We can see that the optimizer performed table expansion transformation, TABLE ACCESS FULL for the second partition p2 and INDEX RANGE SCAN on the first partition which has corresponding LOCAL index.
Lets disable table expansion which is controlled by _\_optimizer_table_expansion_ and see what explain plan we would get.

{% highlight sql %}
SQL> select x from t where x < 20;

Execution Plan
----------------------------------------------------------
Plan hash value: 1571388083

-----------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name  | Rows  | Bytes | Cost (%CPU)| Time     | Pstart| Pstop |
-----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |       |     2 |     6 |   547   (0)| 00:00:01 |       |       |
|   1 |  PARTITION RANGE ITERATOR   |       |     2 |     6 |   547   (0)| 00:00:01 |     1 |     2 |
|   2 |   TABLE ACCESS FULL         | T     |     2 |     6 |   547   (0)| 00:00:01 |     1 |     2 |
-----------------------------------------------------------------------------------------------------
{% endhighlight %}

Expected, CBO couldn’t use the index because not all index partitions where usable and there is no way to access rows for partition p2 through the index except with TABLE ACCESS FULL.
Just to prove that our index partition for partition 2 is unusable:

{% highlight sql %}
SQL> col index_name format a10
SQL> col partition_name format a10
SQL> col status format a10
SQL>
SQL> select index_name,
  2     partition_name,
  3     status
  4 from   user_ind_partitions
  5 where  index_name = 'T_IX'
  6 /

INDEX_NAME PARTITION_ STATUS
---------- ---------- ----------
T_IX       P1         USABLE
T_IX       P3         USABLE
T_IX       P_MAX      USABLE
T_IX     P2         UNUSABLE
{% endhighlight %}

Will this always work ? We need to be careful because if we perform maintenance on table sub/partitions and use UPDATE INDEXES those index partitions which are unusable would be rebuild-ed and set back to USABLE (oracle doesn’t have information that we want those partitions to be UNUSABLE forever). This would result into more space consumption and the optimizer would again consider those partitions since their status will be USABLE.

{% highlight sql %}
SQL> alter table t move partition p2 update indexes;

Table altered.

SQL> select index_name, status, partition_name from user_ind_partitions;

INDEX_NAME STATUS   PARTITION_NAME
---------- -------- --------------------
T_IX       USABLE   P1
T_IX       USABLE   P2
T_IX       USABLE   P3
T_IX       USABLE   P_MAX
{% endhighlight %}

***Partial indexes, 12c***:

As new feature in Oracle 12c, now we have the ability to specify (into the metadata) which partitions we want to be indexed and which to be skipped. It’s just another information for the CBO to be able to generate better plan. By default it’s set to ON which means that when we create an index all table sub/partitions are considered for indexing.
To enable partial indexing at table level we need to specify INDEXING ON|OFF within the DDL or with alter table modify_default_attributes clause for existing tables. Also, we can set INDEXING clause at sub/partition level, where we have option to exclude specific sub/partitions from indexing with INDEXING set to OFF.

{% highlight sql %}
SQL> create table t(x number, col1 date)
  2  indexing on
  3  partition by range(x)
  4  (partition p1 values less than (10),
  5   partition p2 values less than (20) indexing off,
  6   partition p3 values less than (30) indexing on,
  7   partition p_max values less than (maxvalue)
  8  )
  9  /

Table created.
{% endhighlight %}

I’ve created table T as range partitioned table, enabled partial indexing at table level (I could exclude INDEXING ON, because ON is default value) and excluded only partition p2 from indexing. This means that when I’ll create partial global or local index for table T, the data portion from partition p2 should not be included in the global index or if it’s local index, the index partition will be set as UNUSABLE.
We can see which partitions are enabled for indexing from DBA|ALL|USER_TAB_PARTITIONS dictionary view.

{% highlight sql %}
SQL> select partition_name, indexing from user_tab_partitions where table_name = 'T';

PARTITION_ INDEXING
---------- ----------
P1          ON
P2          OFF
P3          ON
P_MAX       ON
{% endhighlight %}

I’ve insert some data to populate the table partitions and created partial index on column x by adding INDEXING PARTIAL clause at index creation DDL.

{% highlight sql %}
SQL> insert into t values (5, sysdate);
1 row created.
SQL> insert into t values (15, sysdate);
1 row created.
SQL> insert into t values (25, sysdate);
1 row created.
SQL> commit;
Commit complete.
SQL>
SQL> exec dbms_stats.gather_table_stats(user,'T');

PL/SQL procedure successfully completed.

SQL> create index t_ix on t(x) local indexing partial;

Index created.
{% endhighlight %}

We can see the index definition from DBA|ALL|USER_INDEXES. It has new column INDEXING which gives information whether the index is PARTIAL or FULL.

{% highlight sql %}
SQL> select index_name,indexing from user_indexes where index_name = 'T_IX';

INDEX_NAME       INDEXING
-------------------- ----------
T_IX             PARTIAL
{% endhighlight %}

Lets see what kind of execution plan the optimizer will generate. Remember, second partition from table T has INDEXING = OFF setting which means that its data is not indexed with T_IX index.

{% highlight sql %}
SQL> select col1  from t where x < 30;

Execution Plan
----------------------------------------------------------
Plan hash value: 674678595

-------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                     | Name    | Rows  | Bytes | Cost (%CPU)| Time     | Pstart| Pstop |
-------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                              |         |     3 |    33 |   278   (0)| 00:00:01 |       |       |
|   1 |  VIEW                                         | VW_TE_2 |     3 |    27 |   278   (0)| 00:00:01 |       |       |
|   2 |   UNION-ALL                                   |         |       |       |            |          |       |       |
|   3 |    CONCATENATION                              |         |       |       |            |          |       |       |
|   4 |     PARTITION RANGE SINGLE                    |         |     1 |    11 |     2   (0)| 00:00:01 |KEY(AP)|KEY(AP)|
|   5 |      TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| T       |     1 |    11 |     2   (0)| 00:00:01 |     3 |     3 |
|*  6 |       INDEX RANGE SCAN                        | T_IX    |     1 |       |     1   (0)| 00:00:01 |     3 |     3 |
|   7 |     PARTITION RANGE SINGLE                    |         |     1 |    11 |     2   (0)| 00:00:01 |KEY(AP)|KEY(AP)|
|*  8 |      TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| T       |     1 |    11 |     2   (0)| 00:00:01 |     1 |     1 |
|*  9 |       INDEX RANGE SCAN                        | T_IX    |     1 |       |     1   (0)| 00:00:01 |     1 |     1 |
|  10 |    PARTITION RANGE SINGLE                     |         |     1 |    11 |   274   (0)| 00:00:01 |     2 |     2 |
|  11 |     TABLE ACCESS FULL                         | T       |     1 |    11 |   274   (0)| 00:00:01 |     2 |     2 |
-------------------------------------------------------------------------------------------------------------------------
{% endhighlight %}

Again, the optimizer performed table expansion transformation by separating partitions 1 and 3 into separate query blocks from partition 2.
Here, important to note is that if we use UPDATE INDEXES the partition still stays UNUSABLE since we’ve set it manually.

{% highlight sql %}
SQL> select index_name, status, partition_name from user_ind_partitions;

INDEX_NAME STATUS   PARTITION_NAME
---------- -------- --------------------
T_IX       USABLE   P1
T_IX       UNUSABLE P2
T_IX       USABLE   P3
T_IX       USABLE   P_MAX

SQL> alter table t move partition p2 update indexes;

Table altered.

SQL> select index_name, status, partition_name from user_ind_partitions;

INDEX_NAME STATUS   PARTITION_NAME
---------- -------- --------------------
T_IX       USABLE   P1
T_IX       UNUSABLE P2
T_IX       USABLE   P3
T_IX       USABLE   P_MAX
{% endhighlight %}

During testing and playing with partial indexes I discovered something strange as effect of TRUNCATE command (which according to me it’s a bug). If we execute TRUNCATE for the table those index partitions which are UNUSABLE will be rebuild-ed to USABLE. Also, if we truncate specific UNUSABLE partition it will be rebuild-ed to USABLE.

{% highlight sql %}
SQL> truncate table t;

Table truncated.

SQL>
SQL> insert into t values (5,sysdate);
1 row created.

SQL> insert into t values (15,sysdate);
1 row created.

SQL> insert into t values (25,sysdate);
1 row created.

SQL>
SQL> commit;

Commit complete.
{% endhighlight %}

If we check the index partitions status in user_ind_partitions we would see that partition p2 is USABLE, meaning that partition data is indexed (ignoring the setting for table partition INDEXING=OFF).

{% highlight sql %}
SQL> select index_name, status, partition_name from user_ind_partitions;

INDEX_NAME STATUS   PARTITION_
---------- -------- ----------
T_IX       USABLE   P1
T_IX       USABLE   P2
T_IX       USABLE   P3
T_IX       USABLE   P_MAX

SQL> select partition_name, indexing from user_tab_partitions where table_name = 'T';

PARTITION_ INDEXING
---------- --------------------
P1         ON
P2         OFF
P3         ON
P_MAX      ON
{% endhighlight %}

Same will happen if we truncate only the table partition.

{% highlight sql %}
SQL> create index t_ix on t(x) local indexing partial;

Index created.

SQL>
SQL> select index_name, status, partition_name from user_ind_partitions;

INDEX_NAME STATUS   PARTITION_
---------- -------- ----------
T_IX       USABLE   P1
T_IX       UNUSABLE P2
T_IX       USABLE   P3
T_IX       USABLE   P_MAX

SQL> alter table t truncate partition p2;

Table truncated.

SQL> select index_name, status, partition_name from user_ind_partitions;

INDEX_NAME STATUS   PARTITION_
---------- -------- ----------
T_IX       USABLE   P1
T_IX       USABLE   P2
T_IX       USABLE   P3
T_IX       USABLE   P_MAX
{% endhighlight %}

Now, because partition p2 is set to USABLE nothing stops the optimizer to take into consideration partition p2 for index access path.

{% highlight sql %}
SQL> select col1  from t where x < 30;

Execution Plan
----------------------------------------------------------
Plan hash value: 2214522014

-------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                  | Name | Rows  | Bytes | Cost (%CPU)| Time     | Pstart| Pstop |
-------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                           |      |     3 |    33 |     4   (0)| 00:00:01 |       |       |
|   1 |  PARTITION RANGE ITERATOR                  |      |     3 |    33 |     4   (0)| 00:00:01 |     1 |     3 |
|   2 |   TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| T    |     3 |    33 |     4   (0)| 00:00:01 |     1 |     3 |
|*  3 |    INDEX RANGE SCAN                        | T_IX |     2 |       |     2   (0)| 00:00:01 |     1 |     3 |
-------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   3 - access("X"<30)
{% endhighlight %}

As workaround/solution we can manually set the index partition to UNUSABLE (which will drop the partition segment) or, not so good solution, drop the index and re-create it again.