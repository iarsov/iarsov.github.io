---
layout: post
title:  "PL/SQL Function Result Cache"
date:   2014-08-21
categories: oracle
---

As you may already know result cache represents nice feature which is used to store results of query execution in the result cache area  (part of the SGA).

From 11g you can also cache pl/sql function results in the SGA and its available to all sessions which runs the application.
It’s easy to use, what you need to do is just mark the pl/sql function that its results should be cached.

You enable result caching for a function by adding the RESULT_CACHE clause in the function definition.

Also you can specify the optionally RELIES_ON clause which specifies tables/views on which the function results depends – since in this example the results depend on the HR.EMPLOYEES table I’ve added the optional keyword RELIES_ON

{% highlight sql %}
SQL> create function getEmpSalary(p_email varchar2)
  2  return number
  3  result_cache relies_on (employees)
  4  is
  5  v_salary employees.salary%type;
  6  begin
  7  select salary into v_salary from employees where lower(email) = lower(p_email);
  8  return v_salary;
  9  end;
 10  /

Function created.
{% endhighlight %}

Since this is test environment I flushed the cache and confirm that there are no cached information by querying the V$RESULT_CACHE_OBJECTS dictionary view.

{% highlight sql %}
SQL> exec dbms_result_cache.flush;

PL/SQL procedure successfully completed.

SQL>
SQL> select count(*) from v$result_cache_objects;

  COUNT(*)
----------
         0
{% endhighlight %}

Execute the function.

{% highlight sql %}
SQL> select HR.GETEMPSALARY('SKING') from dual;

HR.GETEMPSALARY('SKING')
------------------------
                   24000
{% endhighlight %}


Now we should see some information in the V$RESULT_CACHE_OBJECTS regarding our PL/SQL function.

{% highlight sql %}
SQL> col name format a70
SQL> col namespace format a20
SQL> select id, type, namespace, status, name from v$result_cache_objects;

        ID TYPE       NAMESPACE            STATUS    NAME
---------- ---------- -------------------- --------- ----------------------------------------------------------------------
         2 Dependency                      Published HR.EMPLOYEES
         0 Dependency                      Published HR.GETEMPSALARY
         1 Result     PLSQL                Published "HR"."GETEMPSALARY"::8."GETEMPSALARY"#8440831613f0f5d3 #1
{% endhighlight %}

There are some restriction for this feature that prevent caching, and those are:

If the function is pipelined table function
If there are IN OUT or OUT parameters
If the function has IN parameters of the following types (BLOB,CLOB,NCLOB,REF CURSOR,collection,object,record)
If the return type is one of the following (BLOB,CLOB,NCLOB,REF CURSOR,collection,object,record)
If the function is defined in a module that has the invoker’s rights or in an anonymous block

For more advanced topics about how to handle session-specific settings/application context check the online documentation <http://docs.oracle.com/cd/E11882_01/appdev.112/e25519/subprograms.htm#LNPLS706 target="_blank"/>