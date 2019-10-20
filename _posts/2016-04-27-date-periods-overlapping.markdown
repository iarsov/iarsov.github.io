---
layout: post
title:  "Date Periods Overlapping"
date:   2016-04-27
categories: oracle
---

I found this sitting in drafts, short post regarding period overlapping.
With simple logic we can eliminate the OR conditions related to all possible combinations.

The post:

The possible four cases where period overlap can occur are:

{% highlight shell %}
case 1:

----------
--------|------------|---------

case 2:

----------
--------|------------|---------

case 3:

--------------------
--------|-----------|---------

case 4:

-------
--------|-----------|---------
{% endhighlight %}

One way we can determine the overlapping periods is if we use OR conditions for all cases, e.g.:

{% highlight shell %}
(start_date_1 between start_date_2 and end_date_2) -- case 2/case_3
OR
(end_date_1 between start_date_2 and end_date_2) -- case 1/case 3
OR
(start_date_2 between start_date_1 and start_date_2) -- case 4
 {% endhighlight %}

If you are familiar with De Morgan’s laws (from school 🙂 ) then:

{% highlight shell %}
“not (A and B)” is the same as “(not A) or (not B)”
also,
“not (A or B)” is the same as “(not A) and (not B)”.
{% endhighlight %}

With this implemented, we can get the following condition:

{% highlight shell %}
(start_date1 <= end_date2 AND end_date1 >= start_date2)
{% endhighlight %}