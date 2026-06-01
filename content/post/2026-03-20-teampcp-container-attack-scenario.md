---
title: "Linux & Cloud Detection Engineering - TeamPCP Container Attack Scenario"
date: 2026-03-20T12:00:00+02:00
description: "Linux & Cloud Detection Engineering - TeamPCP Container Attack Scenario"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/teampcp.png"
shareImage: "/images/teampcp.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Detection Engineering
  - Linux
  - Elastic
tags:
  - Detection Engineering
  - Linux
  - Containers
  - Elastic
---
At Elastic Security Labs, I published a real-world walkthrough of TeamPCP's multi-stage container compromise, demonstrating how Elastic's Defend for Containers (D4C) surfaces runtime signals across each stage of the attack chain. Rather than analyzing isolated techniques in abstraction, we follow the attack as it unfolds inside a containerized environment based on the TeamPCP cloud-native ransomware operation documented by Flare.

The scenario spans nearly the entire MITRE ATT&CK lifecycle—from initial execution via `curl | bash` and Kubernetes environment discovery, through lateral movement via `kube.py`, persistence via systemd, runtime tooling installation, tunneling with frps and gost, encoded payload execution, miner deployment, and escalation to node control via privileged DaemonSets and Kubernetes API abuse. We also demonstrate how Attack Discovery correlates 130+ individual alerts into a coherent container cryptojacking attack narrative.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/teampcp-container-attack-scenario)!
