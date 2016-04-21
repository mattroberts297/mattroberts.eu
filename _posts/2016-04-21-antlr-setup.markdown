---
layout: post
title:  "ANTLR 4 Setup"
date:   2016-04-21 08:35:00 +0000
tags:
- antlr
---

I use macports so the following directories exist already:

- `/opt/local/lib`
- `/opt/local/bin`

The latter is on my PATH. You can check yours with `echo $PATH`.

```bash
sudo su -

mkdir /opt/local/lib/antlr4

curl --output /opt/local/lib/antlr4/antlr-4.5.3-complete.jar http://www.antlr.org/download/antlr-4.5.3-complete.jar

cat > /opt/local/bin/antlr << EOL
#!/bin/sh
java -cp "/opt/local/lib/antlr4/antlr-4.5.3-complete.jar:$CLASSPATH" org.antlr.v4.Tool $*
EOL
chmod a+x /opt/local/bin/antlr4

cat > /opt/local/bin/grun << EOL
#!/bin/sh
java -cp "/opt/local/lib/antlr4/antlr-4.5.3-complete.jar:$CLASSPATH" org.antlr.v4.gui.TestRig $*
EOL
chmod a+x /opt/local/bin/grun
```

