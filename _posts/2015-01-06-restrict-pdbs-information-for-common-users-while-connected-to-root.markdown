---
layout: post
title:  "Restrict PDBs Information For Common Users While Connected To Root"
date:   2015-01-06
categories: oracle
---

By default common users do not have privileges to query information related to other PDBs when are connected to CDB$ROOT container. If appropriate privileges are given common users can query PDBs information when are connected to specific PDB container.
The point of this post is to show how can information access for specific (or all) PDBs is given to common users when are connected to CDB$ROOT container. Restrictions can be defined on views and tables which are defined as [container data objects](http://docs.oracle.com/database/121/CNCPT/glossary.htm#CNCPT89415){:target="_blank"}. We can list all views which are defined as container data objects from [DBA|ALL|USER]_[VIEWS|TABLES].

{% highlight sql %}
SQL> select count(*) from dba_views where container_data = 'Y';

  COUNT(*)
----------
      2665
{% endhighlight %}

In the release which I’m running (12.1.0.2.0) there are total of 2665 views.
I’ve created a common user with select privileges to CDB_DATA_FILES dictionary view. We’ll see how can we enable and restrict access to information only for XDB1 pluggable database when C##IARSOV is connected to CDB$ROOT container.

{% highlight sql %}
SQL> create user c##iarsov identified by iarsov container=all;

User created.

SQL> grant create session to c##iarsov container=all;

Grant succeeded.

SQL> grant select on cdb_data_files to c##iarsov container=all;

Grant succeeded.
{% endhighlight %}

If I try to query datafile information from CDB_DATA_FILES for XDB1 pluggable database from CDB$ROOT I should get zero results.
With only these settings C##IARSOV is only available to see datafile information (container information) for the container to which is connected.

From CDB$ROOT:

{% highlight sql %}
SQL> select count(*), con_id from cdb_data_files group by con_id;

  COUNT(*)     CON_ID
---------- ----------
     4      1
{% endhighlight %}

From XDB1 pluggable database:

{% highlight sql %}
SQL> conn c##iarsov@xdb1
Enter password: 
Connected.
SQL> 
SQL> select count(*), con_id from cdb_data_files group by con_id;

  COUNT(*)     CON_ID
---------- ----------
     4      3
{% endhighlight %}

In order for C##IARSOV to be able to query information for XDB1 from CDB$ROOT we need to alter the user and set CONTAINER_DATA attribute. With SET CONTAINER_DATA clause we specify for which containers the user can see information and with FOR container_data_object we specify the container data object. In this case to restrict C##IARSOV to be able to see information from CDB_DATA_FILES I’ve used the following alter user definition.

{% highlight sql %}
SQL> conn system@db12c
Enter password: 
Connected.
SQL> 
SQL> alter user c##iarsov
  2  set container_data = (cdb$root, xdb1)
  3  for cdb_data_files
  4  container = current;

User altered.
{% endhighlight %}

Note that CDB$ROOT must be included with SET CONTAINER_DATA.
Further, we can check the definition for already given accesses in CDB_CONTAINER_DATA dictionary view and whether the access is given for all containers (see ALL_CONTAINERS column) or it’s given for specific PDBs (see CON_NAME column).

{% highlight sql %}
SQL> set linesize 150
SQL> 
SQL> col username format a10
SQL> col owner format a8
SQL> col object_name format a15
SQL> col all_containers format a15
SQL> col con_name format a10
SQL> 
SQL> select username, owner, object_name, all_containers, container_name con_name
  2  from cdb_container_data
  3  where username = 'C##IARSOV';

USERNAME   OWNER    OBJECT_NAME     ALL_CONTAINERS  CON_NAME
---------- -------- --------------- --------------- ----------
C##IARSOV  SYS      CDB_DATA_FILES  N          CDB$ROOT
C##IARSOV  SYS      CDB_DATA_FILES  N          XDB1
{% endhighlight %}

Now, C##IARSOV should be able to query XDB1 information from CDB$ROOT container.

{% highlight sql %}
SQL> conn c##iarsov@db12c
Enter password: 
Connected.
SQL> 
SQL> show user;
USER is "C##IARSOV"
SQL> 
SQL> show con_name;

CON_NAME
------------------------------
CDB$ROOT
SQL> 
SQL> 
SQL> select count(*), con_id from cdb_data_files group by con_id;

  COUNT(*)     CON_ID
---------- ----------
     4      1
     4     3
{% endhighlight %}

If we want to define CONTAINER_DATA attribute for all containers we can use SET CONTAINER_DATA = ALL.

{% highlight sql %}
SQL> alter user c##iarsov set container_data=all for cdb_data_files container=current;

User altered.
{% endhighlight %}

After CONTAINER_DATA attribute is set for specific container data object we can use ADD CONTAINER_DATA clause to add additional containers.

{% highlight sql %}
SQL> alter user c##iarsov add container_data=(cdb$root,pdb1) for v$session container=current;

User altered.
{% endhighlight %}

If we try to use ADD CONTAINER_DATA for container data object that is not already defined with SET CONTAINER_DATA or CONTAINER_DATA attribute is already set to ALL we will hit ORA-65060: CONTAINER_DATA attribute is not set.

To remove specific PDB container from container_data attribute we use REMOVE CONTAINER_DATA clause.

{% highlight sql %}
SQL> alter user c##iarsov remove container_data=(xdb1) for v$session container=current;

User altered.
{% endhighlight %}

For more information and syntax details check [container_data_clause](http://docs.oracle.com/database/121/SQLRF/statements_4003.htm#BABFIGJH){:target="_blank"}
