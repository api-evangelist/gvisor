---
title: "Scaling Agentic-RL Sandboxes to the Millions with gVisor at Tencent"
url: "/blog/2026/04/23/scaling-agentic-rl-sandboxes-to-the-millions-with-gvisor-at-tencent/"
date: "2026-04-23T00:00:00-05:00"
author: "yifengtan"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<blockquote>
  <p><em>This article was contributed by <a href="https://www.tencent.com/">Tencent</a>. Yifeng
Tan, Hua Liu, and Hui Chen are engineers at Tencent, responsible for the
internal container infrastructure.</em></p>
</blockquote>

<p>As LLMs evolve from chat interfaces to autonomous agents, building a robust and
secure isolation environment becomes a necessity. We chose
<a href="https://gvisor.dev">gVisor</a> as the default sandbox for our Agentic-RL
scenarios. Today, we run millions of gVisor sandboxes daily for Agentic-RL
training in production, and that scale continues to grow. After more than
<strong>74,000</strong> side-by-side comparisons between <code class="highlighter-rouge">runsc</code> (gVisor) and <code class="highlighter-rouge">runc</code>
(unsandboxed/Linux), combined with targeted fixes driven by real-world
workloads, we have essentially closed the execution correctness gap with <code class="highlighter-rouge">runc</code>,
fully meeting our production-grade business requirements. During this process,
we successfully investigated and resolved gVisor compatibility issues that
accounted for approximately <strong>1.7%</strong> of all test cases.</p>

<p>This post focuses on CPU-centric code execution and testing workloads. We will
discuss gVisor compatibility verification and highlight representative issues,
skipping implementation details like GPU support, image distribution, or cluster
scheduling. We aim to answer three questions:</p>

<ol>
  <li>Why choose gVisor?</li>
  <li>Why doesn’t manual compatibility verification scale?</li>
  <li>How can AI agents analyze compatibility issues, what do typical failures
look like, and what best practices have we established?</li>
</ol>

<!--/excerpt-->

<h2 id="background-why-agentic-rl-needs-gvisor">Background: Why Agentic-RL Needs gVisor</h2>

<p>Over the past two years, benchmarks like SWE-bench have turned “Agents fixing
bugs in real code repositories” from a research concept into an engineering
reality. The agent behavioral model has evolved from <strong>static code generation</strong>
to <strong>dynamic environmental interaction</strong>, spanning the entire lifecycle of
dependency resolution, execution, test feedback, and iterative debugging. We
don’t just need “an environment that runs Docker,” but rather a sandbox that
strictly constrains the kernel attack surface while remaining lightweight and
easy to deploy at scale. <a href="https://gvisor.dev">gVisor</a> is a great fit for this
scenario:</p>

<ul>
  <li>It implements an application-level kernel in user space, intercepting and
re-implementing system calls, significantly reducing the attack surface
where containers directly interact with the host kernel. Its isolation has
been well-recognized by the industry.</li>
  <li>It integrates naturally with existing Docker/Kubernetes infrastructure,
avoiding the need for an entirely new guest kernel operation and maintenance
system.</li>
  <li>Compared to microVM solutions—which must run on bare-metal hosts—gVisor can
run inside regular VMs, making it significantly cheaper while remaining more
flexible with lower startup and resource costs. This makes it far better
suited for large-scale deployments of sandbox containers.</li>
  <li>It is also more friendly to GPU scenarios, facilitating integration with
existing heterogeneous computing environments.</li>
</ul>

<p>However, <strong>re-implementing the Linux ABI means its compatibility must be
rigorously validated.</strong> In an Agentic-RL scenario where “any project can run and
any environment can appear,” compatibility can’t rely on intuition. It requires
large-scale verification against real workloads.</p>

<h2 id="challenge-verifying-tens-of-thousands-of-cases-cannot-rely-entirely-on-manual-effort">Challenge: Verifying Tens of Thousands of Cases Cannot Rely Entirely on Manual Effort</h2>

<p>Compatibility issues are rarely simple. Analyzing a typical SWE-related failure
usually requires answering several questions at once:</p>

<ol>
  <li>Is this failure unique to <code class="highlighter-rouge">runsc</code> (gVisor), or does it also fail under
<code class="highlighter-rouge">runc</code>?</li>
  <li>If it only fails under gVisor, is it a semantic inconsistency in the Linux
