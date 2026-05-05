---
title: "Optimizing seccomp usage in gVisor"
url: "/blog/2024/02/01/seccomp/"
date: "2024-02-01T00:00:00-06:00"
author: "eperot"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<p>gVisor is a multi-layered security sandbox. <a href="https://www.kernel.org/doc/html/v4.19/userspace-api/seccomp_filter.html"><code class="highlighter-rouge">seccomp-bpf</code></a> is
gVisor’s second layer of defense against container escape attacks. gVisor uses
<code class="highlighter-rouge">seccomp-bpf</code> to filter its own syscalls by the host kernel. This significantly
reduces the attack surface to the host that a compromised gVisor process can
access. However, this layer comes at a cost: every legitimate system call that
gVisor makes must be evaluated against this filter by the host kernel before it
is actually executed. <strong>This blog post contains more than you ever wanted to
know about <code class="highlighter-rouge">seccomp-bpf</code>, and explores the past few months of work to optimize
gVisor’s use of it.</strong></p>

<!--/excerpt-->

<p><img alt="gVisor and seccomp" src="/assets/images/2024-02-01-gvisor-seccomp.png" title="gVisor and seccomp" />
<span class="attribution">A diagram showing gVisor’s two main layers of
security: gVisor itself, and <code class="highlighter-rouge">seccomp-bpf</code>. This blog post touches on the
<code class="highlighter-rouge">seccomp-bpf</code> part.
<a href="https://commons.wikimedia.org/wiki/File:Tux.svg">Tux logo by Larry Ewing and The GIMP</a>.</span></p>

<h2 id="performance-considerations">Understanding <code class="highlighter-rouge">seccomp-bpf</code> performance in gVisor</h2>

<p>One challenge with gVisor performance improvement ideas is that it is often very
difficult to estimate how much they will impact performance without first doing
most of the work necessary to actually implement them. Profiling tools help with
knowing where to look, but going from there to numbers is difficult.</p>

<p><code class="highlighter-rouge">seccomp-bpf</code> is one area which is actually much more straightforward to
estimate. Because it is a secondary layer of defense that lives outside of
gVisor, and it is merely a filter, we can simply yank it out of gVisor and
benchmark the performance we get. While running gVisor in this way is strictly
<strong>less secure</strong> and not a mode that gVisor should support, the numbers we get in
this manner do provide an upper bound on the maximum <em>potential</em> performance
gains we could see from optimizations within gVisor’s use of <code class="highlighter-rouge">seccomp-bpf</code>.</p>

<p>To visualize this, we can run a benchmark with the following variants:</p>

<ul>
  <li><strong>Unsandboxed</strong>: Unsandboxed performance without gVisor.</li>
  <li><strong>gVisor</strong>: gVisor from before any of the performance improvements described
later in this post.</li>
  <li><strong>gVisor with empty filter</strong>: Same as <strong>gVisor</strong>, but with the <code class="highlighter-rouge">seccomp-bpf</code>
filter replaced with one that unconditionally approves every system call.</li>
</ul>

<p>From these three variants, we can break down the gVisor overhead that comes from
gVisor itself vs the one that comes from <code class="highlighter-rouge">seccomp-bpf</code> filtering. The difference
between <strong>gVisor</strong> and <strong>unsandboxed</strong> represents the total gVisor performance
overhead, and the difference between <strong>gVisor</strong> and <strong>gVisor with empty filter</strong>
represents the performance overhead of gVisor’s <code class="highlighter-rouge">seccomp-bpf</code> filtering rules.</p>

<p>Let’s run these numbers for the ABSL build benchmark:</p>

<p><img alt="ABSL seccomp-bpf performance" src="/assets/images/2024-02-01-gvisor-seccomp-absl-empty-filter.png" title="ABSL seccomp-bpf performance" /></p>

<p>We can now use these numbers to give a rough breakdown of where the overhead is
coming from:</p>

<p><img alt="ABSL seccomp-bpf performance breakdown" src="/assets/images/2024-02-01-gvisor-seccomp-absl-breakdown.png" title="ABSL seccomp-bpf performance breakdown" /></p>

<p>The <code class="highlighter-rouge">seccomp-bpf</code> overhead is small in absolute terms. The numbers suggest that
the best that can be shaved off by optimizing <code class="highlighter-rouge">seccomp-bpf</code> filters is <strong>up to</strong>
3.4 seconds off from the total ABSL build time, which represents a reduction of
total runtime by ~3.6%. However, when looking at this amount relative to
gVisor’s overhead over unsandboxed time, this means that optimizing the
<code class="highlighter-rouge">seccomp-bpf</code> filters may remove <strong>up to</strong> ~15% of gVisor overhead, which is
significant. <em>(Not all benchmarks have this behavior; some benchmarks show
smaller <code class="highlighter-rouge">seccomp-bpf</code>-related overhead. The overhead is also highly
platform-dependent.)</em></p>

<p>Of course, this level of performance is what was reached with <strong>empty
<code class="highlighter-rouge">seccomp-bpf</code> filtering rules</strong>, so we cannot hope to reach this level of
performance gains. However, it is still useful as an upper bound. Let’s see how
much of it we can recoup without compromising security.</p>

<h2 id="a-primer-on-bpf-and-seccomp-bpf">A primer on BPF and <code class="highlighter-rouge">seccomp-bpf</code></h2>

<h3 id="bpf-cbpf-ebpf-oh-my">BPF, cBPF, eBPF, oh my!</h3>

<p><a href="https://en.wikipedia.org/wiki/Berkeley_Packet_Filter">BPF (Berkeley Packet Filter)</a> is a virtual machine and eponymous machine
language. Its name comes from its original purpose: filtering packets in a
kernel network stack. However, its use has expanded to other domains of the
kernel where programmability is desirable. Syscall filtering in the context of
<code class="highlighter-rouge">seccomp</code> is one such area.</p>

<p>BPF itself comes in two dialects: “Classic BPF” (sometimes stylized as cBPF),
and the now-more-well-known <a href="https://en.wikipedia.org/wiki/EBPF">“Extended BPF” (commonly known as eBPF)</a>.
eBPF is a superset of cBPF and is usable extensively throughout the kernel.
However, <code class="highlighter-rouge">seccomp</code> is not one such area. While
<a href="https://lwn.net/Articles/857228/">the topic has been heavily debated</a>, the
status quo remains that <code class="highlighter-rouge">seccomp</code> filters may only use cBPF, so this post will
focus on cBPF alone.</p>

<h3 id="so-what-is-seccomp-bpf-exactly">So what is <code class="highlighter-rouge">seccomp-bpf</code> exactly?</h3>

<p><code class="highlighter-rouge">seccomp-bpf</code> is a part of the Linux kernel which allows a program to impose
syscall filters on itself. A <code class="highlighter-rouge">seccomp-bpf</code> filter is a cBPF program that is
given syscall data as input, and outputs an “action” (a 32-bit integer) to do as
a result of this system call: allow it, reject it, crash the program, trap
execution, etc. The kernel evaluates the cBPF program on every system call the
application makes. The “input” of this cBPF program is the byte layout of the
<code class="highlighter-rouge">seccomp_data</code> struct, which can be loaded into the registers of the cBPF
virtual machine for analysis.</p>

<p>Here’s what the <code class="highlighter-rouge">seccomp_data</code> struct looks like in
<a href="https://github.com/torvalds/linux/blob/master/include/uapi/linux/seccomp.h">Linux’s <code class="highlighter-rouge">include/uapi/linux/seccomp.h</code></a>:</p>

<div class="language-c highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">struct</span> <span class="n">seccomp_data</span> <span class="p">{</span>
    <span class="kt">int</span> <span class="n">nr</span><span class="p">;</span>                     <span class="c1">// 32 bits</span>
    <span class="n">__u32</span> <span class="n">arch</span><span class="p">;</span>                 <span class="c1">// 32 bits</span>
    <span class="n">__u64</span> <span class="n">instruction_pointer</span><span class="p">;</span>  <span class="c1">// 64 bits</span>
    <span class="n">__u64</span> <span class="n">args</span><span class="p">[</span><span class="mi">6</span><span class="p">];</span>              <span class="c1">// 64 bits × 6</span>
<span class="p">};</span>                              <span class="c1">// Total 512 bits</span>
</code></pre></div></div>

<h3 id="sample-filter">Sample <code class="highlighter-rouge">seccomp-bpf</code> filter</h3>

<p>Here is an example <code class="highlighter-rouge">seccomp-bpf</code> filter, adapted from the
<a href="https://www.kernel.org/doc/Documentation/networking/filter.txt">Linux kernel documentation</a><sup id="fnref:1"><a class="footnote" href="#fn:1">1</a></sup>:</p>

<!-- Markdown note: This uses "javascript" syntax highlighting because that
     happens to work pretty well with this pseudo-assembly-like language.
     It is not actually JavaScript. -->

<div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="mi">00</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">4</span>                <span class="c1">// Load 32 bits at offsetof(struct seccomp_data, arch) (= 4)</span>
                            <span class="c1">//   of the seccomp_data input struct into register A.</span>
<span class="mi">01</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0xc000003e</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">11</span>   <span class="c1">// If A == AUDIT_ARCH_X86_64, jump by 0 instructions [to 02]</span>
                            <span class="c1">//   else jump by 11 instructions [to 13].</span>
<span class="mi">02</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">0</span>                <span class="c1">// Load 32 bits at offsetof(struct seccomp_data, nr) (= 0)</span>
                            <span class="c1">//   of the seccomp_data input struct into register A.</span>
<span class="mi">03</span><span class="p">:</span> <span class="nx">jeq</span>  <span class="mi">15</span><span class="p">,</span>  <span class="mi">10</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// If A == __NR_rt_sigreturn, jump by 10 instructions [to 14]</span>
                            <span class="c1">//   else jump by 0 instructions [to 04].</span>
<span class="mi">04</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">231</span><span class="p">,</span>   <span class="mi">9</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// If A == __NR_exit_group, jump by 9 instructions [to 14]</span>
                            <span class="c1">//   else jump by 0 instructions [to 05].</span>
<span class="mi">05</span><span class="p">:</span> <span class="nx">jeq</span>  <span class="mi">60</span><span class="p">,</span>   <span class="mi">8</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// If A == __NR_exit, jump by 8 instructions [to 14]</span>
                            <span class="c1">//   else jump by 0 instructions [to 06].</span>
<span class="mi">06</span><span class="p">:</span> <span class="nx">jeq</span>   <span class="mi">0</span><span class="p">,</span>   <span class="mi">7</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// Same thing for __NR_read.</span>
<span class="mi">07</span><span class="p">:</span> <span class="nx">jeq</span>   <span class="mi">1</span><span class="p">,</span>   <span class="mi">6</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// Same thing for __NR_write.</span>
<span class="mi">08</span><span class="p">:</span> <span class="nx">jeq</span>   <span class="mi">5</span><span class="p">,</span>   <span class="mi">5</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// Same thing for __NR_fstat.</span>
<span class="mi">09</span><span class="p">:</span> <span class="nx">jeq</span>   <span class="mi">9</span><span class="p">,</span>   <span class="mi">4</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// Same thing for __NR_mmap.</span>
<span class="mi">10</span><span class="p">:</span> <span class="nx">jeq</span>  <span class="mi">14</span><span class="p">,</span>   <span class="mi">3</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// Same thing for __NR_rt_sigprocmask.</span>
<span class="mi">11</span><span class="p">:</span> <span class="nx">jeq</span>  <span class="mi">13</span><span class="p">,</span>   <span class="mi">2</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// Same thing for __NR_rt_sigaction.</span>
<span class="mi">12</span><span class="p">:</span> <span class="nx">jeq</span>  <span class="mi">35</span><span class="p">,</span>   <span class="mi">1</span><span class="p">,</span>   <span class="mi">0</span>       <span class="c1">// If A == __NR_nanosleep, jump by 1 instruction [to 14]</span>
                            <span class="c1">//   else jump by 0 instructions [to 13].</span>
<span class="mi">13</span><span class="p">:</span> <span class="k">return</span> <span class="mi">0</span>                <span class="c1">// Return SECCOMP_RET_KILL_THREAD</span>
<span class="mi">14</span><span class="p">:</span> <span class="k">return</span> <span class="mh">0x7fff0000</span>       <span class="c1">// Return SECCOMP_RET_ALLOW</span>
</code></pre></div></div>

<p>This filter effectively allows only the following syscalls: <code class="highlighter-rouge">rt_sigreturn</code>,
<code class="highlighter-rouge">exit_group</code>, <code class="highlighter-rouge">exit</code>, <code class="highlighter-rouge">read</code>, <code class="highlighter-rouge">write</code>, <code class="highlighter-rouge">fstat</code>, <code class="highlighter-rouge">mmap</code>, <code class="highlighter-rouge">rt_sigprocmask</code>,
<code class="highlighter-rouge">rt_sigaction</code>, and <code class="highlighter-rouge">nanosleep</code>. All other syscalls result in the calling thread
being killed.</p>

