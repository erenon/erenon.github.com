---
title: "Automated, encrypted b2 backups with rclone"
date: 2021-03-03
---

Here's how to create automated, secure backups on [blackblaze](http://blackblaze.com/) using [rclone](https://rclone.org/) and systemd.

First, [register](https://www.backblaze.com/b2/sign-up.html) a new blackblaze b2 account if you don't have one already. Make sure you select the right
region during sign-up - it cannot be changed later. Create a new bucket, name it after your hostname.
Create a b2 rclone remote (see [doc](https://rclone.org/b2/)).
Create a crypt overlay over the b2 remote (see [doc](https://rclone.org/crypt/)). Generate long passwords.

The resulting config is something like this:

    $ cat ~/.config/rclone/rclone.conf
    [b2]
    type = b2
    account = *redacted*
    key = *redacted*

    [b2_HOSTNAME_crypt]
    type = crypt
    remote = b2:HOSTNAME
    filename_encryption = standard
    directory_name_encryption = true
    password = *redacted*
    password2 = *redacted*

Try `rclone listremotes`, `rclone copy`, `rclone mount` to see if it works.
Use the b2 online browser to verify that the uploaded data is encrypted.

If you want to backup your home directory, chances are, you don't want to backup it all (e.g: ignore .cache).
Create an rclone filter file that selects the files to sync:

    $ cat ~/.config/rclone/filter.txt
    - .cache/**
    + **

Use `rclone ncdu --filter-from .config/rclone/filter.txt .` to verify that the right files are selected.

## Automate with systemd

To sync the directory, use `rclone sync`. To avoid overwriting previous backups, use `--backup-directory`.
The following [systemd service](https://www.freedesktop.org/software/systemd/man/systemd.service.html) can be used:

    $ cat ~/.config/systemd/user/rclone.service
    [Unit]
    Description="Sync selected parts of home directory to b2 encrypted remote storage"

    [Service]
    Type=oneshot
    ExecStart=%s -c 'rclone sync --filter-from %E/rclone/filter.txt %h b2_%H_crypt:home --b2-hard-delete --fast-list --transfers 32 --backup-dir b2_%H_crypt:home-$$(date +%%u)'

This will keep 1 week worth of data without overwriting. A simple [systemd timer](https://www.freedesktop.org/software/systemd/man/systemd.timer.html) can make it run daily:

    $ cat ~/.config/systemd/user/rclone.timer
    [Unit]
    Description="Sync daily"

    [Timer]
    OnCalendar=*-*-* 12:15:00
    FixedRandomDelay=true
    Persistent=true

    [Install]
    WantedBy=timers.target

Enable and start the timer:

    $ systemctl --user enable rclone.timer
    $ systemctl --user start rclone.timer

## Backup the config

If the rclone encryption passwords are stored in the directory that is being backed up,
in case of a local failure, the remote content cannot be restored.
Therefore, the rclone config must be stored separately.
The [Paperkey](https://wiki.archlinux.org/index.php/Paperkey) page offers a couple of ideas.
For example, to get a printable version of an encrypted config, do:

    $ qrencode --8bit -t svg -lH -o config.svg < encypted_config

To recover:

    $ zbarcam -1 --raw -q -Sbinary

The encryption can happen with e.g: `gpg`, that requires a similar treatment of the private key.

## Note about browsers

Browsers tend to write their persistent storage frequently, making it more difficult
to create a consistent snapshot of them. To reduce the chance that a browser write
conflicts with the sync action, use [Profile Sync Daemon](https://wiki.archlinux.org/index.php/profile-sync-daemon)
It will move the browser storage to memory, and sync it back periodically.
