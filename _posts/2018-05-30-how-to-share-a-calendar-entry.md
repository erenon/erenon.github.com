---
layout: post
title: "How to share a calendar entry"
description: "Sharing a calendar event over the internet in a platform agnostic way"
category: blog
tags: [javascript, www]
---

Suppose you'd like to share a calendar event over the internet,
e.g: on your website. The goal is to be support as many platforms as possible:

 - Desktop calendars (Thunderbird, macOS iCal, Outlook)
 - Web applications (Google Calendar)
 - Smartphone applications (Google Calendar App, iOS iCal)

The most intuitive way is to create the event in your calendar of
choice, and send the invites in the native way provided by the application.
However, such methods tend to work only if the recipient uses the same
system, which isn't a plausible assumption -- we need something platform independent.

### Using iCalendar

Wikipedia indicates that [iCalendar][] is well supported format, which
_allows Internet users to send meeting requests and tasks to other Internet users by sharing or sending files in this format through various methods._
-- exactly what we are looking for. Let's create an `.ics` file:

{% highlight properties %}
BEGIN:VCALENDAR
VERSION:2.0
METHOD:PUBLISH
X-WR-TIMEZONE:Europe/Budapest
BEGIN:VEVENT
UID:your_eventname@yourdomain.com
DTSTART:20180530T130000Z
DTEND:20180530T143000Z
DESCRIPTION:The description of your event
LOCATION:The location of your event
SEQUENCE:0
SUMMARY:The summary of your event
END:VEVENT
END:VCALENDAR
{% endhighlight %}

There are [other fields][ics_rfc].
The fields are mostly self describing.
In free form text values, `,` and `;` characters must be escaped by a backslash.
The `tzselect` command can be used to get the name of your timezone.
The `DTSTART` and `DTEND` fields are in [UTC][], 20180530T130000 is 2018. 05. 30, 13:00:00.
If all this sound difficult, create your event in a suitable calendar application
and export it to `.ics` from there.

Sharing this file will cover desktop calendars associated with the ics extension
and the iOS calendar. It probably will not work with the Google Calendar website,
and -- surprisingly -- it will not work on Android as expected.

### Fixing Android

On Android, after download, a toast message will be shown, saying:
"Unable to launch event". Internet wisdom blames various components.
On the phones I had access to, Logcat show that the Calendar app does not
have permission to read downloaded files.

Fortunately, there's a way around: Android allows [emitting intents by links][3], in some cases.
The proper syntax and the allowed fields can be discovered by looking at the [source][GCAM],
or [StackOverflow][5].
Here's an example:

{% highlight html %}
<a href="https://www.google.com/calendar/event?action=TEMPLATE&text=your_event&dates=20180530T130000Z/20180530T143000Z&location=location of the event&sprop=yourdomain.com">Calendar event</a>
{% endhighlight %}

Values need to be [urlencoded][4].
Clicking on this link on Android will open the Google Calendar application, if installed,
with the specified event shown to be saved. Fortunately, this link will work for the
browser based Google Calendar as well!

### Solution

Let's put all this together. By default, we should link to the `.ics` file,
as it works on most platforms. On Android, the link has to be changed:

{% highlight html %}
<a href="event.ics" data-gcal="https://www.google.com/calendar/event?..." class="ics-link">Calendar event</a>
{% endhighlight %}

{% highlight js %}
var replaceIcs = function() {
  var as = $('.ics-link');
  for (var i = 0; i < as.length; i++) {
    as[i].href = as[i].getAttribute('data-gcal');
  }
};

document.addEventListener("DOMContentLoaded", function(e) {
  // NB: there are more robust ways to check for android
  var userAgent = navigator.userAgent.toLowerCase();
  if (userAgent.indexOf("android") >= 0) {
    replaceIcs();
  }
});
{% endhighlight %}

What about desktop clients using the Google Calendar web application?
It is difficult if possible to tell what kind of calendar solution such users would like to use.
However, there's a hack, which detects if the [user is logged in to a google account][gahack].
If desired, as an imperfect solution, ics links can be also replaced, if such a login is detected.

### Summary

It is more difficult to share a calendar entry over the internet as expected.
Application native ways do not interoperate, while some platforms are hard to target by standard solutions.
While the solution shown above covers a large percentage of use-cases, it still might fail in others.
A free web needs more support of open standards!

[iCalendar]: https://en.wikipedia.org/wiki/ICalendar
[ics_rfc]: https://tools.ietf.org/html/rfc5545
[UTC]: https://en.wikipedia.org/wiki/Coordinated_Universal_Time
[3]: https://stackoverflow.com/questions/42936576/open-android-calendar-with-intent-from-web-chrome
[GCAM]: https://android.googlesource.com/platform/packages/apps/Calendar/+/donut-release/AndroidManifest.xml
[4]: https://en.wikipedia.org/wiki/Percent-encoding
[5]: https://stackoverflow.com/questions/22757908/google-calendar-render-action-template-parameter-documentation
[gahack]: http://www.tomanthony.co.uk/blog/detect-visitor-social-networks/
