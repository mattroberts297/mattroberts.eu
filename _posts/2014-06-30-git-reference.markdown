---
layout: post
title:  "Git Reference"
date:   2014-06-30 21:30:00
description: "Git reference"
tags:
- git
---

Git is without a doubt my favourite VCS to date. I put this reference together when I first started using it. I still find myself referring back to it when I set up a computer for the first time. Most of the contents is taken from stack overflow articles and the git book. But it's obviously been condensed somewhat.

### Config

```bash
# For completeness!
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com
# Save passwords (OS X).
git config --global credential.helper osxkeychain
# Because you want to use vim. Right?
git config --global core.editor vim
# Because typing commit and status is rubbish.
git config --global alias.cl clone
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
# Setup a merge tool.
git config --global merge.tool diffmerge
git config --global mergetool.diffmerge.cmd "diffmerge --merge
  --result=\$MERGED \$LOCAL \$BASE \$REMOTE"
git config --global mergetool.diffmerge.trustExitCode true
git config --global mergetool.keepBackup false
# This is *not* the default in 1.x
git config --global push.default current
```

### Getting Started

```bash

# Initialize a git repository.
cd ~/
mkdir foo
cd foo
git init
# Make an initial commit.
touch README                       # Modify README.
git add README                     # Stage README.
git ci -m "Initial commit."        # Commit README.
# Convert to bare repository.
git config --bool core.bare true
# Clone it.
cd ~/
git clone foo foo-bar
cd foo-bar
# Make some changes.
echo Foo Bar > README              # Modify README.
git status                         # -|
git diff                           #  |- Diff modified README.
git add README
git status                         # -|
git diff --staged                  #  |- Diff staged README.
git ci -m "Updated README."
git log
git push                           # Push changes to Foo.
```

### Collaborating

#### Merging

```bash
# Clone again.
cd ~/
git clone foo foo-baz
cd foo-baz
# Make some changes and push.
echo Foo Baz! > README
git add README
git ci -m "Updated README (baz)."
git push
# Create a conflict.
cd ~/foo-bar
echo Foo Bar! > README
git add README
git ci -m "Updated README (bar)."
# Get rejected.
git push                           # Rejected!
git pull
git mergetool
git ci -m "Resolving merge."
git push
```

#### Branching

```bash
# Creating a branch.
git branch dev                     # Create dev branch.
git branch                         # Show all branches.
git checkout dev                   # Checkout dev branch.
# Use branch.
echo TODO > LICENSE
git add LICENSE
git ci -m "Created license."
# Pushing.
git push                           # No upstream branch in origin!
git push --set-upstream origin dev
# Pulling branches.
cd ~/foo-baz
git branch                         # No dev here!
git fetch                          # This will get the branch from the origin!
git branch                         # Strange, still no branch.
git branch -a                      # Oh of course, add v for more verbose output.
git branch dev --track origin/dev  # Create a local branch setup to track origin/dev.
git checkout dev
# Delete a local branch.
git checkout master
git branch -d dev
# Try the shorthand.
git checkout -b dev origin/dev
# Delete a branch remotely.
git branch -d dev
git push origin :dev
```

#### Rebasing

```bash
# Pull the remote branch and rebase yours on top of it
git pull --rebase
```
