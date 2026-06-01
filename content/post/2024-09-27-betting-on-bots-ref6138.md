---
title: "Betting on Bots: Investigating Linux malware, crypto mining, and gambling API abuse"
date: 2024-09-27T12:00:00+02:00
description: "Betting on Bots: Investigating Linux malware, crypto mining, and gambling API abuse"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/betting-on-bots.webp"
shareImage: "/images/betting-on-bots.webp"
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
At Elastic Security Labs, we uncovered a sophisticated Linux malware campaign exploiting Apache2 servers since March 2024. Attackers used multiple malware families, including `KAIJI` (DDoS) and `RUDEDEVIL` (crypto miner), along with custom tools for persistence and control. They leveraged C2 channels disguised as kernel processes, Telegram bots, and cron jobs. The investigation suggests a potential Bitcoin/XMR mining scheme tied to gambling APIs, hinting at money laundering. Continuous malware development was observed through a file share hosting fresh `KAIJI` samples. The research provides an in-depth analysis of the attack tactics, persistence methods, and C2 infrastructure.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/betting-on-bots)!
