---
layout: post
title:  "Enable Recovery For Disabled Pluggable Database In Data Guard Environment"
date:   2015-08-08
categories: oracle
---

What I would like to show is how to exclude (disable recovery) specific pluggable database from Data Guard environment, and also how to enable recovery for already excluded pluggable database.
The option for excluding/including pluggable databases from Data Guard is available from 12.1.0.2.

**Configuration**:
I have configured two database in Data Guard environment, simple and straightforward configuration with Data Guard Broker enabled.

Primary database: orcl [pc01edu-dg.lab.com]<br/>
Standby database: orclsb [pc02edu-dg.lab.com]

{% highlight sql %}
DGMGRL for Linux: Version 12.1.0.2.0 - 64bit Production

Copyright (c) 2000, 2013, Oracle. All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected as SYSDG.
DGMGRL>
DGMGRL> show configuration

Configuration - orcldg

  Protection Mode: MaxPerformance
  Members:
  orcl   - Primary database
    orclsb - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 14 seconds ago)
{% endhighlight %}

I’ve created pluggable database PDB3 on the primary database (orcl) without including it in the DG environment.
The trick here is to use STANDBYS=NONE.

{% highlight sql %}
SQL> create pluggable database pdb3 
     admin user pdb3adm identified by pdb3adm 
     default tablespace users datafile '/u01/app/oracle/oradata/orcl/pdb3/users01.dbf' size 100M autoextend ON 
     STANDBYS=NONE 
     file_name_convert = ('pdbseed','pdb3');

Pluggable database created.
{% endhighlight %}

When the pluggable database isn’t included in DG environment its data files are not created at the standby site, rather its data files are only defined in the data dictionary.
We can also confirm this by looking at standby database (orclsb) alert.log.

{% highlight sql %}
...
Recovery created pluggable database PDB3
File #14 added to control file as 'UNNAMED00014'. Originally created as:
'/u01/app/oracle/oradata/orcl/pdb3/system01.dbf'
because the pluggable database was created with nostandby
or the tablespace belonging to the pluggable database is
offline.
File #15 added to control file as 'UNNAMED00015'. Originally created as:
'/u01/app/oracle/oradata/orcl/pdb3/sysaux01.dbf'
because the pluggable database was created with nostandby
or the tablespace belonging to the pluggable database is
offline.
File #16 added to control file as 'UNNAMED00016'. Originally created as:
'/u01/app/oracle/oradata/orcl/pdb3/users01.dbf'
because the pluggable database was created with nostandby
or the tablespace belonging to the pluggable database is
offline.
...
{% endhighlight %}

To see for which pluggable databases recovery is disabled query RECOVERY_STATUS column from V$PDBS.

{% highlight sql %}
SQL> select name,open_mode,recovery_status from v$pdbs;


    CON_ID NAME                           OPEN_MODE  RECOVERY_STATUS
---------- ------------------------------ ---------- ---------------
         2 PDB$SEED                       MOUNTED    ENABLED
         3 PDB1                           MOUNTED    ENABLED
         4 PDB2                           MOUNTED    ENABLED
         5 PDB3                           MOUNTED    DISABLED
{% endhighlight %}

Notice that RECOVERY_STATUS is DISABLED for the new PDB3. This tells us that PDB3 in fact is not enabled for recovery in the Data Guard environment. It is just defined.
We can also confirm that its data files are defined as UNNAMED.

{% highlight sql %}
SQL> select name,status from v$datafile where con_id = 5

NAME                                                         STATUS
------------------------------------------------------------ -------
/u01/app/oracle/product/12.1.0/db/dbs/UNNAMED00014           SYSOFF
/u01/app/oracle/product/12.1.0/db/dbs/UNNAMED00015           RECOVER
/u01/app/oracle/product/12.1.0/db/dbs/UNNAMED00016           RECOVER
{% endhighlight %}

Enable pluggable database recovery in Data Guard 

Before we can enable recovery for PDB3 its data files needs to be put in the right place.

{% highlight sql %}
RMAN> backup pluggable database pdb3 format '/home/oracle/bkp/%U.bkp';

Starting backup at 07-AUG-15
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=48 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00015 name=/u01/app/oracle/oradata/orcl/pdb3/sysaux01.dbf
input datafile file number=00014 name=/u01/app/oracle/oradata/orcl/pdb3/system01.dbf
input datafile file number=00016 name=/u01/app/oracle/oradata/orcl/pdb3/users01.dbf
channel ORA_DISK_1: starting piece 1 at 07-AUG-15
channel ORA_DISK_1: finished piece 1 at 07-AUG-15
piece handle=/home/oracle/bkp/0gqe019l_1_1.bkp tag=TAG20150807T074348 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:25
Finished backup at 07-AUG-15

