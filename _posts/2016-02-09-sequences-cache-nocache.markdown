---
layout: post
title:  "Sequences – Cache/No-Cache"
date:   2016-02-09
categories: oracle
---

Just a short post about sequences. A lot of people don’t pay much attention to sequences, but they can cause big troubles.

Here is an easy and short example:

{% highlight sql %}
Session 1

SQL> create table tab_seqcache(col1 number)
2  /

Table created.

SQL> create sequence seq_cache
2  start with 1 increment by 1 cache 100
3  /

Sequence created.

SQL> insert into tab_seqcache select seq_cache.nextval
2  from dual connect by rownum <= 10e5
3  /

1000000 rows created.

Elapsed: 00:00:02.65
{% endhighlight %}

Session 2

{% highlight sql %}
SQL> create table tab_seqnocache(col1 number)
2  /

Table created.

SQL> create sequence seq_nocache
2  start with 1 increment by 1 nocache
3  /

Sequence created.

SQL> insert into tab_seqnocache select seq_nocache.nextval
2  from dual connect by rownum <= 10e5
3  /

1000000 rows created.

Elapsed: 00:01:39.17
{% endhighlight %}

Next time be careful when you define a sequence, don’t underestimate its power. 😉