---
title: "Store SSH key passphrase in password manager"
date: 2021-02-08
---

The `ssh-agent` program can be used to manage unlocked SSH keys.
The `ssh-add` program can be used to add keys to a running `ssh-agent`.
By default, `ssh-add` uses TTY to ask for a password,
if the selected key is password protected.

The [pass][] program is a password manager that follows the Unix philosophy.
If you want to use `pass` together with ssh-add, first add the private key
password to it:

```shell
$ pass insert ssh/identityname_rsa
```

Then create this wrapper program `/home/user/bin/askpass.sh`:

```shell
#!/bin/sh
# $1 ~= Enter passphrase for '.ssh/$KEYNAME':
exec /usr/bin/pass $(grep -o "ssh/[^:']\+" <<< "$1")
```

Export the following variables, e.g: by adding them to you shell rc:

```shell
export SSH_ASKPASS_REQUIRE=force
export SSH_ASKPASS="/home/user/bin/askpass.sh"
```

Now whenever an SSH key is needed (e.g: for `git` or `ssh` client),
`ssh-add` will turn to `pass`, giving it the key name to be unlocked,
and in case the gpg key is not unlocked at the moment, `pass` will
prompt you to enter your password, then safely transmit the key
passphrase to `ssh-add`, that will add the key to the running `ssh-agent`.

[pass]: https://www.passwordstore.org/

If git does pop up the gpg unlock dialog, but prints
"gpg: decryption failed: No secret key" instead,
make sure the [GPG_TTY environment variable][gpgtty] is exported:

```shell
export GPG_TTY=$TTY
```

If git still does not ask for a password, and no such error is printed:
If the private key is not the default one (e.g: `.ssh/id_*`), the host-key association
must be configured in `.ssh/config`, e.g:

```
Host github.com
  IdentityFile ~/.ssh/your_nondefault_key
```

[gpgtty]: https://www.gnupg.org/documentation/manuals/gnupg/Invoking-GPG_002dAGENT.html