Starting Control File and SPFILE Autobackup at 07-AUG-15
piece handle=/u01/app/oracle/fast_recovery_area/ORCL/autobackup/2015_08_07/o1_mf_s_887096654_bw8kfhkb_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 07-AUG-15
{% endhighlight %}

Copy the backuppiece to the standby site.

{% highlight bash %}
[oracle@pc01edu-dg bkp]$ scp 0gqe019l_1_1.bkp oracle@pc02edu-dg:/home/oracle/bkp/
oracle@pc02edu-dg's password:
0gqe019l_1_1.bkp                                    100%  595MB  85.1MB/s   00:07
{% endhighlight %}

At this stage before we restore PDB3 and enable recovery, DG recovery must be stopped.

{% highlight sql %}
DGMGRL> edit database orclsb set state = 'apply-off';
Succeeded.
{% endhighlight %}

Restore PDB3 data files to its location.

{% highlight sql %}
RMAN> catalog backuppiece '/home/oracle/bkp/0gqe019l_1_1.bkp';

using target database control file instead of recovery catalog
cataloged backup piece
backup piece handle=/home/oracle/bkp/0gqe019l_1_1.bkp RECID=12 STAMP=887096972

RMAN> run {
set newname for datafile 14 to '/u01/app/oracle/oradata/orclsb/pdb3/system01.dbf';
set newname for datafile 15 to '/u01/app/oracle/oradata/orclsb/pdb3/sysaux01.dbf';
set newname for datafile 16 to '/u01/app/oracle/oradata/orclsb/pdb3/users01.dbf';
restore pluggable database pdb3;
switch datafile all;
}

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

Starting restore at 07-AUG-15
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=261 device type=DISK

datafile 14 is already restored to file /u01/app/oracle/oradata/orclsb/pdb3/system01.dbf
datafile 15 is already restored to file /u01/app/oracle/oradata/orclsb/pdb3/sysaux01.dbf
datafile 16 is already restored to file /u01/app/oracle/oradata/orclsb/pdb3/users01.dbf
restore not done; all files read only, offline, or already restored
Finished restore at 07-AUG-15

datafile 14 switched to datafile copy
input datafile copy RECID=18 STAMP=887097380 file name=/u01/app/oracle/oradata/orclsb/pdb3/system01.dbf
datafile 15 switched to datafile copy
input datafile copy RECID=19 STAMP=887097380 file name=/u01/app/oracle/oradata/orclsb/pdb3/sysaux01.dbf
datafile 16 switched to datafile copy
input datafile copy RECID=20 STAMP=887097380 file name=/u01/app/oracle/oradata/orclsb/pdb3/users01.dbf
{% endhighlight %}

Now, recovery for the pluggable database PDB3 can be enabled.

{% highlight sql %}
SQL> alter session set container=pdb3;

Session altered.

SQL> alter pluggable database pdb3 enable recovery;

Pluggable database altered.

SQL> select name,open_mode,recovery_status from v$pdbs;

NAME                           OPEN_MODE  RECOVERY
------------------------------ ---------- --------
PDB3                           MOUNTED    ENABLED
{% endhighlight %}

Enable Data Guard replication.

{% highlight sql %}
DGMGRL> edit database orclsb set state = 'apply-on';
Succeeded.
DGMGRL>
DGMGRL> show configuration

Configuration - orcldg

  Protection Mode: MaxPerformance
  Members:
  orcl   - Primary database
    orclsb - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 23 seconds ago)
{% endhighlight %}

**Test case**

Lets create short test to see if transactions to PDB3 are replied on our standby.

{% highlight sql %}
SQL> alter session set container=pdb3;

Session altered.

SQL> alter database open;

Database altered.

SQL> create user iarsov identified by iarsov;

User created.

SQL> grant create session,create table to iarsov;

Grant succeeded.

SQL> alter user iarsov quota unlimited on users;

User altered.

SQL> conn iarsov/iarsov@pdb3

SQL> create table tab as select rownum col1 from dual connect by rownum <= 10; Table created. 

SQL> select count(*) from tab;

  COUNT(*)
----------
        10
{% endhighlight %}

Check for changes at standby site.

{% highlight sql %}
SQL> alter database open;

Database altered.

SQL> select open_mode from v$database;

OPEN_MODE
--------------------
READ ONLY WITH APPLY

SQL> select name, open_mode from v$pdbs where con_id = 5;

NAME                           OPEN_MODE
------------------------------ ----------
PDB3                           MOUNTED

SQL> alter pluggable database pdb3 open;

Pluggable database altered.

SQL> alter session set container=pdb3;

Session altered.

SQL> select count(*) from iarsov.tab;

  COUNT(*)
----------
        10
{% endhighlight %}