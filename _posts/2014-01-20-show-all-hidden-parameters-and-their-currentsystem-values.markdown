---
layout: post
title:  "Show All Hidden Parameters And Their Current/System Values"
date:   2014-01-20
categories: oracle
---

Here is way to list all hidden parameters and their current session/system values.
You should be careful when changing these parameters and is highly recommended to make change only after consultation with Oracle Support.

{% highlight sql %}
set lines 150
set pages 200
col parameter_name for a30
col parameter_desc for a80
col param_sess_value for a10
col param_system_value for a10

select t.indx,
       t.ksppinm parameter_name,
       t.ksppdesc parameter_desc,
       t1.ksppstvl param_sess_value,
       t2.ksppstvl param_system_value
from   x$ksppi t,
       x$ksppcv t1,
       x$ksppsv t2
where  t.indx = t1.indx
       and t.indx = t2.indx
       and t.ksppinm like '\_%' escape '\'
order by t.indx
/
{% endhighlight %}