ABI, missing procfs / sysfs, file system behavioral differences, or a TOCTOU
(Time-of-Check to Time-of-Use) race condition amplified by system call
overhead?</li>
  <li>What is the actual behavior of the Linux kernel? At which layer did gVisor
deviate?</li>
  <li>Should this issue be addressed by patching gVisor, modifying the test case,
adjusting configurations, or simply avoiding a certain way of running?</li>
</ol>

<p>Engineers can handle a handful of cases manually. But across these datasets, we
are dealing with hundreds of thousands of real-world project instances, over a
dozen programming languages, and numerous build systems (Gradle, Maven, CMake,
Cargo, pip, npm, sbt, SwiftPM). Manual triage simply doesn’t scale.</p>

<p>To solve this, we brought AI coding agents into the verification pipeline to act
as <strong>compatibility analysts</strong>. The process breaks down into four layers:</p>

<ul>
  <li><strong>Baseline Comparison Layer</strong>: Run the same set of test cases in parallel
under <code class="highlighter-rouge">runc</code> and <code class="highlighter-rouge">runsc</code>, collecting complete execution logs and exit
statuses.</li>
  <li><strong>Difference Filtering Layer</strong>: Filter out environmental noise and
non-deterministic outputs unrelated to the runtime, preserving samples that
only fail under gVisor.</li>
  <li><strong>AI Diagnostic Layer</strong>: LLMs output structured root cause analysis reports
by combining logs and relevant source code.</li>
  <li><strong>Decision Routing Layer</strong>: Route the reports into gVisor bugs, user-space
race conditions, environmental differences, or test case issues, providing
suggestions for fixes or workarounds.</li>
</ul>

<p>This creates a neat closed loop: <strong>AI analyzing its own runtime environment.</strong></p>

<pre><code class="language-mermaid">graph TD
    A[Baseline Comparison Layer] --&gt;|Run under runc/runsc in parallel&lt;br&gt;Collect logs &amp; exit status| B(Difference Filtering Layer)
    B --&gt;|Filter environmental noise&lt;br&gt;Keep gVisor-specific failures| C{AI Diagnostic Layer}
    C --&gt;|Combine logs &amp; source code&lt;br&gt;Output structured root cause report| D[Decision Routing Layer]

    D --&gt;|gVisor bug| E[Submit community fix]
    D --&gt;|User-space race condition| F[Workaround strategy]
    D --&gt;|Environmental difference| G[Adjust environment]
    D --&gt;|Test case issue| H[Fix test case]

    subgraph AI-Driven Compatibility Verification Framework
    A
    B
    C
    D
    end
</code></pre>

<p>In our workflow, every deeply analyzed case produces a structured document,
typically containing:</p>

<ul>
  <li>Failure symptoms and minimal reproduction method</li>
  <li><code class="highlighter-rouge">runc</code>/<code class="highlighter-rouge">runsc</code> comparison results</li>
  <li>Root cause classification: gVisor bug, missing feature, environmental
difference, test case issue, or race condition amplification</li>
  <li>Linux kernel behavior comparison and source code evidence</li>
  <li>Fixes or workaround suggestions</li>
  <li>Regression verification results</li>
</ul>

<p>To date, we have used AI to automatically analyze <strong>thousands of test cases
exhibiting behavioral differences</strong>. From these, we extracted and deeply
reviewed <strong>100+ highly representative cases</strong> across <strong>10+ programming
languages</strong> and multiple build systems. These cases help us determine not only
“whether gVisor is usable,” but also “who is actually to blame for a given
failure.”</p>

<h2 id="compatibility-landscape-boundaries-defined-by-batch-comparisons">Compatibility Landscape: Boundaries Defined by Batch Comparisons</h2>

<p>Looking at a small sample of failures makes it easy to misjudge gVisor’s
compatibility. Reliable conclusions require large-scale A/B testing.</p>

<p>Across 10 mainstream code execution datasets in our Agentic-RL infrastructure,
we’ve run <strong>74,379</strong> side-by-side comparisons between <code class="highlighter-rouge">runc</code> and <code class="highlighter-rouge">runsc</code>.</p>

<p>Please see the detailed data in the table below:</p>

<!-- mdformat off(no multiline table support in Kramdown) -->

