---
title: "Rootfs Overlay"
url: "/blog/2023/05/08/rootfs-overlay/"
date: "2023-05-08T00:00:00-05:00"
author: "ayushranjan"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<p>Root filesystem overlay is now the default in runsc. This improves performance
for filesystem-heavy workloads by overlaying the container root filesystem with
a tmpfs filesystem. Learn more about this feature in the following blog that was
<a href="https://opensource.googleblog.com/2023/04/gvisor-improves-performance-with-root-filesystem-overlay.html">originally posted</a>
on <a href="https://opensource.googleblog.com/">Google Open Source Blog</a>.</p>

<!--/excerpt-->

<h2 id="costly-filesystem-access">Costly Filesystem Access</h2>

<p>gVisor uses a trusted filesystem proxy process (“gofer”) to access the
filesystem on behalf of the sandbox. The sandbox process is considered untrusted
in gVisor’s
<a href="https://gvisor.dev/docs/architecture_guide/security/">security model</a>. As a
result, it is not given direct access to the container filesystem and
<a href="https://github.com/google/gvisor/tree/master/runsc/boot/filter">its seccomp filters</a>
do not allow filesystem syscalls.</p>

<p>In gVisor, the container rootfs and
<a href="https://docs.docker.com/storage/bind-mounts/#">bind mounts</a> are configured to
be served by a gofer.</p>

<p><img alt="Figure 1" src="/assets/images/2023-05-08-rootfs-overlay-gofer-diagram.svg" title="Gofer process diagram." width="100%" /></p>

<p>When the container needs to perform a filesystem operation, it makes an RPC to
the gofer which makes host system calls and services the RPC. This is quite
expensive due to:</p>

<ol>
  <li>RPC cost: This is the cost of communicating with the gofer process,
including process scheduling, message serialization and
<a href="https://en.wikipedia.org/wiki/Inter-process_communication">IPC</a> system
calls.
    <ul>
      <li>To ameliorate this, gVisor recently developed a purpose-built protocol
called <a href="https://github.com/google/gvisor/tree/master/pkg/lisafs">LISAFS</a>
which is much more efficient than its predecessor.</li>
      <li>gVisor is also
<a href="https://groups.google.com/g/gvisor-users/c/v-ODHzCrIjE">experimenting</a>
with giving the sandbox direct access to the container filesystem in a
secure manner. This would essentially nullify RPC costs as it avoids the
gofer being in the critical path of filesystem operations.</li>
    </ul>
  </li>
  <li>Syscall cost: This is the cost of making the host syscall which actually
accesses/modifies the container filesystem. Syscalls are expensive, because
they perform context switches into the kernel and back into userspace.
    <ul>
      <li>To help with this, gVisor heavily caches the filesystem tree in memory.
So operations like
<a href="https://man7.org/linux/man-pages/man2/lstat.2.html">stat(2)</a> on cached
files are serviced quickly. But other operations like
<a href="https://man7.org/linux/man-pages/man2/mkdir.2.html">mkdir(2)</a> or
<a href="https://man7.org/linux/man-pages/man2/rename.2.html">rename(2)</a> still
need to make host syscalls.</li>
    </ul>
  </li>
</ol>

<h2 id="container-root-filesystem">Container Root Filesystem</h2>

<p>In Docker and Kubernetes, the container’s root filesystem (rootfs) is based on
the filesystem packaged with the image. The image’s filesystem is immutable. Any
change a container makes to the rootfs is stored separately and is destroyed
with the container. This way, the image’s filesystem can be shared efficiently
with all containers running the same image. This is different from bind mounts,
which allow containers to access the bound host filesystem tree. Changes to bind
mounts are always propagated to the host and persist after the container exits.</p>

<p>Docker and Kubernetes both use the
<a href="https://docs.kernel.org/filesystems/overlayfs.html">overlay filesystem</a> by
default to configure container rootfs. Overlayfs mounts are composed of one
upper layer and multiple lower layers. The overlay filesystem presents a merged
view of all these filesystem layers at its mount location and ensures that lower
layers are read-only while all changes are held in the upper layer. The lower
layer(s) constitute the “image layer” and the upper layer is the “container
layer”. When the container is destroyed, the upper layer mount is destroyed as
well, discarding the root filesystem changes the container may have made.
Docker’s
<a href="https://docs.docker.com/storage/storagedriver/overlayfs-driver/#how-the-overlay2-driver-works">overlayfs driver documentation</a>
has a good explanation.</p>