<h3 id="cbpf-limitations"><code class="highlighter-rouge">seccomp-bpf</code> and cBPF limitations</h3>

<p>cBPF is quite limited as a language. The following limitations all factor into
the optimizations described in this blog post:</p>

<ul>
  <li>The cBPF virtual machine only has 2 32-bit registers, and a tertiary
pseudo-register for a 32-bit immediate value. (Note that syscall arguments
evaluated in the context of <code class="highlighter-rouge">seccomp</code> are 64-bit values, so you can already
foresee that this leads to complications.)</li>
  <li><code class="highlighter-rouge">seccomp-bpf</code> programs are limited to 4,096 instructions.</li>
  <li>Jump instructions can only go forward (this ensures that programs must
halt).</li>
  <li>Jump instructions may only jump by a fixed (“immediate”) number of
instructions. (You cannot say: “jump by whatever this register says”.)</li>
  <li>Jump instructions come in two flavors:
    <ul>
      <li>“Unconditional” jump instructions, which jump by a fixed number of
instructions. This number must fit in 16 bits.</li>
      <li>“Conditional” jump instructions, which include a condition expression
and two jump targets:
        <ul>
          <li>The number of instructions to jump by if the condition is true. This
number must fit in 8 bits, so this cannot jump by more than 255
instructions.</li>
          <li>The number of instructions to jump by if the condition is false.
This number must fit in 8 bits, so this cannot jump by more than 255
instructions.</li>
        </ul>
      </li>
    </ul>
  </li>
</ul>

<h3 id="seccomp-bpf-caching-in-linux"><code class="highlighter-rouge">seccomp-bpf</code> caching in Linux</h3>

<p>Since
<a href="https://www.phoronix.com/news/Linux-5.11-SECCOMP-Performance">Linux kernel version 5.11</a>,
when a program uploads a <code class="highlighter-rouge">seccomp-bpf</code> filter into the kernel,
<a href="https://github.com/torvalds/linux/commit/8e01b51a31a1e08e2c3e8fcc0ef6790441be2f61">Linux runs a BPF emulator</a>
that looks for system call numbers where the BPF program doesn’t do any fancy
operations nor load any bits from the <code class="highlighter-rouge">instruction_pointer</code> or <code class="highlighter-rouge">args</code> fields of
the <code class="highlighter-rouge">seccomp_data</code> input struct, and still returns “allow”. When this is the
case, <strong>Linux will cache this information</strong> in a per-syscall-number bitfield.</p>

<p>Later, when a cacheable syscall number is executed, the BPF program is not
evaluated at all; since the kernel knows that the program is deterministic and
doesn’t depend on the syscall arguments, it can safely allow the syscall without
actually running the BPF program.</p>

<p>This post uses the term “cacheable” to refer to syscalls that match this
criteria.</p>

<h2 id="how-gvisor-builds-its-seccomp-bpf-filter">How gVisor builds its <code class="highlighter-rouge">seccomp-bpf</code> filter</h2>

<p>gVisor imposes a <code class="highlighter-rouge">seccomp-bpf</code> filter on itself as part of Sentry start-up. This
process works as follows:</p>

<ul>
  <li>gVisor gathers bits of configuration that are relevant to the construction
of its <code class="highlighter-rouge">seccomp-bpf</code> filter. This includes which platform is in use, whether
certain features that require looser filtering are enabled (e.g. host
networking, profiling, GPU proxying, etc.), and certain file descriptors
(FDs) which may be checked against syscall arguments that pass in FDs.</li>
  <li>
    <p>gVisor generates a sequence of rulesets from this configuration. A ruleset
is a mapping from syscall number to a predicate that must be true for this
system call, along with an “action” (return code) that is taken should this
predicate be satisfied. For ease of human understanding, the predicate is
often written as a
<a href="https://en.wikipedia.org/wiki/Logical_disjunction">disjunctive rule</a>, for
which each sub-rule is a
<a href="https://en.wikipedia.org/wiki/Logical_conjunction">conjunctive rule</a> that
verifies each syscall argument. In other words, <code class="highlighter-rouge">(fA(args[0]) &amp;&amp; fB(args[1])
&amp;&amp; ...) || (fC(args[0]) &amp;&amp; fD(args[1]) &amp;&amp; ...) || ...</code>. This is represented
<a href="https://github.com/google/gvisor/blob/master/runsc/boot/filter/config/config_main.go">in gVisor code</a>
as follows:</p>

    <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">Or</span><span class="p">{</span>          <span class="c">// Disjunction rule</span>
    <span class="n">PerArg</span><span class="p">{</span>  <span class="c">// Conjunction rule over each syscall argument</span>
        <span class="n">fA</span><span class="p">,</span>  <span class="c">// Predicate for `seccomp_data.args[0]`</span>
        <span class="n">fB</span><span class="p">,</span>  <span class="c">// Predicate for `seccomp_data.args[1]`</span>
        <span class="c">// ... More predicates can go here (up to 6 arguments per syscall)</span>
    <span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span>  <span class="c">// Conjunction rule over each syscall argument</span>
        <span class="n">fC</span><span class="p">,</span>  <span class="c">// Predicate for `seccomp_data.args[0]`</span>
        <span class="n">fD</span><span class="p">,</span>  <span class="c">// Predicate for `seccomp_data.args[1]`</span>
        <span class="c">// ... More predicates can go here (up to 6 arguments per syscall)</span>
    <span class="p">},</span>
<span class="p">}</span>
</code></pre></div>    </div>
  </li>
  <li>
    <p>gVisor performs several optimizations on this data structure.</p>
  </li>
  <li>
    <p>gVisor then renders this list of rulesets into a linear program that looks
close to the final machine language, other than jump offsets which are
initially represented as symbolic named labels during the rendering process.</p>
  </li>
  <li>
    <p>gVisor then resolves all the labels to their actual instruction index, and
computes the actual jump targets of all jump instructions to obtain valid
cBPF machine code.</p>
  </li>
  <li>
    <p>gVisor runs further optimizations on this cBPF bytecode.</p>
  </li>
  <li>Finally, the cBPF bytecode is uploaded into the host kernel and the
<code class="highlighter-rouge">seccomp-bpf</code> filter becomes effective.</li>
</ul>

<p>Optimizing the <code class="highlighter-rouge">seccomp-bpf</code> filter to be more efficient allows the program to
be more compact (i.e. it’s possible to pack more complex filters in the 4,096
instruction limit), and to run faster. While <code class="highlighter-rouge">seccomp-bpf</code> evaluation is
measured in nanoseconds, the impact of any optimization is magnified here,
because host syscalls are an important part of the synchronous “syscall hot
path” that must execute as part of handling certain performance-sensitive
syscall from the sandboxed application. The relationship is not 1-to-1: a single
application syscall may result in several host syscalls, especially due to
<code class="highlighter-rouge">futex(2)</code> which the Sentry calls many times to synchronize its own operations.
Therefore, shaving a nanosecond here and there results in several shaved
nanoseconds in the syscall hot path.</p>

<h2 id="structure">Structural optimizations</h2>

<p>The first optimization done for gVisor’s <code class="highlighter-rouge">seccomp-bpf</code> was to turn its linear
search over syscall numbers into a
<a href="https://en.wikipedia.org/wiki/Binary_search_tree">binary search tree</a>. This
turns the search for syscall numbers from <code class="highlighter-rouge">O(n)</code> to <code class="highlighter-rouge">O(log n)</code> instructions.
This is a very common <code class="highlighter-rouge">seccomp-bpf</code> optimization technique which is replicated
in other projects such as
<a href="https://github.com/seccomp/libseccomp/issues/116">libseccomp</a> and Chromium.</p>

<p>To do this, a cBPF program basically loads the 32-bit <code class="highlighter-rouge">nr</code> (syscall number)
field of the <code class="highlighter-rouge">seccomp_data</code> struct, and does a binary tree traversal of the
<a href="https://chromium.googlesource.com/chromiumos/docs/+/HEAD/constants/syscalls.md#tables">syscall number space</a>.
When it finds a match, it jumps to a set of instructions that check that
syscall’s arguments for validity, and then returns allow/reject.</p>

<p>But why stop here? Let’s go further.</p>

<p>The problem with the binary search tree approach is that it treats all syscall
numbers equally. This is a problem for three reasons:</p>

<ol>
  <li>It does not matter to have good performance for disallowed syscalls, because
such syscalls should never happen during normal program execution.</li>
  <li>It does not matter to have good performance for syscalls which can be cached
by the kernel, because the BPF program will only have to run once for these
system calls.</li>
  <li>For the system calls which are allowed but are not cacheable by the kernel,
there is a
<a href="https://en.wikipedia.org/wiki/Pareto_distribution">Pareto distribution</a> of
their relative frequency. To exploit this we should evaluate the most-often
used syscalls faster than the least-often used ones. The binary tree
structure does not exploit this distribution, and instead treats all
syscalls equally.</li>
</ol>

<p>So gVisor splits syscall numbers into four sets:</p>

<ul>
  <li>🅰: Non-cacheable 🅰llowed, called very frequently.</li>
  <li>🅱: Non-cacheable allowed, called once in a 🅱lue moon.</li>
  <li>🅲: 🅲acheable allowed (whether called frequently or not).</li>
  <li>🅳: 🅳isallowed (which, by definition, is neither cacheable nor expected to
ever be called).</li>
</ul>

<p>Then, the cBPF program is structured in the following layout:</p>

<ul>
  <li>Linear search over allowed frequently-called non-cacheable syscalls (🅰).
These syscalls are ordered in most-frequently-called first (e.g. <code class="highlighter-rouge">futex(2)</code>
is the first one as it is by far the most-frequently-called system call).</li>
  <li>Binary search over allowed infrequently-called non-cacheable syscalls (🅱).</li>
  <li>Binary search over allowed cacheable syscalls (🅲).</li>
  <li>Reject anything else (🅳).</li>
</ul>

<p>This structure takes full advantage of the kernel caching functionality, and of
the Pareto distribution of syscalls.</p>

<details>

  

    <h3 id="binary-search-tree-optimizations">Binary search tree optimizations</h3>

    <p>Beyond classifying syscalls to see which binary search tree they should be a
part of, gVisor also optimizes the binary search process itself.</p>

  

  <p>Each syscall number is a node in the tree. When traversing the tree, there are
three options at each point:</p>

  <ul>
    <li>The syscall number is an exact match</li>
    <li>The syscall number is lower than the node’s value</li>
    <li>The syscall number is higher than the node’s value</li>
  </ul>

  <p>In order to render the BST as cBPF bytecode, gVisor used to render the following
(in pseudocode):</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">if</span> <span class="nx">syscall</span> <span class="nx">number</span> <span class="o">==</span> <span class="nx">current</span> <span class="nx">node</span> <span class="nx">value</span>
    <span class="nx">jump</span> <span class="p">@</span><span class="nd">rules_for_this_syscall</span>
<span class="k">if</span> <span class="nx">syscall</span> <span class="nx">number</span> <span class="o">&lt;</span> <span class="nx">current</span> <span class="nx">node</span> <span class="nx">value</span>
    <span class="nx">jump</span> <span class="p">@</span><span class="nd">left_node</span>
<span class="nx">jump</span> <span class="p">@</span><span class="nd">right_node</span>

<span class="p">@</span><span class="nd">rules_for_this_syscall</span><span class="p">:</span>
  <span class="c1">// Render bytecode for this syscall's filters here...</span>

<span class="p">@</span><span class="nd">left_node</span><span class="p">:</span>
  <span class="c1">// Recursively render the bytecode for the left node value here...</span>

<span class="p">@</span><span class="nd">right_node</span><span class="p">:</span>
  <span class="c1">// Recursively render the bytecode for the right node value here...</span>
</code></pre></div>  </div>

  <p>Keep in mind the <a href="#cbpf-limitations">cBPF limitations</a> here. Because conditional
jumps are limited to 255 instructions, the jump to <code class="highlighter-rouge">@left_node</code> can be further
than 255 instructions away (especially for syscalls with complex filtering rules
like <a href="https://man7.org/linux/man-pages/man2/ioctl.2.html"><code class="highlighter-rouge">ioctl(2)</code></a>). The jump
to <code class="highlighter-rouge">@right_node</code> is almost certainly more than 255 instructions away. This means
in actual cBPF bytecode, we would often need to use conditional jumps followed
by unconditional jumps in order to jump so far forward. Meanwhile, the jump to
<code class="highlighter-rouge">@rules_for_this_syscall</code> would be a very short hop away, but this locality
would only be taken advantage of for a single node of the entire tree for each
traversal.</p>

  <p>Consider this structure instead:</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// Traversal code:</span>
  <span class="k">if</span> <span class="nx">syscall</span> <span class="nx">number</span> <span class="o">&lt;</span> <span class="nx">current</span> <span class="nx">node</span> <span class="nx">value</span>
      <span class="nx">jump</span> <span class="p">@</span><span class="nd">left_node</span>
  <span class="k">if</span> <span class="nx">syscall_number</span> <span class="o">&gt;</span> <span class="nx">current</span> <span class="nx">node</span> <span class="nx">value</span>
      <span class="nx">jump</span> <span class="p">@</span><span class="nd">right_node</span>
  <span class="nx">jump</span> <span class="p">@</span><span class="nd">rules_for_this_syscall</span>
  <span class="p">@</span><span class="nd">left_node</span><span class="p">:</span>
    <span class="c1">// Recursively render only the traversal code for the left node here</span>
  <span class="p">@</span><span class="nd">right_node</span><span class="p">:</span>
    <span class="c1">// Recursively render only the traversal code for the right node here</span>

