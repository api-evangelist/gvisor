---
title: "Releasing Systrap - A high-performance gVisor platform"
url: "/blog/2023/04/28/systrap-release/"
date: "2023-04-28T00:00:00-05:00"
author: "bogomolov"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<p>We are releasing a new gVisor platform: Systrap. Like the existing ptrace
platform, Systrap runs on most Linux machines out of the box without
virtualization. Unlike the ptrace platform, it’s fast 🚀. Go try it by adding
<code class="highlighter-rouge">--platform=systrap</code> to the runsc flags. If you want to know more about it, read
on.</p>

<!--/excerpt-->

<p>gVisor is a security boundary for arbitrary Linux processes. Boundaries do not
come for free, and gVisor imposes some performance overhead on sandboxed
applications. One of the most fundamental performance challenges with the
security model implemented by gVisor is system call interception, which is the
focus of this post.</p>

<p>To recap on the
<a href="https://gvisor.dev/docs/architecture_guide/security/#what-can-a-sandbox-do">security model</a>:
gVisor is an application kernel that implements the Linux ABI. This includes
system calls, signals, memory management, and more. For example, when a
sandboxed application calls
<a href="https://man7.org/linux/man-pages/man2/read.2.html"><code class="highlighter-rouge">read(2)</code></a>, it actually
transparently calls into
<a href="https://github.com/google/gvisor/blob/44e2d0fcfeb641f3b8013c3f93cacdae447cc0f1/pkg/sentry/syscalls/linux/sys_read_write.go#L36">gVisor’s implementation of this system call</a>
This minimizes the attack surface of the host kernel, because sandboxed programs
simply can’t make system calls directly to the host in the first place<sup id="fnref:1"><a class="footnote" href="#fn:1">1</a></sup>. This
interception happens through an internal layer called the Platform interface,
which we have written about in a previous
<a href="https://gvisor.dev/blog/2020/10/22/platform-portability/">blog post</a>. To handle
these interceptions, this interface must also create new address spaces,
allocate memory, and create execution contexts to run the workload.</p>

<p>gVisor had two platform implementations: KVM and ptrace. The KVM platform uses
the kernel’s KVM functionality to allow the Sentry to act as both guest OS and
VMM (Virtual machine monitor). It does system call interception just like a
normal virtual machine would. This gives good performance when using bare-metal
virtualization, but has a noticeable impact with nested virtualization. The
other obvious downside is that it requires support for nested virtualization in
the first place, which is not supported by all hardware (such as ARM CPUs) or
within some Cloud environments.</p>

<p>The ptrace platform was the alternative wherever KVM was not available. It works
through the
<a href="http://man7.org/linux/man-pages/man2/ptrace.2.html"><code class="highlighter-rouge">PTRACE_SYSEMU</code></a> action,
which makes the user process hand back execution to the sentry whenever it
encounters a system call. This is a clean method to achieve system call
interception in any environment, virtualized or not, except that it’s quite
slow. To see how slow, an unrealistic but highly illustrative benchmark to use
is the
<a href="https://github.com/google/gvisor/blob/108410638aa8480e82933870ba8279133f543d2b/test/perf/linux/getpid_benchmark.cc"><code class="highlighter-rouge">getpid</code> benchmark</a><sup id="fnref:2"><a class="footnote" href="#fn:2">2</a></sup>.
This benchmark runs the
<a href="https://man7.org/linux/man-pages/man2/getpid.2.html"><code class="highlighter-rouge">getpid(2)</code></a> system call
in a tight <code class="highlighter-rouge">while</code> loop. No useful application has this behavior, so it is not a
realistic benchmark, but it is well-suited to measure system call latency.</p>

<p><img alt="Figure 1" src="/assets/images/2023-04-28-getpid-ptrace-vs-native.svg" title="Getpid benchmark: ptrace vs. native Linux." width="100%" /></p>

<p>All <code class="highlighter-rouge">getpid</code> runs have been performed on a GCE n2-standard-4 VM, with the
<code class="highlighter-rouge">debian-11-bullseye-v20230306</code> image.</p>

<p>While this benchmark is not applicable to most real-world workloads, just about
any workload will generally suffer from high overhead in system call
performance. Since running in a virtualized environment is the default state for
most cloud users these days, it’s important that gVisor performs well in this
context. Systrap is the new platform targeting this important use case.</p>

<p>Systrap relies on multiple techniques to implement the Platform interface. Like
the ptrace platform, Systrap uses Linux’s ptrace subsystem to initialize
workload executor threads, which are started as child processes of the main
gVisor sentry process. Systrap additionally sets a very restrictive seccomp
filter, installs a custom signal handler, and allocates chunks of memory shared
between user threads and runsc sentry. This shared memory is what serves as the
main form of communication between the sentry and sandboxed programs: whenever
the sandboxed process attempts to execute a system call, it triggers a <code class="highlighter-rouge">SIGSYS</code>
signal which is handled by our signal handler. The signal handler in turn
populates shared memory regions, and requests the sentry to handle the requested
system call. This alone proved to be faster than using <code class="highlighter-rouge">PTRACE_SYSEMU</code>, as
demonstrated by the <code class="highlighter-rouge">getpid</code> benchmark:</p>

<p><img alt="Figure 2" src="/assets/images/2023-04-28-getpid-ptrace-vs-systrap-unoptimized.svg" title="Getpid benchmark: ptrace vs. Systrap." width="100%" /></p>

