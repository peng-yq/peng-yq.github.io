---
layout: post
title: How GitHub contributions counts
subtitle: why-are-your-contributions-not-showing-up-on-my-profile?
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - Git
---

## 前言

 “TL;DR”：给自己的repo提交了，但是并未显示在GitHub的profile中。一开始以为是private repo的原因，但是在profile中开启了private contributions也没有解决问题。

“RDTM”：[why-are-my-contributions-not-showing-up-on-my-profile?](https://docs.github.com/zh/account-and-profile/setting-up-and-managing-your-github-profile/managing-contribution-settings-on-your-profile/why-are-my-contributions-not-showing-up-on-my-profile)

## Why?

只有如下操作计入contributions：

1. Issues, pull requests and discussions (opened in a standalone repository, not a fork)

2. commits will appear on your contributions graph if they meet **all** of the following conditions:

   - The email address used for the commits is associated with your account on GitHub.com.
   - The commits were made in a standalone repository, not a fork.
   - The commits were made:
     - In the repository's default branch
     - In the `gh-pages` branch (for repositories with project sites)

   In addition, **at least one** of the following must be true:

   - You are a collaborator on the repository or are a member of the organization that owns the repository.
   - You have forked the repository.
   - You have opened a pull request or issue in the repository.
   - You have starred the repository.

官方还列举了一些常见的不会计入contributions的情况：

1. Commit was made less than 24 hours ago (这个似乎很少见)
2. Your local Git commit email isn't connected to your account
3. Commit was not made in the default or gh-pages branch (破案了，因为提交的不是默认分支或者gh-pages分支)
4. Commit was made in a fork (这个也遇到过)
