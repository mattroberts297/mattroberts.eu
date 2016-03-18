---
layout: post
title:  "Git Rebase then Merge"
date:   2015-08-05 07:00:00
summary: "Reduce the number of merges in your git log by rebasing local branches (never force push though)."
tags:
- git
---

Whilst git's `rebase` command can be dangerous. It can also lead to cleaner commit logs than just using `merge`.

```bash
git init
touch README
git add .
git commit -m "Add README"
git log --oneline
# 9e1b782 Add README
```

<img src="//assets.mattro.be/rts/img/git-rebase-add-readme.png" alt="README Added" class="img-responsive max-height-125">

```bash
git branch alice
git checkout alice
touch A
git add .
git commit -m "Add A"
touch B
git add .
git commit -m "Add B"
git log --oneline
# a718f5a Add B
# c5849fd Add A
# 9e1b782 Add README
```

<img src="//assets.mattro.be/rts/img/git-rebase-alice-branch.png" alt="Alice Branched" class="img-responsive max-height-125">

```bash
git checkout master
git branch charlie
git checkout charlie
touch C
git add .
git commit -m "Add C"
touch D
git add .
git commit -m "Add D"
git log --oneline
# 0c047d3 Add D
# fa7d087 Add C
# 9e1b782 Add README
```

<img src="//assets.mattro.be/rts/img/git-rebase-charlie-branch.png" alt="Charlie Branched" class="img-responsive max-height-125">

```bash
git checkout master
git merge alice
git log --oneline
# a718f5a Add B
# c5849fd Add A
# 9e1b782 Add README
```

<img src="//assets.mattro.be/rts/img/git-rebase-alice-merged.png" alt="Alice Merged" class="img-responsive max-height-125">

```bash
git checkout charlie
git log --no-merges --oneline charlie..master
# a718f5a Add B
# c5849fd Add A
git log --oneline
# 0c047d3 Add D
# fa7d087 Add C
# 9e1b782 Add README

git rebase master
git log --no-merges --oneline charlie..master
git log --oneline
# 7ddd032 Add D
# d51a0d8 Add C
# a718f5a Add B
# c5849fd Add A
# 9e1b782 Add README
```

<img src="//assets.mattro.be/rts/img/git-rebase-charlie-rebased.png" alt="Charlie Rebased" class="img-responsive max-height-125">

```bash
git checkout master
git merge charlie
git log --oneline
# 7ddd032 Add D
# d51a0d8 Add C
# a718f5a Add B
# c5849fd Add A
# 9e1b782 Add README
```

<img src="//assets.mattro.be/rts/img/git-rebase-charlie-merged.png" alt="Charlie Merged" class="img-responsive max-height-125">

 If you find yourself rebasing master onto your feature branch then something has probably gone wrong. You will know if you have accidentally done this because git will politely object to pushing your newly ruined master. In short, do not force push.