<p>Can we make it even faster? Recall what the main purpose of our signal handler
is: to send a request to the sentry via shared memory. To do that, the sandboxed
process must first incur the overhead of executing the seccomp filter<sup id="fnref:3"><a class="footnote" href="#fn:3">3</a></sup>, and
then generating a full signal stack before being able to run the signal handler.
What if there was a way to simply have the sandboxed process jump to another
user-space function when it wanted to perform a system call? Well, turns out,
there is<sup id="fnref:4"><a class="footnote" href="#fn:4">4</a></sup>! There is a popular x86 instruction pattern that’s used to perform
system calls, and it goes a little something like this: <strong><code class="highlighter-rouge">mov sysno, %eax;
syscall</code></strong>. The size of the mov instruction is 5 bytes and the size of the
syscall instruction is 2 bytes. Luckily this is just enough space to fit in a
<strong><code class="highlighter-rouge">jmp *%gs:offset</code></strong> instruction. When the signal handler sees this instruction
pattern, it signals to the sentry that the original instructions can be replaced
with a <strong><code class="highlighter-rouge">jmp</code></strong> to trampoline code that performs the same function as the
regular <code class="highlighter-rouge">SIGSYS</code> signal handler. The system call number is not lost, but rather
encoded in the offset. The results are even more impressive:</p>

<p><img alt="Figure 3" src="/assets/images/2023-04-28-getpid-ptrace-vs-systrap-opt.svg" title="Getpid benchmark: ptrace vs. Optimized Systrap." width="100%" /></p>

<p>As mentioned, the <code class="highlighter-rouge">getpid</code> benchmark is not representative of real-world
performance. To get a better picture of the magnitude of improvement, here are
some real-world workloads:</p>

<ul>
  <li>The
<a href="https://github.com/google/gvisor/blob/master/blob/master/test/benchmarks/fs/bazel_test.go">Build ABSL benchmark</a>
measures compilation performance by compiling
<a href="https://abseil.io/">abseil.io</a>; this is a highly system call dependent
workload due to needing to do a lot of I/O filesystem operations (gVisor’s
file system overhead is also dependent upon file system isolation it
implements, which is something you can learn about
<a href="https://gvisor.dev/docs/user_guide/filesystem/">here</a>).</li>
  <li>The
<a href="https://github.com/google/gvisor/blob/master/blob/master/test/benchmarks/media/ffmpeg_test.go">ffmpeg benchmark</a>
runs a multimedia processing tool, to perform video stream encoding/decoding
for example; this workload does not require a significant amount of system
calls and there are very few userspace to kernel mode switches.</li>
  <li>The
<a href="https://github.com/google/gvisor/blob/master/blob/master/test/benchmarks/ml/tensorflow_test.go">Tensorflow benchmark</a>
trains a variety of machine learning models on CPU; the system-call usage of
this workload is in between compilation and ffmpeg, due to needing to
retrieve training and validation data, but the majority of time is still
spent just running userspace computations.</li>
  <li>Finally, the Redis benchmark performs SET RPC calls with 5 concurrent
clients, measures the latency that each call takes to execute, and reports
the median (scaled by 250,000 to fit the graph’s axis); this workload is
heavily bounded by system call performance due to high network stack usage.</li>
</ul>

<p><img alt="Figure 4" src="/assets/images/2023-04-28-systrap-sample-workloads.svg" title="Comparison of sample workloads running on ptrace, Systrap, and native Linux." width="100%" /></p>

<p>Systrap will replace the ptrace platform by September 2023 and become the
default. Until then, we are working really hard to make it production-ready,
which includes working on additional performance and stability improvements, and
making sure we maintain a high bar for security through targeted fuzz-testing
for Systrap specifically.</p>

<p>In the meantime, we would like gVisor users to try it out, and give us feedback!
If you run gVisor using ptrace today (either by specifying <code class="highlighter-rouge">--platform ptrace</code>
or not specifying the <code class="highlighter-rouge">--platform</code> flag at all), or you use the KVM platform with
nested virtualization, switching to Systrap should be a drop-in performance
upgrade. All you have to do is specify <code class="highlighter-rouge">--platform systrap</code> to runsc. If you
encounter any issues, please let us know at
<a href="https://github.com/google/gvisor/issues">gvisor.dev/issues</a>.
<br />
<br /></p>

<hr />

<!-- mdformat off(Footnotes need to be separated by linebreaks to be rendered) -->

<!-- mdformat on -->
<div class="footnotes">
  <ol>
    <li id="fn:1">
      <p>Even if the sandbox itself is compromised, it will still be bound by
several defense-in-depth layers, including a restricted set of seccomp
filters. You can find more details here:
<a href="https://gvisor.dev/blog/2020/09/18/containing-a-real-vulnerability/">https://gvisor.dev/blog/2020/09/18/containing-a-real-vulnerability/</a>. <a class="reversefootnote" href="#fnref:1">&#8617;</a></p>
    </li>
    <li id="fn:2">
      <p>Once the system call has been intercepted by gVisor (or in the case of
Linux, once the process has entered kernel-mode), actually executing the
getpid system call itself is very fast, so this benchmark effectively
measures single-thread syscall-interception overhead. <a class="reversefootnote" href="#fnref:2">&#8617;</a></p>
    </li>
    <li id="fn:3">
      <p>Seccomp filters are known to have a “not insubstantial” overhead:
<a href="https://lwn.net/Articles/656307/">https://lwn.net/Articles/656307/</a>. <a class="reversefootnote" href="#fnref:3">&#8617;</a></p>
    </li>
    <li id="fn:4">
      <p>On the x86_64 architecture. ARM does not have this optimization as of the
time of writing. <a class="reversefootnote" href="#fnref:4">&#8617;</a></p>
    </li>
  </ol>
</div>
