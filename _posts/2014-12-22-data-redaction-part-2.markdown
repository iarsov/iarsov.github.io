---
layout: post
title:  "Data Redaction – Part 2"
date:   2014-12-22
categories: oracle
---

Finally I’ve found some time to finish this part 2 which covers date and number data types. If you are interested for redaction of character data type you can check [part 1](https://iarsov.github.io/oracle/data-redaction-part-1.html){:target="_blank"}.

**Preparation**

For this post I'm using TIME_ID and AMOUNT_SOLD columns from sales table – SH schema.

{% highlight sql %}
SQL> desc sales;
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 PROD_ID                                   NOT NULL NUMBER
 CUST_ID                                   NOT NULL NUMBER
 TIME_ID                                   NOT NULL DATE
 CHANNEL_ID                                NOT NULL NUMBER
 PROMO_ID                                  NOT NULL NUMBER
 QUANTITY_SOLD                             NOT NULL NUMBER(10,2)
 AMOUNT_SOLD                               NOT NULL NUMBER(10,2)
{% endhighlight %}

**FULL REDACT**

* Numbers

Policy definition is simple, the only “special” parameter that needs to be set is function_type parameter (to DBMS_REDACT.FULL). We don’t need to set other specific parameters for this type of redaction. Oracle will perform full redaction with predefined default value.

{% highlight sql %}
begin
  DBMS_REDACT.ADD_POLICY (
    object_schema  => 'SH',
    object_name    => 'SALES',
    policy_name    => 'SALES_AMOUNT_FULL',
    column_name    => 'AMOUNT_SOLD',
    function_type  => DBMS_REDACT.FULL,
    expression     => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
{% endhighlight %}

I've tested this policy with another user who has select privilege over sh.sales table. Note that I’ve enabled this policy only for specific user (in this case IARSOV user) by setting expression to SYS_CONTEXT(‘USERENV’,’CURRENT_USER’) = ‘IARSOV’

{% highlight sql %}
SQL> conn iarsov@xdb1
Enter password:
Connected.
SQL>
SQL>
SQL> select prod_id,
cust_id,
amount_sold
from sh.sales
fetch first 5 rows only;

   PROD_ID    CUST_ID AMOUNT_SOLD
---------- ---------- -----------
        13        987           0
        13       1660           0
        13       1762           0
        13       1843           0
        13       1948           0
{% endhighlight %}

* Dates

As in previous example there aren’t any special settings. Just specify DBMS_REDACT.FULL for function_type.

Policy definition:

{% highlight sql %}
DBMS_REDACT.DROP_POLICY('SH','SALES','SALES_AMOUNT_FULL');
DBMS_REDACT.ADD_POLICY (
       object_schema  => 'SH',
       object_name    => 'SALES',
       policy_name    => 'SALES_TIME_FULL',
       column_name    => 'TIME_ID',
       function_type  => DBMS_REDACT.FULL,
       expression     => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
/
{% endhighlight %}

Result:

{% highlight sql %}
SQL> select distinct to_char(time_id,'dd.mm.yyyy') time_id from sh.sales;

TIME_ID
----------
01.01.2001
{% endhighlight %}

TIME_ID was replaced with default value (01.01.2001) for all 918843 rows.

**PARTIAL REDACT**

* Numbers

For this redaction type I chose policy definition that replaces first two numbers with the number 9. Format defined for function_parameters:  ‘9,1,2’

9 – what number to use as replacement<br/>
1 – start position<br/>
2 – end position<br/>

{% highlight sql %}
begin
    DBMS_REDACT.DROP_POLICY('SH','SALES','SALES_TIME_FULL');
    DBMS_REDACT.ADD_POLICY (
       object_schema  => 'SH',
       object_name    => 'SALES',
       policy_name    => 'SALES_AMOUNT_PARTIAL',
       function_parameters => '9,1,2',
       column_name    => 'AMOUNT_SOLD',
       function_type  => DBMS_REDACT.PARTIAL,
       expression     => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
{% endhighlight %}

Result:

{% highlight sql %}
SQL> select distinct amount_sold from sh.sales fetch first 5 rows only;

AMOUNT_SOLD
-----------
    9932.16
     9964.4
    9939.99
    9981.09
    9948.76
{% endhighlight %}

* Dates

For dates partial redact, function_parameters requires specific format. Fields used for formatting are M (m#), D (d#), Y(y#), H(h#), M(m#), S(s#) where:

M – month (skip redaction)<br/>
m# – replace month with the value defined for #. (# can be between 1 and 12)<br/>
D – day (skip redaction)<br/>
d# – replace day with the value defined for #. (# can be between 1 and 31)<br/>
Y – year (skip redaction)<br/>
y# – replace year with the value defined for #. (# can be between 1 and 9999)<br/>
H – hour (skip redaction)<br/>
h# – replace hours with the value defined for #. (# can be between 0 and 23)<br/>
M – minute (skip redaction)<br/>
m# – replace minutes with the value defined for #. (# can be between 0 and 59)<br/>
S – second (skip redaction)<br/>
s# – replace seconds with the value defined for #. (# can be between 0 and 59)<br/>

If we want to redact the year we have to specify y# where # specifies the year we want our policy to use as replacement. For example, in order to redact the following date 21.12.2014 to 21.12.2000 we would set function_parameters parameter as ‘MDy2000’.

Policy definition:

{% highlight sql %}
begin
 DBMS_REDACT.DROP_POLICY('SH','SALES','SALES_AMOUNT_PARTIAL');
 DBMS_REDACT.ADD_POLICY (
     object_schema  => 'SH',
     object_name    => 'SALES',
     policy_name    => 'SALES_TIME_ID_FULL',
     column_name    => 'TIME_ID',
     function_type  => DBMS_REDACT.PARTIAL,
     function_parameters => 'MDy2000HMS',
     expression     => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
 end;
{% endhighlight %}

Result:

{% highlight sql %}
SQL> select distinct to_char(time_id,'yyyy') time_id_year from sh.sales;

TIME
----
2000
SQL>
SQL> select count(*) from sh.sales;

  COUNT(*)
----------
    918843
{% endhighlight %}

The year was redacted to 2000 for all 918843 rows.

**RANDOM REDACT**

Random redaction is simple to set (no need for specific format), something like FULL redaction just with different value for function_type.

* Numbers

{% highlight sql %}
begin
     DBMS_REDACT.DROP_POLICY('SH','SALES','SALES_AMOUNT_PARTIAL');
     DBMS_REDACT.ADD_POLICY (
        object_schema  => 'SH',
        object_name    => 'SALES',
        policy_name    => 'SALES_AMOUNT_RANDOM',
        column_name    => 'AMOUNT_SOLD',
        function_type  => DBMS_REDACT.RANDOM,
        expression     => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
{% endhighlight %}

Result:

{% highlight sql %}
SQL> select distinct amount_sold from sh.sales order by 1 fetch first 5 rows only;

AMOUNT_SOLD
-----------
          1
       2.84
       5.79
       5.78
       2.08

SQL> /

AMOUNT_SOLD
-----------
        1.8
       3.02
        .66
       5.28
        .13
{% endhighlight %}

We are getting different values for every execution.

* Dates

Same is for dates.

Policy definition:

{% highlight sql %}
begin
     DBMS_REDACT.DROP_POLICY('SH','SALES','SALES_AMOUNT_RANDOM');
     DBMS_REDACT.ADD_POLICY (
        object_schema  => 'SH',
        object_name    => 'SALES',
        policy_name    => 'SALES_TIME_RANDOM',
        column_name    => 'TIME_ID',
        function_type  => DBMS_REDACT.RANDOM,
        expression     => 'SYS_CONTEXT(''USERENV'',''CURRENT_USER'') = ''IARSOV''');
end;
{% endhighlight %}

Result:

{% highlight sql %}
SQL> conn iarsov@xdb1
Enter password:
Connected.
SQL>
SQL> select to_char(time_id,'dd.mm.yyyy') time_id from sh.sales fetch first 5 rows only;

TIME_ID
----------
20.01.0101
27.12.0206
11.06.0216
12.03.0031
07.01.0198

SQL> /

TIME_ID
----------
16.12.0164
02.04.0232
20.08.0154
25.06.0047
13.05.0051
{% endhighlight %}

As you can see the redaction kicked in and we received strange dates (specially for the year).

**REGEXP REDACT**

* Numbers

not supported data type.

* Dates

not supported data type.