<span class="c1">// Filtering code:</span>
  <span class="p">@</span><span class="nd">rules_for_this_syscall</span><span class="p">:</span>
    <span class="c1">// Render bytecode for this syscall's filters here</span>
  <span class="c1">// Recursively render only the filtering code for the left node here</span>
  <span class="c1">// Recursively render only the filtering code for the right node here</span>
</code></pre></div>  </div>

  <p>This effectively separates the per-syscall rules from the traversal of the BST.
This ensures that the traversal can be done entirely using conditional jumps,
and that for any given execution of the cBPF program, there will be at most one
unconditional jump to the syscall-specific rules.</p>

  <p>This structure is further improvable by taking advantage of the fact that
syscall numbers are a dense space, and so are syscall filter rules. This means
we can often avoid needless comparisons. For example, given the following tree:</p>

  <div class="highlighter-rouge"><div class="highlight"><pre class="highlight"><code>      22
     /  \
    9    24
   /    /  \
  8   23    50
</code></pre></div>  </div>

  <p>Notice that the tree contains <code class="highlighter-rouge">22</code>, <code class="highlighter-rouge">23</code>, and <code class="highlighter-rouge">24</code>. This means that if we get to
node <code class="highlighter-rouge">23</code>, we do not need to check for syscall number equality, because we’ve
already established from the traversal that the syscall number must be <code class="highlighter-rouge">23</code>.</p>

</details>

<h2 id="cbpf-bytecode-optimizations">cBPF bytecode optimizations</h2>

<p>gVisor now implements a
<a href="https://github.com/google/gvisor/blob/master/pkg/bpf/optimizer.go">bytecode-level cBPF optimizer</a>
running a few lossless optimizations. These optimizations are run repeatedly
until the bytecode no longer changes. This is because each type of optimization
tends to feed on the fruits of the others, as we’ll see below.</p>

<p><img alt="gVisor sentry seccomp-bpf filter program size" src="/assets/images/2024-02-01-gvisor-seccomp-sentry-filter-size.png" title="gVisor sentry seccomp-bpf filter program size" /></p>

<p>gVisor’s <code class="highlighter-rouge">seccomp-bpf</code> program size is reduced by over a factor of 4 using the
optimizations below.</p>

<details>

  

    <h3 id="optimizing-cbpf-jumps">Optimizing cBPF jumps</h3>

    <p>The <a href="#cbpf-limitations">limitations of cBPF jump instructions described earlier</a>
means that typical BPF bytecode rendering code will usually favor unconditional
jumps even when they are not necessary. However, they can be optimized after the
fact.</p>

  

  <p>Typical BPF bytecode rendering code for a simple condition is usually rendered
as follows:</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nx">jif</span> <span class="o">&lt;</span><span class="nx">condition</span><span class="o">&gt;</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">1</span>     <span class="c1">// If &lt;condition&gt; is true, continue,</span>
                          <span class="c1">//   otherwise skip over 1 instruction.</span>
<span class="nx">jmp</span> <span class="p">@</span><span class="nd">condition_was_true</span>   <span class="c1">// Unconditional jump to label @condition_was_true.</span>
<span class="nx">jmp</span> <span class="p">@</span><span class="nd">condition_was_false</span>  <span class="c1">// Unconditional jump to label @condition_was_false.</span>
</code></pre></div>  </div>

  <p>… or as follows:</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nx">jif</span> <span class="o">&lt;</span><span class="nx">condition</span><span class="o">&gt;</span><span class="p">,</span> <span class="mi">1</span><span class="p">,</span> <span class="mi">0</span>     <span class="c1">// If &lt;condition&gt; is true, jump by 1 instruction,</span>
                          <span class="c1">//   otherwise continue.</span>
<span class="nx">jmp</span> <span class="p">@</span><span class="nd">condition_was_false</span>  <span class="c1">// Unconditional jump to label @condition_was_false.</span>
<span class="c1">// Flow through here if the condition was true.</span>
</code></pre></div>  </div>

  <p>… In other words, the generated code always uses unconditional jumps, and
conditional jump offsets are always either 0 or 1 instructions forward. This is
because conditional jumps are limited to 8 bits (255 instructions), and it is
not always possible at BPF bytecode rendering time to know ahead of time that
the jump targets (<code class="highlighter-rouge">@condition_was_true</code>, <code class="highlighter-rouge">@condition_was_false</code>) will resolve to
an instruction that is close enough ahead that the offset would fit in 8 bits.
The safe thing to do is to always use an unconditional jump. Since unconditional
jump targets have 16 bits to play with, and <code class="highlighter-rouge">seccomp-bpf</code> programs are limited
to 4,096 instructions, it is always possible to encode a jump using an
unconditional jump instruction.</p>

  <p>But of course, the jump target often <em>does</em> fit in 8 bits. So gVisor looks over
the bytecode for optimization opportunities:</p>

  <ul>
    <li><strong>Conditional jumps that jump to unconditional jumps</strong> are rewritten to
their final destination, so long as this fits within the 255-instruction
conditional jump limit.</li>
    <li><strong>Unconditional jumps that jump to other unconditional jumps</strong> are rewritten
to their final destination.</li>
    <li><strong>Conditional jumps where both branches jump to the same instruction</strong> are
replaced by an unconditional jump to that instruction.</li>
    <li><strong>Unconditional jumps with a zero-instruction jump target</strong> are removed.</li>
  </ul>

  <p>The aim of these optimizations is to clean up after needless indirection that is
a byproduct of cBPF bytecode rendering code. Once they all have run, all jumps
are as tight as they can be.</p>

</details>

<details>

  

    <h3 id="removing-dead-code">Removing dead code</h3>

    <p>Because cBPF is a very restricted language, it is possible to determine with
certainty that some instructions can never be reached.</p>

  

  <p>In cBPF, each instruction either:</p>

  <ul>
    <li><strong>Flows</strong> forward (e.g. <code class="highlighter-rouge">load</code> operations, math operations).</li>
    <li><strong>Jumps</strong> by a fixed (immediate) number of instructions.</li>
    <li><strong>Stops</strong> the execution immediately (<code class="highlighter-rouge">return</code> instructions).</li>
  </ul>

  <p>Therefore, gVisor runs a simple program traversal algorithm. It creates a
bitfield with one bit per instruction, then traverses the program and all its
possible branches. Then, all instructions that were never traversed are removed
from the program, and all jump targets are updated to account for these
removals.</p>

  <p>In turn, this makes the program shorter, which makes more jump optimizations
possible.</p>

</details>

<details>

  

    <h3 id="redundant-loads">Removing redundant <code class="highlighter-rouge">load</code> instructions</h3>

    <p>cBPF programs filter system calls by inspecting their arguments. To do these
comparisons, this data must first be loaded into the cBPF VM registers. These
load operations can be optimized.</p>

  

  <p>cBPF’s conditional operations (e.g. “is equal to”, “is greater than”, etc.)
operate on a single 32-bit register called “A”. As such, a <code class="highlighter-rouge">seccomp-bpf</code> program
typically consists of many load operations (<code class="highlighter-rouge">load32</code>) that loads a 32-bit value
from a given offset of the <code class="highlighter-rouge">seccomp_data</code> struct into register A, then performs
a comparative operation on it to see if it matches the filter.</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="mi">00</span><span class="p">:</span> <span class="nx">load32</span> <span class="o">&lt;</span><span class="nx">offset</span><span class="o">&gt;</span>
<span class="mi">01</span><span class="p">:</span> <span class="nx">jif</span> <span class="o">&lt;</span><span class="nx">condition1</span><span class="o">&gt;</span><span class="p">,</span> <span class="p">@</span><span class="nd">condition1_was_true</span><span class="p">,</span> <span class="p">@</span><span class="nd">condition1_was_false</span>
<span class="mi">02</span><span class="p">:</span> <span class="nx">load32</span> <span class="o">&lt;</span><span class="nx">offset</span><span class="o">&gt;</span>
<span class="mi">03</span><span class="p">:</span> <span class="nx">jif</span> <span class="o">&lt;</span><span class="nx">condition2</span><span class="o">&gt;</span><span class="p">,</span> <span class="p">@</span><span class="nd">condition2_was_true</span><span class="p">,</span> <span class="p">@</span><span class="nd">condition2_was_false</span>
<span class="c1">// ...</span>
</code></pre></div>  </div>

  <p>But when a syscall rule is of the form “this syscall argument must be one of the
following values”, we don’t need to reload the same value (from the same offset)
multiple times. So gVisor looks for redundant loads like this, and removes them.</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="mi">00</span><span class="p">:</span> <span class="nx">load32</span> <span class="o">&lt;</span><span class="nx">offset</span><span class="o">&gt;</span>
<span class="mi">01</span><span class="p">:</span> <span class="nx">jif</span> <span class="o">&lt;</span><span class="nx">condition1</span><span class="o">&gt;</span><span class="p">,</span> <span class="p">@</span><span class="nd">condition1_was_true</span><span class="p">,</span> <span class="p">@</span><span class="nd">condition1_was_false</span>
<span class="mi">02</span><span class="p">:</span> <span class="nx">jif</span> <span class="o">&lt;</span><span class="nx">condition2</span><span class="o">&gt;</span><span class="p">,</span> <span class="p">@</span><span class="nd">condition2_was_true</span><span class="p">,</span> <span class="p">@</span><span class="nd">condition2_was_false</span>
<span class="c1">// ...</span>
</code></pre></div>  </div>

  <p>Note that syscall arguments are <strong>64-bit</strong> values, whereas the A register is
only 32-bits wide. Therefore, asserting that a syscall argument matches a
predicate usually involves at least 2 <code class="highlighter-rouge">load32</code> operations on different offsets,
thereby making this optimization useless for the “this syscall argument must be
one of the following values” case. We’ll get back to that.</p>

</details>

<details>

  

    <h3 id="minimizing-the-number-of-return-instructions">Minimizing the number of <code class="highlighter-rouge">return</code> instructions</h3>

    <p>A typical syscall filter program consists of many predicates which return either
“allowed” or “rejected”. These are encoded in the bytecode as either <code class="highlighter-rouge">return</code>
instructions, or jumps to <code class="highlighter-rouge">return</code> instructions. These instructions can show up
dozens or hundreds of times in the cBPF bytecode in quick succession, presenting
an optimization opportunity.</p>

  

  <p>Since two <code class="highlighter-rouge">return</code> instructions with the same immediate return code are exactly
equivalent to one another, it is possible to rewrite jumps to all <code class="highlighter-rouge">return</code>
instructions that return “allowed” to go to a single <code class="highlighter-rouge">return</code> instruction that
returns this code, and similar for “rejected”, so long as the jump offsets fit
within the limits of conditional jumps (255 instructions). In turn, this makes
the program shorter, and therefore makes more jump optimizations possible.</p>

  <p>To implement this optimization, gVisor first replaces all unconditional jump
instructions that go to <code class="highlighter-rouge">return</code> statements with a copy of that <code class="highlighter-rouge">return</code>
statement. This removes needless indirection.</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code>    <span class="nx">Original</span> <span class="nx">bytecode</span>                      <span class="nx">New</span> <span class="nx">bytecode</span>
<span class="mi">00</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">1</span>                        <span class="mi">00</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">1</span>
<span class="mi">01</span><span class="p">:</span> <span class="nx">jmp</span> <span class="p">@</span><span class="nd">good</span>                    <span class="o">--&gt;</span>   <span class="mi">01</span><span class="p">:</span> <span class="k">return</span> <span class="nx">allowed</span>
<span class="mi">02</span><span class="p">:</span> <span class="nx">jmp</span> <span class="p">@</span><span class="nd">bad</span>                     <span class="o">--&gt;</span>   <span class="mi">02</span><span class="p">:</span> <span class="k">return</span> <span class="nx">rejected</span>
<span class="p">...</span>                                    <span class="p">...</span>
<span class="mi">10</span><span class="p">:</span> <span class="nx">jge</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">1</span>                        <span class="mi">10</span><span class="p">:</span> <span class="nx">jge</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">1</span>
<span class="mi">11</span><span class="p">:</span> <span class="nx">jmp</span> <span class="p">@</span><span class="nd">good</span>                    <span class="o">--&gt;</span>   <span class="mi">11</span><span class="p">:</span> <span class="k">return</span> <span class="nx">allowed</span>
<span class="mi">12</span><span class="p">:</span> <span class="nx">jmp</span> <span class="p">@</span><span class="nd">bad</span>                     <span class="o">--&gt;</span>   <span class="mi">12</span><span class="p">:</span> <span class="k">return</span> <span class="nx">rejected</span>
<span class="p">...</span>                                    <span class="p">...</span>
<span class="mi">100</span> <span class="p">[@</span><span class="nd">good</span><span class="p">]:</span> <span class="k">return</span> <span class="nx">allowed</span>            <span class="mi">100</span> <span class="p">[@</span><span class="nd">good</span><span class="p">]:</span> <span class="k">return</span> <span class="nx">allowed</span>
<span class="mi">101</span> <span class="p">[@</span><span class="nd">bad</span><span class="p">]:</span>  <span class="k">return</span> <span class="nx">rejected</span>           <span class="mi">101</span> <span class="p">[@</span><span class="nd">bad</span><span class="p">]:</span>  <span class="k">return</span> <span class="nx">rejected</span>
</code></pre></div>  </div>

  <p>gVisor then searches for <code class="highlighter-rouge">return</code> statements which can be entirely removed by
