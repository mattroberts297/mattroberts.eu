---
layout: post
title:  "Gmail and Mac OS X Mail"
date:   2016-03-21 08:00:00 +0000
tags:
- osx
---

I wanted to be able to read and write emails on the train. I do tether my laptop to my phone, but the signal drops out pretty regularly. That makes using Gmail in a browser difficult. When not on the train I usually use Gmail in a browser because it has better search than Mail.

In Gmail -> Settings -> Forwarding and POP/IMAP:

- Check 'Enable IMAP'
- Check 'Auto-expunge-off'
- Check 'Archive the message (default)'
- Check 'Do not limit the number of messages in an IMAP folder (default)'

The second setting is required to enable the third setting.

In Mail -> Preferences -> Accounts -> Google -> Mailbox Behaviors:

- Uncheck 'Store draft messages on the server'
- Uncheck 'Store sent messages on the server'
- Uncheck 'Store deleted messages on the server'

The first setting stops lots of drafts appearing in Gmail. The second and third settings prevent duplicates (and probably reduces traffic).
