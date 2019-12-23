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

   A----------B
--------|------------|---------
        C            D
case 2:

                A----------B
--------|------------|---------
        C            D
case 3:

     A-----------------B
--------|-----------|---------
        C           D
case 4:

          A-------B
--------|-----------|---------
        C           D
{% endhighlight %}

One way we can determine the overlapping periods is if we use OR conditions for all cases, e.g.:

{% highlight shell %}
(start_date_C between start_date_A and end_date_B) -- cases 1,3
OR
(end_date_D between start_date_A and end_date_B) -- cases 2,3
OR
(start_date_A between start_date_C and start_date_D) -- case 4
 {% endhighlight %}

If you are familiar with De Morgan’s laws (from school 🙂 ) then:

{% highlight shell %}
“not (A and B)” is the same as “(not A) or (not B)”
also,
“not (A or B)” is the same as “(not A) and (not B)”.
{% endhighlight %}

With this implemented, we can get the following condition:

{% highlight shell %}
(start_date_A <= end_date_D AND end_date_B >= start_date_C)
{% endhighlight %}
