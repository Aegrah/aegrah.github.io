---
title: "Illuminating VoidLink: Technical analysis of the VoidLink rootkit framework"
date: 2026-03-26T12:00:00+02:00
description: "Illuminating VoidLink: Technical analysis of the VoidLink rootkit framework"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/voidlink.webp"
shareImage: "/images/voidlink.webp"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Malware Analysis
  - Linux
  - Elastic
tags:
  - Malware Analysis
  - Linux
  - Rootkit
  - Elastic
---
At Elastic Security Labs, Remco Sprooten and I analyzed a data dump containing source code, compiled binaries, and deployment scripts for the kernel rootkit components of VoidLink—a cloud-native Linux malware framework first documented by Check Point Research. The dump revealed a multigenerational rootkit framework actively developed and tested across real targets, spanning CentOS 7 through Ubuntu 22.04.

VoidLink's architecture immediately stood out: rather than relying on a single technique, it combines a traditional Loadable Kernel Module with eBPF programs in a hybrid design rarely encountered in the wild. The LKM handles deep kernel manipulation, syscall hooking via ftrace, and an ICMP-based covert command channel, while a companion eBPF program hides network connections from the `ss` utility by manipulating Netlink socket responses in userspace memory. We trace its evolution across four generations, dissect its most technically interesting features—including the eBPF "swallowing" technique for `ss` hiding—and provide actionable detection strategies including a YARA signature.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/illuminating-voidlink)!