<table>
  <thead>
    <tr>
      <th>Dataset</th>
      <th style="text-align: right;">Total cases</th>
      <th style="text-align: right;">Native <code class="highlighter-rouge">runc</code> accuracy</th>
      <th style="text-align: right;">gVisor pre-fix <code class="highlighter-rouge">runsc</code> accuracy</th>
      <th style="text-align: right;">gVisor post-fix <code class="highlighter-rouge">runsc</code> accuracy</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="highlighter-rouge">terminal-bench2</code></td>
      <td style="text-align: right;">89</td>
      <td style="text-align: right;">100.00%</td>
      <td style="text-align: right;">94.38%</td>
      <td style="text-align: right;">97.75%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/Multi-SWE-bench</code></td>
      <td style="text-align: right;">1,632</td>
      <td style="text-align: right;">70.16%</td>
      <td style="text-align: right;">72.49%</td>
      <td style="text-align: right;">73.16%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/Multi-SWE-RL</code></td>
      <td style="text-align: right;">7,046</td>
      <td style="text-align: right;">27.73%</td>
      <td style="text-align: right;">20.49%</td>
      <td style="text-align: right;">26.81%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/SWE-bench_Multilingual</code></td>
      <td style="text-align: right;">300</td>
      <td style="text-align: right;">93.00%</td>
      <td style="text-align: right;">92.67%</td>
      <td style="text-align: right;">93.00%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/SWE-bench_Not_Verified</code></td>
      <td style="text-align: right;">1,794</td>
      <td style="text-align: right;">97.94%</td>
      <td style="text-align: right;">97.94%</td>
      <td style="text-align: right;">97.94%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/SWE-bench_Pro</code></td>
      <td style="text-align: right;">731</td>
      <td style="text-align: right;">90.15%</td>
      <td style="text-align: right;">90.97%</td>
      <td style="text-align: right;">90.97%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/SWE-bench_Verified</code></td>
      <td style="text-align: right;">500</td>
      <td style="text-align: right;">100.00%</td>
      <td style="text-align: right;">99.60%</td>
      <td style="text-align: right;">100.00%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/SWE-Gym</code></td>
      <td style="text-align: right;">2,438</td>
      <td style="text-align: right;">86.75%</td>
      <td style="text-align: right;">88.27%</td>
      <td style="text-align: right;">88.27%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/SWE-rebench</code></td>
      <td style="text-align: right;">21,336</td>
      <td style="text-align: right;">83.33%</td>
      <td style="text-align: right;">83.33%</td>
      <td style="text-align: right;">83.77%</td>
    </tr>
    <tr>
      <td><code class="highlighter-rouge">swe-public/SWE-smith</code></td>
      <td style="text-align: right;">38,513</td>
      <td style="text-align: right;">99.37%</td>
      <td style="text-align: right;">97.42%</td>
      <td style="text-align: right;">99.31%</td>
    </tr>
    <tr>
      <td><strong>Total</strong></td>
      <td style="text-align: right;"><strong>74,379</strong></td>
      <td style="text-align: right;"><strong>86.78%</strong></td>
      <td style="text-align: right;"><strong>85.18%</strong></td>
      <td style="text-align: right;"><strong>86.91%</strong></td>
    </tr>
  </tbody>
</table>

<!-- mdformat on -->

<p>Three key takeaways emerge from this data:</p>

<ul>
  <li><strong><code class="highlighter-rouge">runsc</code> (gVisor) and <code class="highlighter-rouge">runc</code> (Linux native) are now effectively on par.</strong>
Across 74,379 runs, the correctness gap between <code class="highlighter-rouge">runsc</code> and <code class="highlighter-rouge">runc</code> is only
about <strong>0.13 percentage points</strong> (86.91% vs 86.78%). We also performed
retries and cross-validation on core datasets to rule out one-off flakiness.
We have improved <code class="highlighter-rouge">runsc</code>’s overall pass rate by approximately <strong>1.7
percentage points</strong>. This correctness gain largely stemmed from highly
concentrated failures in a small number of repositories—such as trio,
cloud-custodian, asciidoctor, and syncthing. Once a root cause was
identified, a single fix could often resolve hundreds of failing cases at
once.</li>
  <li><strong>Most “compatibility issues” should not be attributed to gVisor.</strong> The
