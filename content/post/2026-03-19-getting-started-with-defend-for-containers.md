---
title: "Linux & Cloud Detection Engineering - Getting Started with Defend for Containers (D4C)"
date: 2026-03-19T12:00:00+02:00
description: "Linux & Cloud Detection Engineering - Getting Started with Defend for Containers (D4C)"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/teampcp-2.png"
shareImage: "/images/teampcp-2.png"
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
At Elastic Security Labs, I published a comprehensive walkthrough of Elastic's Defend for Containers (D4C) integration, covering Kubernetes-based deployment, BPF-enriched runtime telemetry analysis, and the practical application of policy-driven security controls for containerized Linux environments.

Defend for Containers arrived in Elastic Stack 9.3.0 as a runtime security integration that captures process execution and file access events enriched with container and orchestration context. This post provides a practical starting point for detection engineers: how to deploy D4C via Elastic Agent in Kubernetes, how its selector-response policy model works, which fields matter for detection logic (capabilities, interactive execution, container privilege context), and how to enable the pre-built detection ruleset. We also cover Beta limitations and a recommended workflow for validating policies before enabling blocking responses.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/getting-started-with-defend-for-containers)!