seeing if it is possible to rewrite the rest of the program to jump or flow
through to an equivalent <code class="highlighter-rouge">return</code> statement (without making the program longer
in the process). In the above example:</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code>    <span class="nx">Original</span> <span class="nx">bytecode</span>                      <span class="nx">New</span> <span class="nx">bytecode</span>
<span class="mi">00</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">1</span>                  <span class="o">--&gt;</span>   <span class="mi">00</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">99</span><span class="p">,</span> <span class="mi">100</span>   <span class="c1">// Targets updated</span>
<span class="mi">01</span><span class="p">:</span> <span class="k">return</span> <span class="nx">allowed</span>                     <span class="mi">01</span><span class="p">:</span> <span class="k">return</span> <span class="nx">allowed</span>   <span class="c1">// Now dead code</span>
<span class="mi">02</span><span class="p">:</span> <span class="k">return</span> <span class="nx">reject</span>                      <span class="mi">02</span><span class="p">:</span> <span class="k">return</span> <span class="nx">rejected</span>  <span class="c1">// Now dead code</span>
<span class="p">...</span>                                    <span class="p">...</span>
<span class="mi">10</span><span class="p">:</span> <span class="nx">jge</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">1</span>                  <span class="o">--&gt;</span>   <span class="mi">10</span><span class="p">:</span> <span class="nx">jge</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">89</span><span class="p">,</span> <span class="mi">90</span>    <span class="c1">// Targets updated</span>
<span class="mi">11</span><span class="p">:</span> <span class="nx">jmp</span> <span class="p">@</span><span class="nd">good</span>                          <span class="mi">11</span><span class="p">:</span> <span class="k">return</span> <span class="nx">allowed</span>   <span class="c1">// Now dead code</span>
<span class="mi">12</span><span class="p">:</span> <span class="nx">jmp</span> <span class="p">@</span><span class="nd">bad</span>                           <span class="mi">12</span><span class="p">:</span> <span class="k">return</span> <span class="nx">rejected</span>  <span class="c1">// Now dead code</span>
<span class="p">...</span>                                    <span class="p">...</span>
<span class="mi">100</span> <span class="p">[@</span><span class="nd">good</span><span class="p">]:</span> <span class="k">return</span> <span class="nx">allowed</span>            <span class="mi">100</span> <span class="p">[@</span><span class="nd">good</span><span class="p">]:</span> <span class="k">return</span> <span class="nx">allowed</span>
<span class="mi">101</span> <span class="p">[@</span><span class="nd">bad</span><span class="p">]:</span>  <span class="k">return</span> <span class="nx">rejected</span>           <span class="mi">101</span> <span class="p">[@</span><span class="nd">bad</span><span class="p">]:</span>  <span class="k">return</span> <span class="nx">rejected</span>
</code></pre></div>  </div>

  <p>Finally, the dead code removal pass cleans up the dead <code class="highlighter-rouge">return</code> statements and
the program becomes shorter.</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code>    <span class="nx">Original</span> <span class="nx">bytecode</span>                      <span class="nx">New</span> <span class="nx">bytecode</span>
<span class="mi">00</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">99</span><span class="p">,</span> <span class="mi">100</span>               <span class="o">--&gt;</span>   <span class="mi">00</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">95</span><span class="p">,</span> <span class="mi">96</span>  <span class="c1">// Targets updated</span>
<span class="mi">01</span><span class="p">:</span> <span class="k">return</span> <span class="nx">allowed</span>               <span class="o">--&gt;</span>   <span class="cm">/* Removed */</span>
<span class="mi">02</span><span class="p">:</span> <span class="k">return</span> <span class="nx">reject</span>                <span class="o">--&gt;</span>   <span class="cm">/* Removed */</span>
<span class="p">...</span>                                    <span class="p">...</span>
<span class="mi">10</span><span class="p">:</span> <span class="nx">jge</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">89</span><span class="p">,</span> <span class="mi">90</span>                <span class="o">--&gt;</span>   <span class="mi">08</span><span class="p">:</span> <span class="nx">jge</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">87</span><span class="p">,</span> <span class="mi">88</span>  <span class="c1">// Targets updated</span>
<span class="mi">11</span><span class="p">:</span> <span class="k">return</span> <span class="nx">allowed</span>               <span class="o">--&gt;</span>   <span class="cm">/* Removed */</span>
<span class="mi">12</span><span class="p">:</span> <span class="k">return</span> <span class="nx">rejected</span>              <span class="o">--&gt;</span>   <span class="cm">/* Removed */</span>
<span class="p">...</span>                                    <span class="p">...</span>
<span class="mi">100</span> <span class="p">[@</span><span class="nd">good</span><span class="p">]:</span> <span class="k">return</span> <span class="nx">allowed</span>            <span class="mi">96</span> <span class="p">[@</span><span class="nd">good</span><span class="p">]:</span> <span class="k">return</span> <span class="nx">allowed</span>
<span class="mi">101</span> <span class="p">[@</span><span class="nd">bad</span><span class="p">]:</span>  <span class="k">return</span> <span class="nx">rejected</span>           <span class="mi">97</span> <span class="p">[@</span><span class="nd">bad</span><span class="p">]:</span>  <span class="k">return</span> <span class="nx">rejected</span>
</code></pre></div>  </div>

  <p>While this search is expensive to perform, in a program full of predicates —
which is exactly what <code class="highlighter-rouge">seccomp-bpf</code> programs are — this approach massively
reduces program size.</p>

</details>

<h2 id="optimize-rulesets">Ruleset optimizations</h2>

<p>Bytecode-level optimizations are cool, but why stop here? gVisor now also
performs
<a href="https://github.com/google/gvisor/blob/master/pkg/seccomp/seccomp_optimizer.go"><code class="highlighter-rouge">seccomp</code> ruleset optimizations</a>.</p>

<p>In gVisor, a <code class="highlighter-rouge">seccomp</code> <code class="highlighter-rouge">RuleSet</code> is a mapping from syscall number to a logical
expression named <code class="highlighter-rouge">SyscallRule</code>, along with a <code class="highlighter-rouge">seccomp-bpf</code> action (e.g. “allow”)
if a syscall with a given number matches its <code class="highlighter-rouge">SyscallRule</code>.</p>

<details>

  

    <h3 id="basic-ruleset-simplifications">Basic ruleset simplifications</h3>

    <p>A <code class="highlighter-rouge">SyscallRule</code> is a predicate over the data contained in the <code class="highlighter-rouge">seccomp_data</code>
struct (beyond its <code class="highlighter-rouge">nr</code>). A trivial implementation is <code class="highlighter-rouge">MatchAll</code>, which simply
matches any <code class="highlighter-rouge">seccomp_data</code>. Other implementations include <code class="highlighter-rouge">Or</code> and <code class="highlighter-rouge">And</code> (which
do what they sound like), and <code class="highlighter-rouge">PerArg</code> which applies predicates to each specific
argument of a <code class="highlighter-rouge">seccomp_data</code>, and forms the meat of actual syscall filtering
rules. Some basic simplifications are already possible with these building
blocks.</p>

  

  <p>gVisor implements the following basic optimizers, which look like they may be
useless on their own but end up simplifying the logic of the more complex
optimizer described in other sections quite a bit:</p>

  <ul>
    <li><code class="highlighter-rouge">Or</code> and <code class="highlighter-rouge">And</code> rules with a single predicate within them are replaced with
just that predicate.</li>
    <li>Duplicate predicates within <code class="highlighter-rouge">Or</code> and <code class="highlighter-rouge">And</code> rules are removed.</li>
    <li><code class="highlighter-rouge">Or</code> rules within <code class="highlighter-rouge">Or</code> rules are flattened.</li>
    <li><code class="highlighter-rouge">And</code> rules within <code class="highlighter-rouge">And</code> rules are flattened.</li>
    <li>An <code class="highlighter-rouge">Or</code> rule which contains a <code class="highlighter-rouge">MatchAll</code> predicate is replaced with
<code class="highlighter-rouge">MatchAll</code>.</li>
    <li><code class="highlighter-rouge">MatchAll</code> predicates within <code class="highlighter-rouge">And</code> rules are removed.</li>
    <li><code class="highlighter-rouge">PerArg</code> rules with <code class="highlighter-rouge">MatchAll</code> predicates for each argument are replaced
with a rule that matches anything.</li>
  </ul>

  <p>As with the bytecode-level optimizations, gVisor runs these in a loop until the
structure of the rules no longer change. With the basic optimizations above,
this silly-looking rule:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">Or</span><span class="p">{</span>
    <span class="n">Or</span><span class="p">{</span>
        <span class="n">And</span><span class="p">{</span>
            <span class="n">MatchAll</span><span class="p">,</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">EqualTo</span><span class="p">(</span><span class="m">2</span><span class="p">),</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="p">},</span>
        <span class="n">MatchAll</span><span class="p">,</span>
    <span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">EqualTo</span><span class="p">(</span><span class="m">2</span><span class="p">),</span> <span class="n">AnyValue</span><span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">EqualTo</span><span class="p">(</span><span class="m">2</span><span class="p">),</span> <span class="n">AnyValue</span><span class="p">},</span>
<span class="p">}</span>
</code></pre></div>  </div>

  <p>… is simplified down to just <code class="highlighter-rouge">PerArg{AnyValue, EqualTo(2), AnyValue}</code>.</p>

</details>

<details>

  

    <h3 id="extracting-repeated-argument-matchers">Extracting repeated argument matchers</h3>

    <p>This is the main optimization that gVisor performs on rulesets. gVisor looks for
common argument matchers that are repeated across all combinations of <em>other</em>
argument matchers in branches of an <code class="highlighter-rouge">Or</code> rule. It removes them from these
<code class="highlighter-rouge">PerArg</code> rules, and <code class="highlighter-rouge">And</code> the overall syscall rule with a single instance of
that argument matcher. Sound complicated? Let’s look at an example.</p>

  

  <p>In the
<a href="https://github.com/google/gvisor/blob/master/runsc/boot/filter/config/">gVisor Sentry <code class="highlighter-rouge">seccomp-bpf</code> configuration</a>,
these are the rules for the
<a href="https://man7.org/linux/man-pages/man2/fcntl.2.html"><code class="highlighter-rouge">fcntl(2)</code> system call</a>:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">rules</span> <span class="o">=</span> <span class="o">...</span><span class="p">(</span><span class="k">map</span><span class="p">[</span><span class="kt">uintptr</span><span class="p">]</span><span class="n">SyscallRule</span><span class="p">{</span>
    <span class="n">SYS_FCNTL</span><span class="o">:</span> <span class="n">Or</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span>
            <span class="n">NonNegativeFD</span><span class="p">,</span>
            <span class="n">EqualTo</span><span class="p">(</span><span class="n">F_GETFL</span><span class="p">),</span>
        <span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span>
            <span class="n">NonNegativeFD</span><span class="p">,</span>
            <span class="n">EqualTo</span><span class="p">(</span><span class="n">F_SETFL</span><span class="p">),</span>
        <span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span>
            <span class="n">NonNegativeFD</span><span class="p">,</span>
            <span class="n">EqualTo</span><span class="p">(</span><span class="n">F_GETFD</span><span class="p">),</span>
        <span class="p">},</span>
    <span class="p">},</span>
<span class="p">})</span>
</code></pre></div>  </div>

  <p>… This means that for the <code class="highlighter-rouge">fcntl(2)</code> system call, <code class="highlighter-rouge">seccomp_data.args[0]</code> may