table clearly demonstrates that even under the native <code class="highlighter-rouge">runc</code> environment,
there is an inherent failure rate of about 13% (with an average correctness
of 86.91%). These failures largely stem from flaky test code, build
environment deficiencies, or limitations within the underlying datasets.
Evaluating gVisor without a <code class="highlighter-rouge">runc</code> baseline could easily lead to
misattributing this 13% background failure rate as sandbox
incompatibilities.</li>
  <li><strong>The overall pass rate for Multi-SWE-RL is relatively low (around ~27% for
both runtimes).</strong> This is because our internal evaluation framework and some
case-execution methods are still being adapted, so it is not a standalone
compatibility problem in gVisor itself. The same bias affects both <code class="highlighter-rouge">runc</code>
and <code class="highlighter-rouge">runsc</code>, and therefore does not change the comparative conclusion.</li>
</ul>

<p>At the production scale we described earlier—<strong>millions of gVisor sandboxes
running every day</strong>—this data answers the real question: how much correctness do
we lose by replacing <code class="highlighter-rouge">runc</code> with <code class="highlighter-rouge">runsc</code>? The answer is: <strong>almost none.</strong></p>

<h2 id="representative-cases-six-types-of-issues-and-corresponding-fix-paths">Representative Cases: Six Types of Issues and Corresponding Fix Paths</h2>

<p>After filtering out cases where both <code class="highlighter-rouge">runc</code> and <code class="highlighter-rouge">runsc</code> failed simultaneously,
we conducted in-depth reviews of the remaining cases that exhibited behavioral
differences. Using these 100+ representative cases as a sample, their final
root-cause attribution can roughly be divided into the following categories:</p>

<!-- mdformat off(no multiline table support in Kramdown) -->

<table>
  <thead>
    <tr>
      <th>Root Cause Category</th>
      <th>Requires gVisor Modification?</th>
      <th>Typical Examples</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>Genuine gVisor bugs</strong></td>
      <td>Yes</td>
      <td><code class="highlighter-rouge">poll</code> incorrectly modifying <code class="highlighter-rouge">events</code>, inconsistent <code class="highlighter-rouge">execve</code> <code class="highlighter-rouge">errno</code> returns, <code class="highlighter-rouge">O_TRUNC</code> missing <code class="highlighter-rouge">IN_MODIFY</code> inotify events</td>
    </tr>
    <tr>
      <td><strong>Missing syscalls and virtual FS entries</strong></td>
      <td>Yes</td>
      <td>Unimplemented <code class="highlighter-rouge">copy_file_range</code> syscall, missing <code class="highlighter-rouge">/proc/sys/fs/pipe-max-size</code> configuration file, and absence of <code class="highlighter-rouge">/sys/dev/block</code> directory</td>
    </tr>
    <tr>
      <td><strong>Clock and timer precision differences</strong></td>
      <td>Partially</td>
      <td>CPU clock measurement precision, monotonic clock start value differences, sleep duration jitter</td>
    </tr>
    <tr>
      <td><strong>Amplified race conditions</strong></td>
      <td>No</td>
      <td>Gradle <code class="highlighter-rouge">clean test</code> parallel execution concurrency race, CMake <code class="highlighter-rouge">copy_if_different</code> TOCTOU race</td>
    </tr>
    <tr>
      <td><strong>Environmental or config differences</strong></td>
      <td>No</td>
      <td>External network access restrictions, JDK version mismatches, missing dynamic library paths</td>
    </tr>
    <tr>
      <td><strong>Test case issues</strong></td>
      <td>No</td>
      <td>Test execution order dependencies, underlying dataset defects, inherently flaky tests</td>
    </tr>
  </tbody>
</table>

<!-- mdformat on -->

<p>This shows that aside from genuine bugs or missing Linux ABI implementations in
gVisor, a significant portion of behavioral differences stems from
timing-sensitive tests, amplified user-space race conditions, or environmental
setup differences. This is especially crucial for Agentic-RL scenarios. Without
<code class="highlighter-rouge">runc</code> baselines and root cause analysis, these failures could easily be
misattributed as sandbox incompatibilities, leading to systematically
pessimistic conclusions.</p>

<p>These cases highlight the different types of compatibility issues we see in
Agentic-RL: system call semantic deviations, Linux ABI gaps, VFS implementation
gaps, and user-space race conditions.</p>

<h3 id="case-1-poll-behavior-inconsistency-causes-tmux-busy-loop">Case 1: <code class="highlighter-rouge">poll</code> Behavior Inconsistency Causes <code class="highlighter-rouge">tmux</code> Busy-Loop</h3>

