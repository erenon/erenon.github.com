---
layout: post
title: "Bootstrap Typeahead submit on select"
description: "A javascript snippet on autosubmitting form using bootstrap typeahead"
category: development
tags: [javascript, bootstrap]
---
{% include JB/setup %}

The [Twitter Bootstrap][1]s [Typeahead][2] is a very nice way to provide autocomplete functionality on your text inputs. However, the default configuration might be a bit confusing. When the user clicks on a suggestion in the dropdown menu, the utility populates the input but doesn't submit the form. It's usually ok, but sometimes (e.g: search boxes) it's frustrating. Here's how to change it:

{% highlight js linenos %}
var input = $('#your-input-box');
input.typeahead({
    'source' : ['foo', 'bar', 'baz'],
    'updater' : function(item) {
        this.$element[0].value = item;
        this.$element[0].form.submit();
        return item;
    }
});
{% endhighlight %}

This snippet autosubmits the form if the user clicks on a suggestion or selects it by keyboard.

[1]: http://twitter.github.com/bootstrap/index.html
[2]: http://twitter.github.com/bootstrap/javascript.html#typeahead
