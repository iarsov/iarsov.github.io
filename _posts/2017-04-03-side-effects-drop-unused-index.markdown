---
layout: post
title:  "The Side Effects Of Drop 'UNUSED' Index"
date:   2017-04-03
categories: oracle
---

Few weeks ago I’ve published a post on Pythian’s blog regarding a scenario where dropping a potentially ‘unused’ index can have a negative influence on the optimizer’s cardinality estimation.

I will post here the introduction only, you can find link to the full post at the bottom.
<hr/>

_THE SIDE EFFECTS OF DROP 'UNUSED' INDEX_

In this blog post I’ll demonstrate a scenario where dropping a potentially ‘unused’ index can have a negative influence on the optimizer’s cardinality estimation. Having columns with logical correlation and skewed data can cause difficulties for the optimizer when calculating cardinality. This post will also address what we can do to help the optimizer with better cardinality estimates.

The inspiration for this post was derived from a recent index usage review. One of the requirements was to determine how to confirm which unused indexes qualify to be dropped. I decided to do some tests regarding extended statistics and the effect of potentially dropping an ‘unused’ index. You will observe what kind of result may be seen from the drop of an index which has not been used. It’s important to remember that it does not apply in all cases. Occasionally, even if the index is used, it doesn’t mean that it’s needed.

This is more or less linked to columns with skewed data and which might have logical relationship.
Hopefully, it can help you answer some of the following questions:

Is the optimizer using the indexes behind scenes?
While there are methods to determine if an index has been used in an execution plan, can an index be dropped on this basis only?
If we drop composite index (constructed from correlated columns), can we do anything to avoid performance degradation?
Before we start with the use case, let’s briefly review some concepts.

The basic formula for selectivity is 1/NDV. The cardinality (CDN) is calculated as selectivity * total number of rows.

The selectivity of a join is defined as the selectivity of the most selective join column adjusted by the proportion of not null values in each join column.

{% highlight sql %}
Join Selectivity:
Sel = 1/max[NDV(t1.c1),NDV(t2.c2)] *
( (Card t1 - # t1.c1 NULLs) / Card t1) *
( (Card t2 - # t2.c2 NULLs) / Card t2)

Join Cardinality:
Card(Pj) = Card(T1) * Card(T2) * Sel(Pj)
{% endhighlight %}

In Oracle’s Doc ID 68992.1 you can find a more detailed explanation about different selectivity calculations based on different predicates. For simplicity, I will use equality predicate.

This blog post is divided in three sections.

Use case where we demonstrate how drop of an “unused” index can mess up optimizer cardinality calculation.
How to help optimizer for better cardinality estimation with extended statistics.
More explanation on column correlation (CorStregth).

The full post you can read [here](https://www.pythian.com/blog/drop-unused-index-side-effects){:target="_blank"}.
