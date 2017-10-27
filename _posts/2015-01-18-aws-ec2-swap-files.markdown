---
layout: post
title:  "AWS EC2 Swap Files"
date:   2015-01-18 11:30:00
description: "Create swap files on t2.micro instances to stop the linux OS killing your JVM process."
tags:
- aws
- linux
- bash
---

At the time of writing, the AWS EC2 t2.micro instance comes with: 1 vCPU; 6 CPU credits / hour; 1 GiB of RAM and some EBS storage. If you're using a t2.micro, performance probably isn't your thing. They're great for testing little applications out however. The only problem is that if you start to run out of memory then Linux will start killing processes (probably yours).

If you're creating an instance from scratch you should probably attach an extra volume to it. Whilst you'll have to use EBS storage with the t2.micro you can choose ephemeral instance storage for the bigger, better instances. Assuming, for now, that you're using EBS storage and the volume is attached at `/dev/sdg` then you can do the folowing:

{% highlight bash %}
sudo su
umount /dev/sdg
mkswap /dev/sdg
swapon /dev/sdg
echo '/dev/sdg none swap defaults 0 0' >> /etc/fstab
{% endhighlight %}

If you're using instance storage then AWS will probably have already added an entry for it into fstab. You can use sed to find and replace that entry like so:

{% highlight bash %}
sudo su
umount /dev/sdg
mkswap /dev/sdg
swapon /dev/sdg
sed -i 's/^\/dev\/sdg.*$/\/dev\/sdg none swap defaults 0 0/' /etc/fstab
{% endhighlight %}

Finally, if you've already created the instance and really don't want to stop it then you can use a swap file instead:

{% highlight bash %}
sudo su
dd if=/dev/zero of=/var/swap bs=1024 count=131072
mkswap /var/swap
swapon /var/swap
echo '/swapfile none swap defaults 0 0' >> /etc/fstab
{% endhighlight %}
