---
title: "Declawing PUMAKIT"
date: 2024-12-12T12:00:00+02:00
description: "Declawing PUMAKIT"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/pumakit.webp"
shareImage: "/images/pumakit.webp"
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
  - Elastic
---
At Elastic Security Labs, we uncovered `PUMAKIT`, a sophisticated multi-stage Linux malware with advanced rootkit capabilities. Initially identified through routine threat hunting on VirusTotal, `PUMAKIT` consists of a dropper (`cron`), two memory-resident executables, an LKM rootkit module, and a userland shared object (SO) rootkit.

The rootkit, internally named `PUMA` by its authors, employs ftrace to hook 18 syscalls and multiple kernel functions, enabling stealthy privilege escalation, file and process hiding, and anti-debugging measures. Uniquely, it interacts with the system through unconventional mechanisms, such as leveraging the `rmdir()` syscall for privilege escalation. The malware ensures activation only under specific conditions, including secure boot status and kernel symbol availability, making it highly evasive.

This research delves into `PUMAKIT`’s technical architecture, stealth techniques, and persistence mechanisms, highlighting its ability to manipulate core system behaviors while maintaining control over infected hosts.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/declawing-pumakit)!
