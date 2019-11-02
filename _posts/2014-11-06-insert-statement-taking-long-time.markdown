---
layout: post
title:  "INSERT Statement Taking Long Time – Sequence Problem"
date:   2014-11-06
categories: oracle
---

Today I encountered a problem with an insert statement which was executing slowly (the client started to complain).
It was about sql statement which had to insert something about 592 000 rows into table that already had ~2 million rows.
The process was taking about 90 seconds for simple insert into _… select … from … where …_.

I’ve used tkprof to trace the session in order to see which statement was causing the problem.
From tkprof output (sort by elapsed time) I got this:

{% highlight sql %}
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.02       0.02          0          0          0           0
Execute      1     74.95      77.38      14353     201088    1910040      642400
Fetch        0      0.00       0.00          0          0          0           0
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        2     74.98      77.40      14353     201088    1910040      642400
{% endhighlight %}

As you can see 77.40 seconds were used for execution of that insert statement.
When I reproduced this in new session I run query for V$SESS_TIME_MODEL to see the time model statistics:

{% highlight sql %}
SQL> @tune

... insert statement ...

       SID STAT_NAME                                               VALUE
---------- -------------------------------------------------- ----------
       357 DB time                                              56.27379
       357 sql execute elapsed time                            56.258497
       357 DB CPU                                                 54.913
       357 sequence load elapsed time                          50.660571
       357 repeated bind elapsed time                            .748207
       357 connection management call elapsed time                .00347
       357 parse time elapsed                                    .001308
       357 PL/SQL execution elapsed time                         .000323
{% endhighlight %}

From the output we can see that ~51 seconds are spent on sequence load elapsed time which is amount of elapsed time spent getting the next sequence number from the data dictionary. This shows that the problem is with the sequence which is used for this particular statement.

I checked the sequence in USER_SEQUENCES in order to see if cache is used. But as I assumed cache was not used.

{% highlight sql %}
SQL> select sequence_name,cache_size from user_sequences where lower(sequence_name) = 'asc_restriction_lists_s';

SEQUENCE_NAME                  CACHE_SIZE
------------------------------ ----------
ASC_RESTRICTION_LISTS_S                 0
{% endhighlight %}

Next what I did was to set cache for this sequence.
With CACHE we are pre-allocating sequence numbers in memory so those cached numbers can be accessed faster. Be careful because in some circumstances those numbers in memory can be lost and you can end up with gaps.

{% highlight sql %}
SQL> alter sequence ASC_RESTRICTION_LISTS_S cache 100;

Sequence altered.
{% endhighlight %}

And when I re-run the .sql script (insert statement) it finished for ~2 seconds.

{% highlight sql %}
SQL> @tune

592007 rows created.

Elapsed: 00:00:02.09
{% endhighlight %}