<h2 id="rootfs-configuration-before">Rootfs Configuration Before</h2>

<p>Let’s consider an example where the image has files <code class="highlighter-rouge">foo</code> and <code class="highlighter-rouge">baz</code>. The
container overwrites <code class="highlighter-rouge">foo</code> and creates a new file <code class="highlighter-rouge">bar</code>. The diagram below shows
how the root filesystem used to be configured in gVisor earlier. We used to go
through the gofer and access/mutate the overlaid directory on the host. It also
shows the state of the host overlay filesystem.</p>

<p><img alt="Figure 2" src="/assets/images/2023-05-08-rootfs-overlay-before.svg" title="Rootfs state before." width="100%" /></p>

<h2 id="opportunity-sandbox-internal-overlay">Opportunity! Sandbox Internal Overlay</h2>

<p>Given that the upper layer is destroyed with the container and that it is
expensive to access/mutate a host filesystem from the sandbox, why keep the
upper layer on the host at all? Instead we can move the upper layer <strong>into the
sandbox</strong>.</p>

<p>The idea is to overlay the rootfs using a sandbox-internal overlay mount. We can
use a tmpfs upper (container) layer and a read-only lower layer served by the
gofer client. Any changes to rootfs would be held in tmpfs (in-memory).
Accessing/mutating the upper layer would not require any gofer RPCs or syscalls
to the host. This really speeds up filesystem operations on the upper layer,
which contains newly created or copied-up files and directories.</p>

<p>Using the same example as above, the following diagram shows what the rootfs
configuration would look like using a sandbox-internal overlay.</p>

<p><img alt="Figure 3" src="/assets/images/2023-05-08-rootfs-overlay-memory.svg" title="Memory-backed rootfs overlay." width="100%" /></p>

<h2 id="host-backed-overlay">Host-Backed Overlay</h2>

<p>The tmpfs mount by default will use the sandbox process’s memory to back all the
file data in the mount. This can cause sandbox memory usage to blow up and
exhaust the container’s memory limits, so it’s important to store all file data
from tmpfs upper layer on disk. We need to have a tmpfs-backing “filestore” on
the host filesystem. Using the example from above, this filestore on the host
will store file data for <code class="highlighter-rouge">foo</code> and <code class="highlighter-rouge">bar</code>.</p>

<p>This would essentially flatten all regular files in tmpfs into one host file.
The sandbox can <a href="https://man7.org/linux/man-pages/man2/mmap.2.html">mmap(2)</a> the
filestore into its address space. This allows it to access and mutate the
filestore very efficiently, without incurring gofer RPCs or syscalls overheads.</p>

<h2 id="self-backed-overlay">Self-Backed Overlay</h2>

<p>In Kubernetes, you can set
<a href="https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#setting-requests-and-limits-for-local-ephemeral-storage">local ephemeral storage limits</a>.
The upper layer of the rootfs overlay (writeable container layer) on the host
<a href="https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-emphemeralstorage-consumption">contributes towards this limit</a>.
The kubelet enforces this limit by
<a href="https://github.com/containerd/containerd/blob/bbcfbf2189f15c9e9e2ce0775c3caf2e8642274c/vendor/github.com/containerd/continuity/fs/du_unix.go#L57-L58">traversing</a>
the entire
<a href="https://github.com/containerd/containerd/blob/bbcfbf2189f15c9e9e2ce0775c3caf2e8642274c/snapshots/overlay/overlay.go#L189-L190">upper layer</a>,
<code class="highlighter-rouge">stat(2)</code>-ing all files and
<a href="https://github.com/containerd/containerd/blob/bbcfbf2189f15c9e9e2ce0775c3caf2e8642274c/vendor/github.com/containerd/continuity/fs/du_unix.go#L69-L74">summing up</a>
their <code class="highlighter-rouge">stat.st_blocks*block_size</code>. If we move the upper layer into the sandbox,
then the host upper layer is empty and the kubelet will not be able to enforce
these limits.</p>

