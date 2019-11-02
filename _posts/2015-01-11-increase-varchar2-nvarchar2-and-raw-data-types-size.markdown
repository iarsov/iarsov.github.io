---
layout: post
title:  "Increase VARCHAR2, NVARCHAR2, And RAW Data Types Size"
date:   2015-01-11
categories: oracle
---

Starting with 12c we can extend the size for VARCHAR2, NVARCHAR2 and RAW data types. The modification is done by setting MAX_STRING_SIZE init parameter. Possible values are { STANDARD \| EXTENDED } where:

STANDARD = same as pre-12c configuration. Maximum size is 4000 bytes for VARHCAR2, NVARCHAR2 and 2000 bytes for RAW.<br/>
EXTENDED = maximum size is 32767 bytes.

In order to be able to set MAX_STRING_SIZE = EXTENDED the database must be 12c with init parameter COMPATIBLE set to 12.0.0.0 or higher. The trick when performing the change is that the database/container must be opened in UPGRADE mode. If you are using multitenant architecture remember that this is container based change (in other words, if you want to make the change in all containers than you’ll have to make the change in each of the containers separately).

When column data type size is declared with up to 4000 bytes for VARCHAR2, NVARCHAR2) and up to 2000 bytes for RAW, the column is known to be stored as inline. If the column data type is declared with size higher than 4000 up to 32767 the column is known to be stored as out-of-line. When the column is stored out-of-line internally LOB segment is created for column storage.

**Important note**: You can’t go back from EXTENDED to STANDARD mode.

In this post I’ll cover how to change MAX_STRING_SIZE for specific PDB. If you’re interested on how can you perform this change for non-CDB, RAC or Logical DG take a look at Oracle Reference Guide MAX_STRING_SIZE.

With default setting MAX_STRING_SIZE = STANDARD, if we try to specify higher size e.g. for VARCHAR2 we would get ORA-00910 error.

{% highlight sql %}
SQL> create table t(col1 varchar2(32767));
create table t(col1 varchar2(32767))
                             *
ERROR at line 1:
ORA-00910: specified length too long for its datatype
{% endhighlight %}

***How to change MAX_STRING_SIZE in a PDB?***

From ROOT container close the PDB (if it’s already opened) and startup in UPGRADE mode.

{% highlight sql %}
SQL> alter pluggable database pdb1 close;

Pluggable database altered.

SQL> 
SQL> alter pluggable database pdb1 open upgrade;

Pluggable database altered.
{% endhighlight %}

From PDB container set MAX_STRING_SIZE to ‘EXTENDED’.

{% highlight sql %}
SQL> conn sys@pdb1 as sysdba
Enter password: 
Connected.
SQL> 
SQL> select con_uid,name from v$pdbs;

   CON_UID NAME
---------- ------------------------------
2377496040 PDB1

SQL>
SQL> alter system set max_string_size = 'EXTENDED' scope=both;

System altered.
{% endhighlight %}

Confirm that you’ve set the MAX_STRING_SIZE correctly. If you are not familiar with init parameters for PDBs read this post for more details.

{% highlight sql %}
SQL> conn / as sysdba
Connected.
SQL>
SQL> select con_uid,name from v$pdbs where name = 'PDB1';

   CON_UID NAME
---------- ------------------------------
2377496040 PDB1

SQL>
SQL> set linesize 150
SQL> col value$ format a20
SQL> col name format a20
SQL>
SQL> select db_uniq_name,
  2         pdb_uid,
  3         name,
  4         value$
  5  from   pdb_spfile$
  6  where  pdb_uid = '2377496040'
  7  and name = 'max_string_size'
  8 /

DB_UNIQ_NAME              PDB_UID NAME             VALUE$
------------------------------ ---------- -------------------- --------------------
cdb                2377496040 max_string_size      'EXTENDED'
{% endhighlight %}

Run _$ORACLE_HOME/rdbms/admin/utl32k.sql_ from PDB container logged as SYSDBA.

{% highlight sql %}
SQL> conn sys@pdb1 as sysdba
Enter password: 
Connected.
SQL> 
SQL> @?/rdbms/admin/utl32k.sql

Session altered.

DOC>#######################################################################
DOC>#######################################################################
DOC>   The following statement will cause an "ORA-01722: invalid number"
DOC>   error if the database has not been opened for UPGRADE.
DOC>
DOC>   Perform a "SHUTDOWN ABORT"  and
DOC>   restart using UPGRADE.
DOC>#######################################################################
DOC>#######################################################################
DOC>#

no rows selected

DOC>#######################################################################
DOC>#######################################################################
DOC>   The following statement will cause an "ORA-01722: invalid number"
DOC>   error if the database does not have compatible >= 12.0.0
DOC>
DOC>   Set compatible >= 12.0.0 and retry.
DOC>#######################################################################
DOC>#######################################################################
DOC>#

PL/SQL procedure successfully completed.


Session altered.


2 rows updated.


Commit complete.


System altered.


PL/SQL procedure successfully completed.


Commit complete.


System altered.


Session altered.


PL/SQL procedure successfully completed.

No errors.

Session altered.


PL/SQL procedure successfully completed.


Commit complete.


Package altered.


Package altered.
{% endhighlight %}

After you’ve run the script open the PDB in normal mode.

{% highlight sql %}
SQL> alter pluggable database pdb1 close;

Pluggable database altered.

SQL> alter pluggable database pdb1 open;

Pluggable database altered.
{% endhighlight %}

