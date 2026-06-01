---
title: "Hooked on Linux: Rootkit Detection Engineering"
date: 2026-04-02T12:00:00+02:00
description: "Hooked on Linux: Rootkit Detection Engineering"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/hooked-on-linux-rootkits-2.webp"
shareImage: "/images/hooked-on-linux-rootkits-2.webp"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Malware Analysis
  - Detection Engineering
  - Linux
  - Elastic
tags:
  - Malware Analysis
  - Detection Engineering
  - Linux
  - Elastic
---
In the second part of our two-part Linux rootkit series at Elastic Security Labs, Remco Sprooten and I turn from theory to detection engineering. We begin by demonstrating why static detection is often unreliable against Linux rootkits—even trivial modifications like stripping binaries or appending a single null byte can significantly degrade VirusTotal detection rates.

From there, we cover practical behavioral detection across userland rootkit loading (`LD_PRELOAD`, `/etc/ld.so.preload`, dynamic linker configuration), kernel-space LKM loading via `init_module`/`finit_module` syscalls, out-of-tree and unsigned module taint signals, kill-signal abuse, eBPF rootkits, io_uring-based evasion, persistence mechanisms, and defense evasion techniques such as masquerading as kernel threads and log cleansing. Each section includes detection rules and Auditd configuration guidance defenders can apply in production environments.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/linux-rootkits-2-caught-in-the-act)!
