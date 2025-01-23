---
title: The best dotfiles manager, chezmoi
description: Description
date: 2021-10-12
author: leon h
draft: true
favorite: false
tags:
  - brew
  - chezmoi
  - dotfiles
  - homebrew
  - onepassword
---

manage secretes in dotfiles with chezmoi

best dotfiles manager

https://blog.arkey.fr/2020/04/01/manage_dotfiles_with_chezmoi/

https://www.chezmoi.io/docs/comparison/

The most useful chezmoi pattern

<!--more-->

```bash
#!/bin/bash
{{ if (eq .chezmoi.os "darwin") -}}

if ! [ -x "$(command -v brew)" ]; then
    echo "Installing Homebrew"
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    [[ $? ]] && echo "Done"
fi

echo "- Updating Homebrew"
brew update
[[ $? ]] && echo "Done"

echo "- Upgrading existing cask and formulae"
brew upgrade
[[ $? ]] && echo "Done"

echo "- Installing missing brews and casks"
brew bundle --no-lock --file=- <<EOF
brew "awscli"
brew "borgbackup"
brew "chezmoi"
brew "dockutil"
brew "gh"
brew "git"
brew "go"
brew "hugo"
brew "jq"
brew "ripgrep"
brew "rustup"
brew "nvm"
cask "1password"
cask "amethyst"
cask "docker"
cask "firefox"
cask "google-chrome"
cask "iterm2"
cask "slack"
cask "spotify"
cask "vlc"
cask "visual-studio-code"
EOF
[[ $? ]] && echo "Done"

echo "- Clean up"
brew cleanup
[[ $? ]] && echo "Done"
{{ end -}}
```

brew list

brew list --cask

brew leaves

brew leaves | xargs -n1 brew desc

- different approaches to manage dotfiles
- chezmoi = integration with password managers

```bash
#!/bin/bash

echo "- Check OnePassord"

if ! [ -x "$(command -v op)" ]; then
  echo "Please install the OnePassord CLI first"
  exit 1
fi

if ! [ $(op get account) ]; then
  echo "Please sign in to the OnePassword CLI"
  exit 1
fi

echo "Signed in to OnePassword"
```

```
[default]
aws_access_key_id={{ (onepasswordItemFields "mgjlbgynirfu7jloxsaryqnrey").credential.v }}
aws_secret_access_key={{ (onepasswordItemFields "msc5zkzuw5cklohc43aqxujphi").credential.v }}
```
