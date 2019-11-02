---
layout: post
title:  "View Merging And DBMS_RADNOM Anomaly"
date:   2016-02-07
categories: oracle
---

I have a function called generate_random_number which returns random number if it’s called without parameters. If parameter value is specified it returns the same value as the input parameter. The random number that is returned is build with dbms_random and it can be any number between 0 and 10.

{% highlight sql %}
create or replace function generate_random_number(p_num1 number)
return number
is
v_result number;
begin

if p_num1 is null then
v_result := dbms_random.value(0,10);
else
v_result := p_num1;
end if;

return round(v_result);

end generate_random_number;
{% endhighlight %}

If I run the following statement where I’m trying to build random value by calling generate_random_number and using it as projection in the upper query block, I would get inconsistent results. Notice, how I am using the value from the inner block in projection, _number1_ and _number1_ as number2.

{% highlight sql %}
select  number1 as number1
,number1 as number2
from    (select generate_random_number(null) number1 from dual)
/

NUMBER1    NUMBER2
---------- ----------
5          8

NUMBER1    NUMBER2
---------- ----------
7         10

NUMBER1    NUMBER2
---------- ----------
7          9
{% endhighlight %}

Same “value” different results, interesting. Let see the execution plan, +outlines so we can see what the optimizer is trying to accomplish.

{% highlight sql %}
SQL_ID  a642huv84jmur, child number 0
-------------------------------------
select  number1 as number1         ,number1 as number2 from    (select
generate_random_number(null) number1 from dual)

Plan hash value: 1388734953

-----------------------------------------------------------------
| Id  | Operation        | Name | Rows  | Cost (%CPU)| Time     |
-----------------------------------------------------------------
|   0 | SELECT STATEMENT |      |       |     2 (100)|          |
|   1 |  FAST DUAL       |      |     1 |     2   (0)| 00:00:01 |
-----------------------------------------------------------------

Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------

1 - SEL$F5BB74E1 / DUAL@SEL$2

Outline Data
-------------

/*+
BEGIN_OUTLINE_DATA
IGNORE_OPTIM_EMBEDDED_HINTS
OPTIMIZER_FEATURES_ENABLE('10.2.0.5')
DB_VERSION('12.1.0.2')
ALL_ROWS
OUTLINE_LEAF(@"SEL$F5BB74E1")
MERGE(@"SEL$2")
OUTLINE(@"SEL$1")
OUTLINE(@"SEL$2")
END_OUTLINE_DATA
*/
{% endhighlight %}

Notice the MERGE(@”SEL$2″), oracle is clever enough to detect that inner result is used in the upper query block and it performs simple view merging. But, what it doesn’t know is that the function generate_random_number uses dbms_random package and therefore might return different value for each call.

Here is the final plan used by the optimizer after query transformation. This information is produced with dbms_sqldiag.dump_trace.

{% highlight sql %}
Final query after transformations:******* UNPARSED QUERY IS *******

SELECT "HR"."GENERATE_RANDOM_NUMBER"(NULL) "NUMBER1"
,"HR"."GENERATE_RANDOM_NUMBER"(NULL) "NUMBER2"
FROM    "SYS"."DUAL" "DUAL"
{% endhighlight %}

This explains from where the different (inconsistent) values are coming. The generate_random_number is called twice therefore we get two different values.

***Solution***:

There are different workarounds for this behavior. You can put DISTINCT in the inner block to restrict the optimizer from doing view merging transformation. It will do the job for this particular example since I am generating only one row.

{% highlight sql %}
select  number1 as number1
,number1 as number2
from    (select distinct generate_random_number(null) number1 from dual)
/

NUMBER1    NUMBER2
---------- ----------
9          9

NUMBER1    NUMBER2
---------- ----------
1          1

NUMBER1    NUMBER2
---------- ----------
3          3
{% endhighlight %}

If you are not allowed to modify the SQL statement, set “_simple_view_merging” = false at session level.

{% highlight sql %}
alter session set "_simple_view_merging" = false;

select  number1 as number1
,number1 as number2
from    (select generate_random_number(null) number1 from dual)
/

NUMBER1    NUMBER2
---------- ----------
6          6

NUMBER1    NUMBER2
---------- ----------
0          0

NUMBER1    NUMBER2
---------- ----------
5          5
{% endhighlight %}

You can also hint the inner query block statement with /*+ NO_MERGE */.