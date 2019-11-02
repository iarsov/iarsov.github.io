---
layout: post
title:  "LATERAL Inline Views / GROUP BY And Scalar Subquery Caching"
date:   2015-06-27
categories: oracle
---

I'll be short into an introduction and will go straight to the point.
Oracle 12c introduced new SQL clause LATERAL which enables us to refer tables in the FROM clause which appear left from the lateral inline view.

***Update***:
Most probably the bug it's not related to LATERAL clause and it is related to the optimzer "grouping and scalar subquery caching" mechanism in 12c.
At first my assumptions were that it is related to LATERAL clause because I was able to reproduce the "bad" result set, but still LATERAL can lead the optimizer to buggy execution plan which most likely at this stage will produce wrong results if it's used with scalar subquery caching.

The examples bellow will definitely give you an example what happens and be careful when implementing scalar subquery caching.

Also this is only 12c behavior:

<img src="/assets/lateral_bug.jpg" />

As workaround you can disable 12c adaptive features (which I wouldn't recommend) or add ORDER BY clause which will eliminate this buggy behavior.

Basically you can do something like

{% highlight sql %}
select ... from table1, lateral (select ... from table2 where table2.column = table1.column) ...
{% endhighlight %}

Pretty nice for better syntax and less code.

However these day I've seen something strange with LATERAL views and so far I couldn't fully understand why it is happening.

Here is the tricky test case:

{% highlight sql %}
create table t1 as select rownum id from dual connect by rownum <= 2;
create table t2 as select rownum id from dual connect by rownum <= 2;
create table t3 as select rownum id, rownum * 10 id2 from dual connect by rownum <= 2;
{% endhighlight %}

Next I want to run the following sql:

{% highlight sql %}
select  /*+ LEADING(t1, t2, t3) MERGE(t2) */
        t3.id2 id,
        (select t3.id2 from dual) id1,
        (select (select t3.id2 from dual) from dual) id2
        
from    t1,
        lateral (select min(t2.id) from t2 where t2.id = t1.id) t2,
        lateral (select t3.id2 from t3 where t3.id = t1.id) t3;
{% endhighlight %}

I've added the hints in order reproduce the "bad" plan, but In the database where I've discovered this behavior the statement was executed without any hints and the optimizer still was using the "bad" plan. Also I needed couple of hours to successfully create a test case (I had to derive the bad behavior from a big sql 🙂 )

First try:

{% highlight sql %}
Result:

        ID        ID1        ID2
---------- ---------- ----------
        20         20         20
        10         10         10


Execution Plan
----------------------------------------------------------
Plan hash value: 2668642390

-----------------------------------------------------------------------------
| Id  | Operation            | Name | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |      |     2 |    24 |    16   (7)| 00:00:01 |
|   1 |  FAST DUAL           |      |     1 |       |     2   (0)| 00:00:01 |
|   2 |  FAST DUAL           |      |     1 |       |     2   (0)| 00:00:01 |
|   3 |  FAST DUAL           |      |     1 |       |     2   (0)| 00:00:01 |
|   4 |  HASH GROUP BY       |      |     2 |    24 |    16   (7)| 00:00:01 |
|*  5 |   HASH JOIN          |      |     2 |    24 |     9   (0)| 00:00:01 |
|*  6 |    HASH JOIN OUTER   |      |     2 |    12 |     6   (0)| 00:00:01 |
|   7 |     TABLE ACCESS FULL| T1   |     2 |     6 |     3   (0)| 00:00:01 |
|   8 |     TABLE ACCESS FULL| T2   |     2 |     6 |     3   (0)| 00:00:01 |
|   9 |    TABLE ACCESS FULL | T3   |     2 |    12 |     3   (0)| 00:00:01 |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   5 - access("T3"."ID"="T1"."ID")
   6 - access("T2"."ID"(+)="T1"."ID")
{% endhighlight %}

Sofar so good, I've got correct results.
But look what happens when I create an index on T3 join column T3.ID

{% highlight sql %}
SQL> create index t3_id_ix on t3(id);

Index created.

select  /*+ LEADING(t1, t2, t3) MERGE(t2) */
        lateral (select min(t2.id) from t2 where t2.id = t1.id) t2,
        t3.id2 id,
        (select t3.id2 from dual) id1,
        (select (select t3.id2 from dual) from dual) id2
from    t1,
        lateral (select min(t2.id) from t2 where t2.id = t1.id) t2,
        lateral (select t3.id2 from t3 where t3.id = t1.id) t3;


        ID        ID1        ID2
---------- ---------- ----------
        20         20         20
        10         10         20


Execution Plan
----------------------------------------------------------
Plan hash value: 3502536382

------------------------------------------------------------------------------------------
| Id  | Operation                     | Name     | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |          |     2 |    24 |    15   (7)| 00:00:01 |
|   1 |  FAST DUAL                    |          |     1 |       |     2   (0)| 00:00:01 |
|   2 |  FAST DUAL                    |          |     1 |       |     2   (0)| 00:00:01 |
|   3 |  FAST DUAL                    |          |     1 |       |     2   (0)| 00:00:01 |
|   4 |  HASH GROUP BY                |          |     2 |    24 |    15   (7)| 00:00:01 |
|   5 |   NESTED LOOPS                |          |     2 |    24 |     8   (0)| 00:00:01 |
|   6 |    NESTED LOOPS               |          |     2 |    24 |     8   (0)| 00:00:01 |
|*  7 |     HASH JOIN OUTER           |          |     2 |    12 |     6   (0)| 00:00:01 |
|   8 |      TABLE ACCESS FULL        | T1       |     2 |     6 |     3   (0)| 00:00:01 |
|   9 |      TABLE ACCESS FULL        | T2       |     2 |     6 |     3   (0)| 00:00:01 |
|* 10 |     INDEX RANGE SCAN          | T3_ID_IX |     1 |       |     0   (0)| 00:00:01 |
|  11 |    TABLE ACCESS BY INDEX ROWID| T3       |     1 |     6 |     1   (0)| 00:00:01 |
------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("T2"."ID"(+)="T1"."ID")
  10 - access("T3"."ID"="T1"."ID")
{% endhighlight %}

The plan clearly shows when this behavior happens, but I couldn't understand what causes this (maybe some kind of logical bug with lateral views and scalar subquery caching) ?

And this is happening only if we select column different from the join column in this case (T3.ID2).
It's kind of specific case but it's definitely wrong if I'm not missing something.
Also, if we remove the aggregate function the optimizer jumps from HASH OUTER JOIN to HASH JOIN producing correct results.

{% highlight sql %}
SQL> select  /*+ LEADING(t1, t2, t3) MERGE(t2) */
  2          t3.id2 id,
  3          (select t3.id2 from dual) id1,
  4          (select (select t3.id2 from dual) from dual) id2
  5  from       t1,
  6          lateral (select t2.id from t2 where t2.id = t1.id) t2,
  7          lateral (select t3.id2 from t3 where t3.id = t1.id) t3
  8  /

        ID        ID1        ID2
---------- ---------- ----------
        10         10         10
        20         20         20


Execution Plan
----------------------------------------------------------
Plan hash value: 1447545244

-----------------------------------------------------------------------------------------
| Id  | Operation                    | Name     | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |          |     2 |    24 |    14   (0)| 00:00:01 |
|   1 |  FAST DUAL                   |          |     1 |       |     2   (0)| 00:00:01 |
|   2 |  FAST DUAL                   |          |     1 |       |     2   (0)| 00:00:01 |
|   3 |  FAST DUAL                   |          |     1 |       |     2   (0)| 00:00:01 |
|   4 |  NESTED LOOPS                |          |     2 |    24 |     8   (0)| 00:00:01 |
|   5 |   NESTED LOOPS               |          |     2 |    24 |     8   (0)| 00:00:01 |
|*  6 |    HASH JOIN                 |          |     2 |    12 |     6   (0)| 00:00:01 |
|   7 |     TABLE ACCESS FULL        | T1       |     2 |     6 |     3   (0)| 00:00:01 |
|   8 |     TABLE ACCESS FULL        | T2       |     2 |     6 |     3   (0)| 00:00:01 |
|*  9 |    INDEX RANGE SCAN          | T3_ID_IX |     1 |       |     0   (0)| 00:00:01 |
|  10 |   TABLE ACCESS BY INDEX ROWID| T3       |     1 |     6 |     1   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - access("T2"."ID"="T1"."ID")
   9 - access("T3"."ID"="T1"."ID")
{% endhighlight %}
