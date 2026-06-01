---
title: "Linux Detection Engineering -  Approaching the Summit on Persistence Mechanisms"
date: 2025-02-11T12:00:00+02:00
description: "Linux Detection Engineering -  Approaching the Summit on Persistence Mechanisms"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/approaching-summit-persistence.webp"
shareImage: "/images/approaching-summit-persistence.webp"
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
In the fourth part of the Linux Persistence Detection Engineering series, I continue exploring advanced Linux persistence techniques, expanding on the foundation set in previous publications.

This latest installment delves into additional creative and complex methods adversaries use to maintain persistence on Linux systems. We explore the abuse of Pluggable Authentication Modules (PAM), specifically how pam_exec can be leveraged to execute malicious code during authentication events. We also analyze installer package manipulation via RPM and DPKG, where lifecycle scripts are weaponized to establish persistence through package installations and updates. Finally, we examine malicious Docker containers, detailing how attackers exploit privileged containers and host-level access for persistence and container escapes.

Using [PANIX](https://github.com/Aegrah/PANIX), a Linux persistence tool I developed, we will simulate these techniques, analyze system logs, and identify detection opportunities. With tailored ES|QL and OSQuery detection queries, defenders can strengthen their ability to uncover and respond to these advanced threats.

By the end of this research, you’ll have a deeper understanding of both common and stealthy Linux persistence mechanisms and how to engineer robust detections against real-world adversary tactics.

Are you interested in this research? The full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/approaching-the-summit-on-persistence)!
