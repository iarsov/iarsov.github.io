---
layout: post
title:  "Empty Blocks Affecting ROWNUM Performance"
date:   2017-03-26
categories: oracle
---

ROWNUM is a pseudocolumn which is used to uniquely assign numbers (1,2,3,4,5…) to result rows as they are retrieved. In this blog post I will demonstrate a situation where using a ROWNUM filter on a table which has lot of empty blocks can negatively impact on the execution time.
The root cause is not the ROWNUM itself, however lot of people are surprised when using a filter simple as ROWNUM <= 1 takes minutes to execute. Using ROWNUM <= N as a predicate triggers the optimizer to use COUNT STOPKEY operation, meaning it will stop the table scan once N rows are identified.

In the case of a full table scan all blocks below HWM are read. Oracle has no way to determine which blocks contain rows and which don’t before hand, all blocks will be read until N (ROWNUM) rows are identified. If there are empty blocks, it will result in many reads. That might be the case if you do lot of direct path inserts where the rows are placed after HWM in new unformatted blocks.
Usually, Oracle will reuse empty blocks for regular inserts. However, if the data is loaded by direct path then Oracle won’t reuse any of the empty blocks.

Test case.

{% highlight sql %}
create table t1 nologging tablespace arsov_tbs_ssd
as
select rownum id, mod(rownum,200) id2, rpad('x',1500,'x') str from dual connect by rownum <= 1e6
/
insert /*+ append */ into t1 select * from t1;
commit;
insert /*+ append */ into t1 select * from t1;
commit;

insert /*+ append */ into t1
select -1 id, mod(rownum,200) id2, rpad('x',1500,'x') str from dual
/
commit;
{% endhighlight %}

Having T1 fully packed without any deletes ROWNUM <= 1 completes in less than a second.
Once it finds one row which satisfies ROWNUM <= 1 it stops with execution, that’s the purpose of COUNT STOPKEY operation.

{% highlight sql %}
16:15:51 IARSOV@orcl11g > select count(*) from t1 where rownum <= 1;

COUNT(*)
----------
1
{% endhighlight %}

Having lot of empty blocks from the beginning of the segment will result in longer execution time due to the fact that Oracle will spend some time until it finds a block with a row. We can see that after we delete all rows except the latest which was inserted with APPEND hint.

{% highlight sql %}
16:15:55 IARSOV@orcl11g > delete t1 where id > 0;

4000000 rows deleted.

16:31:19 IARSOV@orcl11g > select count(*) from t1 where rownum <= 1;

COUNT(*)
----------
1
{% endhighlight %}

Execution Plan
----------------------------------------------------------
Plan hash value: 912793033

--------------------------------------------------------------------
| Id  | Operation       | Name | Rows  | Cost (%CPU)| Time     |
--------------------------------------------------------------------
|   0 | SELECT STATEMENT    |      |     1 |   271K  (1)| 00:54:18 |
|   1 |  SORT AGGREGATE     |      |     1 |        |      |
|*  2 |   COUNT STOPKEY     |      |       |        |      |
|   3 |    TABLE ACCESS FULL| T1   |     1 |   271K  (1)| 00:54:18 |
--------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

2 - filter(ROWNUM<=1)

Note
-----
- dynamic sampling used for this statement (level=2)

Statistics
----------------------------------------------------------
0  recursive calls
0  db block gets
1000010  consistent gets
1000002  physical reads
0  redo size
526  bytes sent via SQL*Net to client
524  bytes received via SQL*Net from client
2  SQL*Net roundtrips to/from client
0  sorts (memory)
0  sorts (disk)
1  rows processed
{% endhighlight %}

It scanned all segment blocks, 1000002 physical reads is 7812mb (8k block size) and T1 segment is 7872mb in size (I previously flushed the buffer cache).
Further we can confirm what’s the data distribution between the blocks with DBMS_SPACE.SPACE_USAGE procedure.

{% highlight sql %}
16:32:13 SYS@orcl11g AS SYSDBA> @dbms_space_usage arsov t1 table
...
unformatted_blocks: 0
unformatted_bytes: 0
fs1_blocks  (25%): 0
fs1_bytes  (25%): 0
fs2_blocks (50%): 0
fs2_bytes (50%): 0
fs3_blocks (75%): 0
fs3_bytes (75%): 0
fs4_blocks (100%): 1000000
fs4_bytes (100%): 8192000000
full_blocks: 1
full_bytes: 8192
v_total_blocks: 1007616
v_total_bytes: 8254390272

PL/SQL procedure successfully completed.
{% endhighlight %}

We have 7812mb free space (fs4_blocks – number of blocks fully empty), the same amount which was processed until a row was encountered.

This is expected behavior if lot of DELETE’s are performed followed by INSERT’s with APPEND hint, in which case empty blocks are not re-used.
There are multiple approaches to release unused space, we can use online table redefinition, alter table move, data pump export/import, segment shrink, etc.
