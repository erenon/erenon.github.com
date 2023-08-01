---
title: "Selfhosted RSS Reader"
date: 2023-08-01
---

For some time I was using a simple [RSS][] reader on my phone.
However, I wanted to:

 - Read the same set of feeds on multiple devices
 - Archive the scraped content
 - Search the archived content

So I looked around for different options.
I ruled out proprietary options, fearing their end of life (e.g: Google Reader)
or gradual degradation - I wanted something stable.

The chosen architecture consists of (1) a server component, that periodically
scrapes and archives the configured feeds, exports the content to authorized clients,
and tracks read status (So if I read an article on one device, it'll appear as
read on all the connected devices), and (2) client programs on different devices with screens.

For the server side, I chose [miniflux][]. It is easy to deploy (requires PostgreSQL),
sufficiently fast on a Raspberry Pi4 (written in Go), and provides a pleasingly simple
web interface, for desktop use. Note that miniflux offers a [paid, hosted service][minipay].

For mobile, there are several options with different qualities.
On Android, I found [FeedMe] really good. Enable the Fever API in miniflux
(Settings/Integrations/Fever) to make them connect.

On iOS, [NetNewsWire][] is an absolute treat. Enable the Google Reader API in
miniflux, and use the FreshRSS option in the client to make them connect.

All of these components (miniflux, FeedMe, NetNewsWire) can also get the original HTML
content and convert it to simplified text (with images), in case the content in the RSS
feed is missing or incomplete. A non-trivial, and really useful feature.

The server runs on a network that doesn't have a public IP address.
I made it available to other devices using [tailscale][], that is
also warmly recommended, and surprisingly easy to set up.
It uses [various tricks][tailscale-how] to bypass NAT based firewalls,
and works in every scenario I tested so far.

The result is a securely available, robust, fast, clean news feed.

## Considered Alternatives

 - FreshRSS: I didn't like the HTML interface
 - Microflux: Slow and confusing UI
 - Miniflutt: Does not store content locally, always needs to re-fetch everything on start (no offline mode)

[RSS]: https://en.wikipedia.org/wiki/RSS
[miniflux]: https://miniflux.app/
[minipay]: https://miniflux.app/hosting.html
[FeedMe]: https://play.google.com/store/apps/details?id=com.seazon.feedme&hl=en&gl=US
[NetNewsWire]: https://netnewswire.com/
[tailscale]: https://tailscale.com/
[tailscale-how]: https://tailscale.com/blog/how-nat-traversal-works/
