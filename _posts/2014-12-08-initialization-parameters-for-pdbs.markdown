---
layout: post
title:  "Initialization Parameters For PDBs"
date:   2014-12-28
categories: oracle
---

In 12c we also have one SPFILE which store information for the initialization parameters, it’s global for the entire CDB. We can also modify some of the parameters for specific PDB, but does that information is written into the SPFILE ?

Lets see which parameters we can modify at PDB level.

{% highlight sql %}
SQL> select name, ispdb_modifiable
  2  from v$system_parameter
  3  where ispdb_modifiable = 'TRUE';

NAME                                     ISPDB
---------------------------------------- -----
sessions                                 TRUE
timed_statistics                         TRUE
timed_os_statistics                      TRUE
resource_limit                           TRUE
nls_language                             TRUE
nls_territory                            TRUE
nls_sort                                 TRUE
nls_date_language                        TRUE
nls_date_format                          TRUE
nls_currency                             TRUE
nls_numeric_characters                   TRUE
nls_iso_currency                         TRUE
nls_calendar                             TRUE
nls_time_format                          TRUE
nls_timestamp_format                     TRUE
nls_time_tz_format                       TRUE
nls_timestamp_tz_format                  TRUE
nls_dual_currency                        TRUE
nls_comp                                 TRUE
nls_length_semantics                     TRUE
nls_nchar_conv_excp                      TRUE
resource_manager_plan                    TRUE
db_performance_profile                   TRUE
log_archive_dest_1                       TRUE
log_archive_dest_2                       TRUE
log_archive_dest_3                       TRUE
log_archive_dest_4                       TRUE
log_archive_dest_5                       TRUE
log_archive_dest_6                       TRUE
.....

185 rows selected.
{% endhighlight %}

Till this release 12.1.0.2 there are 185 init parameters that can be modified at PDB level.
In order do modify parameters at PDB level we need to be connected to PDB container for which we want to make the modification. If we are connected at ROOT container the parameter modification will take place for entire CDB.

{% highlight sql %}
SQL> show con_name;

CON_NAME
------------------------------
CDB$ROOT
SQL> show con_id;

CON_ID
------------------------------
1
SQL>
SQL> alter system set open_cursors=100 scope=both;

System altered.
{% endhighlight %}

Now lets modify the same parameter at PDB level

{% highlight sql %}
SQL> alter pluggable database pdb1 open;

Pluggable database altered.

SQL> alter session set container=pdb1;

Session altered.

SQL> alter system set open_cursors=50 scope=both;

System altered.
{% endhighlight %}

I’ve modified successfully the parameter at ROOT container and PDB1 container, now lets create PFILE and see the changes.

{% highlight sql %}
SQL> create pfile='/export/home/oracle/initdb12c.ora' from spfile;

File created.
{% endhighlight %}

This is the content from initdb12c.ora

{% highlight sql %}
....
*.db_files=800
*.db_name='db12c'
*.diagnostic_dest='/oracle/app/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=db12cXDB)'
*.enable_pluggable_database=true
*.inmemory_size=1073741824
*.memory_max_target=3G
*.memory_target=3G
*.open_cursors=100
*.processes=300
*.remote_login_passwordfile='EXCLUSIVE'
*.undo_tablespace='UNDOTBS1'
{% endhighlight %}

As we can see only the information about CDB is stored in the SPFILE, so the next question is where does the init parameters for the PDBs are stored?
They are stored in the data dictionary PDB_SPFILE$ table. The values are written in PDB_SPFILE$ only if we specify scope=spfile/both.

{% highlight sql %}
SQL> desc pdb_spfile$;
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 DB_UNIQ_NAME                              NOT NULL VARCHAR2(30)
 PDB_UID                                   NOT NULL NUMBER
 SID                                       NOT NULL VARCHAR2(80)
 NAME                                      NOT NULL VARCHAR2(80)
 VALUE$                                             VARCHAR2(4000)
 COMMENT$                                           VARCHAR2(255)
 SPARE1                                             NUMBER
 SPARE2                                             NUMBER
 SPARE3                                             VARCHAR2(128)

SQL>

SQL> col db_uniq_name format a10
SQL> col name format a15
SQL> col value$ format a10
SQL> select db_uniq_name, pdb_uid, name, value$ from pdb_spfile$;

DB_UNIQ_NA    PDB_UID NAME            VALUE$
---------- ---------- --------------- ----------
db12c      3485584667 open_cursors    50
{% endhighlight %}

Every time when we open a PDB if that particular PDB has its own values for some of the init parameters the values from SPFILE are overwritten with the values from PDB_SPFILE$.
Also, when we unplug a PDB if that particular PDB has some its own init parameters defined the values are extracted from PDB_SPFILE$.

Update:

In order to reset the value we use alter system reset …

{% highlight sql %}
SQL> alter session set container=pdb1;

Session altered.

SQL> alter system reset open_cursors scope=spfile;

System altered.

SQL> select db_uniq_name, pdb_uid, name, value$ from pdb_spfile$;

no rows selected

SQL>
SQL> show parameter open_cursors

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
open_cursors                         integer     100
{% endhighlight %}