Because utl32k.sql invalidates some system views and synonyms we need to run _@?/rdbms/admin/utlrp.sql_ to re-validate invalid objects.

{% highlight sql %}
SQL> conn sys@pdb1 as sysdba
Enter password: 
Connected.
SQL> 
SQL> @?/rdbms/admin/utlrp.sql

TIMESTAMP
------------------------------------------------------------------------------------------------------------------------------------------------------
COMP_TIMESTAMP UTLRP_BGN  2015-01-11 14:28:37

DOC>   The following PL/SQL block invokes UTL_RECOMP to recompile invalid
DOC>   objects in the database. Recompilation time is proportional to the
DOC>   number of invalid objects in the database, so this command may take
DOC>   a long time to execute on a database with a large number of invalid
DOC>   objects.
DOC>
DOC>   Use the following queries to track recompilation progress:
DOC>
DOC>   1. Query returning the number of invalid objects remaining. This
DOC>   number should decrease with time.
DOC>      SELECT COUNT(*) FROM obj$ WHERE status IN (4, 5, 6);
DOC>
DOC>   2. Query returning the number of objects compiled so far. This number
DOC>   should increase with time.
DOC>      SELECT COUNT(*) FROM UTL_RECOMP_COMPILED;
DOC>
DOC>   This script automatically chooses serial or parallel recompilation
DOC>   based on the number of CPUs available (parameter cpu_count) multiplied
DOC>   by the number of threads per CPU (parameter parallel_threads_per_cpu).
DOC>   On RAC, this number is added across all RAC nodes.
DOC>
DOC>   UTL_RECOMP uses DBMS_SCHEDULER to create jobs for parallel
DOC>   recompilation. Jobs are created without instance affinity so that they
DOC>   can migrate across RAC nodes. Use the following queries to verify
DOC>   whether UTL_RECOMP jobs are being created and run correctly:
DOC>
DOC>   1. Query showing jobs created by UTL_RECOMP
DOC>      SELECT job_name FROM dba_scheduler_jobs
DOC>     WHERE job_name like 'UTL_RECOMP_SLAVE_%';
DOC>
DOC>   2. Query showing UTL_RECOMP jobs that are running
DOC>      SELECT job_name FROM dba_scheduler_running_jobs
DOC>     WHERE job_name like 'UTL_RECOMP_SLAVE_%';
DOC>#

PL/SQL procedure successfully completed.


TIMESTAMP
------------------------------------------------------------------------------------------------------------------------------------------------------
COMP_TIMESTAMP UTLRP_END  2015-01-11 14:28:47

DOC> The following query reports the number of objects that have compiled
DOC> with errors.
DOC>
DOC> If the number is higher than expected, please examine the error
DOC> messages reported with each object (using SHOW ERRORS) to see if they
DOC> point to system misconfiguration or resource constraints that must be
DOC> fixed before attempting to recompile these objects.
DOC>#

OBJECTS WITH ERRORS
-------------------
          0

DOC> The following query reports the number of errors caught during
DOC> recompilation. If this number is non-zero, please query the error
DOC> messages in the table UTL_RECOMP_ERRORS to see if any of these errors
DOC> are due to misconfiguration or resource constraints that must be
DOC> fixed before objects can compile successfully.
DOC>#

ERRORS DURING RECOMPILATION
---------------------------
              0


Function created.


PL/SQL procedure successfully completed.


Function dropped.

...Database user "SYS", database schema "APEX_040200", user# "98" 14:29:00
...Compiled 0 out of 3014 objects considered, 0 failed compilation 14:29:00
...271 packages
...263 package bodies
...452 tables
...11 functions
...16 procedures
...3 sequences
...457 triggers
...1320 indexes
...211 views
...0 libraries
...6 types
...0 type bodies
...0 operators
...0 index types
...Begin key object existence check 14:29:00
...Completed key object existence check 14:29:00
...Setting DBMS Registry 14:29:00
...Setting DBMS Registry Complete 14:29:01
...Exiting validate 14:29:01

PL/SQL procedure successfully completed.
{% endhighlight %}

At the end if you like you can perform small test to confirm that the change is successful.

{% highlight sql %}
SQL> conn hr@pdb1
Enter password: 
Connected.
SQL> 
SQL> 
SQL> 
SQL> create table t(col1 varchar2(32767));

Table created.
{% endhighlight %}

As previously specified when the column is stored out-of-place LOB segment is internally created for column storage.

{% highlight sql %}
SQL> select table_name, column_name, segment_name from user_lobs;

TABLE_NAME COLUMN_NAM SEGMENT_NAME
---------- ---------- ------------------------------
T      COL1       SYS_LOB0000092744C00001$
{% endhighlight %}

***Restrictions***

There are several restrictions with extended character data types.
– If we have already existing indexed columns in order to be able to increase the size we have to drop the indexes, increase the columns size and as last step re-create the indexes on those columns.

* It’s not supported for index-organized tables and clustered tables

{% highlight sql %}
SQL> create table t2(id number primary key, col1 varchar2(4001)) organization index overflow;
create table t2(id number primary key, col1 varchar2(4001)) organization index overflow
*
ERROR at line 1:
ORA-14691: Extended character types are not allowed in this table.
{% endhighlight %}

However, we can still specify size up to 4000 bytes (column is stored inline) without any problems.

{% highlight sql %}
SQL> create table t1(id number primary key, col1 varchar2(4000)) organization index overflow;

Table created.
{% endhighlight %}

* There is also restriction on intrapartition parallel operations on DDL, DML, DELETE DML and direct-path inserts for tables in ASSM managed tablespace.

