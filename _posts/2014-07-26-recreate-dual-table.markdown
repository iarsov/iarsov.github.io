---
layout: post
title:  "Recreate DUAL Table"
date:   2014-07-26
categories: oracle
---

Just a little experiment with the DUAL table.
Someone might find it useful.

{% highlight sql %}
[oracle@rac1edu ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.3.0 Production on Mon Jul 21 08:21:59 2014

Copyright (c) 1982, 2011, Oracle. All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL>
SQL> select count(*) from dual;

COUNT(*)
----------
1

SQL> desc dual;
Name Null? Type
----------------------------------------- -------- ----------------------------
DUMMY VARCHAR2(1)

SQL> drop table dual;

Table dropped.

SQL> select * from dual;
select * from dual
*
ERROR at line 1:
ORA-01775: looping chain of synonyms

SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup
ORACLE instance started.

Total System Global Area 849530880 bytes
Fixed Size 1348244 bytes
Variable Size 524291436 bytes
Database Buffers 318767104 bytes
Redo Buffers 5124096 bytes
Database mounted.
ORA-01092: ORACLE instance terminated. Disconnection forced
ORA-01775: looping chain of synonyms
Process ID: 27181
Session ID: 129 Serial number: 3
{% endhighlight %}

Re-create the dual table:

Start up the database in upgrade mode with replication_dependency_tracking= false

{% highlight sql %}
SQL> startup upgrade pfile='/home/oracle/pfile.ora';
ORACLE instance started.

Total System Global Area 849530880 bytes
Fixed Size 1348244 bytes
Variable Size 524291436 bytes
Database Buffers 318767104 bytes
Redo Buffers 5124096 bytes
Database mounted.
Database opened.
SQL>
SQL> create table dual
(dummy varchar2(1))
storage (initial 1)
/

Table created.

SQL> insert into dual values('X')
/

1 row created.

SQL> shutdown immediate;
ORA-01097: cannot shutdown while in a transaction - commit or rollback first
SQL>
SQL> commit;

Commit complete.

SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle@rac1edu ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.3.0 Production on Mon Jul 21 08:33:42 2014

Copyright (c) 1982, 2011, Oracle. All rights reserved.

Connected to an idle instance.

SQL> startup
ORACLE instance started.

Total System Global Area 849530880 bytes
Fixed Size 1348244 bytes
Variable Size 524291436 bytes
Database Buffers 318767104 bytes
Redo Buffers 5124096 bytes
Database mounted.
Database opened.
SQL> select count(*) from dual;

COUNT(*)
----------
1

SQL>
SQL> grant select on dual to public;
Grant succeeded.
{% endhighlight %}

