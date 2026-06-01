---
title: "Copy Fail and DirtyFrag: Linux Page Cache Bugs in the Wild"
date: 2026-05-09T12:00:00+02:00
description: "Copy Fail and DirtyFrag: Linux Page Cache Bugs in the Wild"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
thumbnail: "/images/copyfail_dirtyfrag.webp"
shareImage: "/images/copyfail_dirtyfrag.webp"
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
  - Elastic
---
At Elastic Security Labs, together with Eric Forte and Samir Bousseaden, we analyzed the Linux kernel privilege escalation vulnerabilities Copy Fail (CVE-2026-31431), Copy Fail 2, and DirtyFrag. These issues exploit subtle page cache corruption bugs to create reliable paths to root access, using legitimate kernel interfaces such as `AF_ALG`, `splice()`, and in DirtyFrag's case, networking stack primitives via `AF_NETLINK` and `AF_RXRPC`.

Copy Fail has been reported as exploited in the wild and was added to CISA's Known Exploited Vulnerabilities catalog. DirtyFrag expands the same bug class with variants that do not depend on the `algif_aead` module, meaning systems that only applied Copy Fail mitigations may still be exposed. Rather than matching specific proof-of-concept implementations, we focused detection logic on the underlying exploitation primitives and behavior—syscall sequences, namespace manipulation, and suspicious SUID binary abuse—along with ES|QL hunting queries and mitigation guidance.

Are you interested in this research? Our full paper is available at [Elastic Security Labs](https://www.elastic.co/security-labs/copy-fail-dirtyfrag-linux-page-bugs-in-the-wild)!
