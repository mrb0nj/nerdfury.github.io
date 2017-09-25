---
layout: post
title: "Freenas Install: Error 19"
subtitle: "Mounting from ufs:/dev/md0.uzip failed with error 19"
tags: [home]
---

# Background

Recently, by my own neglegence, suffered a fairly significant data loss after the failure of 2 drives in a RAID5 array. 
I ignored daily warnings from smartd about 'degraded array' and 'bad blocks' on one of my drives for about 6 months, 
then we had a storm which caused the smallest of 'flickers' in power, but it was still enough to do some serious damage to another drive. 
Fortunately, the contents of the array was mostly replaceable media files and old obsolete backups, no big loss, but inconvenient all the same.
Anyway, enough of how stupid I am...


# Decisions, decisions

So, I'd been happily running Ubuntu server with software raid and lvm and probably
would have rebuilt it identically had a friend not challenged me to try something else... Freenas!
Not really for any other reason than his own personal enjoyment, knowing that I wouldn't be able to 'not'
try something new and shiney!

New RAM ordered, new USB boot drive ordered... happy days...

# The hair pulling started

So, I followed the [instructions](http://doc.freenas.org/11/install.html#preparing-the-media) and create my bootable
USB installation drive using the command:

```terminal
$ sudo dd if=FreeNAS-11.0-U3.iso of=/dev/sdc bs=64k
```

I start the installation as described in the docs, but it gets stuck repeating the same few lines over and over again

```terminal
md0 attached to /data/base.ufs.uzip
Trying to mount root from ufs:/dev/md0.uzip [rw]...
mountroot: waiting for device /dev/md0.uzip
Mounting from ufs:/dev/md0.uzip failed with error 19
```

After many hours of google-fu and tearing my hair out, I stumbled on a [Reddit post](https://www.reddit.com/r/freenas/comments/5k5846/cannot_install_freenas_error_19/) 
which talked about creating the installation media with a block size of 32k instead of 64k.

```terminal
$ sudo dd if=FreeNAS-11.0-U3.iso of=/dev/sdc bs=32k
```

I quickly recreated the install drive, setting the block size to 32k and the installation completed exactly as the documentation 
described, just in time for dinner. Life was great again!

Hope this helps anyone sufferring with the same error and currently wading through the 10 billion forum posts that are similar but not __quite__ the same issue!


#### .nerdfury
