---
title: "Quarantine Post: Make Homebrew Work in China"
date: 2021-05-16
draft: false
toc: false
images:
tags:
  - homebrew
  - macos
  - technical
  - quarantine
---

While you are reading this post, I have probably been in quarantine for several days. Yes! I'm home, but in a hotel for now.

I have a lot of ideas that I want to work on. I'm not disclosing any of them, but the gist is that by realizing those ideas, I get to experience the same workflow as before but in a different country with [some internet censorships](https://en.m.wikipedia.org/wiki/Great_Firewall). I will have to naturally get through them if I want to work in the same way as before.

## The problem

The first issue I encountered, right off the bat, was not being able to use Homebrew normally.

By default, Homebrew relies on GitHub for their repo distributions. In China, accessing GitHub is extremely slow. I immediately realized this when I attempted to run `brew update` and it timed out with "443" errors.

## The solution

Luckily, Homebrew has a huge Chinese community that has established repo sources in China. All I needed to do was to set the sources to one of the Chinese ones for `homebrew`, `homebrew-core`, and `homebrew-cask`. I'm using the ones set up by the University of Science and Technology of China:

```bash
git -C "$(brew --repo)" remote set-url origin https://mirrors.ustc.edu.cn/brew.git

git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git

git -C "$(brew --repo homebrew/cask)" remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git

```

I also need to change the environment variable `HOMEBREW_BOTTLE_DOMAIN` by adding this line to my `.zshrc`:

```bash
export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles/
```

To sum up, I can put the above code into a shell script `change_source.sh`:

```bash
#!/bin/sh

git -C "$(brew --repo)" remote set-url origin https://mirrors.ustc.edu.cn/brew.git

git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git

git -C "$(brew --repo homebrew/cask)" remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git

if [ $SHELL = "/bin/bash" ]; then # 如果你的是bash
  echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles/' >>~/.bash_profile
  source ~/.bash_profile
elif [ $SHELL = "/bin/zsh" ]; then # 如果用的shell 是zsh 的话
  echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles/' >>~/.zshrc
  source ~/.zshrc
fi
```

To restore the sources, I have created a neat shell script `restore_sources.sh` that sets all sources back to GitHub:

```bash
git -C "$(brew --repo)" remote set-url origin https://github.com/Homebrew/brew.git

git -C "$(brew --repo homebrew/core)" remote set-url origin https://github.com/Homebrew/homebrew-core.git

git -C "$(brew --repo homebrew/cask)" remote set-url origin https://github.com/Homebrew/homebrew-cask.git
```

## Closing thoughts

That's it for Homebrew - I will likely encounter more issues in the coming days as I spend more time working, so stay alerted for more :smiley:!.