<p>To address this issue, we
<a href="https://github.com/google/gvisor/commit/a53b22ad5283b00b766178eff847c3193c1293b7">introduced “self-backed” overlays</a>,
which create the filestore in the host upper layer. This way, when the kubelet
scans the host upper layer, the filestore will be detected and its
<code class="highlighter-rouge">stat.st_blocks</code> should be representative of the total file usage in the
sandbox-internal upper layer. It is also important to hide this filestore from
the containerized application to avoid confusing it. We do so by
<a href="https://github.com/google/gvisor/commit/09459b203a532c24fbb76cc88484d533356b8b91">creating a whiteout</a>
in the sandbox-internal upper layer, which blocks this file from appearing in
the merged directory.</p>

<p>The following diagram shows what rootfs configuration would finally look like
today in gVisor.</p>

<p><img alt="Figure 4" src="/assets/images/2023-05-08-rootfs-overlay-self.svg" title="Self-backed rootfs overlay." width="100%" /></p>

<h2 id="performance-gains">Performance Gains</h2>

<p>Let’s look at some filesystem-intensive workloads to see how rootfs overlay
impacts performance. These benchmarks were run on a gLinux desktop with
<a href="https://gvisor.dev/docs/architecture_guide/platforms/#kvm">KVM platform</a>.</p>

<h3 id="micro-benchmark">Micro Benchmark</h3>

<p><a href="https://linux-test-project.github.io/">Linux Test Project</a> provides a
<a href="https://github.com/linux-test-project/ltp/tree/master/testcases/kernel/fs/fsstress">fsstress binary</a>.
This program performs a large number of filesystem operations concurrently,
creating and modifying a large filesystem tree of all sorts of files. We ran
this program on the container’s root filesystem. The exact usage was:</p>

<p>    <code class="highlighter-rouge">sh -c "mkdir /test &amp;&amp; time fsstress -d /test -n 500 -p
20 -s 1680153482 -X -l 10"</code></p>

<p>You can use the -v flag (verbose mode) to see what filesystem operations are
being performed.</p>

<p>The results were astounding! Rootfs overlay reduced the time to run this
fsstress program <strong>from 262.79 seconds to 3.18 seconds</strong>! However, note that
such microbenchmarks are not representative of real-world applications and we
should not extrapolate these results to real-world performance.</p>

<h3 id="real-world-benchmark">Real-world Benchmark</h3>

<p>Build jobs are very filesystem intensive workloads. They read a lot of source
files, compile and write out binaries and object files. Let’s consider building
the <a href="https://github.com/abseil/abseil-cpp">abseil-cpp project</a> with
<a href="https://bazel.build/">bazel</a>. Bazel performs a lot of filesystem operations in
rootfs; in bazel’s cache located at <code class="highlighter-rouge">~/.cache/bazel/</code>.</p>

<p>This is representative of the real-world because many other applications also
use the container root filesystem as scratch space due to the handy property
that it disappears on container exit. To make this more realistic, the
abseil-cpp repo was attached to the container using a bind mount, which does not
have an overlay.</p>

<p>When measuring performance, we care about reducing the sandboxing overhead and
bringing gVisor performance as close as possible to unsandboxed performance.
Sandboxing overhead can be calculated using the formula <em>overhead = (s-n)/n</em>
where <code class="highlighter-rouge">s</code> is the amount of time taken to run a workload inside gVisor sandbox
and <code class="highlighter-rouge">n</code> is the time taken to run the same workload natively (unsandboxed). The
following graph shows that rootfs overlay <strong>halved the sandboxing overhead</strong> for
abseil build!</p>

<p><img alt="Figure 5" src="/assets/images/2023-05-08-rootfs-overlay-benchmark-result.svg" title="Sandbox Overhead: rootfs overlay vs no overlay." width="100%" /></p>

<h2 id="conclusion">Conclusion</h2>

<p>Rootfs overlay in gVisor substantially improves performance for many
filesystem-intensive workloads, so that developers no longer have to make large
tradeoffs between performance and security. We recently made this optimization
<a href="https://github.com/google/gvisor/commit/38750cdedcce19a3039da10e515f5852565d2c7e">the default</a>
in runsc. This is part of our ongoing efforts to improve gVisor performance. You
can use gVisor in GKE with GKE Sandbox. Happy sandboxing!</p>