be any non-negative number, <code class="highlighter-rouge">seccomp_data.args[1]</code> may be either <code class="highlighter-rouge">F_GETFL</code>,
<code class="highlighter-rouge">F_SETFL</code>, or <code class="highlighter-rouge">F_GETFD</code>, and all other <code class="highlighter-rouge">seccomp_data</code> fields may be any value.</p>

  <p>If rendered naively in BPF, this would iterate over each branch of the <code class="highlighter-rouge">Or</code>
expression, and re-check the <code class="highlighter-rouge">NonNegativeFD</code> each time. Clearly wasteful.
Conceptually, the ideal expression is something like this:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">rules</span> <span class="o">=</span> <span class="o">...</span><span class="p">(</span><span class="k">map</span><span class="p">[</span><span class="kt">uintptr</span><span class="p">]</span><span class="n">SyscallRule</span><span class="p">{</span>
    <span class="n">SYS_FCNTL</span><span class="o">:</span> <span class="n">PerArg</span><span class="p">{</span>
        <span class="n">NonNegativeFD</span><span class="p">,</span>
        <span class="n">AnyOf</span><span class="p">(</span><span class="n">F_GETFL</span><span class="p">,</span> <span class="n">F_SETFL</span><span class="p">,</span> <span class="n">F_GETFD</span><span class="p">),</span>
    <span class="p">},</span>
<span class="p">})</span>
</code></pre></div>  </div>

  <p>… But going through all the syscall rules to look for this pattern would be
quite tedious, and some of them are actually <code class="highlighter-rouge">Or</code>‘d from multiple
<code class="highlighter-rouge">map[uintptr]SyscallRule</code> in different files (e.g. platform-dependent syscalls),
so they cannot be all specified in a single location with a single predicate on
<code class="highlighter-rouge">seccomp_data.args[1]</code>. So gVisor needs to detect this programmatically at
optimization time.</p>

  <p>Conceptually, gVisor goes from:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">Or</span><span class="p">{</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">B1</span><span class="p">,</span> <span class="n">C1</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">B1</span><span class="p">,</span> <span class="n">C1</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">B2</span><span class="p">,</span> <span class="n">C2</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">B2</span><span class="p">,</span> <span class="n">C2</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">B3</span><span class="p">,</span> <span class="n">C3</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">B3</span><span class="p">,</span> <span class="n">C3</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
<span class="p">}</span>
</code></pre></div>  </div>

  <p>… to (after one pass):</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">And</span><span class="p">{</span>
    <span class="n">Or</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
    <span class="p">},</span>
    <span class="n">Or</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B1</span><span class="p">,</span> <span class="n">C1</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B1</span><span class="p">,</span> <span class="n">C1</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B2</span><span class="p">,</span> <span class="n">C2</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B2</span><span class="p">,</span> <span class="n">C2</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B3</span><span class="p">,</span> <span class="n">C3</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B3</span><span class="p">,</span> <span class="n">C3</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
    <span class="p">},</span>
<span class="p">}</span>
</code></pre></div>  </div>

  <p>Then the <a href="#basic-ruleset-simplifications">basic optimizers</a> will kick in and
detect duplicate <code class="highlighter-rouge">PerArg</code> rules in <code class="highlighter-rouge">Or</code> expressions, and delete them:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">And</span><span class="p">{</span>
    <span class="n">Or</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
    <span class="p">},</span>
    <span class="n">Or</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B1</span><span class="p">,</span> <span class="n">C1</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B2</span><span class="p">,</span> <span class="n">C2</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B3</span><span class="p">,</span> <span class="n">C3</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
    <span class="p">},</span>
<span class="p">}</span>
</code></pre></div>  </div>

  <p>… Then, on the next pass, the second inner <code class="highlighter-rouge">Or</code> rule gets recursively
optimized:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">And</span><span class="p">{</span>
    <span class="n">Or</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
    <span class="p">},</span>
    <span class="n">And</span><span class="p">{</span>
        <span class="n">Or</span><span class="p">{</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
        <span class="p">},</span>
        <span class="n">Or</span><span class="p">{</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B1</span><span class="p">,</span> <span class="n">C1</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B2</span><span class="p">,</span> <span class="n">C2</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B3</span><span class="p">,</span> <span class="n">C3</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="p">},</span>
    <span class="p">},</span>
<span class="p">}</span>
</code></pre></div>  </div>

  <p>… which, after other basic optimizers clean this all up, finally becomes:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">And</span><span class="p">{</span>
    <span class="n">Or</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A1</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">A2</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
    <span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">,</span> <span class="n">D</span><span class="p">},</span>
    <span class="n">Or</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B1</span><span class="p">,</span> <span class="n">C1</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B2</span><span class="p">,</span> <span class="n">C2</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">B3</span><span class="p">,</span> <span class="n">C3</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
    <span class="p">},</span>
<span class="p">}</span>
</code></pre></div>  </div>

  <p>This has turned what would be 24 comparisons into just 9:</p>

  <ul>
    <li><code class="highlighter-rouge">seccomp_data[0]</code> must either match predicate <code class="highlighter-rouge">A1</code> or <code class="highlighter-rouge">A2</code>.</li>
    <li><code class="highlighter-rouge">seccomp_data[3]</code> must match predicate <code class="highlighter-rouge">D</code>.</li>
    <li>At least one of the following must be true:
      <ul>
        <li><code class="highlighter-rouge">seccomp_data[1]</code> must match predicate <code class="highlighter-rouge">B1</code> and <code class="highlighter-rouge">seccomp_data[2]</code> must
match predicate <code class="highlighter-rouge">C1</code>.</li>
        <li><code class="highlighter-rouge">seccomp_data[1]</code> must match predicate <code class="highlighter-rouge">B2</code> and <code class="highlighter-rouge">seccomp_data[2]</code> must
match predicate <code class="highlighter-rouge">C2</code>.</li>
        <li><code class="highlighter-rouge">seccomp_data[1]</code> must match predicate <code class="highlighter-rouge">B3</code> and <code class="highlighter-rouge">seccomp_data[2]</code> must
match predicate <code class="highlighter-rouge">C3</code>.</li>
      </ul>
    </li>
  </ul>

  <p>To go back to our <code class="highlighter-rouge">fcntl(2)</code> example, the rules would therefore be rewritten to:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">rules</span> <span class="o">=</span> <span class="o">...</span><span class="p">(</span><span class="k">map</span><span class="p">[</span><span class="kt">uintptr</span><span class="p">]</span><span class="n">SyscallRule</span><span class="p">{</span>
    <span class="n">SYS_FCNTL</span><span class="o">:</span> <span class="n">And</span><span class="p">{</span>
        <span class="c">// Check for args[0] exclusively:</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">NonNegativeFD</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="c">// Check for args[1] exclusively:</span>
        <span class="n">Or</span><span class="p">{</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">EqualTo</span><span class="p">(</span><span class="n">F_GETFL</span><span class="p">)},</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">EqualTo</span><span class="p">(</span><span class="n">F_SETFL</span><span class="p">)},</span>
            <span class="n">PerArg</span><span class="p">{</span><span class="n">AnyValue</span><span class="p">,</span> <span class="n">EqualTo</span><span class="p">(</span><span class="n">F_GETFD</span><span class="p">)},</span>
        <span class="p">},</span>
    <span class="p">},</span>
<span class="p">})</span>
</code></pre></div>  </div>

  <p>… thus we’ve turned 6 comparisons into 4. But we can do better still!</p>

</details>

<details>

  

    <h3 id="extracting-repeated-32-bit-match-logic-from-64-bit-argument-matchers">Extracting repeated 32-bit match logic from 64-bit argument matchers</h3>

    <p>We can apply the same optimization, but down to the 32-bit matching logic that
underlies the 64-bit syscall argument matching predicates.</p>

  

  <p>As you may recall,
<a href="#cbpf-limitations">cBPF instructions are limited to 32-bit math</a>. This means
that when rendered, each of these argument comparisons are actually 2 operations
each: one for the first 32-bit half of the argument, and one for the second
32-bit half of the argument.</p>

  <p>Let’s look at the <code class="highlighter-rouge">F_GETFL</code>, <code class="highlighter-rouge">F_SETFL</code>, and <code class="highlighter-rouge">F_GETFD</code> constants:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">F_GETFL</span> <span class="o">=</span> <span class="m">0x3</span>
<span class="n">F_SETFL</span> <span class="o">=</span> <span class="m">0x4</span>
<span class="n">F_GETFD</span> <span class="o">=</span> <span class="m">0x1</span>
</code></pre></div>  </div>

  <p>The cBPF bytecode for checking the arguments of this syscall may therefore look
something like this:</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// Check for `seccomp_data.args[0]`:</span>
  <span class="mi">00</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">16</span>                <span class="c1">// Load the first 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[0]` into register A.</span>
  <span class="mi">01</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>           <span class="c1">// If A == 0, continue, otherwise jump to @bad.</span>
  <span class="mi">02</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">20</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[0]` into register A.</span>
  <span class="mi">03</span><span class="p">:</span> <span class="nx">jset</span> <span class="mh">0x80000000</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span><span class="p">,</span> <span class="mi">0</span> <span class="c1">// If A &amp; 0x80000000 != 0, jump to @bad,</span>
                               <span class="c1">//   otherwise continue.</span>

<span class="c1">// Check for `seccomp_data.args[1]`:</span>
  <span class="mi">04</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">24</span>                <span class="c1">// Load the first 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">05</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">next1</span>         <span class="c1">// If A == 0, continue, otherwise jump to @next1.</span>
  <span class="mi">06</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">28</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">07</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x3</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">next1</span>   <span class="c1">// If A == 0x3, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @next1.</span>

<span class="p">@</span><span class="nd">next1</span><span class="p">:</span>
  <span class="mi">08</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">24</span>                <span class="c1">// Load the first 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">09</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">next2</span>         <span class="c1">// If A == 0, continue, otherwise jump to @next2.</span>
  <span class="mi">10</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">28</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">11</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x4</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">next2</span>   <span class="c1">// If A == 0x3, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @next2.</span>

<span class="p">@</span><span class="nd">next2</span><span class="p">:</span>
  <span class="mi">12</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">24</span>                <span class="c1">// Load the first 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">13</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>           <span class="c1">// If A == 0, continue, otherwise jump to @bad.</span>
  <span class="mi">14</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">28</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">15</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x1</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>     <span class="c1">// If A == 0x1, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @bad.</span>

<span class="c1">// Good/bad jump targets for the checks above to jump to:</span>
<span class="p">@</span><span class="nd">good</span><span class="p">:</span>
  <span class="mi">16</span><span class="p">:</span> <span class="k">return</span> <span class="nx">ALLOW</span>
<span class="p">@</span><span class="nd">bad</span><span class="p">:</span>
  <span class="mi">17</span><span class="p">:</span> <span class="k">return</span> <span class="nx">REJECT</span>
</code></pre></div>  </div>

  <p>Clearly this could be better. The first 32 bits must be zero in all possible
cases. So the syscall argument value-matching primitives (e.g. <code class="highlighter-rouge">EqualTo</code>) may be
split into 2 32-bit value matchers:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">rules</span> <span class="o">=</span> <span class="o">...</span><span class="p">(</span><span class="k">map</span><span class="p">[</span><span class="kt">uintptr</span><span class="p">]</span><span class="n">SyscallRule</span><span class="p">{</span>
    <span class="n">SYS_FCNTL</span><span class="o">:</span> <span class="n">And</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">NonNegativeFD</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">Or</span><span class="p">{</span>
            <span class="n">PerArg</span><span class="p">{</span>
                <span class="n">AnyValue</span><span class="p">,</span>
                <span class="n">splitMatcher</span><span class="p">{</span>
                    <span class="n">high32bits</span><span class="o">:</span> <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_GETFL</span> <span class="o">&amp;</span> <span class="m">0xffffffff00000000</span> <span class="c">/* = 0 */</span><span class="p">),</span>
                    <span class="n">low32bits</span><span class="o">:</span>  <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_GETFL</span> <span class="o">&amp;</span> <span class="m">0x00000000ffffffff</span> <span class="c">/* = 0x3 */</span><span class="p">),</span>
                <span class="p">},</span>
            <span class="p">},</span>
            <span class="n">PerArg</span><span class="p">{</span>
                <span class="n">AnyValue</span><span class="p">,</span>
                <span class="n">splitMatcher</span><span class="p">{</span>
                    <span class="n">high32bits</span><span class="o">:</span> <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_SETFL</span> <span class="o">&amp;</span> <span class="m">0xffffffff00000000</span> <span class="c">/* = 0 */</span><span class="p">),</span>
                    <span class="n">low32bits</span><span class="o">:</span>  <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_SETFL</span> <span class="o">&amp;</span> <span class="m">0x00000000ffffffff</span> <span class="c">/* = 0x4 */</span><span class="p">),</span>
                <span class="p">},</span>
            <span class="p">},</span>
            <span class="n">PerArg</span><span class="p">{</span>
                <span class="n">AnyValue</span><span class="p">,</span>
                <span class="n">splitMatcher</span><span class="p">{</span>
                    <span class="n">high32bits</span><span class="o">:</span> <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_GETFD</span> <span class="o">&amp;</span> <span class="m">0xffffffff00000000</span> <span class="c">/* = 0 */</span><span class="p">),</span>
                    <span class="n">low32bits</span><span class="o">:</span>  <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_GETFD</span> <span class="o">&amp;</span> <span class="m">0x00000000ffffffff</span> <span class="c">/* = 0x1 */</span><span class="p">),</span>
                <span class="p">},</span>
            <span class="p">},</span>
        <span class="p">},</span>
    <span class="p">},</span>
