---
layout: post
title: "Neptun Keepalive"
description: "How to keep alive your neptun session"
category: blog
tags: [neptun, hack]
---

Közeleg a tárgyfelvétel, és senkinek nincs ideje órákat az F5-ön ülni. Egy lehetséges megoldás,
hogy fenntartsuk a sessiont, ha a kezdés előtt néhány órával sikerül bejelentkezni:

{% highlight js linenos %}
javascript:(
  function(){
      window.setInterval(
          function(){
            $('#upBoxes_upRSS_gdgRSS_gdgRSS_RefreshBtn')
            .click();
          }, 
          20000
      );
  }()
);
{% endhighlight %}

How to use
===

 - Bookmarkoljuk a [Keep Alive][1] bookmarkletet.
 - Jelentkezzünk be a neptunba
 - Kattintsunk az előbb felvett könyvjelzőre
 
A script 20 másodpercenként megnyom egy gombot, ami a tétlenségi idő újraindulását eredményezi.

_Korlátozott körökben terjesztendő_

[1]: javascript:(function(){window.setInterval(function(){$('#upBoxes_upRSS_gdgRSS_gdgRSS_RefreshBtn').click();},20000);}());
