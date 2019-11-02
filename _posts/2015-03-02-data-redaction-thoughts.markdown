---
layout: post
title:  "Data Redaction Thoughts"
date:   2015-03-02
categories: oracle
---

This is continuation of previous [post](https://iarsov.github.io/oracle/function-based-indexes-and-data-redaction){:target="_blank"} about Data Redaction behavior with function-based indexes and virtual columns which breaks in some cases when virtual columns are involved. Most likely function based indexes are affected because in background a virtual column is defined to store metadata. After short discussion on Tweeter, Jonathan Lewis mentioned about ORA-28083 error which according documentations it says:

> ORA-28083: A redacted column was referenced in a virtual column expression.
  Cause: This redacted column was referenced in a virtual column expression.
  Action: Ensure the column expression definition for any virtual column does not refer to any redacted columns. To check for columns with redaction policies, use the REDACTION_COLUMNS catalog view which lists all data redaction policies.

There is an error ORA-28083 which *should* prevent data redaction policy creation on column already involved in virtual column definition and vice versa, restriction for virtual column definition on already redacted columns.

I agree on this and very likely there is some kind of relation disruption between FBIs – Virtual Columns – Data Redaction.
Again, after some tests, 12c redaction partially failed and this time *no indexes* were involved.

{% highlight sql %}
SQL> begin
2 dbms_redact.ADD_POLICY(OBJECT_SCHEMA=>'HR',
OBJECT_NAME=>'EMPLOYEES',
POLICY_NAME=>'SALARY_FULL_REDACT',
FUNCTION_TYPE=>DBMS_REDACT.FULL,
COLUMN_NAME=>'SALARY',
EXPRESSION=>'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
/

PL/SQL procedure successfully completed.
{% endhighlight %}

Now, If we try to create virtual column it should fail with ORA-28083, which is expected since it’s not supported (documented).

{% highlight sql %}
SQL> alter table hr.employees add sal number as (salary+0) virtual;
alter table hr.employees add sal number as (salary+0) virtual
*
ERROR at line 1:
ORA-28083: A redacted column was referenced in a virtual column expression.
{% endhighlight %}

So far so good, but what happens if we already have virtual column (created by developer and we’re not aware) defined and we try to create redaction policy on original column? You will be surprised because ORA-28083 won’t be raised (revealing sensitive data).

{% highlight sql %}
SQL> exec dbms_redact.drop_policy('HR','EMPLOYEES','SALARY_FULL_REDACT');

PL/SQL procedure successfully completed.

SQL>
SQL> alter table hr.employees add sal number as (salary+0) virtual;

Table altered.

SQL>
SQL> begin
2 dbms_redact.ADD_POLICY(OBJECT_SCHEMA=>'HR',
OBJECT_NAME=>'EMPLOYEES',
POLICY_NAME=>'SALARY_FULL_REDACT',
FUNCTION_TYPE=>DBMS_REDACT.FULL,
COLUMN_NAME=>'SALARY',
EXPRESSION=>'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
/

PL/SQL procedure successfully completed.    <----- ORA-28083 NOT RAISED
{% endhighlight %}

And, again other users in this case IARSOV is able to see salary

{% highlight sql %}
SQL> select salary+0 from hr.employees where email = 'SKING';

SALARY+0
----------
24000
{% endhighlight %}
