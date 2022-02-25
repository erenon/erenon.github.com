---
title: "Snapcast on Demand"
date: 2022-02-21
---

[Snapcast][] is a wonderful piece of software: a multiroom client-server audio
player, where all clients are time synchronized with the server to play
perfectly synced audio. It allows you to play the same music
in different rooms, at the same time, using multiple devices.

Normally, I listen to music on my desktop. Sometimes, however,
I'd like to broadcast music to another room. In that room, there's
a raspberry pi connected to a set of speakers.
I wanted to make the switch between local and multiroom playback
as easy as possible. This post describes my approach.

Normally, my [music player][cmus] uses the pulse-audio interface
to interact with the audio system (PipeWire, currently) that uses
ALSA to talk to the actual device:

```
cmus -> pulse -> pipewire -> alsa -> device
```

We are going to add a virtual sink to pulse audio, a file sink,
where the Snapcast server will pick up the music from:

```
cmus -> pulse -> file -> snapcast server -> snapcast client -> alsa -> device
```

To this chain, we can add multiple snapcast clients, to achieve a multiroom sound system.
It is important that the snapcast client should not use the pulse audio interface,
to avoid infinite loops. Read more about these pieces in this [post from badaix][b1].
By changing the active pulse sink, we can switch between local and multiroom audio.
The advantage of this setup is that it is player agnostic: it works with
every music player that can talk to pulse, and requires no additional player configuration.

Theory aside, let's get to work.
First, build snapcast from [AUR][], with a change to minimize dependencies:

```diff
diff --git a/PKGBUILD b/PKGBUILD
index e5033a8..5efe228 100644
--- a/PKGBUILD
+++ b/PKGBUILD
@@ -7,7 +7,7 @@ pkgdesc="Synchronous multi-room audio player"
 arch=('x86_64' 'armv6h' 'armv7h' 'aarch64')
 url="https://github.com/badaix/snapcast"
 license=('GPL')
-depends=(alsa-lib avahi libvorbis flac opus expat libsoxr libpulse)
+depends=(alsa-lib)
 makedepends=(cmake alsa-utils boost)
 install="snapcast.install"
 backup=('etc/default/snapserver' 'etc/default/snapclient' 'etc/snapserver.conf')
@@ -25,6 +25,12 @@ build() {
     cmake -B build -S . \
           -DCMAKE_BUILD_TYPE=None \
           -DCMAKE_INSTALL_PREFIX=/usr \
+          -DBUILD_WITH_FLAC=Off \
+          -DBUILD_WITH_VORBIS=Off \
+          -DBUILD_WITH_TREMOR=Off \
+          -DBUILD_WITH_OPUS=Off \
+          -DBUILD_WITH_AVAHI=Off \
+          -DBUILD_WITH_EXPAT=Off \
           -Wno-dev
     make -C build
 }
```

Create systemd units:

```SYSTEMD
# .config/systemd/user/snapserver.service
[Unit]
Description=Snapcast server
Documentation=man:snapserver(1)
After=network-online.target time-sync.target

[Service]
ExecStart=/usr/bin/snapserver --logging.sink=system --server.datadir=/tmp/ --stream.codec=pcm --http.enabled=false --tcp.enabled=false
Restart=on-failure
```

```SYSTEMD
# .config/systemd/user/snapclient.service
[Unit]
Description=Snapcast client
Documentation=man:snapclient(1)
After=network-online.target time-sync.target sound.target

[Service]
ExecStart=/usr/bin/snapclient --logsink=system -h my_desktop_hostname
Restart=on-failure
```

Copy the snapclient service file to the raspberry pi.
To allow running user services via ssh, [enable PAM][]:

```
# /etc/ssh/sshd_config
UsePAM yes
```

Add the virtual pulse audio sink:

```
# .config/pulse/default.pa
load-module module-pipe-sink file=/tmp/snapfifo sink_name=Snapcast format=s16le rate=48000
set-default-sink alsa_output.pci-0000_00_14.2.analog-stereo
```

Create a script that switches to the virtual sink and starts the required services,
then undoes everything on exit:

```shell
#!/bin/sh

set -xeu

LOCAL_SINK=$(pactl get-default-sink)

trap cleanup EXIT INT

cleanup() {
  ssh my_remote_host 'systemctl --user stop snapclient.service'
  systemctl stop --user snapserver.service snapclient.service
  pactl set-default-sink $LOCAL_SINK
}

systemctl start --user snapserver.service snapclient.service
ssh my_remote_host 'systemctl --user start snapclient.service'
pactl set-default-sink Snapcast

read
```

I decided not to run snapclient on the pi all the time:
if the server is down (i.e: I'm not using it, or the desktop is powered off),
it tries to reconnect every second, as expected, and I considered that a waste of resources.

To avoid systemd stopping the remote service after ssh disconnects,
on the remote machine, do:

```sh
$ sudo loginctl enable-linger $USER
```


[Snapcast]: https://github.com/badaix/snapcast
[AUR]: https://aur.archlinux.org/packages/snapcast
[enable PAM]: https://bbs.archlinux.org/viewtopic.php?id=232424
[cmus]: https://cmus.github.io/
[b1]: https://github.com/badaix/snapcast/issues/821#issuecomment-789538906
