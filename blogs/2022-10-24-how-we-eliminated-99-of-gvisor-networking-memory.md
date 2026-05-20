---
title: "How we Eliminated 99% of gVisor Networking Memory Allocations with Enhanced Buffer Pooling"
url: "/blog/2022/10/24/buffer-pooling/"
date: "2022-10-24T00:00:00-05:00"
author: "lucasmanning"
feed_url: "https://gvisor.dev/blog/index.xml"
---
In an earlier blog post about networking security, we described how and why gVisor implements its own userspace network stack in the Sentry (gVisor kernel). In summary, we’ve implemented our networking stack – aka Netstack – in Go to minimize exposure to unsafe code and avoid using an unsafe Foreign Function Interface. With Netstack, gVisor can do all packet processing internally and only has to enable a few host I/O syscalls for near-complete networking capabilities.
