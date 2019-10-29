---
layout: post
title:  "How To Get DDL For Scheduler Jobs Owned By SYS"
date:   2014-12-05
categories: oracle
---

As you may already now we can’t export scheduler jobs which are owned by SYS user. This is because the package/function dbms_metadata.get_ddl which is used to export the DDL is working similarly to the datapump export, and since SYS objects are marked non-exportable we can’t export them.

If you try to get DDL for scheduler job owned by SYS you’ll hit this error ORA-31603 as you can see from the example bellow

{% highlight sql %}
SQL> show user;
USER is "SYS"
SQL>
SQL>
SQL> begin
  2  dbms_scheduler.create_job(job_name => 'my_job',
  3  job_type => 'plsql_block',
  4  job_action => 'begin null; end;',
  5  start_date => (trunc(sysdate) + 10));
  6  end;
  7  /

PL/SQL procedure successfully completed.

SQL>
SQL>
SQL> select dbms_metadata.get_ddl('PROCOBJ','MY_JOB','SYS') from dual;
ERROR:
ORA-31603: object "MY_JOB" of type PROCOBJ not found in schema "SYS"
ORA-06512: at "SYS.DBMS_METADATA", line 5805
ORA-06512: at "SYS.DBMS_METADATA", line 8344
ORA-06512: at line 1

no rows selected
{% endhighlight %}

Solution for this is to copy the scheduler job to another user and then get the DDL.

{% highlight sql %}
SQL> show user;
USER is "SYS"
SQL>
SQL> exec dbms_scheduler.copy_job('SYS.MY_JOB','IARSOV.MY_JOB');

PL/SQL procedure successfully completed.

SQL> conn iarsov/iarsov
Connected.
SQL>
SQL>
SQL> set long 300
SQL>
SQL> select dbms_metadata.get_ddl('PROCOBJ','MY_JOB','IARSOV') from dual;

DBMS_METADATA.GET_DDL('PROCOBJ','MY_JOB','IARSOV')
--------------------------------------------------------------------------------
BEGIN
dbms_scheduler.create_job('"MY_JOB"',
job_type=>'PLSQL_BLOCK', job_action=>
'begin null; end;'
, number_of_arguments=>0,
start_date=>TO_TIMESTAMP_TZ('15-DEC-2014 12.00.00.000000000 AM +01:00','DD-MON-R
RRR HH.MI.SSXFF AM TZR','NLS_DATE_LANGUAGE=english'), repeat_interval=> NULL

...
{% endhighlight %}>

