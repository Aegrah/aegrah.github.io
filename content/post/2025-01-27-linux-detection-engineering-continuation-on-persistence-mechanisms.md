---
title: "Linux Detection Engineering -  A Continuation on Persistence Mechanisms"
date: 2025-01-27T12:00:00+02:00
description: "Linux Detection Engineering -  A Continuation on Persistence Mechanisms"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/continuation-persistence.webp"
shareImage: "/images/continuation-persistence.webp"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Detection Engineering
  - Persistence
  - Elastic
tags:
  - Detection Engineering
  - Hunting
  - Linux
  - Persistence
  - Elastic
---
In the third part of the Linux Persistence Detection Engineering series, I continue exploring advanced Linux persistence techniques, expanding on the foundation set in previous publications.

This latest installment dives into more creative and complex persistence methods, providing security researchers and defenders with a deeper understanding of how adversaries maintain access on Linux systems. We explore techniques such as dynamic linker hijacking, where adversaries manipulate the dynamic linker through `LD_PRELOAD` to execute malicious code persistently. We also examine loadable kernel modules (LKMs), which allow attackers to embed malicious code directly into the kernel for deep system control. Additionally, we analyze web shells, a persistent threat in web-exposed environments, and demonstrate how default system users with non-interactive shells can be leveraged for stealthy persistence without creating new user entries.

Using [PANIX](https://github.com/Aegrah/PANIX), a Linux persistence tool I developed, we will simulate these techniques, analyze system logs, and uncover new detection opportunities. By leveraging tailored ES|QL and OSQuery detection queries, defenders can strengthen their ability to identify and respond to these advanced threats.

By the end of this research, you’ll have practical insights into both common and rare persistence mechanisms, along with the knowledge needed to build effective detections.

Are you interested in this research? The full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/continuation-on-persistence-mechanisms)!
