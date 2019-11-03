---
layout: post
title:  "Data Redaction – Virtual Column Bug Fixed In Oracle 12.2"
date:   2017-02-14
categories: oracle
---

If you’ve used virtual columns with data redaction you may have encountered the issue where the redacted data is visible through the virtual column. It is a known bug and with the release of Oracle 12.2 it has been fixed! I think I saw somewhere on MOS a note regarding this bug, but it seems it has been revoked as internal bug (couldn’t find it).

I've wrote couple of blog posts regarding Data Redaction

[Data Redaction – part 1](https://iarsov.github.io/oracle/data-redaction-part-1.html){:target="_blank"}<br/>
[Data Redaction – part 2](https://iarsov.github.io/oracle/data-redaction-part-2.html){:target="_blank"}

More on this virtual column bug with data redaction you can read at:

[Function Based Indexes and Data Redaction](https://iarsov.github.io/oracle/function-based-indexes-and-data-redaction.html){:target="_blank"}<br/>
[Data Redaction thoughts](https://iarsov.github.io/oracle/data-redaction-thoughts.html){:target="_blank"}

Before Oracle 12.2 if you had virtual column with data redaction policy on the original column you could see the data through the virtual column.
The below example is from Oracle 12.1 version.

{% highlight sql %}
SQ> select * from t1;

    ID    ID2
---------- ----------
    10     10

SQL>  begin
    dbms_redact.ADD_POLICY(OBJECT_SCHEMA=>user,
    OBJECT_NAME=>'T1',
    POLICY_NAME=>'FULL_REDACT',
    FUNCTION_TYPE=>DBMS_REDACT.FULL,
    COLUMN_NAME=>'ID',
    EXPRESSION=>'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
/

PL/SQL procedure successfully completed.

SQL>
SQL> select * from t1;

    ID    ID2
---------- ----------
     0     10
{% endhighlight %}

However, with 12.2 the issue has been resolved and now accessing a column which has redaction policy and it’s included in virtual column definition throws an ORA-28081.

{% highlight sql %}
SQL> select * from t1;

    ID    ID2
---------- ----------
    10     10

SQL>  begin
    dbms_redact.ADD_POLICY(OBJECT_SCHEMA=>user,
    OBJECT_NAME=>'T1',
    POLICY_NAME=>'FULL_REDACT',
    FUNCTION_TYPE=>DBMS_REDACT.FULL,
    COLUMN_NAME=>'ID',
    EXPRESSION=>'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
/

PL/SQL procedure successfully completed.

SQL>
SQL> select * from t1;
select * from t1
*
ERROR at line 1:
ORA-28081: Insufficient privileges - the command references a redacted object.
{% endhighlight %}