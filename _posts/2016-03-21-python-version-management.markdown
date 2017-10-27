---
layout: post
title:  "Python version management"
description: "How to use pyenv instead of virtualenv"
date:   2016-03-21 18:32:00 +0000
tags:
- python
- versioning
---

I stumbled upon [pyenv][pyenv] today. It lets you:

- change the **global** python version per user
- change the python version per **project**
- **override** the python version with an environment variable

It doesn't depend on python (unlike [virtualenv][virtualenv]).

There's also a [pyenv plugin][plugin] to manage [virtualenv][virtualenv].

It's a fork of [rbenv][rbenv] and works in the same way i.e. shims and shell functions.

[pyenv]: https://github.com/yyuu/pyenv
[rbenv]: https://github.com/rbenv/rbenv
[virtualenv]: https://pypi.python.org/pypi/virtualenv
[plugin]: https://github.com/yyuu/pyenv-virtualenv
