---
layout: post
title: "Symfony2: Huge transactions cause memory leak"
description: ""
category: development
tags: [symfony2, doctrine2, php, memoryleak]
---
{% include JB/setup %}

...or they seem like to cause one -- I noticed while I was testing the latest [ComPPI][1] build. As it turned out, the continuously
increasing memory consumption was caused by the built in logger component. The following snippet takes care of it:

{% highlight php linenos %}
<?php
use Doctrine\ORM\EntityManager;

/**
 * Disables SQL logging.
 *
 * Call this before the first large transaction
 */
function initEntityManager(EntityManager $em) {
    $em->connection->getConfiguration()->setSQLLogger(null);
}
{% endhighlight %}

[1]: https://github.com/erenon/ComPPI
