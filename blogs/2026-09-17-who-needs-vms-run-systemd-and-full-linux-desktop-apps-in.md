---
title: "Who needs VMs? Run systemd and full Linux desktop apps in gVisor"
url: "/blog/2026/09/17/systemd-in-gvisor/"
date: "2026-09-17"
author: "relkochta"
feed_url: "https://gvisor.dev/blog/index.xml"
---
As an OCI runtime, gVisor is traditionally used to sandbox application containers containing a single service. These containers have little to no userspace running in them except the service itself. While this is great for efficiency and scalability, there are some use cases that can benefit from having something closer to a full Linux system inside the sandbox, with a “normal” service manager and system daemons.
