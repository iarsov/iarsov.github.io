---
layout: post
title:  "TEMP Table Transformation Errors"
date:   2017-04-13
categories: oracle
---

Today, I’ve encountered an interesting bug for what I believe it’s related to TEMP TABLE TRANSFORMATION operation.

I was able to reproduce this on 11.2.0.3, 11.2.0.4 and 12.1.0.2, but not on 12.2.0.1 (looks like it’s been fixed).
I didn’t find any related MOS note/bug regarding this issue, if you’re aware of any please let me know.

Let’s observe the following statement.

{% highlight sql %}
create table t as select * from all_objects;

with subq as
(select /*+ materialize */ t1.* from t t1, t t2, t t3)
select subq.* from subq;
{% endhighlight %}

During statement execution if you press CTRL-C on the keyboard to stop (break) the execution it it will report “ORA-01013: user requested cancel of current operation”, which is fine. However, the bug is that this error is written in the alert.log and trace file is generated.

{% highlight bash %}
*** 2017-04-13 15:45:20.208
ORA-01013: user requested cancel of current operation
Dump of memory from 0x0000000152981E18 to 0x0000000152981E74
152981E10                   68746977 62757320          [with sub]
152981E20 73612071 6573280A 7463656C 2B2A2F20  [q as.(select /*+]
152981E30 74616D20 61697265 657A696C 202F2A20  [ materialize */ ]
152981E40 2A2E3174 6F726620 2074206D 202C3174  [t1.* from t t1, ]
152981E50 32742074 2074202C 0A293374 656C6573  [t t2, t t3).sele]
152981E60 73207463 2E716275 7266202A 73206D6F  [ct subq.* from s]
152981E70 00716275                             [ubq.]
{% endhighlight %}

This should happen for every error if one occurs during execution of the query.

The problem looks like to be at TEMP TABLE TRANSFORMATION operation. Probably it’s some code defect where an exception is missing and every error is traced. Note, the error has to occur during query execution, not fetching phase.

{% highlight sql %}
--------------------------------------------------------------------------------------------------------
| Id  | Operation          | Name              | Rows  | Bytes | Cost (%CPU)| Time                      |
--------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                           |  1016T|   142P|  9320G  (1)|999:59:59 |
|   1 |  TEMP TABLE TRANSFORMATION  |                           |       |       |            |          |
|   2 |   LOAD AS SELECT            | SYS_TEMP_0FD9D6615_4AAABB |       |       |            |          |
|   3 |    MERGE JOIN CARTESIAN     |                           |  1016T|   142P|  3402G  (1)|999:59:59 |
|   4 |     MERGE JOIN CARTESIAN    |                           |    10G|  1487G|    33M  (1)|112:47:16 |
|   5 |      TABLE ACCESS FULL      | T                         |   100K|    15M|    338  (1)| 00:00:05 |
|   6 |      BUFFER SORT            |                           |   100K|       |    33M  (1)|112:47:12 |
|   7 |       TABLE ACCESS FULL     | T                         |   100K|       |    336  (0)| 00:00:05 |
|   8 |     BUFFER SORT             |                           |   100K|       |  3402G  (1)|999:59:59 |
|   9 |      TABLE ACCESS FULL      | T                         |   100K|       |    336  (0)| 00:00:05 |
|  10 |   VIEW                      |                           |  1016T|   142P|  5917G  (1)|999:59:59 |
|  11 |    TABLE ACCESS FULL        | SYS_TEMP_0FD9D6615_4AAABB |  1016T|   142P|  5917G  (1)|999:59:59 |
--------------------------------------------------------------------------------------------------------
{% endhighlight %}

In my case, the monitoring system raised an alert for “ORA-01722: invalid number” in the alert.log file.
