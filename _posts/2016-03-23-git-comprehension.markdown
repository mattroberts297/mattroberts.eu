---
layout: post
title:  "Git Comprehension"
description: "Helpful commands to navigate a large of fast moving repository"
date:   2016-03-23 08:35:00 +0000
tags:
- git
---

Some commands to help navigate a large and / or fast moving repository:


```bash
# Show first line of commit message only
git log --oneline
# Show the files touched by a commit
git log --status
# Do not show merges (particularly useful when rebasing is frowned upon)
git log --no-merges
# Show details of the last commit (HEAD)
git show
# Show details of the penultimate commit
git show HEAD~1
# Show details of a particular commit e.g. c6a105e
git show c6a105e
# Show the commits that touched a file and where
git blame README
# Show changed files across commits
git diff-tree --no-commit-id --name-only -r HEAD..HEAD~3
# Show commits that match some regex e.g. foo
git log --grep foo
# Show commits by an author (use email too if more than one MR)
git log --author "Matt Roberts"
```