<span class="p">})</span>
</code></pre></div>  </div>

  <p>gVisor then applies the same optimization as earlier, but this time going into
each 32-bit half of each argument. This means it can extract the
<code class="highlighter-rouge">EqualTo32Bits(0)</code> matcher from the <code class="highlighter-rouge">high32bits</code> part of each <code class="highlighter-rouge">splitMatcher</code> and
move it up to the <code class="highlighter-rouge">And</code> expression like so:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">rules</span> <span class="o">=</span> <span class="o">...</span><span class="p">(</span><span class="k">map</span><span class="p">[</span><span class="kt">uintptr</span><span class="p">]</span><span class="n">SyscallRule</span><span class="p">{</span>
    <span class="n">SYS_FCNTL</span><span class="o">:</span> <span class="n">And</span><span class="p">{</span>
        <span class="n">PerArg</span><span class="p">{</span><span class="n">NonNegativeFD</span><span class="p">,</span> <span class="n">AnyValue</span><span class="p">},</span>
        <span class="n">PerArg</span><span class="p">{</span>
            <span class="n">AnyValue</span><span class="p">,</span>
            <span class="n">splitMatcher</span><span class="p">{</span>
                <span class="n">high32bits</span><span class="o">:</span> <span class="n">EqualTo32Bits</span><span class="p">(</span><span class="m">0</span><span class="p">),</span>
                <span class="n">low32bits</span><span class="o">:</span>  <span class="n">Any32BitsValue</span><span class="p">,</span>
            <span class="p">},</span>
        <span class="p">},</span>
        <span class="n">Or</span><span class="p">{</span>
            <span class="n">PerArg</span><span class="p">{</span>
                <span class="n">AnyValue</span><span class="p">,</span>
                <span class="n">splitMatcher</span><span class="p">{</span>
                    <span class="n">high32bits</span><span class="o">:</span> <span class="n">Any32BitsValue</span><span class="p">,</span>
                    <span class="n">low32bits</span><span class="o">:</span>  <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_GETFL</span> <span class="o">&amp;</span> <span class="m">0x00000000ffffffff</span> <span class="c">/* = 0x3 */</span><span class="p">),</span>
                <span class="p">},</span>
            <span class="p">},</span>
            <span class="n">PerArg</span><span class="p">{</span>
                <span class="n">AnyValue</span><span class="p">,</span>
                <span class="n">splitMatcher</span><span class="p">{</span>
                    <span class="n">high32bits</span><span class="o">:</span> <span class="n">Any32BitsValue</span><span class="p">,</span>
                    <span class="n">low32bits</span><span class="o">:</span>  <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_SETFL</span> <span class="o">&amp;</span> <span class="m">0x00000000ffffffff</span> <span class="c">/* = 0x4 */</span><span class="p">),</span>
                <span class="p">},</span>
            <span class="p">},</span>
            <span class="n">PerArg</span><span class="p">{</span>
                <span class="n">AnyValue</span><span class="p">,</span>
                <span class="n">splitMatcher</span><span class="p">{</span>
                    <span class="n">high32bits</span><span class="o">:</span> <span class="n">Any32BitsValue</span><span class="p">,</span>
                    <span class="n">low32bits</span><span class="o">:</span>  <span class="n">EqualTo32Bits</span><span class="p">(</span>
                      <span class="n">F_GETFD</span> <span class="o">&amp;</span> <span class="m">0x00000000ffffffff</span> <span class="c">/* = 0x1 */</span><span class="p">),</span>
                <span class="p">},</span>
            <span class="p">},</span>
        <span class="p">},</span>
    <span class="p">},</span>
<span class="p">})</span>
</code></pre></div>  </div>

  <p>This looks bigger as a tree, but keep in mind that the <code class="highlighter-rouge">AnyValue</code> and
<code class="highlighter-rouge">Any32BitsValue</code> matchers do not produce any bytecode. So now let’s render that
tree to bytecode:</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// Check for `seccomp_data.args[0]`:</span>
  <span class="mi">00</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">16</span>                <span class="c1">// Load the first 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[0]` into register A.</span>
  <span class="mi">01</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>           <span class="c1">// If A == 0, continue, otherwise jump to @bad.</span>
  <span class="mi">02</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">20</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[0]` into register A.</span>
  <span class="mi">03</span><span class="p">:</span> <span class="nx">jset</span> <span class="mh">0x80000000</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span><span class="p">,</span> <span class="mi">0</span> <span class="c1">// If A &amp; 0x80000000 != 0, jump to @bad,</span>
                               <span class="c1">//   otherwise continue.</span>

<span class="c1">// Check for `seccomp_data.args[1]`:</span>
  <span class="mi">04</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">24</span>                <span class="c1">// Load the first 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">05</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>           <span class="c1">// If A == 0, continue, otherwise jump to @bad.</span>
  <span class="mi">06</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">28</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">07</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x3</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">next1</span>   <span class="c1">// If A == 0x3, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @next1.</span>

<span class="p">@</span><span class="nd">next1</span><span class="p">:</span>
  <span class="mi">08</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">28</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">09</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x4</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">next2</span>   <span class="c1">// If A == 0x3, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @next2.</span>

<span class="p">@</span><span class="nd">next2</span><span class="p">:</span>
  <span class="mi">10</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">28</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">11</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x1</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>     <span class="c1">// If A == 0x1, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @bad.</span>

<span class="c1">// Good/bad jump targets for the checks above to jump to:</span>
<span class="p">@</span><span class="nd">good</span><span class="p">:</span>
  <span class="mi">12</span><span class="p">:</span> <span class="k">return</span> <span class="nx">ALLOW</span>
<span class="p">@</span><span class="nd">bad</span><span class="p">:</span>
  <span class="mi">13</span><span class="p">:</span> <span class="k">return</span> <span class="nx">REJECT</span>
</code></pre></div>  </div>

  <p>This is where the bytecode-level optimization to remove redundant loads
<a href="#redundant-loads">described earlier</a> finally becomes relevant. We don’t need to
load the second 32 bits of <code class="highlighter-rouge">seccomp_data.args[1]</code> multiple times in a row:</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// Check for `seccomp_data.args[0]`:</span>
  <span class="mi">00</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">16</span>                <span class="c1">// Load the first 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[0]` into register A.</span>
  <span class="mi">01</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>           <span class="c1">// If A == 0, continue, otherwise jump to @bad.</span>
  <span class="mi">02</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">20</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[0]` into register A.</span>
  <span class="mi">03</span><span class="p">:</span> <span class="nx">jset</span> <span class="mh">0x80000000</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span><span class="p">,</span> <span class="mi">0</span> <span class="c1">// If A &amp; 0x80000000 != 0, jump to @bad,</span>
                               <span class="c1">//   otherwise continue.</span>

<span class="c1">// Check for `seccomp_data.args[1]`:</span>
  <span class="mi">04</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">24</span>                <span class="c1">// Load the first 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">05</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>           <span class="c1">// If A == 0, continue, otherwise jump to @bad.</span>
  <span class="mi">06</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">28</span>                <span class="c1">// Load the second 32 bits of</span>
                               <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
  <span class="mi">07</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x3</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">next1</span>   <span class="c1">// If A == 0x3, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @next1.</span>

<span class="p">@</span><span class="nd">next1</span><span class="p">:</span>
  <span class="mi">08</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x4</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">next2</span>   <span class="c1">// If A == 0x3, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @next2.</span>

<span class="p">@</span><span class="nd">next2</span><span class="p">:</span>
  <span class="mi">09</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mh">0x1</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>     <span class="c1">// If A == 0x1, jump to @good,</span>
                               <span class="c1">//   otherwise jump to @bad.</span>

<span class="c1">// Good/bad jump targets for the checks above to jump to:</span>
<span class="p">@</span><span class="nd">good</span><span class="p">:</span>
  <span class="mi">10</span><span class="p">:</span> <span class="k">return</span> <span class="nx">ALLOW</span>
<span class="p">@</span><span class="nd">bad</span><span class="p">:</span>
  <span class="mi">11</span><span class="p">:</span> <span class="k">return</span> <span class="nx">REJECT</span>
</code></pre></div>  </div>

  <p>Of course, in practice the <code class="highlighter-rouge">@good</code>/<code class="highlighter-rouge">@bad</code> jump targets would also be unified
with rules from other system call filters in order to cut down on those too. And
by having reduced the number of instructions in each individual filtering rule,
the jumps to these targets can be deduplicated against that many more rules.</p>

  <p>This example demonstrates how <strong>optimizations build on top of each other</strong>,
making each optimization more likely to make <em>other</em> optimizations useful in
turn.</p>

</details>

<h2 id="other-optimizations">Other optimizations</h2>

<p>Beyond these, gVisor also has the following minor optimizations.</p>

<details>

  

    <h3 id="making-futex2-rules-faster">Making <code class="highlighter-rouge">futex(2)</code> rules faster</h3>

    <p><a href="https://man7.org/linux/man-pages/man2/futex.2.html"><code class="highlighter-rouge">futex(2)</code></a> is by far the
most-often-called system call that gVisor calls as part of its operation. It is
used for synchronization, so it needs to be very efficient.</p>

  

  <p>Its rules used to look like this:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">SYS_FUTEX</span><span class="o">:</span> <span class="n">Or</span><span class="p">{</span>
    <span class="n">PerArg</span><span class="p">{</span>
        <span class="n">AnyValue</span><span class="p">,</span>
        <span class="n">EqualTo</span><span class="p">(</span><span class="n">FUTEX_WAIT</span> <span class="o">|</span> <span class="n">FUTEX_PRIVATE_FLAG</span><span class="p">),</span>
    <span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span>
        <span class="n">AnyValue</span><span class="p">,</span>
        <span class="n">EqualTo</span><span class="p">(</span><span class="n">FUTEX_WAKE</span> <span class="o">|</span> <span class="n">FUTEX_PRIVATE_FLAG</span><span class="p">),</span>
    <span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span>
        <span class="n">AnyValue</span><span class="p">,</span>
        <span class="n">EqualTo</span><span class="p">(</span><span class="n">FUTEX_WAIT</span><span class="p">),</span>
    <span class="p">},</span>
    <span class="n">PerArg</span><span class="p">{</span>
        <span class="n">AnyValue</span><span class="p">,</span>
        <span class="n">EqualTo</span><span class="p">(</span><span class="n">FUTEX_WAKE</span><span class="p">),</span>
    <span class="p">},</span>
<span class="p">},</span>
</code></pre></div>  </div>

  <p>Essentially a 4-way <code class="highlighter-rouge">Or</code> between 4 different values allowed for
<code class="highlighter-rouge">seccomp_data.args[1]</code>. This is all well and good, and the above optimizations
already optimize this down to the minimum amount of <code class="highlighter-rouge">jeq</code> comparison operations.</p>

  <p>But looking at the actual bit values of the <code class="highlighter-rouge">FUTEX_*</code> constants above:</p>

  <div class="language-go highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">FUTEX_WAIT</span>         <span class="o">=</span> <span class="m">0x00</span>
<span class="n">FUTEX_WAKE</span>         <span class="o">=</span> <span class="m">0x01</span>
<span class="n">FUTEX_PRIVATE_FLAG</span> <span class="o">=</span> <span class="m">0x80</span>
</code></pre></div>  </div>

  <p>… We can see that this is equivalent to checking that no bits other than
<code class="highlighter-rouge">0x01</code> and <code class="highlighter-rouge">0x80</code> may be set. Turns out that cBPF has an instruction for that.
This is now optimized down to two comparison operations:</p>

  <div class="language-javascript highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="mi">01</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">24</span>                     <span class="c1">// Load the first 32 bits of</span>
                                  <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
<span class="mi">02</span><span class="p">:</span> <span class="nx">jeq</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span>                <span class="c1">// If A == 0, continue,</span>
                                  <span class="c1">//   otherwise jump to @bad.</span>
<span class="mi">03</span><span class="p">:</span> <span class="nx">load32</span> <span class="mi">28</span>                     <span class="c1">// Load the second 32 bits of</span>
                                  <span class="c1">//   `seccomp_data.args[1]` into register A.</span>
<span class="mi">04</span><span class="p">:</span> <span class="nx">jset</span> <span class="mh">0xffffff7e</span><span class="p">,</span> <span class="p">@</span><span class="nd">bad</span><span class="p">,</span> <span class="p">@</span><span class="nd">good</span>  <span class="c1">// If A &amp; ^(0x01 | 0x80) != 0, jump to @bad,</span>
                                  <span class="c1">//   otherwise jump to @good.</span>
