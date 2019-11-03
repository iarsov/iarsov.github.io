---
layout: post
title:  "Blog Migrated to GitHub Pages"
date:   2019-11-03
categories: general
---

I have decided to ditch Wordpress and all activities that come with it (plugins, scripts, hosting maintenance etc.) and to move to GitHub Pages.

GitHub Pages offer very easy, fast and clean environment if you have blog similar to mine where you mainly post text based (_with few images now and then_) posts.

I will explain the steps I took to configure Jekyll and setup GitHub Pages.

* You need to have GitHub account. If you don't have one you can create at [GitHub](https://github.com){:target="_blank"}
* Create public repository named _username_.github.io
    * In general the repository needs to be public. I have read that you can have your repository private if you have paid plan, but I haven't tried it.
* Go to [Jekyll](https://jekyllrb.com/docs/installation){:target="_blank"} installation page to learn how to install Jekyll and to get familiar with the framework.
* Once you have Jekyll installed (I installed in on an Ubuntu VM) you need to create your site structure with:

{% highlight bash %}
jekyll new myblog
{% endhighlight %}

* Push the folder contents to the GitHub repository you created in step #2. I use [GitHub Desktop](https://desktop.github.com){:target="_blank"} on daily basis to push/pull changes to/from GitHub.
* Open your favourite browser and go to _username_.github.io

Now, you should be able to access your page via _youraccount_.github.io

* By default GitHub Pages uses _minima_ theme (that's what I use as well).

**Custom domain**

If you want to use custom domain for you site, like I do, then you need to let GitHub know what domain/subdomain you want to use and update your DNS settings.

Provide your domain/subdomain to GitHub Pages via _your\_repository_ -> Settings -> GitHub Pages -> _Custom domain_. This will create CNAME file in the root directory and it will store the custom domain.

At this point you won't be able to _Enforce HTTPS_

***Apex domain***

If you want to use an Apex domain for you site (i.e. no subdomain) then you need to put an A record in your DNS configuraiton to point the IP addresses for GitHub Pages.

More you can read at [Configuring an apex domain](https://help.github.com/en/github/working-with-github-pages/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain){:target="_blank"}

***Subdomain***

If, like me, you want to use subdomain for your site then you just need to create CNAME record in your DNS configuration to point to _youraccount_.github.io

More you can read at [Configuring a subdomain](https://help.github.com/en/github/working-with-github-pages/managing-a-custom-domain-for-your-github-pages-site#configuring-a-subdomain){:target="_blank"}

Once you have properly configured Apex domain or subdomain you can enforce HTTPS for your site. To do that you need to go to _your\_repository_ -> Settings -> GitHub Pages -> _Enforce HTTPS_

**Posts migration**

The migration of the posts was done manually. I did not bother to automate that part of the process. It also gave me a chance to clean some of the posts.

**Comments**

For comments I decided to use Twitter. On each post (_at the end of the post_) there is a link _comment_ which if clicked will redirect you to Twitter. The link contains pre-defined text (the post url) which gets included into the _compose tweet box_. The downside is that you need to be already logged in to Twitter in order for the link to work. It seems that Twitter will not redirect to login page if you're not singed in.

You can integrate Disqus comments but since I don't have an account there and I want to simplify the things I decided to go with Twitter.

**How do I create blog posts**

Now, in order to publish a blog post I simply open my favourite editor (Visual Studio Code) write my post in Markdown (and/or HTML) syntax and commit to GitHub.

To create a post, add a file to your _posts directory with the following format:

{% highlight liquid %}
YEAR-MONTH-DAY-title.MARKUP
{% endhighlight %}

Each post needs to have metadata definition at the top defined as:

{% highlight liquid %}{% include blog_migrate_liquid_snippet.html %}{% endhighlight %}

You can read more at [Posts](https://jekyllrb.com/docs/posts){:target="_blank"}. 

For example, to format code blocks you use Liquid tags ( without the \\ ):

{% highlight liquid %}
{\% highlight sql \%}
_code_
{\% endhighlight \%}
{% endhighlight %}

That's it. The site will automatically build itself and publish the post. In case of an errors you'll receive _Page build failure_ email with details on what line it failed. You'll have to fix the error and commit again (which will trigger new build). In case of _Page build failure_ you're site will be still available! You just won't see the new changes at the build failed.