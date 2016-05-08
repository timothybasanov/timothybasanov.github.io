---
title:  "Fixing sbin for Homebrew on macOS"
---

Homebrew installs `iftop` into `/usr/local/sbin` and not in `/usr/local/bin` so it's not a part of your `$PATH` by default.

Yes, you can add it manually via `.zprofile` or something similar, but there is a native way to add new paths to all shells of all users:

```sh
echo '/usr/local/sbin' | tee -a /etc/paths
```

Or you can add a new entry to `paths.d`:

```sh
echo '/usr/local/sbin' | tee -a /etc/paths.d/homebrew
```