</code></pre></div>  </div>

</details>

<details>

  

    <h3 id="optimizing-non-negative-fd-checks">Optimizing non-negative FD checks</h3>

    <p>A lot of syscall arguments are file descriptors (FD numbers), which we need to
filter efficiently.</p>

  

  <p>An FD is a 32-bit positive integer, but is passed as a 64-bit value as all
syscall arguments are. Instead of doing a “less than” operation, we can simply
turn it into a bitwise check. We simply check that the first half of the 64-bit
value is zero, and that the 31st bit of the second half of the 64-bit value is
not set.</p>

</details>

<details>

  

    <h3 id="enforcing-consistency-of-argument-wise-matchers">Enforcing consistency of argument-wise matchers</h3>

    <p>When one syscall argument is checked consistently across all branches of an
<code class="highlighter-rouge">Or</code>, enforcing that this is the case ensures that the
<a href="#optimize-rulesets">optimization for such matchers</a> remains effective.</p>

  

  <p>The <code class="highlighter-rouge">ioctl(2)</code> system call takes an FD as one of its arguments. Since it is a
“grab bag” of a system call, gVisor’s rules for <code class="highlighter-rouge">ioctl(2)</code> were similarly spread
across many files and rules, and not all of them checked that the FD argument
was non-negative; some of them simply accepted any value for the FD argument.</p>

  <p>Before this optimization work, this meant that the BPF program did less work for
the rules which didn’t check the value of the FD argument. However, now that
gVisor <a href="#optimize-rulesets">optimizes repeated argument-wise matchers</a>, it is
now actually <em>cheaper</em> if <em>all</em> <code class="highlighter-rouge">ioctl(2)</code> rules verify the value of the FD
argument consistently, as that argument check can be performed exactly once for
all possible branches of the <code class="highlighter-rouge">ioctl(2)</code> rules. So now gVisor has a test that
verifies that this is the case. This is a good example that shows that
<strong>optimization work can lead to improved security</strong> due to the efficiency gains
that comes from applying security checks consistently.</p>

</details>

<h2 id="secbench"><code class="highlighter-rouge">secbench</code>: Benchmarking <code class="highlighter-rouge">seccomp-bpf</code> programs</h2>

<p>To measure the effectiveness of the above improvements, measuring gVisor
performance itself would be very difficult, because each improvement is a rather
tiny part of the syscall hot path. At the scale of each of these optimizations,
we need to zoom in a bit more.</p>

<p>So now gVisor has
<a href="https://github.com/google/gvisor/blob/master/test/secbench/">tooling for benchmarking <code class="highlighter-rouge">seccomp-bpf</code> programs</a>.
It works by taking a
<a href="https://github.com/google/gvisor/blob/master/runsc/boot/filter/filter_bench_test.go">cBPF program along with several possible syscalls</a>
to try with it. It runs a subprocess that installs this program as <code class="highlighter-rouge">seccomp-bpf</code>
filter for itself, replacing all actions (other than “approve syscall”) with
“return error” in order to avoid crashing. Then it measures the latency of each
syscall. This is then measured against the latency of the very same syscalls in
a subprocess that has an empty <code class="highlighter-rouge">seccomp-bpf</code> (i.e. the only instruction within
it is <code class="highlighter-rouge">return ALLOW</code>).</p>

<p>Let’s measure the effect of the above improvements on a gVisor-like workload.</p>

<details>

  

    <h3 id="modeling-gvisor-seccomp-bpf-behavior-for-benchmarking">Modeling gVisor <code class="highlighter-rouge">seccomp-bpf</code> behavior for benchmarking</h3>

    <p>This can be done by running gVisor under <code class="highlighter-rouge">ptrace</code> to see what system calls the
gVisor process is doing.</p>

  

  <p>Note that <code class="highlighter-rouge">ptrace</code> here refers to the mechanism by which we can inspect the
system call that the gVisor Sentry is making. This is distinct from the system
calls the <em>sandboxed</em> application is doing. It has also nothing to do with
gVisor’s former “ptrace” platform.</p>

  <p>For example, after running a Postgres benchmark inside gVisor with Systrap, the
<code class="highlighter-rouge">ptrace</code> tool generated the following summary table:</p>

  <div class="language-markdown highlighter-rouge"><div class="highlight"><pre class="highlight"><code>% time     seconds  usecs/call     calls    errors syscall
<span class="p">------ ----------- ----------- --------- --------- ----------------</span>
<span class="p"> 62.</span>10  431.799048         496    870063     46227 futex
<span class="p">  4.</span>23   29.399526         106    275649        38 nanosleep
<span class="p">  0.</span>87    6.032292          37    160201           sendmmsg
<span class="p">  0.</span>28    1.939492          16    115769           fstat
<span class="p"> 27.</span>96  194.415343        2787     69749       137 ppoll
<span class="p">  1.</span>05    7.298717         315     23131           fsync
<span class="p">  0.</span>06    0.446930          31     14096           pwrite64
<span class="p">  3.</span>37   23.398106        1907     12266         9 epoll_pwait
<span class="p">  0.</span>00    0.019711           9      1991         6 close
<span class="p">  0.</span>02    0.116739          82      1414           tgkill
<span class="p">  0.</span>01    0.068481          48      1414       201 rt_sigreturn
<span class="p">  0.</span>02    0.147048         104      1413           getpid
<span class="p">  0.</span>01    0.045338          41      1080           write
<span class="p">  0.</span>01    0.039876          37      1056           read
<span class="p">  0.</span>00    0.015637          18       836        24 openat
<span class="p">  0.</span>01    0.066699          81       814           madvise
<span class="p">  0.</span>00    0.029757         111       267           fallocate
<span class="p">  0.</span>00    0.006619          15       420           pread64
<span class="p">  0.</span>00    0.013334          35       375           sched_yield
<span class="p">  0.</span>00    0.008112         114        71           pwritev2
<span class="p">  0.</span>00    0.003005          57        52           munmap
<span class="p">  0.</span>00    0.000343          18        19         6 unlinkat
<span class="p">  0.</span>00    0.000249          15        16           shutdown
<span class="p">  0.</span>00    0.000100           8        12           getdents64
<span class="p">  0.</span>00    0.000045           4        10           newfstatat
...
<span class="p">------ ----------- ----------- --------- --------- ----------------</span>
<span class="p">100.</span>00  695.311111         447   1552214     46651 total
</code></pre></div>  </div>

  <p>To mimic the syscall profile of this gVisor sandbox from the perspective of
<code class="highlighter-rouge">seccomp-bpf</code> overhead, we need to have it call these system calls with the same
relative frequency. Therefore, the dimension that matters here isn’t <code class="highlighter-rouge">time</code> or
<code class="highlighter-rouge">seconds</code> or even <code class="highlighter-rouge">usecs/call</code>; it is actually just the number of system calls
(<code class="highlighter-rouge">calls</code>). In graph form:</p>

  <p><img alt="Sentry syscall profile" src="/assets/images/2024-02-01-gvisor-seccomp-sentry-syscall-profile.png" title="Sentry syscall profile" /></p>

  <p>The Pareto distribution of system calls becomes immediately clear.</p>

</details>

<h3 id="seccomp-bpf-filtering-overhead-reduction"><code class="highlighter-rouge">seccomp-bpf</code> filtering overhead reduction</h3>

<p>The <code class="highlighter-rouge">secbench</code> library lets us take the top 10 system calls and measure their
<code class="highlighter-rouge">seccomp-bpf</code> filtering overhead individually, as well as building a weighted
aggregate of their overall overhead. Here are the numbers from before and after
the filtering optimizations described in this post:</p>

<p><img alt="Systrap seccomp-bpf performance" src="/assets/images/2024-02-01-gvisor-seccomp-systrap.png" title="Systrap seccomp-bpf performance" /></p>

<p>The <code class="highlighter-rouge">nanosleep(2)</code> system call is a bit of an oddball here. Unlike the others,
this system call causes the current thread to be descheduled. To make the
results more legible, here is the same data with the duration normalized to the
<code class="highlighter-rouge">seccomp-bpf</code> filtering overhead from before optimizations:</p>

<p><img alt="Systrap seccomp-bpf performance" src="/assets/images/2024-02-01-gvisor-seccomp-systrap-normalized.png" title="Systrap seccomp-bpf performance" /></p>

<p>This shows that most system calls have had their filtering overhead reduced, but
others haven’t significantly changed (10% or less change in either direction).
This is to be expected: those that have not changed are the ones that are
cacheable: <code class="highlighter-rouge">nanosleep(2)</code>, <code class="highlighter-rouge">fstat(2)</code>, <code class="highlighter-rouge">ppoll(2)</code>, <code class="highlighter-rouge">fsync(2)</code>, <code class="highlighter-rouge">pwrite64(2)</code>,
<code class="highlighter-rouge">close(2)</code>, <code class="highlighter-rouge">getpid(2)</code>. The non-cacheable syscalls
<a href="#structure">which have dedicated checks</a> before the main BST, <code class="highlighter-rouge">futex(2)</code> and
<code class="highlighter-rouge">sendmmsg(2)</code>, experienced the biggest boost. Lastly, <code class="highlighter-rouge">epoll_pwait(2)</code> is
non-cacheable but doesn’t have a dedicated check before the main BST, so while
it still sees a small performance gain, that gain is lower than its
counterparts.</p>

<p>The “Aggregate” number comes from the <code class="highlighter-rouge">secbench</code> library and represents the
total time difference spent in system calls after calling them using weighted
randomness. It represents the average system call overhead that a Sentry using
Systrap would incur. Therefore, per these numbers, these optimizations removed
~29% from gVisor’s overall <code class="highlighter-rouge">seccomp-bpf</code> filtering overhead.</p>

<p>Here is the same data for KVM, which has a slightly different syscall profile
with <code class="highlighter-rouge">ioctl(2)</code> and <code class="highlighter-rouge">rt_sigreturn(2)</code> being critical for performance:</p>

<p><img alt="KVM seccomp-bpf performance" src="/assets/images/2024-02-01-gvisor-seccomp-kvm-normalized.png" title="KVM seccomp-bpf performance" /></p>

<p>Lastly, let’s look at GPU workload performance. This benchmark enables gVisor’s
<a href="/blog/2023/06/20/gpu-pytorch-stable-diffusion/">experimental <code class="highlighter-rouge">nvproxy</code> feature for GPU support</a>.
What matters for this workload is <code class="highlighter-rouge">ioctl(2)</code> performance, as this is the system
call used to issue commands to the GPU. Here is the <code class="highlighter-rouge">seccomp-bpf</code> filtering
overhead of various CUDA control commands issued via <code class="highlighter-rouge">ioctl(2)</code>:</p>

<p><img alt="nvproxy ioctl seccomp-bpf performance" src="/assets/images/2024-02-01-gvisor-seccomp-nvproxy-ioctl.png" title="nvproxy ioctl seccomp-bpf performance" /></p>

<p>As <code class="highlighter-rouge">nvproxy</code> adds a lot of complexity to the <code class="highlighter-rouge">ioctl(2)</code> filtering rules, this is
where we see the most improvement from these optimizations.</p>

<h2 id="secfuzz"><code class="highlighter-rouge">secfuzz</code>: Fuzzing <code class="highlighter-rouge">seccomp-bpf</code> programs</h2>

<p>To ensure that the optimizations above don’t accidentally end up producing a
cBPF program that has different behavior from the unoptimized one used to do,
gVisor also has
<a href="https://github.com/google/gvisor/blob/master/test/secfuzz/"><code class="highlighter-rouge">seccomp-bpf</code> fuzz tests</a>.</p>

<p>Because gVisor knows which high-level filters went into constructing the
<code class="highlighter-rouge">seccomp-bpf</code> program, it also
<a href="https://github.com/google/gvisor/blob/master/runsc/boot/filter/filter_fuzz_test.go">automatically generates test cases</a>
from these filters, and the fuzzer verifies that each line and every branch of
the optimized cBPF bytecode is executed, and that the result is the same as
giving the same input to the unoptimized program.</p>

<p>(Line or branch coverage of the unoptimized program is not enforceable, because
without optimizations, the bytecode contains many redundant checks for which
later branches can never be reached.)</p>

<h2 id="optimizing-in-gvisor-seccomp-bpf-filtering">Optimizing in-gVisor <code class="highlighter-rouge">seccomp-bpf</code> filtering</h2>

<p>gVisor supports sandboxed applications adding <code class="highlighter-rouge">seccomp-bpf</code> filters onto
themselves, and
<a href="https://github.com/google/gvisor/blob/master/pkg/bpf/interpreter.go">implements its own cBPF interpreter</a>
for this purpose.</p>

<p>Because the cBPF bytecode-level optimizations are lossless and are generally
applicable to any cBPF program, they are applied onto programs uploaded by
sandboxed applications to make filter evaluation faster in gVisor itself.</p>

