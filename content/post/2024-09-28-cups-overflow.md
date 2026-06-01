---
title: "Cups Overflow: When your printer spills more than Ink"
date: 2024-09-28T12:00:00+02:00
description: "Cups Overflow: When your printer spills more than Ink"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/cups.webp"
shareImage: "/images/cups.webp"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - CVE
  - Detection Engineering
  - Linux
  - Elastic
tags:
  - CVE
  - Detection Engineering
  - Linux
  - Elastic
---
At Elastic Security Labs, we analyzed a critical set of vulnerabilities in the CUPS printing system, disclosed by security researcher Simone Margaritelli (@evilsocket) on September 26, 2024. These flaws, affecting CUPS versions ≤ 2.0.1, enable unauthenticated remote attackers to achieve remote code execution (RCE) via the Internet Printing Protocol (IPP) and mDNS, exploiting UDP port 631. Key weaknesses include input validation flaws in cups-browsed, libcupsfilters, and libppd, as well as the long-unpatched foomatic-rip filter. Many UNIX-based systems, including Linux, BSDs, ChromeOS, and Solaris, are impacted, with cups-browsed often enabled by default.

Our research delves into the technical details of the exploitation chain, providing insights into the attack methods and detection strategies. We also outline mitigation steps to help organizations secure their systems against these threats.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/cups-overflow)!
