---
layout: post
title:  "Identify Timed-Out SQL From SQL Tuning Task"
date:   2016-12-05
categories: oracle
---

In this post I would like to share a quick way of identifying the timed-out SQLs from a SQL tuning task report.

Usually you would do that from EM Express (12c) or Cloud control. But, having an access to EM/Cloud control is not always possible and you might be stuck in identifying the correct statements.

Here is an example, the portion of the SQL Auto Tuning Task report which shows how many statements have timed-out.

{% highlight sql %}
...
-----------------------------------------------------------
Global SQL Tuning Result Statistics
-----------------------------------------------------------
Number of SQLs Analyzed                      : 130
Number of SQLs in the Report                 : 23
Number of SQLs with Findings                 : 22
Number of SQLs with Statistic Findings       : 2
Number of SQLs with Alternative Plan Findings: 5
Number of SQLs with SQL profiles recommended : 14
Number of SQLs with Index Findings           : 11
Number of SQLs with SQL Restructure Findings : 3
Number of SQLs with Timeouts                 : 2
Number of SQLs with Errors                   : 1
...
{% endhighlight %}

I've extracted the following SQL from the EM Express – SQL Tuning Advisor report (page).
The original statement is much bigger, but the following portion should be good enough to give you the statements which have timed-out.

All you need to do is to specify the execution name. It has been tested on 11.2 and 12.1.

{% highlight sql %}
SELECT oe.*,
f.id finding_id,
f.type finding_type,
f.flags finding_flags
FROM
(SELECT
o.exec_name ,
o.id object_id,
o.attr1 sql_id,
o.attr3 parsing_schema,
to_number(NVL(o.attr5, '0')) phv,
NVL(o.attr8,0) obj_attr8
FROM  wri$_adv_objects o
WHERE o.exec_name = 'exectuion_name'
AND o.type      = 7
) oe,
wri$_adv_findings f
WHERE  f.exec_name = oe.exec_name
AND f.obj_id = oe.object_id
AND type = 3
AND bitand(flags,2) <> 0
/
{% endhighlight %}