<p>The evaluation cluster’s CPU utilization was unusually high. Investigation
revealed that the <code class="highlighter-rouge">tmux</code> server in each Agent container was pegging a CPU core:
under gVisor, CPU usage hovered at <strong>96.6%</strong>, while under <code class="highlighter-rouge">runc</code> it was
practically <strong>0%</strong>.</p>

<p>The root cause was <code class="highlighter-rouge">poll</code> write-back semantics. gVisor internally appended
<code class="highlighter-rouge">POLLHUP|POLLERR</code> to <code class="highlighter-rouge">pollfd.events</code> and wrote the entire <code class="highlighter-rouge">pollfd</code> struct back
to user space. Linux, however, only writes to <code class="highlighter-rouge">revents</code> and <strong>never modifies the
user’s original <code class="highlighter-rouge">events</code></strong>. This discrepancy prevented libevent from properly
removing closed file descriptors. Subsequent <code class="highlighter-rouge">poll</code> calls immediately returned
<code class="highlighter-rouge">POLLNVAL</code>, triggering a busy-loop.</p>

<p>After fixing this, the <code class="highlighter-rouge">tmux</code> CPU dropped from 96.6% to 0%. The impact goes far
beyond <code class="highlighter-rouge">tmux</code> — any program relying on the <code class="highlighter-rouge">libevent</code> <code class="highlighter-rouge">poll</code> backend benefits
from this.</p>

<h3 id="case-2-syncthing-test-case-exposes-two-independent-linux-abi-gaps-unimplemented-syscalls-or-virtual-files">Case 2: syncthing Test Case Exposes Two Independent Linux ABI Gaps (Unimplemented Syscalls or Virtual Files)</h3>

