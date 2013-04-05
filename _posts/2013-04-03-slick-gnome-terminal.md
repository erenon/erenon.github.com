---
layout: post
title: "Slick Gnome Terminal"
description: "How to create a good looking gnome-terminal"
category: hack
tags: [hack, desktop, ubuntu]
---
{% include JB/setup %}

So, you want me to belive, the default gnome-terminal in ubuntu doesn't look _that_ good. Here it is:

<img 
    style="margin: 1em auto;" 
    src="https://lh5.googleusercontent.com/-smSXpT9_WQU/UV9LWIz8soI/AAAAAAAAAm0/I6yZEBqDgL4/s696/01.png" 
    title="gnome-terminal default look and feel in Ubuntu 12.10" 
    alt="gnome-terminal default look and feel in Ubuntu 12.10"
/>

I aggree. Fortunately, there is a way to make it look more _slick_.

Remove window borders
===

To remove window borders, install (if not yet installed) and launch <abbr title="Compiz Config Settings Manager">ccsm</abbr>.

{% highlight bash linenos %}
$ sudo apt-get install compizconfig-settings-manager
$ ccsm &
{% endhighlight %}

In **ccsm**, open the `window decorations` settings window and set `Decoration windows:` preference to:

{% highlight bash %}
(any) & !(class=Gnome-terminal)
{% endhighlight %}

This will remove window decorations (e.g: window borders) from all gnome-terminal windows. Here is how it looks now:

<img 
    style="margin: 1em auto;" 
    src="https://lh3.googleusercontent.com/-_qtLqny_YyE/UV9LWSRh2dI/AAAAAAAAAm4/jteGIWJQimQ/s696/02.png" 
    title="gnome-terminal without borders" 
    alt="gnome-terminal without borders"
/>

Adjust colors
===

Finally change the color theme and add a bit of transparency. Create a new gnome-terminal profile (`edit` / `Profiles...`) and tailor it to your needs. Find a sample setting below:

<img 
    style="margin: 1em auto;" 
    src="https://lh4.googleusercontent.com/-AYiacF9Rc6k/UV9LcDdTjzI/AAAAAAAAAnI/FWt3iVcWOPU/s696/03.png" 
    title="transparent gnome-terminal without borders and settings window" 
    alt="transparent gnome-terminal without borders and settings window"
/>

Such a nice terminal deserves a colored promt. Edit your `~/.bashrc` file, locate the colored `PS1` definition and change it to something more cool:

{% highlight bash %}
PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
{% endhighlight %}

Final version:

<img 
    style="margin: 1em auto;" 
    src="https://lh5.googleusercontent.com/-IpSKVCkWilA/UV9Lbmk-DaI/AAAAAAAAAnA/cDZkRjTaNxc/s696/04.png" 
    title="transparent gnome-terminal without borders, with colored promt" 
    alt="transparent gnome-terminal without borders, with colored promt"
/>

By the end of all this tweaks, you might change that <del>ugly</del> default wallpaper as well.
