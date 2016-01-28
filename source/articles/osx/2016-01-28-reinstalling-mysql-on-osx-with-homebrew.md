---
title: Reinstalling MySQL on OSX using Homebrew
category: Rails
---

If you find that you need to have a particular version of MySQL installed on
your OSX machine (e.g. to match the version running in production) but you
already have a different version of MySQL installed via Homebrew, then the notes
below may help you to replace the installed version with the one you need.

**DISCLAIMER:** The following instructions involve the complete removal of the
existing MySQL installation, including any databases. Proceed with caution, and
double check every command--this is just meant to be a rough guide to an
approach you can take to achieve the desired result.

## Uninstalling

 1. Check `~/Library/LaunchAgents`, if `homebrew.mxcl.mysql.plist` is present then:
   1. Unload: `launchctl unload -w ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist`
   2. Remove: `rm ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist`
 1. Stop the MySQL server process: `sudo mysql.server stop`
 1. `brew remove mysql`
 1. `brew cleanup` (note that this command will clean up after more than just MySQL)
 1. Remove if present:
    1. `/usr/local/var/mysql`

## Reinstalling

 1. Select the version that you want to install (in my case, `5.5.x`)
 1. `brew tap homebrew/versions`
 1. `brew install mysql55`
 1. Follow the post-install instructions as provided by the `brew install` output
 1. Add the appropriate directoy to your `PATH`, e.g. `/usr/local/opt/mysql55/bin`

### Post-install instructions

Yours may differ from the below.

```
To have launchd start homebrew/versions/mysql55 at login:
  ln -sfv /usr/local/opt/mysql55/*.plist ~/Library/LaunchAgents
Then to load homebrew/versions/mysql55 now:
  launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql55.plist
Or, if you don't want/need launchctl, you can just run:
  /usr/local/opt/mysql55/bin/mysql.server start

WARNING: launchctl will fail when run under tmux.
```
