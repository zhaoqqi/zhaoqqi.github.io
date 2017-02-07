---
layout: post
title: nova-compute 无损重启
categories: [openstack]
description: How to restart nova-compute gracefully.
keywords: openstack, nova-compute, restart, graceful
---

nova stop 的时候，变更发出 SIGTERM 为 SIGHUP，这样就不会造成一stop服务，所有线程就全部退出了。
