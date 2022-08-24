---
layout: post
title:  "What is web minimalism?"
date:   2022-08-22 20:00:00 +0100
categories: frontend minimalism
author: "Darren Foley"
description: "Post discussing the perils of modern website UI."
tags: ["Web", "Frontend", "Minimalism", "UX"]
hero_darken: true
---

<p><b>Web Minimalism</b> is a design principal which focuses on the simplification of web user interfaces. Its an approach to website UI design that places emphasis on clarity, purpose, functionality and intentionality over adding features that provide little benefit. Everything else is redundant; popups, trackers, banners or anything else that could increase cognitive overhead for end users. 
</p>
<p>
In addition to visual bloat, modern websites are growing in size due to extra javascript logic and image files. Website designers have become more reliant on end users with high bandwidth internet connection which is more common but not given, and the use of Content Delivery Networks (CDNs) to improve performance. This is becoming easier now with numerious cloud CDN services and providers. However, page size should not be neglected at the expense of functionality which provide little to no value.
</p>
<p>
Given that a large proportion of the web is serving static content e.g. browsing products, recipes, news articles and so on, a large proportion of the modern web could be radically simplified. This is best shown visually, so I've included some examples below to explain what I mean along with some principals of UX that I think are very important.
</p>

<br>

### Bloated Website Example

<p><b>Example</b>: <a href="https://www.harveynorman.ie/">Harvey Norman</a></p>

<p><b>Business Type</b>: Electronics and furniture retailer.</p>

<p>Home page</p>

![HN_Site1](/images/harvey_norman_2.png)

<br>

<p>Product Page</p>

![HN_Site2](/images/harvey_norman_3.png)

<br>

<p>First off, visually this website throws a lot at you; multiple color schemes, Banners, multiple navigation bars, popup windows etc. None of this is really what the customer is looking for. In most cases the customer just wants to browse products, which is really just static content, images and text.</p>

<br>

<p>Performance wise, including images and javascript files etc. the site is around the ~10.9MB mark and takes approximately 1.8 seconds for DOM ready and about 3.32 seconds for the page to be fully rendered. - Thats with no cached files measuring from an incognito browser and with high bandwidth wifi connection approx 30MB/sec.</p>

![PagePerformance](/images/performance_hn.png)

<p>Lets compare that to a minimal website.</p>

<br>

### Minimalist Websites

<p>Example 2: <a href="https://tinkerwatches.com/">Tinker Watches</a>
</p>

<p>Business Type: Custom Wrist Watch brand.</p>

<br>

<p>Home Page</p>

![tinkerSite](/images/tinker_site_1.png)

<p>Product Page</p>

![tinkerSite2](/images/tinker_site_2.png)

<br>

<p>First off, this is a noticably superior UI design. A simple color scheme that doesn't overload the eyes along with a intuitive navigation side bar that really makes things easy for the customer. As for the products page itself, again a very simple design, showing images and text for browsing and no unecessary popups or banners.</p>

<br>

![tinkerSite2](/images/tinker_site_perf.png)

<p>In terms of performance, this is a much smaller site than the first example at around 3.1MB (still quite big). DOM takes 1 second with full load around 1.9 seconds.</p>

<br>

### Summary

<p>You could argue that this was not a fair comparison due to differences in business model, company size and product range between sites. However, its clear that the tinker watch site adheres to minimalist principals much better than the harvey Norman web site, both in terms of visual minimalism and performance. </p>

