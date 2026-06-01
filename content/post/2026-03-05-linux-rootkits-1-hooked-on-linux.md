---
title: "Hooked on Linux: Rootkit Taxonomy, Hooking Techniques and Tradecraft"
date: 2026-03-05T12:00:00+02:00
description: "Hooked on Linux: Rootkit Taxonomy, Hooking Techniques and Tradecraft"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/hooked-on-linux-rootkits-1.jpg"
shareImage: "/images/hooked-on-linux-rootkits-1.jpg"
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
In the first part of our two-part Linux rootkit series at Elastic Security Labs, Remco Sprooten and I explore the theory behind how rootkits work: their taxonomy, evolution, and the hooking techniques they use to subvert the kernel. We trace the progression from early userland shared object rootkits through LKM-based implants, eBPF rootkits, and emerging io_uring-based evasion.

The publication covers rootkit loader and payload components, kernel hooking techniques including IDT hooking, syscall table patching, inline hooking, VFS hooking, ftrace and kprobes abuse, the KHOOK framework, userspace `LD_PRELOAD` interposition, and eBPF program attachment. We also discuss how modern rootkits like PUMAKIT and Diamorphine combine multiple hooking techniques, and how the recently published FlipSwitch technique demonstrates that syscall table patching remains viable even on Linux kernel 6.9+.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/linux-rootkits-1-hooked-on-linux)!
