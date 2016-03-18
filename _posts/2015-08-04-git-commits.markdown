---
layout: post
title:  "Git Commit Messages in Vim"
date:   2015-08-04 20:30:00
summary: "Keep your fellow terminal goers onside by hard wrapping your commit messages."
tags:
- git
---

A colleague stumbled upon Chris Beam's "[How to Write a Git Commit Message](http://chris.beams.io/posts/git-commit/)" post yesterday. His suggestions might not be to everyone's tastes (see the comments at the end), but I certainly found them useful. In particular, I liked the suggestion that commit messages should be multi-lined and that no commit should be greater than 72 characters.

Use the following to configure vim to wrap commit messages at 72 characters:

```bash
cat >> ~/.vimrc <<EOF
au FileType gitcommit set tw=72
EOF
```

And, if you haven't already, set vim as your git editor:

```bash
git config --global core.editor vim
```
