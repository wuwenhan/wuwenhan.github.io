---
layout: post
title:  "ubuntu添加ftp用户"
date:   2015-01-26 15:22:46
categories: wuwenhan update
---

groupadd ftptest

useradd -G ftptest -d /var/www -M user1

passwd user1