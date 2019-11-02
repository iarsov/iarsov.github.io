---
layout: post
title:  "Data Redaction Thoughts"
date:   2015-03-02
categories: oracle
---

With Data Redaction we can define policy for specific table/view columns where its data is redacted when it’s queried by end/application users. I won’t go into details about how policies are created, the available types for redaction etc… For that topic you can check Data Redaction [part 1](https://iarsov.github.io/oracle/data-redaction-part-1.html){:target="_blank"} and [part 2](https://iarsov.github.io/oracle/data-redaction-part-2.html){:target="_blank"}

What I want to point is that Data Redaction doesn’t work properly when function based indexes or indexes based on expression are used for the redacted column. I couldn’t find any logic explanation why it is like that, if you have any opinions feel free to comment.

As example I took table EMPLOYEES from HR schema.

{% highlight sql %}
SQL> desc employees;
 Name                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 EMPLOYEE_ID                   NOT NULL NUMBER(6)
 FIRST_NAME                     VARCHAR2(20)
 LAST_NAME                 NOT NULL VARCHAR2(25)
 EMAIL                     NOT NULL VARCHAR2(25)
 PHONE_NUMBER                       VARCHAR2(20)
 HIRE_DATE                 NOT NULL DATE
 JOB_ID                    NOT NULL VARCHAR2(10)
 SALARY                       NUMBER(8,2)
 COMMISSION_PCT                     NUMBER(2,2)
 MANAGER_ID                     NUMBER(6)
 DEPARTMENT_ID                      NUMBER(4)
{% endhighlight %}

I’ve defined Data Redaction policy SALARY_FULL_REDACT (with FULL redaction type) which redacts SALARY column to 0 (zero).
If you didn’t know, default value for DBMS_REDACT.FULL is 0 (zero).

{% highlight sql %}
begin
    dbms_redact.ADD_POLICY(OBJECT_SCHEMA=>'HR',
                   OBJECT_NAME=>'EMPLOYEES',
                   POLICY_NAME=>'SALARY_FULL_REDACT',
                   FUNCTION_TYPE=>DBMS_REDACT.FULL,
                   COLUMN_NAME=>'SALARY',
                   EXPRESSION=>'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
/

PL/SQL procedure successfully completed.
{% endhighlight %}

As you can see this policy is in effect only for iarsov user. If i log in as iarsov and query SALARY column from HR.EMPLOYEES I should get 0 as result.

{% highlight sql %}
SQL> show user;
USER is "IARSOV"
SQL> 
SQL> 
SQL> select first_name,last_name, salary from hr.employees where email = 'SKING';

FIRST_NAME       LAST_NAME             SALARY
-------------------- ------------------------- ----------
Steven           King               0
{% endhighlight %}

So far so good, as expected data redaction took place.
Now, lets create normal index on SALARY and check if we have any difference.

{% highlight sql %}
SQL> create index ss_ix on employees(salary);

Index created.

SQL>
SQL> 
SQL> conn iarsov@pdb1
Enter password: 
Connected.
SQL> 
SQL> 
SQL> 
SQL> select first_name,last_name, salary from hr.employees where email = 'SKING';

FIRST_NAME       LAST_NAME             SALARY
-------------------- ------------------------- ----------
Steven           King               0
{% endhighlight %}

Again, data redaction took place. It seems perfect.

Next I would like to show whether Data Redaction will be used if we use user-defined function to access SALARY values. For that purpose I’ve created function f as simple (dummy) function which returns the same value passed as parameter. The function is defined as deterministic because later it’s used for the function-based index definition.

{% highlight sql %}
SQL> create function f(p_val number)
  2  return number deterministic
  3  is
  4  begin
  5  return p_val;
  6  end f;
  7  /

Function created.
SQL>
SQL> conn iarsov@pdb1
Enter password: 
Connected.
SQL> 
SQL> show user;
USER is "IARSOV"
SQL>
SQL> select first_name,last_name, hr.f(salary) from hr.employees where email = 'SKING';
FIRST_NAME       LAST_NAME             HR.F(SALARY)
-------------------- ------------------------- ------------
Steven           King                 0
{% endhighlight %}

Great, everything works fine.
Now, lets create function based index on SALARY and try the query again.

{% highlight sql %}
SQL> create index ss_ix_f on employees(f(salary));

Index created.

SQL>
SQL> conn iarsov@pdb1
Enter password: 
Connected.
SQL> 
SQL> show user;
USER is "IARSOV"
SQL> 
SQL> select first_name,last_name, hr.f(salary) from hr.employees e where email = 'SKING';

FIRST_NAME       LAST_NAME             HR.F(SALARY)
-------------------- ------------------------- ------------
Steven           King                 24000
{% endhighlight %}

We’ve just enabled access to every employee salary (sensitive data).
So, is this a bug or normal behavior ?
Me personally, I think its a bug because data redaction works fine if we don’t have the index, but with the index somehow data redaction is disabled or something is messing up …

GROUP BY clause

According database documentation SQL expression are not allowed on redacted column if used in GROUP BY clause, ORA-00979 will be raised.
Just to prove, as expected:

{% highlight sql %}
SQL> select first_name,last_name, hr.f(salary) 
     from hr.employees 
     where email = 'SKING' 
     group by first_name, last_name, hr.f(salary);
select first_name,last_name, hr.f(salary)
                                  *
ERROR at line 1:
ORA-00979: not a GROUP BY expression

That is not the case if we have defined function-based index or index based on expression on the redacted column.

SQL> select first_name,last_name, hr.f(salary) 
     from hr.employees 
     where email = 'SKING' 
     group by first_name, last_name, hr.f(salary);

FIRST_NAME       LAST_NAME             HR.F(SALARY)
-------------------- ------------------------- ------------
Steven           King                 24000
{% endhighlight %}

Be careful when implementing Data Redaction because you might not intentionally reveal sensitive data.
* I haven’t tested if this behavior is same for all data redaction types.

***Update***:

Data redaction also breaks with virtual columns (no indexes involved): [Data Redaction thoughts](https://iarsov.github.io/oracle/data-redaction-thoughts.html){:target="_blank"}
