---
title: Back up your data with borgbackup
description: Description
date: 2021-10-12
author: leon h
draft: true
favorite: false
tags:
  - backup
  - borgackup
  - chezmoi
  - dotfiles
---

## borg

```sh
# init repo
borg init --remote-path=borg1 \
  --make-parent-dirs \
  -e repokey-blake2 \
  zh1820@zh1820.rsync.net:macos/Official
# create first backup
borg create --remote-path=borg1 \
  --progress --stats \
  zh1820@zh1820.rsync.net:macos/Official::backup-init ~/Official
```

<!--more-->

## scripts

```sh
export BORG_PASSPHRASE="passphrase"
export BORG_REPO="zh1820@zh1820.rsync.net:macos/Official"
borg create ::$(hostname)-$(date '+%Y-%m-%d') ~/Official
```

https://jstaf.github.io/2018/03/12/backups-with-borg-rsync.html

https://magnusson.io/post/backups/

https://manpages.debian.org/testing/borgbackup/borg-patterns.1.en.

https://www.rsync.net/products/borg.html

https://rafaelc.org/posts/mount-borg-backups-with-ease/

~/bin/executable_cronjob_backup.tmpl

```bash
#!/bin/bash

export BORG_PASSPHRASE="{{ (onepasswordItemFields "hiracdbrhfgopncgodestpn7y4").credential.v }}"
export BORG_REPO="zh1820@zh1820.rsync.net:macos"
echo $(date '+%Y-%m-%d')
borg create \
  --remote-path=borg1 \
  --pattern=!**/.* \
  --pattern=-**/.* \
 {{ range .borgbackup.include }} --pattern=+{{ $.chezmoi.homeDir }}/{{ . }}{{ end }} \
 {{ range .borgbackup.exclude }} --pattern=!{{ $.chezmoi.homeDir }}/{{ . }}{{ end }} \
  --pattern=-{{ .chezmoi.homeDir }} \
  ::$(hostname)-$(date '+%Y-%m-%d') {{ .chezmoi.homeDir }}
borg prune \
  --keep-daily 7 \
  --keep-weekly 8 \
  --keep-monthly 12
```

run_once_after_00_bootstrap.sh.tmpl

```bash
#!/bin/bash

source {{ joinPath .chezmoi.sourceDir "lib/utils.sh" }}

p_header "- Setting up cron jobs"

crontab -r
crontab - <<EOF
30 9 * * * $HOME/.local/bin/cronjob_backup &> $HOME/.cron.backup.log
45 9 * * * $HOME/.local/bin/cronjob_update &> $HOME/.cron.update.log
EOF

[[ $? ]] && p_success "Done"
```

```toml
# .chezmoidata.toml
[general]
  name = "Léon Hollender"
[email]
  userzoom = "lhollender@userzoom.com"
  personal = "hello@leonh.fr"
[github]
  userzoom = "lhollender-uz"
  personal = "leonhfr"
[macos]
  computer_name = "lhollender-macbook"
[borgbackup]
  include = [
    "calibre-kobo",
    "calibre-tablet",
    "Official",
    "Pictures"
  ]
  exclude = [
    "go",
    "leonhfr",
    "Library",
    "uz"
  ]
```
