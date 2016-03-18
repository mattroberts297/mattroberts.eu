---
layout: post
title:  "Move . to foo excluding foo"
date:   2016-03-16 18:32:02 +0000
tags:
- bash
- linux
---

I don't think you can do this with the `mv` command alone, so I use `find`.

Test first:

```bash
find  . -depth 1 ! -name foo -exec echo {} \;
```

Do the move:

```bash
find  . -depth 1 ! -name foo -exec mv {} foo \;
```

Stack excludes as required:

```bash
find  . -depth 1 ! -name foo ! -name bar -exec mv {} foo \;
```
