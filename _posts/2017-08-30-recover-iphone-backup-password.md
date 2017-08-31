---
title:  "How to recover iPhone backup's password?"
---

If you backup your iPhone or iPad onto your iMac via iTunes you probably use
encrypted backups. Sometimes it happens that you forget your encryption
password. As it turns out
there is no way to disable backups or change their password without knowing
the original password. See Apple support article
[About encrypted backups in iTunes](https://support.apple.com/en-us/HT205220).

There are several paid solutions that can decrypt your password that could
be run inside a Windows virtual machine. But there is little guarantee that
they would not share all your data through the Internet. So, the obvious
solution is to use a command-line utility that does it for free.

> Time to try one password is roughly **a minute per core** on my 2014 iMac, so
> this solution does not work if **really** don't remember your password.

<!--more-->

# Why does it take a minute to try only one password?

See StackOverflow for
[How to decrypt an encrypted Apple iTunes iPhone backup?](https://stackoverflow.com/questions/1498342/how-to-decrypt-an-encrypted-apple-itunes-iphone-backup)
In short it's a usual [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2)
scheme with lots of iterations. So there is
not shortcut, unless SHA is completely broken by the time you're
reading this article.

# Is it important which iOS version do you have?

Yes. New versions of iOS/macOS/iTunes often bring new schemes for password
encryption, usually it only gets stronger over time. It also means
that unless somebody spent their time reverse engineering new encryption
scheme, you're out of luck.

I've tried it with iOS 10.3.3 backups in iTunes 12.6.2.20 on macOS 10.12.15
and it worked fine.

# What have I used to decrypt it?

Collection of python scripts that can decrypt iPhone backups as one of the
many features that they support:
[iphone-dataprotection](https://github.com/dinosec/iphone-dataprotection).
One particularly easy to use python file is `python_scripts/backup_tool.py`

I my case I knew prefix for my password, and I was pretty sure about specific
letters for the rest, so my search was very short.
I used this simple shell script to try to figure out the last 3 letters:

```sh
#!/bin/bash

try_pass() {
  END="$1$2$3"
  # Print password in the logs to make it easier to read them
  echo "Trying password: '*****$END'"
  # Agree to typing in a password and type in the password when tool prompts
  ( echo y ; echo 'known_pa$$w0rd_prefix'"$END" ) \
  | ./backup_tool.py \
        ~/Library/Application\ Support/MobileSync/Backup/00_LONG_BACKUP_HASH_00 \
        ./extract
  # If extract directory was created, exit with non-0 status
  [ ! -d ./extract ]
}
# Make it available to parallel, only works in bash
export -f try_pass

# show progress, tag output, exit when non-0 status, pipe output
parallel --progress --tag --halt soon,fail=1 \
      try_pass ::: {a..z} ::: '' {a..z} ::: '' {a..z} \
  > getpass.out
```

> **xargs** on macOS has less options than its GNU variant.
> **parallel** is more convenient, but requires
> [Homebrew](https://brew.sh) installation:
> `brew install parallel`, help available at `man parallel_tutorial`.

The only thing left is run it and wait until finished and check
`getpass.out` file for the extraction password.
