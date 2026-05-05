---
title: "How we Eliminated 99% of gVisor Networking Memory Allocations with Enhanced Buffer Pooling"
url: "/blog/2022/10/24/buffer-pooling/"
date: "2022-10-24T00:00:00-05:00"
author: "lucasmanning"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<p>In an
<a href="https://gvisor.dev/blog/2020/04/02/gvisor-networking-security/">earlier blog post</a>
about networking security, we described how and why gVisor implements its own
userspace network stack in the Sentry (gVisor kernel). In summary, we’ve
implemented our networking stack – aka Netstack – in Go to minimize exposure to
unsafe code and avoid using an unsafe Foreign Function Interface. With Netstack,
gVisor can do all packet processing internally and only has to enable a few host
I/O syscalls for near-complete networking capabilities. This keeps gVisor’s
exposure to host vulnerabilities as narrow as possible.</p>

<!--/excerpt-->

<p>Although writing Netstack in Go was important for runtime safety, up until now
it had an undeniable performance cost. iperf benchmarks showed Netstack was
spending between 20-30% of its processing time allocating memory and pausing for
garbage collection, a slowdown that limited gVisor’s ability to efficiently
sandbox networking workloads. In this blog we will show how we crafted a cure
for Netstack’s allocation addiction, reducing them by 99%, while also increasing
gVisor networking throughput by 30+%.</p>

<p><img alt="Figure 1" src="/assets/images/2022-10-24-buffer-pooling-figure1.png" title="Buffer pooling results." width="100%" /></p>

<h2 id="a-waste-management-problem">A Waste Management Problem</h2>

<p>Go guarantees a basic level of memory safety through the use of a garbage
collector (GC), which is described in great detail by the Go team
<a href="https://tip.golang.org/doc/gc-guide">here</a>. The Go runtime automatically tracks
and frees objects allocated from the heap, relieving the programmer of the often
painful and error-prone process of manual memory management. Unfortunately,
tracking and freeing memory during runtime comes at a performance cost. Running
the GC adds scheduling overhead, consumes valuable CPU time, and occasionally
pauses the entire program’s progress to track down garbage.</p>

<p>Go’s GC is highly optimized, tunable, and sufficient for a majority of
workloads. Most of the other parts of gVisor happily use Go’s GC with no
complaints. However, under high network stress, Netstack needed to aggressively
allocate buffers used for processing TCP/IP data and metadata. These buffers
often had short lifespans, and once the processing was done they were left to be
cleaned up by the GC. This meant Netstack was producing tons of garbage that
needed to be tracked and freed by GC workers.</p>

<h2 id="recycling-to-the-rescue">Recycling to the Rescue</h2>

<p>Luckily, we weren’t the only ones with this problem. This pattern of small,
frequently allocated and discarded objects was common enough that the Go team
introduced <a href="https://pkg.go.dev/sync#Pool"><code class="highlighter-rouge">sync.Pool</code></a> in Go1.3. <code class="highlighter-rouge">sync.Pool</code> is
designed to relieve pressure off the Go GC by maintaining a thread-safe cache of
previously allocated objects. <code class="highlighter-rouge">sync.Pool</code> can retrieve an object from the cache
if it exists or allocate a new one according to a user specified allocation
function. Once the user is finished with an object they can safely return it to
the cache to be reused again.</p>

<p>While <code class="highlighter-rouge">sync.Pool</code> was exactly what we needed to reduce allocations,
incorporating it into Netstack wasn’t going to be as easy as just replacing all
our <code class="highlighter-rouge">make()</code>s with <code class="highlighter-rouge">pool.Get()</code>s.</p>

<h2 id="netstack-challenges">Netstack Challenges</h2>

<p>Netstack uses a few different types of buffers under the hood. Some of these are
specific to protocols, like
<a href="https://github.com/google/gvisor/blob/master/pkg/tcpip/transport/tcp/segment.go"><code class="highlighter-rouge">segment</code></a>
for TCP, and others are more widely shared, like
<a href="https://github.com/google/gvisor/blob/master/pkg/tcpip/stack/packet_buffer.go"><code class="highlighter-rouge">PacketBuffer</code></a>,
which is used for IP, ICMP, UDP, etc. Although each of these buffer types are
slightly different, they generally share a few common traits that made it
difficult to use <code class="highlighter-rouge">sync.Pool</code> out of the box:</p>

<ul>
  <li>The buffers were originally built with the assumption that a garbage
collector would clean them up automatically – there was little (if any)
effort put into tracking object lifetimes. This meant that we had no way to
know when it was safe to return buffers to a pool.</li>
  <li>Buffers have dynamic sizes that are determined during creation, usually
depending on the size of the packet holding them. A <code class="highlighter-rouge">sync.Pool</code> out of the
box can only accommodate buffers of a single size. One common solution to
this is to fill a pool with
<a href="https://pkg.go.dev/bytes#Buffer"><code class="highlighter-rouge">bytes.Buffer</code></a>, but even a pooled
<code class="highlighter-rouge">bytes.Buffer</code> could incur allocations if it were too small and had to be
grown to the requested size.</li>
  <li>Netstack splits, merges, and clones buffers at various points during
processing (for example, breaking a large segment into smaller MTU-sized
packets). Modifying a buffer’s size during runtime could mean lots of
reallocating from the pool in a one-size-fits-all setup. This would limit
the theoretical effectiveness of a pooled solution.</li>
</ul>

<p>We needed an efficient, low-level buffer abstraction that had answers for the
Netstack specific challenges and could be shared by the various intermediate
buffer types. By sharing a common buffer abstraction, we could maximize the
benefits of pooling and avoid introducing additional allocations while minimally
changing any intermediate buffer processing logic.</p>

