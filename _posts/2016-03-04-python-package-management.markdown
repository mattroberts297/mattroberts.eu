---
layout: post
title:  "Python package management"
date:   2016-03-04 18:59:00 +0000
tags:
- python
- versioning
- dependencies
---

Python ships with [easy_install][easy-install] but that is limited so people use [pip][pip]:

```bash
sudo easy_install pip.
```

Whilst pip is better than easy_install it still installs dependencies globally by default (unlike npm, sbt, mvn etc.). So, everyone uses [virtualenv][virtualenv]:

```bash
sudo pip install virtualenv.
```

Using virtualenv is a little involved:

```bash
mkdir ~/my-python-project
cd ~/my-python-project
virtualenv env
source env/bin/activate # Do this for every session
```

In python, the standard is to create a requirements.txt to specify dependencies:

```bash
echo "requests==2.9.1" > requirements.txt
pip install -r requirements.txt
```

Assuming one wants to use python 3 then it should be as simple as:

```bash
virtualenv -p python3.5 env
# Assumes: sudo port install python35
```

Note that env can be something more descriptive. Perhaps the (short) project name.

To deactivate the virtual environment just run:

```bash
deactivate
```

[easy-install]: http://peak.telecommunity.com/DevCenter/EasyInstall
[pip]: https://pypi.python.org/pypi/pip
[virtualenv]: https://pypi.python.org/pypi/virtualenv