<p>Additionally, gVisor removed the use of Go interfaces previously used for
loading data from the BPF “input” (i.e. the <code class="highlighter-rouge">seccomp_data</code> struct for
<code class="highlighter-rouge">seccomp-bpf</code>). This used to require an endianness-specific interface due to how
the BPF interpreter was used in two places in gVisor: network processing (which
uses network byte ordering), and <code class="highlighter-rouge">seccomp-bpf</code> (which uses native byte
ordering). This interface has now been replaced with
<a href="https://go.dev/doc/tutorial/generics">Go templates</a>, yielding to a 2x speedup
on <a href="#sample-filter">the reference simplistic <code class="highlighter-rouge">seccomp-bpf</code> filter</a>. The more
<code class="highlighter-rouge">load</code> instructions are in the filter, the better the effect. <em>(Naturally, this
also benefits network filtering performance!)</em></p>

<h3 id="gvisor-cbpf-interpreter-performance">gVisor cBPF interpreter performance</h3>

<p>The graph below shows the gVisor cBPF interpreter’s performance against three
sample filters: <a href="#sample-filter">the reference simplistic <code class="highlighter-rouge">seccomp-bpf</code> filter</a>,
and optimized vs unoptimized versions of gVisor’s own syscall filter (to
represent a more complex filter).</p>

<p><img alt="gVisor cBPF interpreter performance" src="/assets/images/2024-02-01-gvisor-seccomp-interpreter.png" title="gVisor cBPF interpreter performance" /></p>

<h3 id="seccomp-bpf-filter-result-caching-for-sandboxed-applications"><code class="highlighter-rouge">seccomp-bpf</code> filter result caching for sandboxed applications</h3>

<p>Lastly, gVisor now also implements an in-sandbox caching mechanism for syscalls
which do not depend on the <code class="highlighter-rouge">instruction_pointer</code> or syscall arguments. Unlike
Linux’s <code class="highlighter-rouge">seccomp-bpf</code> cache, gVisor’s implementation also handles actions other
than “allow”, and supports the entire set of cBPF instructions rather than the
restricted emulator Linux uses for caching evaluation purposes. This removes the
interpreter from the syscall hot path entirely for cacheable syscalls, further
speeding up system calls from applications that use <code class="highlighter-rouge">seccomp-bpf</code> within gVisor.</p>

<p><img alt="gVisor seccomp-bpf cache" src="/assets/images/2024-02-01-gvisor-seccomp-cache.png" title="gVisor seccomp-bpf cache" /></p>

<h2 id="faster-gvisor-startup-via-filter-precompilation">Faster gVisor startup via filter precompilation</h2>

<p>Due to these optimizations, the overall process of building the syscall
filtering rules, rendering them to cBPF bytecode, and running all the
optimizations, can take quite a while (~10ms). As one of gVisor’s strengths is
its startup latency being much faster than VMs, this is an unacceptable delay.</p>

<p>Therefore, gVisor now
<a href="https://github.com/google/gvisor/blob/master/pkg/seccomp/precompiledseccomp/">precompiles the rules</a>
to optimized cBPF bytecode for most possible gVisor configurations. This means
the <code class="highlighter-rouge">runsc</code> binary contains cBPF bytecode embedded in it for some subset of
popular configurations, and it will use this bytecode rather than compiling the
cBPF program from scratch during startup. If <code class="highlighter-rouge">runsc</code> is invoked with a
configuration for which the cBPF bytecode isn’t embedded in the <code class="highlighter-rouge">runsc</code> binary,
it will fall back to compiling the program from scratch.</p>

<details>

  

    <h3 id="dealing-with-dynamic-values-in-precompiled-rules">Dealing with dynamic values in precompiled rules</h3>

  

  <p>One challenge with this approach is to support parts of the configuration that
are only known at <code class="highlighter-rouge">runsc</code> startup time. For example, many filters act on a
specific file descriptor used for interacting with the <code class="highlighter-rouge">runsc</code> process after
startup over a Unix Domain Socket (called the “controller FD”). This is an
integer that is only known at runtime, so its value cannot be embedded inside
the optimized cBPF bytecode prepared at <code class="highlighter-rouge">runsc</code> compilation time.</p>

  <p>To address this, the <code class="highlighter-rouge">seccomp-bpf</code> precompilation tooling actually supports the
notions of 32-bit “variables”, and takes as input a function to render cBPF
bytecode for a given key-value mapping of variables to placeholder 32-bit
values. The precompiler calls this function <em>twice</em> with different arbitrary
value mappings for each variable, and observes where these arbitrary values show
up in the generated cBPF bytecode. This takes advantage of the fact that
gVisor’s <code class="highlighter-rouge">seccomp-bpf</code> program generation is deterministic.</p>

  <p>If the two cBPF programs are of the same byte length, and the placeholder values
show up at exactly the same byte offsets within the cBPF bytecode both times,
and the rest of the cBPF bytecode is byte-for-byte equivalent, the precompiler
has very high confidence that these offsets are where the 32-bit variables are
represented in the cBPF bytecode. It then stores these offsets as part of the
embedded data inside the <code class="highlighter-rouge">runsc</code> binary. Finally, at <code class="highlighter-rouge">runsc</code> execution time, the
bytes at these offsets are replaced with the now-known values of the variables.</p>

</details>

<h2 id="performance">OK that’s great and all, but is gVisor actually faster?</h2>

<p>The short answer is: <strong>yes, but only slightly</strong>. As we
<a href="#performance-considerations">established earlier</a>, <code class="highlighter-rouge">seccomp-bpf</code> is only a
small portion of gVisor’s total overhead, and the <code class="highlighter-rouge">secbench</code> benchmark shows
that this work only removes a portion of that overhead, so we should not expect
large differences here.</p>

<p>Let’s come back to the trusty ABSL build benchmark, with a new build of gVisor
with all of these optimizations turned on:</p>

<p><img alt="ABSL build performance" src="/assets/images/2024-02-01-gvisor-seccomp-absl-vs-unsandboxed.png" title="ABSL build performance" /></p>

<p>Let’s zoom the vertical axis in on the gVisor variants to see the difference
better:</p>

<p><img alt="ABSL build performance" src="/assets/images/2024-02-01-gvisor-seccomp-absl.png" title="ABSL build performance" /></p>

<p>This is about in line with what the earlier benchmarks showed. The initial
benchmarks showed that <code class="highlighter-rouge">seccomp-bpf</code> filtering overhead for this benchmark was
on the order of ~3.6% of total runtime, and the <code class="highlighter-rouge">secbench</code> benchmarks showed
that the optimizations reduced <code class="highlighter-rouge">seccomp-bpf</code> filter evaluation time by ~29% in
aggregate. The final absolute reduction in total runtime should then be around
~1%, which is just about what this result shows.</p>

<p>Other benchmarks show a similar pattern. Here’s gRPC build, similar to ABSL:</p>

<p><img alt="gRPC build performance" src="/assets/images/2024-02-01-gvisor-seccomp-grpc-vs-unsandboxed.png" title="gRPC build performance" /></p>

<p><img alt="gRPC build performance" src="/assets/images/2024-02-01-gvisor-seccomp-grpc.png" title="gRPC build performance" /></p>

<p>Here’s a benchmark running the
<a href="https://github.com/fastlane/fastlane">Ruby Fastlane</a> test suite:</p>

<p><img alt="Ruby Fastlane performance" src="/assets/images/2024-02-01-gvisor-seccomp-rubydev-vs-unsandboxed.png" title="Ruby Fastlane performance" /></p>

<p><img alt="Ruby Fastlane performance" src="/assets/images/2024-02-01-gvisor-seccomp-rubydev.png" title="Ruby Fastlane performance" /></p>

<p>Here’s the 50th percentile of nginx serving latency for an empty webpage.
<a href="https://www.prnewswire.com/news-releases/akamai-online-retail-performance-report-milliseconds-are-critical-300441498.html">Every microsecond counts when it comes to web serving</a>,
and here we’ve shaven off 20 of them.</p>

<p><img alt="nginx performance" src="/assets/images/2024-02-01-gvisor-seccomp-nginx-vs-unsandboxed.png" title="nginx performance" /></p>

<p><img alt="nginx performance" src="/assets/images/2024-02-01-gvisor-seccomp-nginx.png" title="nginx performance" /></p>

<p>CUDA workloads also get a boost from this work. Since their gVisor-related
overhead is already relatively small, <strong><code class="highlighter-rouge">seccomp-bpf</code> filtering makes up a
higher proportion of their overhead</strong>. Additionally, as the performance
improvements described in this post disproportionately help the <code class="highlighter-rouge">ioctl(2)</code>
system call, this cuts a larger portion of the <code class="highlighter-rouge">seccomp-bpf</code> filtering overhead
of these workload, since CUDA uses the <code class="highlighter-rouge">ioctl(2)</code> system call to communicate
with the GPU.</p>

<p><img alt="PyTorch performance" src="/assets/images/2024-02-01-gvisor-seccomp-pytorch-vs-unsandboxed.png" title="PyTorch performance" /></p>

<p><img alt="PyTorch performance" src="/assets/images/2024-02-01-gvisor-seccomp-pytorch.png" title="PyTorch performance" /></p>

<p>While some of these results may not seem like much in absolute terms, it’s
important to remember:</p>

<ul>
  <li>These improvements have resulted in gVisor being able to enforce <strong>more</strong>
<code class="highlighter-rouge">seccomp-bpf</code> filters than it previously could; gVisor’s <code class="highlighter-rouge">seccomp-bpf</code>
filter was nearly half the maximum <code class="highlighter-rouge">seccomp-bpf</code> program size, so it could
at most double in complexity. After optimizations, it is reduced to less
than a fourth of this size.</li>
  <li>These improvements allow the gVisor filters to <strong>scale better</strong>. This is
visible from the effects on <code class="highlighter-rouge">ioctl(2)</code> performance with <code class="highlighter-rouge">nvproxy</code> enabled.</li>
  <li>The resulting work has produced useful libraries for <code class="highlighter-rouge">seccomp-bpf</code> tooling
which may be helpful for other projects: testing, fuzzing, and benchmarking
<code class="highlighter-rouge">seccomp-bpf</code> filters.</li>
  <li>This overhead could not have been addressed in another way. Unlike other
areas of gVisor, such as network overhead or file I/O, overhead from the
host kernel evaluating <code class="highlighter-rouge">seccomp-bpf</code> filter lives outside of gVisor itself
and therefore it can only be improved upon by this type of work.</li>
</ul>

<h2 id="further-work">Further work</h2>

<p>One potential source of work is to look into the performance gap between no
<code class="highlighter-rouge">seccomp-bpf</code> filter at all versus performance with an empty <code class="highlighter-rouge">seccomp-bpf</code>
filter (equivalent to an all-cacheable filter). This points to a potential
inefficiency in the Linux kernel implementation of the <code class="highlighter-rouge">seccomp-bpf</code> cache.</p>

<p>Another potential point of improvement is to port over the optimizations that
went into searching for a syscall number into the
<a href="https://man7.org/linux/man-pages/man2/ioctl.2.html"><code class="highlighter-rouge">ioctl(2)</code> system call</a>. <code class="highlighter-rouge">ioctl(2)</code> is a “grab-bag” kind of system call,
used by many drivers and other subsets of the Linux kernel to extend the syscall
interface without using up valuable syscall numbers. For example, the
<a href="https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine">KVM</a> subsystem is
almost entirely controlled through <code class="highlighter-rouge">ioctl(2)</code> system calls issued against
<code class="highlighter-rouge">/dev/kvm</code> or against per-VM file descriptors.</p>

<p>For this reason, the first non-file-descriptor argument of <a href="https://man7.org/linux/man-pages/man2/ioctl.2.html"><code class="highlighter-rouge">ioctl(2)</code></a>
(“request”) usually encodes something analogous to what the syscall number
usually represents: the type of request made to the kernel. Currently, gVisor
performs a linear scan through all possible enumerations of this argument. This
is usually fine, but with features like <code class="highlighter-rouge">nvproxy</code> which massively expand this
list of possible values, this can take a long time. <code class="highlighter-rouge">ioctl</code> performance is also
critical for gVisor’s KVM platform. A binary search tree would make sense here.</p>

<p>gVisor welcomes further contributions to its <code class="highlighter-rouge">seccomp-bpf</code> machinery. Thanks for
reading!</p>

<div class="footnotes">
  <ol>
    <li id="fn:1">
      <p>cBPF does not have a canonical assembly-style representation. The
assembly-like code in this blog post is close to
<a href="https://man7.org/linux/man-pages/man8/bpfc.8.html">the one used in <code class="highlighter-rouge">bpfc</code></a>
but diverges in ways to make it hopefully clearer as to what’s happening,
and all code is annotated with <code class="highlighter-rouge">// comments</code>. <a class="reversefootnote" href="#fnref:1">&#8617;</a></p>
    </li>
  </ol>
</div>
