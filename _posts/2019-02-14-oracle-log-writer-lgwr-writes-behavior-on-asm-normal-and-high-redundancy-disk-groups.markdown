---
layout: post
title:  "Oracle Log Writer (LGWR) Writes Behavior On ASM – NORMAL And HIGH Redundancy Disk Groups"
date:   2019-02-14
categories: oracle
---

Sometime ago we had an internal discuss at Pythian about how LGWR writes are handled on ASM diskgroup configured with NORMAL or HIGH redundancy. Does LGWR waits for write acknowledgment from the writes on primary and secondary (mirror) extents? Some people had divided opinions.

If you think about it, it’s natural to wait for the write operations to complete on primary and secondary extents, otherwise what’s the point of having redundancy in the first place.

Imagine what would happen if disk where the first write happend experience failure and the system already retrieved confirmation of the write without writing to all fail-groups.

**Test case**:

I have 2 available partitioned disks /dev/sdd1 and /dev/sde1.

{% highlight bash %}
ll /dev/sd*
...
brw-rw---- 1 root disk 8, 48 May 10 08:47 /dev/sdd
brw-rw---- 1 root disk 8, 49 May 10 08:47 /dev/sdd1
brw-rw---- 1 root disk 8, 64 May 10 08:47 /dev/sde
brw-rw---- 1 root disk 8, 65 May 10 08:47 /dev/sde1
{% endhighlight %}

Let’s create device on top of /dev/sdd1 for which we will use dm_delay to delay writes for 5 seconds.

{% highlight bash %}
./delay_writes.sh /dev/sdd1 --create delay device with dm_delay
{% endhighlight %}

For IO delay I am using the delay option from dmsetup.

delay Delays reads and/or writes to different devices. Useful for testing

It offers a way to delay reads and/or writes against devices. More information you can find at <https://www.kernel.org/doc/Documentation/device-mapper/delay.txt>{:target="_blank"}

The new device will be created in /dev/mapper directory.

{% highlight bash %}
ll /dev/mapper/d1-delay
lrwxrwxrwx 1 root root 7 May 10 08:48 /dev/mapper/d1-delay -> ../dm-8
{% endhighlight %}

Next is to create ASM disks. I am using asmlib libraries. One ASM disk against newly created device with delayed IO and the other one against /dev/sde1 device.

{% highlight sql %}
/usr/sbin/oracleasm createdisk TEST_D1_DELAYED /dev/mapper/d1-delay
Writing disk header: done
Instantiating disk: done

/usr/sbin/oracleasm createdisk TEST_D2_NORMAL /dev/sde1
Writing disk header: done
Instantiating disk: done
{% endhighlight %}

Create the ASM diskgroup.

{% highlight sql %}
SQL> create diskgroup DELAYED_IO_DG normal redundancy disk 'ORCL:TEST_D1_DELAYED', 'ORCL:TEST_D2_NORMAL';

Diskgroup created.
{% endhighlight %}

First let’s confirm log switch is fast.

{% highlight sql %}
SQL> set lines 200
col member for a60
select vl.group#, vl.status, vlf.member
from v$log vl, v$logfile vlf
where vl.group# = vlf.group#
order by vl.group#;

GROUP# STATUS       MEMBER
---------- ---------------- ------------------------------------------------------------
1 CURRENT       /oracle/app/oracle/oradata/oraee/redo01.log
2 INACTIVE      /oracle/app/oracle/oradata/oraee/redo02.log
3 INACTIVE      /oracle/app/oracle/oradata/oraee/redo03.log

Elapsed: 00:00:00.01
SQL>
SQL> alter system archive log current;

System altered.

Elapsed: 00:00:00.08
SQL>
SQL> r
1* alter system archive log current

System altered.

Elapsed: 00:00:00.05
SQL> r
1* alter system archive log current

System altered.

Elapsed: 00:00:00.46
{% endhighlight %}

Add logfile group to the new diskgroup.

{% highlight sql %}
SQL> ALTER DATABASE ADD LOGFILE ('+DELAYED_IO_DG') SIZE 50M;

Database altered.

Elapsed: 00:02:23.26 --> TOOK LONG DUE TO DELAYED IO
SQL>

SQL> set lines 200
col member for a60
select vl.group#, vl.status, vlf.member
from v$log vl, v$logfile vlf
where vl.group# = vlf.group#
order by vl.group#;

GROUP# STATUS       MEMBER
---------- ---------------- ------------------------------------------------------------
1 CURRENT       /oracle/app/oracle/oradata/oraee/redo01.log
2 INACTIVE      /oracle/app/oracle/oradata/oraee/redo02.log
3 INACTIVE      /oracle/app/oracle/oradata/oraee/redo03.log
8 UNUSED        +DELAYED_IO_DG/oraee/onlinelog/group_8.256.975747739

Elapsed: 00:00:00.01
{% endhighlight %}

Switch logfile for the new logfile group to become current.

{% highlight sql %}
SQL> alter system archive log current;

System altered.

Elapsed: 00:00:05.15 --> WRITES TO REDO LOGS LOCATED ON DISKGROUP WITH DELAYED IO

SQL> set lines 200
col member for a60
select vl.group#, vl.status, vlf.member
from v$log vl, v$logfile vlf
where vl.group# = vlf.group#
order by vl.group#;

GROUP# STATUS       MEMBER
---------- ---------------- ------------------------------------------------------------
1 ACTIVE        /oracle/app/oracle/oradata/oraee/redo01.log
2 INACTIVE      /oracle/app/oracle/oradata/oraee/redo02.log
3 INACTIVE      /oracle/app/oracle/oradata/oraee/redo03.log
8 CURRENT       +DELAYED_IO_DG/oraee/onlinelog/group_8.256.975747739

Elapsed: 00:00:00.00
{% endhighlight %}

Next is to test COMMIT.

{% highlight sql %}
SQL> create table t1(id number);

Table created.

SQL>
SQL> set timi on
SQL>

--CONFIRM CURRENT REDO LOG GROUP IS ON DISKGROUP WITH DELAYED IO
SQL> r
1 select vl.group#, vl.status, vlf.member
2 from v$log vl, v$logfile vlf
3 where vl.group# = vlf.group#
4* order by vl.group#

GROUP# STATUS MEMBER

1 INACTIVE /oracle/app/oracle/oradata/oraee/redo01.log
2 INACTIVE /oracle/app/oracle/oradata/oraee/redo02.log
3 INACTIVE /oracle/app/oracle/oradata/oraee/redo03.log
8 CURRENT +DELAYED_IO_DG/oraee/onlinelog/group_8.256.975747739

SQL> insert into t1 values (10);

1 row created.

Elapsed: 00:00:00.00
SQL> commit;

Commit complete.

Elapsed: 00:00:05.98
{% endhighlight %}

As you can see COMMIT took almost 6 seconds to complete.