<h2 id="introducing-bufferv2">Introducing bufferv2</h2>

<p>Our solution was
<a href="https://github.com/google/gvisor/tree/1ceb81454444981448ad57612139adfc0def1b85/pkg/bufferv2">bufferv2</a>.
Bufferv2 is a non-contiguous, reference counted, pooled, copy-on-write,
buffer-like data structure.</p>

<p>Internally, a bufferv2 <code class="highlighter-rouge">Buffer</code> is a linked list of <code class="highlighter-rouge">View</code>s. Each <code class="highlighter-rouge">View</code> has
start/end indices and holds a pointer to a <code class="highlighter-rouge">Chunk</code>. A <code class="highlighter-rouge">Chunk</code> is a
reference-counted structure that’s allocated from a pool and holds data in a
byte slice. There are several <code class="highlighter-rouge">Chunk</code> pools, each of which allocates chunks with
different sized byte slices. These sizes start at 64 and double until 64k.</p>

<p><img alt="Figure 2" src="/assets/images/2022-10-24-buffer-pooling-figure2.png" title="bufferv2 implementation diagram." width="100%" /></p>

<p>The design of bufferv2 has a few key advantages over simpler object pooling:</p>

<ul>
  <li><strong>Zero-cost copies and copy-on-write</strong>: Cloning a Buffer only increments the
reference count of the underlying chunks instead of reallocating from the
pool. Since buffers are much more frequently read than modified, this saves
allocations. In the cases where a buffer is modified, only the chunk that’s
changed has to be cloned, not the whole buffer.</li>
  <li><strong>Fast buffer transformations</strong>: Truncating and merging buffers or appending
and prepending Views to Buffers are fast operations. Thanks to the
non-contiguous memory structure these operations are usually as quick as
adding a node to a linked list or changing the indices in a View.</li>
  <li><strong>Tiered pools</strong>: When growing a Buffer or appending data, the new chunks
come from different pools of previously allocated chunks. Using multiple
pools means we are flexible enough to efficiently accommodate packets of all
sizes with minimal overhead. Unlike a one-size-fits-all solution, we don’t
have to waste lots of space with a chunk size that is too big or loop
forever allocating small chunks.</li>
</ul>

<h2 id="trade-offs">Trade-offs</h2>

<p>Shifting Netstack to bufferv2 came with some costs. To start, rewriting all
buffers to use bufferv2 was a sizable effort that took many months to fully roll
out. Any place in Netstack that allocated or used a byte slice needed to be
rewritten. Reference counting had to be introduced so all the aforementioned
intermediate buffer types (<code class="highlighter-rouge">PacketBuffer</code>, <code class="highlighter-rouge">segment</code>, etc) could accurately
track buffer lifetimes, and tests had to be modified to ensure reference
counting correctness.</p>

<p>In addition to the upfront cost, the shift to bufferv2 also increased the
engineering complexity of future Netstack changes. Netstack contributors must
adhere to new rules to maintain memory safety and maximize the benefits of
pooling. These rules are strict – there needs to be strong justification to
break them. They are as follows:</p>

<ul>
  <li>Never allocate a byte slice; always use <code class="highlighter-rouge">NewView()</code> instead.</li>
  <li>Use a <code class="highlighter-rouge">View</code> for simple data operations (e.g writing some data of a fixed
size) and a <code class="highlighter-rouge">Buffer</code> for more complex I/O operations (e.g appending data of
variable size, merging data, writing from an <code class="highlighter-rouge">io.Reader</code>).</li>
  <li>If you need access to the contents of a <code class="highlighter-rouge">View</code> as a byte slice, use
<code class="highlighter-rouge">View.AsSlice()</code>. If you need access to the contents of a <code class="highlighter-rouge">Buffer</code> as a byte
slice, consider refactoring, as this will cause an allocation.</li>
  <li>Never write or modify the slices returned by <code class="highlighter-rouge">View.AsSlice()</code>; they are
still owned by the view.</li>
  <li>Release bufferv2 objects as close to where they’re created as possible. This
is usually most easily done with defer.</li>
  <li>Document function ownership of bufferv2 object parameters. If there is no
documentation, it is assumed that the function does not take ownership of
its parameters.</li>
  <li>If a function takes ownership of its bufferv2 parameters, the bufferv2
objects must be cloned before passing them as arguments.</li>
  <li>All new Netstack tests must enable the leak checker and run a final leak
check after the test is complete.</li>
</ul>

<h2 id="give-it-a-try">Give it a Try</h2>

<p>Bufferv2 is enabled by default as of
<a href="https://github.com/google/gvisor/releases/tag/release-20221017.0">gVisor 20221017</a>,
and will be rolling out to
<a href="https://cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods">GKE Sandbox</a>
soon, so no action is required to see a performance boost. Network-bound
workloads, such as web servers or databases like Redis, are the most likely to
see benefits. All the code implementing bufferv2 is public
<a href="https://github.com/google/gvisor/tree/master/pkg/bufferv2">here</a>, and
contributions are welcome! If you’d like to run the iperf benchmark for
yourself, you can run:</p>

<div class="highlighter-rouge"><div class="highlight"><pre class="highlight"><code>make run-benchmark BENCHMARKS_TARGETS=//test/benchmarks/network:iperf_test \
  RUNTIME=your-runtime-here BENCHMARKS_OPTIONS=-test.benchtime=60s
</code></pre></div></div>

<p>in the base gVisor directory. If you experience any issues, please feel free to
let us know at <a href="https://github.com/google/gvisor/issues">gvisor.dev/issues</a>.</p>