<p>In real-world workloads, it’s not uncommon for a single test case to hit two
independent gVisor compatibility issues at once. The <code class="highlighter-rouge">syncthing__syncthing-7828</code>
test case in the Multi-SWE-RL dataset passes normally under <code class="highlighter-rouge">runc</code>, but
consistently fails under <code class="highlighter-rouge">runsc</code>: 16 <code class="highlighter-rouge">TestCopyRange/*</code> subtests report <code class="highlighter-rouge">function
not implemented</code>, and another <code class="highlighter-rouge">TestTruncateFileOnly</code> times out waiting for an
inotify event.</p>

<p>This was caused by two independent Linux ABI gaps:</p>

<ul>
  <li><strong><code class="highlighter-rouge">copy_file_range</code> (syscall 326) was unimplemented.</strong> gVisor registered it
as <code class="highlighter-rouge">ErrorWithEvent(ENOSYS)</code>, so any program using this syscall received
<code class="highlighter-rouge">function not implemented</code>.</li>
  <li><strong><code class="highlighter-rouge">open(O_TRUNC)</code> was missing the <code class="highlighter-rouge">IN_MODIFY</code> inotify event.</strong> The Linux
kernel generates <code class="highlighter-rouge">IN_MODIFY</code> along the <code class="highlighter-rouge">do_open()</code> → <code class="highlighter-rouge">handle_truncate()</code> →
<code class="highlighter-rouge">notify_change()</code> path. However, gVisor VFS’s <code class="highlighter-rouge">OpenAt</code> only generated
<code class="highlighter-rouge">IN_OPEN</code>, causing programs listening for file modification events to be
“deaf” to the truncation action.</li>
</ul>

<p>The fix proceeded along two lines: implementing <code class="highlighter-rouge">copy_file_range</code> for both amd64
(326) and arm64 (285), and issuing <code class="highlighter-rouge">IN_MODIFY</code> at the VFS layer for <code class="highlighter-rouge">O_TRUNC</code> on
non-newly created files (skipping it for newly created files via the
<code class="highlighter-rouge">FMODE_CREATED</code> flag, consistent with Linux). After the fix, this test case
passed consistently under <code class="highlighter-rouge">runsc</code> just like under <code class="highlighter-rouge">runc</code>.</p>

<h3 id="case-3-gradle-clean-test-concurrency-raceroot-cause-in-user-space-not-gvisor">Case 3: Gradle clean test Concurrency Race—Root Cause in User Space, Not gVisor</h3>

<p>Not all issues that “only reproduce under gVisor” are actually gVisor bugs.</p>

<p>A Thunderbird Android test running <code class="highlighter-rouge">./gradlew clean test --max-workers 8
--continue</code> under <code class="highlighter-rouge">runsc</code> frequently failed with <code class="highlighter-rouge">Unable to delete directory</code>.
However, running it 7 times under <code class="highlighter-rouge">runc</code> yielded <strong>5 failures</strong> (71%). This
pointed to a user-space TOCTOU race condition in Gradle’s parallel build: one
subproject was still writing to <code class="highlighter-rouge">build/</code>, while another subproject’s clean task
was already trying to delete it.</p>

<p>gVisor’s higher system call overhead amplified the probability of triggering
this race, but it did not introduce new semantic errors. Splitting the command
into <code class="highlighter-rouge">./gradlew clean</code> and <code class="highlighter-rouge">./gradlew test ...</code> fixed it completely. <strong>This is
also a fundamental principle we follow in compatibility analysis: always use
<code class="highlighter-rouge">runc</code> as a baseline first, then determine whether the issue should be
attributed to the sandbox itself.</strong></p>

<h3 id="case-4-missing-procfs--sysfs-causes-real-applications-to-take-abnormal-paths">Case 4: Missing procfs / sysfs Causes Real Applications to Take Abnormal Paths</h3>

<p>Agentic-RL workloads are full of paths that are not usually tested in isolation
but are relied upon by real projects, such as <code class="highlighter-rouge">/proc/sys/fs/pipe-max-size</code>,
<code class="highlighter-rouge">/proc/sys/kernel/randomize_va_space</code>, <code class="highlighter-rouge">/sys/dev/block</code>, <code class="highlighter-rouge">/proc/[pid]/fdinfo</code>,
etc. Once missing, these typically manifest as <code class="highlighter-rouge">ENOENT</code> or cause upper-layer
libraries to take abnormal code paths.</p>

<p>These are usually cheap to fix by wiring up static files or directory
structures. They perfectly illustrate the value of real-world workloads: <strong>we
aren’t adding these paths to satisfy a benchmark, we’re adding them because real
applications actually read them.</strong></p>

<h3 id="case-5-inconsistent-pty-implementation-causes-interactive-agents-to-error">Case 5: Inconsistent PTY Implementation Causes Interactive Agents to Error</h3>

<p>Interactive terminals are easily overlooked but heavily used in Agent systems
(tmux, screen, expect, REPLs, etc.). All rely on PTYs. We fixed several
inconsistencies here:</p>

<ul>
  <li>The <code class="highlighter-rouge">ISIG</code> flag was not checked correctly, causing signals to still be
generated after <code class="highlighter-rouge">stty -isig</code>.</li>
  <li>When the master closed, it did not send <code class="highlighter-rouge">SIGHUP</code> to the foreground process
group as Linux does.</li>
  <li><code class="highlighter-rouge">TCSBRK</code> / <code class="highlighter-rouge">TCFLSH</code> and other ioctls were missing or had incorrect
directional semantics, affecting programs like pyserial.</li>
</ul>

<p>Notably, <code class="highlighter-rouge">TCFLSH</code> semantics must be evaluated from the <strong>caller’s perspective</strong>
rather than hardcoding internal queue names. Otherwise, the flush directions
seen by the master and replica are reversed compared to Linux.</p>

<h3 id="case-6-jekyll-test-order-dependency-causes-flaky-failuresa-pure-test-case-issue">Case 6: Jekyll Test Order Dependency Causes Flaky Failures—A Pure Test Case Issue</h3>

<p>Sometimes, a test failing under gVisor has nothing to do with the runtime
environment at all.</p>

<p>During evaluation, a Jekyll test case (<code class="highlighter-rouge">jekyll-7637</code>) failed under <code class="highlighter-rouge">runsc</code> but
coincidentally passed under <code class="highlighter-rouge">runc</code>. After a deep dive, we found that this test
actually had a roughly 33% chance of failing in <em>any</em> environment.</p>

<p>The root cause was rather dramatic: the test code itself had a bug where it
passed a configuration value as a Ruby <code class="highlighter-rouge">Symbol</code> type, while the underlying
source code incorrectly compared it as a <code class="highlighter-rouge">String</code>. As a result, this test could
<strong>never</strong> load its required syntax highlighting plugin as intended. So why did
it sometimes pass? Because the testing framework (<code class="highlighter-rouge">minitest</code>) executes tests in
a randomized order. If this buggy test happened to run <strong>after</strong> another test
that correctly loaded the plugin into memory, it would “freeload” off that
global state and pass. But if the randomized order happened to put this test
first, it would genuinely fail. It just so happened that gVisor hit that 1-in-3
failure chance during our evaluation.</p>

<p>This perfectly illustrates why we need large-scale A/B testing and deep
analysis: without them, sporadic test flakiness like this can easily be
misdiagnosed as “sandbox instability.”</p>

<h2 id="best-practices-suggestions-for-using-gvisor-in-agentic-rl-scenarios">Best Practices: Suggestions for Using gVisor in Agentic-RL Scenarios</h2>

<p>If you’re building an Agent execution environment with gVisor, here are some
practical tips.</p>

<h3 id="suggestions-for-different-build-systems">Suggestions for Different Build Systems</h3>

<!-- mdformat off(no multiline table support in Kramdown) -->

<table>
  <thead>
    <tr>
      <th>Build System</th>
      <th>Common Risks</th>
      <th>Suggestions</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>Gradle</strong></td>
      <td>clean test concurrency race</td>
      <td>Split into clean and test steps</td>
    </tr>
    <tr>
      <td><strong>Maven</strong></td>
      <td>Remote dependency download timeout or 403</td>
      <td>Pre-populate local repo cache, minimize online downloads</td>
    </tr>
    <tr>
      <td><strong>CMake</strong></td>
      <td><code class="highlighter-rouge">copy_if_different</code> race conditions</td>
      <td>Lower parallelism, avoid over-reliance on extremely short time windows</td>
    </tr>
    <tr>
      <td><strong>sbt / Scala</strong></td>
      <td>Deep stack, slow startup, test flakiness</td>
      <td>Increase <code class="highlighter-rouge">-Xss</code>, give the first compilation a more generous timeout</td>
    </tr>
    <tr>
      <td><strong>pip / pytest</strong></td>
      <td>Differences in CPU count vs cgroup quota perception</td>
      <td>Be aware of the relationship between <code class="highlighter-rouge">os.cpu_count()</code> and actual quotas</td>
    </tr>
    <tr>
      <td><strong>Cargo / npm / yarn</strong></td>
      <td>Generally good compatibility</td>
      <td>Usually do not require special handling</td>
    </tr>
  </tbody>
</table>

<!-- mdformat on -->

<h3 id="debugging-procedure-when-encountering-failures">Debugging Procedure When Encountering Failures</h3>

<p>When a test fails, we recommend this debugging flow:</p>

<ol>
  <li>First reproduce the same command under <code class="highlighter-rouge">runc</code> to confirm if the failure is
specific to gVisor.</li>
  <li>If <code class="highlighter-rouge">runc</code> also fails, prioritize investigating test case issues,
environmental differences, or race conditions.</li>
  <li>If it only fails under gVisor, check for obvious missing syscalls, procfs,
or sysfs.</li>
  <li>For issues with no obvious missing features, compare logs, strace, and
runtime behavior to distinguish between semantic inconsistencies, amplified
race conditions, or environmental configuration differences.</li>
  <li>Only after confirming it is a gVisor semantic issue, proceed to locate the
code path, create a minimal reproduction, and add regression tests.</li>
</ol>

<p>Note: Many perceived “gVisor compatibility issues” are ultimately reclassified
as test case issues during this step.</p>

<h2 id="ai-driven-compatibility-analysis-why-this-path-is-feasible">AI-Driven Compatibility Analysis: Why This Path Is Feasible</h2>

<p>Large-scale compatibility analysis is well suited to AI assistance because it
involves a large amount of repetitive, context-heavy work:</p>

<ul>
  <li>Reading project source code and build scripts</li>
  <li>Comparing behavioral differences between two runtimes</li>
  <li>Comparing syscall, procfs, sysfs, PTY, network, and VFS semantics</li>
  <li>Turning conclusions into executable patches, PRs, or workaround suggestions</li>
  <li>Running regression validation and re-investigating the issue when validation
fails</li>
</ul>

<p>Manual analysis does not scale, while hardcoded rules often break down on
complex cases. AI agents fit naturally in the middle: they can take on most of
the “read logs → categorize → locate → report” work, while human engineers still
review the proposed approach and code.</p>

<p>The real value here is not just saving time; it is making our conclusions
<strong>scalable, traceable, and continuously improvable</strong>:</p>

<ul>
  <li>Every case has standardized analysis artifacts rather than scattered chat
logs.</li>
  <li>Every fix can be validated again against the original real-world test case.</li>
  <li>Every case that is “not a gVisor issue” can still be turned into a concrete
workaround playbook.</li>
  <li>As new datasets, images, or build systems arrive, the same analysis
framework can be reused.</li>
</ul>

<p>Through this method, we already have more than ten fixes merged into the gVisor
mainline, covering multiple areas such as file systems, networking, proc/sysfs,
PTY, and system call semantics. Some representative PRs are listed below:</p>

<!-- mdformat off(no multiline table support in Kramdown) -->

<table>
  <thead>
    <tr>
      <th>PR</th>
      <th>Fix Content</th>
      <th>Typical Agentic-RL Scenario</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><a href="https://github.com/google/gvisor/pull/12851">#12851</a></td>
      <td>poll: Only write back <code class="highlighter-rouge">revents</code></td>
      <td>tmux, libevent poll backend</td>
    </tr>
    <tr>
      <td><a href="https://github.com/google/gvisor/pull/12911">#12911</a></td>
      <td>proc: Add <code class="highlighter-rouge">/proc/sys/fs/pipe-max-size</code></td>
      <td>Python libraries like wurlitzer</td>
    </tr>
    <tr>
      <td><a href="https://github.com/google/gvisor/pull/12915">#12915</a></td>
      <td>pty: Implement <code class="highlighter-rouge">TCSBRK</code> / <code class="highlighter-rouge">TCFLSH</code></td>
      <td>pyserial, interactive PTY programs</td>
    </tr>
    <tr>
      <td><a href="https://github.com/google/gvisor/pull/12814">#12814</a></td>
      <td>proc: Add <code class="highlighter-rouge">randomize_va_space</code></td>
      <td>Performance and security inspection tools</td>
    </tr>
    <tr>
      <td><a href="https://github.com/google/gvisor/pull/12813">#12813</a></td>
      <td>sysfs: Add <code class="highlighter-rouge">/sys/dev/block</code> and <code class="highlighter-rouge">/sys/dev/char</code></td>
      <td>lsblk, device-related tools</td>
    </tr>
    <tr>
      <td><a href="https://github.com/google/gvisor/pull/12819">#12819</a></td>
      <td>proc: Fill in <code class="highlighter-rouge">fdinfo</code> fields</td>
      <td>lsof, fuser, diagnostic tools</td>
    </tr>
    <tr>
      <td><a href="https://github.com/google/gvisor/pull/12786">#12786</a></td>
      <td>devpts: Fix <code class="highlighter-rouge">ISIG</code> check</td>
      <td>Interactive shells / terminal-based agents</td>
    </tr>
    <tr>
      <td><a href="https://github.com/google/gvisor/pull/12853">#12853</a></td>
      <td>vfs: <code class="highlighter-rouge">FICLONE*</code> returns <code class="highlighter-rouge">EOPNOTSUPP</code></td>
      <td>file copying tools</td>
    </tr>
  </tbody>
</table>

<!-- mdformat on -->

<p>In this sense, Agentic-RL is not just a new use case for gVisor; it has also
pushed our compatibility engineering toward a more AI-driven workflow.</p>

<h2 id="conclusion">Conclusion</h2>

<p>Agentic-RL is both a proving ground for gVisor and, in practice, a <strong>large-scale
regression suite</strong>: it continuously drives real-world projects through the
sandbox and exposes compatibility boundaries that standard unit tests struggle
to cover. By bringing AI agents into this verification loop, we can evaluate
gVisor’s production readiness with data rather than intuition.</p>

<p>Our conclusions are simple:</p>

<ol>
  <li><strong>gVisor’s compatibility has proven to be production-ready.</strong></li>
  <li><strong>Most “compatibility issues” should not actually be attributed to gVisor.</strong></li>
  <li><strong>Real-world workloads are better than handpicked tests at revealing
critical problems.</strong></li>
  <li><strong>AI-driven compatibility analysis is practical.</strong></li>
</ol>

<p>As AI agents take on heavier tasks, the code-execution sandbox will become an
indispensable security foundation. We will continue refining this AI-driven
verification system, applying it to new datasets and language stacks, and
upstreaming our findings to the gVisor community. For Agentic-RL, a good sandbox
is not just secure—it also needs to be <strong>highly compatible, debuggable, and able
to evolve alongside real-world workloads.</strong></p>
