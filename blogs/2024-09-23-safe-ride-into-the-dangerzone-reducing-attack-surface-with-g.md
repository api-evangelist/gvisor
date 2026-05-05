---
title: "Safe Ride into the Dangerzone: Reducing attack surface with gVisor"
url: "/blog/2024/09/23/safe-ride-into-the-dangerzone/"
date: "2024-09-23T00:00:00-05:00"
author: "almet"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<p><em>This article was written in collaboration with the
<a href="https://freedom.press">Freedom of the Press Foundation</a> and
<a href="https://dangerzone.rocks/news/2024-09-23-gvisor">cross-posted on the Dangerzone blog</a>.</em></p>

<p>One of the oft-repeated sound bites of computer security advice is: “Don’t open
random attachments from strangers.” If you are a journalist, however, opening
attachments and documents is part of your job description. Since journalists
already have a lot of security threats to worry about in dealing with sources,
the safe opening of documents should not be one of them.
<a href="https://dangerzone.rocks">Dangerzone</a> was developed to solve this problem. It
lets you open suspicious documents with confidence and gets out of your way.</p>

<p>For the past few months, members of the Dangerzone team and the
<a href="https://gvisor.dev">gVisor project</a> collaborated on significantly improving the
security properties of Dangerzone. We’re excited to announce that <strong>as of
version 0.7.0, Dangerzone uses gVisor to secure its document conversion
process</strong>. It is already trusted by Google
<a href="https://gvisor.dev/users">and others</a> to secure cloud products, scan Gmail
attachments for viruses, etc.</p>

<!--/excerpt-->

<p>If you’re an existing Dangerzone user on 0.7.0 scratching your head and thinking
“Well, I haven’t noticed anything different,” then first of all, “yay!” That was
the plan. And second, because the plan worked so deviously well, this change has
probably flown under the radar, so here are more than 3,000 words to amend this.</p>

<p>The rest of the article dives deep into Dangerzone’s security, describes how
gVisor works as a technology, and explains how Dangerzone’s security profile has
changed after this integration. Expect some technical terms and nerdery.</p>

<h2 id="how-dangerzone-works">How Dangerzone works</h2>

<p>Dangerzone’s purpose is to sanitize documents of any elements that can
compromise your computer or the source’s identity (think malware and document
metadata). To do this, it first renders the document into visual data (pixels)
and then turns this visual representation back into a readable document file.
The first part of this process (rendering the document into pixel data) is the
most security-critical part and, for the purpose of this article, we will zoom
in on just this.</p>

<blockquote>
  <p>💡 For a broader understanding of how Dangerzone works, we encourage you to
read the <a href="https://dangerzone.rocks/about/">“About Dangerzone”</a> section on the
Dangerzone website. Props to the <a href="https://www.qubes-os.org/">Qubes OS</a> team,
who first popularized the concept that is now their
<a href="https://blog.invisiblethings.org/2013/02/21/converting-untrusted-pdfs-into-trusted.html">TrustedPDF feature</a>.</p>
</blockquote>

<p>In order to support a wide variety of document formats (PDF, office documents,
image formats, etc.), Dangerzone needs to open them with software that
potentially has security bugs. That may result in compromise of the user’s
device, personal files, and communication. This is the same risk you face when
you use your computer to open attachments from unknown sources. Dangerzone needs
to somehow isolate this process from the rest of your computer, so that anything
it does cannot “get out of the box”.</p>

<p>Dangerzone’s isolation relies on <strong>Linux containers</strong>. Containers are very handy
for two things: ensuring that they work the same way across operating systems
and separating the container from the rest of the machine.</p>

<figure>
<img alt="Diagram showing the Dangerzone UI sending a document to a document renderer, which converts it to pixels, and then receives the pixels back." src="/assets/images/2024-09-23-dangerzone-outline.svg" />
<figcaption>Outline of how Dangerzone uses containers to render a document into pixels.</figcaption>
</figure>

<p>Dangerzone benefits from both of these aspects: Development and testing are made
easy by using containers’ cross-platform compatibility; and containers’
security, especially how Dangerzone configured them, offers strong isolation
guarantees. The
<a href="https://freedom.press/news/dangerzone-receives-favorable-audit/">security audit Dangerzone passed recently</a>
is a testament to this.</p>

<p>In computer security, the gold standard of isolation is <strong>virtual machines</strong>.
VMs are what they sound like: a computer running within a computer. When running
a virtual machine, the “host” (outer) machine is protected from the action of
the “guest” (inner) virtual machine. This is why the TrustedPDF feature of
QubesOS uses disposable VMs as its isolation mechanism. Dangerzone also tried to
use VMs in the past, but implementing them in a multiplatform way proved
high-maintenance. Thus, Dangerzone switched back to containers, but the team
always wanted to improve Dangerzone’s security properties.</p>

<blockquote>
  <p>💡 How does Dangerzone use Linux containers on Windows and Mac OS? It requires
<a href="https://www.docker.com/products/docker-desktop/">Docker Desktop</a>, which runs
Linux inside a virtual machine and then runs Linux containers in it.</p>
</blockquote>

<h2 id="dangerzones-attack-surface">Dangerzone’s attack surface</h2>

<p>To understand how to protect Dangerzone users from exploits, it’s useful to
think like an attacker. When Dangerzone processes a malicious document within a
container, the first point of the attack is the application that opens the
document. Dangerzone is designed with the assumption that determined attackers
will find a vulnerability in such applications and take control of them (check
out this <a href="https://github.com/freedomofpress/dangerzone/blob/main/docs/advisories/2023-12-07.md">security advisory from the Dangerzone team about a recent, critical
LibreOffice
vulnerability</a>).
From there on, the next point of attack is to circumvent the Linux kernel
protections for the container or directly compromise the Linux kernel.</p>

<p>The Linux kernel, even in Docker Desktop VMs, is a very privileged component. It
has access to sensitive data, such as other files on the user’s machine or the
user’s browser history, and to your computer’s network.</p>

<p>Processes in containers interface with the Linux kernel through
<a href="https://en.wikipedia.org/wiki/System_call"><strong>system calls</strong></a> and
<a href="https://opensource.com/article/19/3/virtual-filesystems-linux"><strong>virtual filesystems</strong></a>.
Attackers can try to take advantage of security bugs in the above interfaces. So
it is critical to limit the container’s access to the Linux kernel. We call this
the container’s
<a href="https://en.wikipedia.org/wiki/Attack_surface"><strong>attack surface</strong></a>. The smaller
it is, the more secure a system is.</p>

<p>Dangerzone tries to reduce its attack surface by multiple mechanisms available
to Linux containers:</p>

<ul>
  <li>Removal of
<a href="https://en.wikipedia.org/wiki/Capability-based_security">process capabilities</a>.
This reduces the set of permissions the container has in the kernel.</li>
  <li>Removal of network access. This prevents the container from accessing the
internet to exfiltrate document data.</li>
  <li>Filtering of allowed system calls through
<a href="https://en.wikipedia.org/wiki/Seccomp">seccomp</a>. This reduces the set of
system calls (i.e., types of actions) that the container is allowed to make
to the kernel.</li>
  <li>Minimal <a href="https://en.wikipedia.org/wiki/User_identifier">user ID</a> mapping.
This reduces the risk that the container may access files belonging to users
other than the Dangerzone user on the same computer.</li>
</ul>

<blockquote>
  <p>💡 Check out the above protection measures in
<a href="https://github.com/freedomofpress/dangerzone/blob/88a2d151ab4a3cb2f769998f27f251518d93bb45/dangerzone/isolation_provider/container.py#L188-L213">Dangerzone’s codebase</a>.</p>
</blockquote>

<figure>
<img alt="Diagram showing that the renderer and LibreOffice make system calls to the Linux kernel, to which several filters are applied." src="/assets/images/2024-09-23-dangerzone-protections.svg" />
<figcaption>Container protections employed by Dangerzone prior to 0.7.0.</figcaption>
</figure>

<p>This provides the container with a fair degree of isolation from the Linux
kernel. However, some attack surface remains, since:</p>

<ul>
  <li>The computer’s user is still mapped in the container. This means that a
container escape would allow the attacker to access the user’s personal
files (browser data, documents, etc.); it would be more isolated if that
were not the case.</li>
  <li>The system call filter is still relatively permissive. The specific system
calls that are blocked are dependent on the container manager and version in
use (see
<a href="https://github.com/microsoft/docker/blob/master/docs/security/seccomp.md">Docker’s filters, for example</a>),
but in general, the system call filter only blocks obscure or
system-admin-only system calls (e.g., rebooting, modifying systemwide
settings). It does not block containers from opening arbitrary files or
interacting with the network stack, which can still be vectors for security
bugs.</li>
  <li>The container’s root filesystem, while ephemeral, is still writable. This
allows attackers to exploit potential vulnerabilities in Linux’s filesystem
stack.</li>
  <li>The Linux kernel is still exposed to the container. While it is possible to
reduce the attack surface available to the container to a minimum, this
architecture still requires that the container have direct access to Linux
via system calls. So if a Linux security bug can be triggered within the set
of filtered system calls, an attack may still be successful.</li>
</ul>

<figure>
<img alt="Diagram highlighting how access to the Linux kernel and the relatively permissive system filter may create exposure to bugs or vulnerabilities." src="/assets/images/2024-09-23-dangerzone-protections-annotated.svg" />
<figcaption>Dangerzone's attack surface prior to 0.7.0, illustrated.</figcaption>
</figure>

<p>We’ve wanted to mitigate these risks for a while now, but we had to do so in a
cross-platform way and without burdening the user with administrative tasks.</p>

<p>Enter gVisor.</p>

<h2 id="what-is-gvisor">What is gVisor?</h2>

<p><a href="https://gvisor.dev"><strong>gVisor</strong></a> is a container security solution. In short, it
makes it much harder for malicious code to break out of the container boundary.
This was a great fit for Dangerzone’s security needs.</p>

<p>An open source project written in Go, gVisor was released in May 2018 by Google
under the Apache 2.0 license. It runs on Linux and integrates with all popular
container management software, such as Docker, Podman, or Kubernetes. At its
core, gVisor is an <strong>application kernel</strong> that implements a substantial portion
of the Linux system call interface. This means gVisor sits between a container
and the Linux kernel and plays both roles: from the container’s perspective,
gVisor acts as a <strong>kernel</strong>, but from Linux’s perspective, gVisor is just a
regular <strong>application</strong>. That means the container can no longer directly
interface with the Linux kernel. This is a massive reduction in attack surface.</p>

<p>If you’re new to gVisor, the concept of not interfacing with the Linux kernel at
all may seem either quite vague or overly restrictive. That’s normal, so let’s
toy with this concept a bit for fun and illustrative purposes. Here’s a
perfectly normal sentence:</p>

<blockquote>
  <p>“A process opens a document on the filesystem”</p>
</blockquote>

<p>And here’s how gVisor warps every single word in that sentence:</p>

<ul>
  <li>“on the filesystem”: Nope, no such thing. The gVisor container runs in an
empty filesystem.</li>
  <li>“opens a document”: Nuh-uh, the gVisor container does not even have the
permission to perform the <code class="highlighter-rouge">open</code> system call. Also, there are no files to
open in the first place.</li>
  <li>“A process”: Amusingly, the gVisor container does not even have the ability
to perform the <code class="highlighter-rouge">exec</code> system calls. From the Linux kernel’s perspective, the
gVisor “process” looks like a typical multithreaded program, even while many
independent processes are running within the gVisor sandbox.</li>
</ul>

<p>And yet, gVisor can containerize most applications without issue. For example,
the Dangerzone container image was not altered at all for the gVisor
integration.</p>

<p>So what’s going on here?</p>

<p>gVisor manages to pull the above trick with the help of two components:</p>

<ol>
  <li>
    <p><strong>Sentry</strong> is the component that runs the containerized application. It
intercepts every system call that the application makes and reimplements it
in Go. As part of this, it may decide to do one or more system calls to the
host Linux kernel. However, it’s heavily restricted with a strict seccomp
filter (that’s why system calls like <code class="highlighter-rouge">open</code>, <code class="highlighter-rouge">socket</code>, or <code class="highlighter-rouge">exec</code> are not
allowed).</p>
  </li>
  <li>
    <p><strong>Gofer</strong> is a component that runs outside the container and is responsible
for filesystem operations. The sentry may make I/O requests to the gofer.
The gofer will independently validate them, then perform these I/O
operations on the container’s behalf (that’s how the container can read
files from the host filesystem, even though <code class="highlighter-rouge">open</code> is not allowed from the
sentry).</p>
  </li>
</ol>

<p>The above components are managed by a container runtime called <code class="highlighter-rouge">runsc</code>, which
exposes the same interface as other container runtimes. This means it can be
integrated in other container management software like Podman, Docker, or
Kubernetes.</p>

<figure>
<img alt="Diagram showing a potentially vulnerable application running in the gVisor sandbox. gVisor Sentry implements the sandbox and intercepts all system calls. It services them either by making limited system calls of its own, or by asking gVisor Gofer to perform I/O system calls on its behalf. Both components are further restricted by a tailored kernel filter, along with other kernel protections." src="/assets/images/2024-09-23-gvisor-outline.svg" />
<figcaption>gVisor intercepting system calls from a sandboxed application</figcaption>
</figure>

<p>With the above architecture, gVisor blue-pills the application into thinking
that it interacts with a regular Linux kernel. In practice, gVisor reimplements
most basic features that Linux provides (memory management, scheduling, system
call interface, I/O, networking), and only issues system calls to the Linux
kernel when truly necessary, such as when it needs information from it (e.g.,
reading the document to be converted by Dangerzone).</p>

<p>The gVisor kernel is designed to be difficult to break out of. gVisor is written
in Go. Many of Linux’s security woes stem from its use of C, which is a
memory-unsafe language. By contrast, gVisor is a regular Go application and
inherits Go’s memory safety features. This eliminates a large class of security
vulnerabilities.</p>

<p>The gVisor kernel also has a much smaller code footprint, because unlike a
traditional kernel like Linux, it does not have to deal with things like
hardware devices, and only implements a subset of the Linux kernel interface
that is sufficient for most applications to work in practice. Because of its
smaller implementation, there are fewer moving parts to juggle between, and thus
fewer opportunities for bugs to exist.</p>

<p>Beyond its kernel indirection, gVisor also hardens itself through a bunch of
security measures on startup, some of which are similar to regular containers:</p>

<ul>
  <li><strong>Isolation</strong>: Running in its own set of namespaces (user namespace, process
namespace, network namespace, etc.) to further isolate it from the host.</li>
  <li><strong>File access prevention</strong>: Running in its own root with exactly zero host
files initially visible to it.</li>
  <li><strong>Privilege revocation</strong>: Dropping all capabilities it has to ensure it runs
with the least privileges.</li>
  <li><strong>System call filtering</strong>: Setting a strict system call filter tuned for the
gVisor Sentry specifically.
    <ul>
      <li>As mentioned, unlike Docker or Podman’s default system call filter, this
is a <em>very restricted set</em> of system calls. This filter blocks basic
operations like opening files, creating network connections, or
executing other processes. The presence of this filter does <em>not</em>
prevent use of these system calls from within the gVisor sandbox;
instead, the gVisor kernel <em>intercepts and reimplements</em> system calls
internally without needing to make a “real” system call out to the Linux
kernel.</li>
    </ul>
  </li>
  <li>The gofer also uses all of the above techniques to isolate itself as much as
possible.</li>
</ul>

<p>The gVisor kernel has been battle-tested by Google and other large companies
like Ant and Cloudflare. For example, searching for the text “GKE Sandbox”
(which uses gVisor) on the
<a href="https://cloud.google.com/kubernetes-engine/security-bulletins">GKE security bulletin</a>
shows how often Linux kernel vulnerabilities occur but that gVisor prevents.
gVisor is also continuously <a href="https://en.wikipedia.org/wiki/Fuzzing">fuzz-tested</a>
for bugs using <a href="https://github.com/google/syzkaller/">Syzkaller</a>, an automated
kernel security testing tool.</p>

<p>What’s the catch here? Applications that perform lots of system calls and heavy
I/O will have some degraded performance. Also, applications that rely on exotic
features by the Linux kernel may not work. In practice,
<a href="https://gvisor.dev/docs/user_guide/compatibility">the majority of applications do not suffer from this issue</a>.</p>

<h2 id="integrating-gvisor-with-dangerzone">Integrating gVisor with Dangerzone</h2>

<p>So, gVisor looks like a strong candidate for Dangerzone, which is a relatively
simple application that does not perform a heavy amount of system calls. Also,
gVisor conveniently offers a container runtime that is a drop-in replacement for
use with Docker/Podman. Therefore, integrating these two projects should be
really simple, right?</p>

<p>Well, not so fast.</p>

<p>Dangerzone is a <em>multiplatform</em> application, and most of its users are on
Windows and macOS. Integrating gVisor just for Linux would not cut it. At the
same time, gVisor works strictly on Linux systems, so we are at an impasse.</p>

<p>In what is, in retrospect, a classic case of
<a href="https://en.wikipedia.org/wiki/Law_of_the_instrument">Maslow’s hammer</a>, we
decided to solve our container problems with yet another container. The idea is
simple; why not containerize gVisor and make it run on Docker Desktop? After
all, as we already pointed out, Docker Desktop runs Linux inside a virtual
machine.</p>

<p>By doing so, Dangerzone now has two containers with different responsibilities:</p>

<ul>
  <li>The <strong>outer</strong> Docker/Podman container acts as the <strong>portability</strong> layer for
Dangerzone. Its main responsibility is to bundle the necessary config files,
scripts, and programs to run gVisor. It’s also responsible for bundling the
container image that gVisor will spawn a container from.</li>
  <li>The <strong>inner</strong> gVisor container acts as the <strong>isolation</strong> layer for
Dangerzone. Its sole responsibility is to run the actual Dangerzone logic
for rendering documents to pixels.</li>
</ul>

<figure>
<img alt="Diagram showing the Dangerzone UI sending a document to a document renderer within an inner container, which is protected by gVisor's Sentry. The Sentry intercepts system calls, allowing only limited system calls to pass to the Linux kernel with strict security settings. I/O system calls are handled by gVisor Gofer in an outer container, with less strict but controlled permissions" src="/assets/images/2024-09-23-dangerzone-with-gvisor.svg" />
<figcaption>Outline of how gVisor integrates with Dangerzone. There are now two nested containers, and each one brings its own protections. Usage of LibreOffice is implied.</figcaption>
</figure>

<p>Running gVisor inside a container came with its own set of challenges:</p>

<ul>
  <li>The Docker/Podman’s seccomp filter must allow the <code class="highlighter-rouge">ptrace</code> system call. We
found that recent Docker Desktop versions and Podman version &gt;= 4.0 have a
seccomp filter that allows this system call. For older versions, we
specified a custom seccomp filter that allowed it.</li>
  <li>gVisor cannot run under SELinux in enforcing mode under default settings, so
we labeled the container with <code class="highlighter-rouge">container_engine_t</code> (see GitHub issue
<a href="https://github.com/freedomofpress/dangerzone/issues/880">#880</a>).</li>
  <li>The Docker/Podman container must run with the <code class="highlighter-rouge">SYS_CHROOT</code> capability. This
is needed by gVisor to restrict its own access to the filesystem before it
starts document processing. Other than that, the <strong>outer</strong> container drops
all other capabilities and privileges.</li>
</ul>

<blockquote>
  <p>💡 You can find more details about this integration in the Dangerzone’s
<a href="https://github.com/freedomofpress/dangerzone/blob/main/docs/developer/gvisor.md">gVisor design doc</a>.</p>
</blockquote>

<h2 id="dangerzone-protections">Dangerzone protections</h2>

<p>We talked about Dangerzone’s original attack surface, and how we integrated
gVisor to reduce it. In practice though, in what ways is Dangerzone better off
than before? Well, if the Matryoshka containers are giving you a headache, or
you just skimmed to this section (no shade), here’s how the new Dangerzone
protections fare against the previous version, and the default protections of
Linux containers:</p>

<table>
  <thead>
    <tr>
      <th>🛡️ <strong>Protections</strong></th>
      <th><strong>Default</strong></th>
      <th><strong>Dangerzone (0.6.1)</strong></th>
      <th><strong>Dangerzone + gVisor (0.7.0)</strong></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>🐧 <strong>Linux kernel</strong></td>
      <td><span style="color: #505050;">Exposed</span></td>
      <td><span style="color: #990000;">👎 Exposed</span></td>
      <td><span style="color: #38761d;">🎉 Not exposed</span></td>
    </tr>
    <tr>
      <td>🛠️ <strong>System call filter</strong></td>
      <td><span style="color: #505050;">Moderate</span></td>
      <td><span style="color: #990000;">👎 Moderate</span></td>
      <td><span style="color: #38761d;">👍 Strict</span></td>
    </tr>
    <tr>
      <td>🛠️ <strong>Capabilities</strong></td>
      <td><span style="color: #505050;">Default</span></td>
      <td><span style="color: #38761d;">👍 None</span></td>
      <td><span style="color: #38761d;">👍 None</span></td>
    </tr>
    <tr>
      <td>👤 <strong>Host user</strong></td>
      <td><span style="color: #505050;">Mapped</span></td>
      <td><span style="color: #990000;">👎 Mapped</span></td>
      <td><span style="color: #38761d;">👍 Unmapped</span></td>
    </tr>
    <tr>
      <td>📁 <strong>Filesystem</strong></td>
      <td><span style="color: #505050;">Exposed</span></td>
      <td><span style="color: #990000;">👎 Writable</span></td>
      <td><span style="color: #38761d;">👍 Read-only</span></td>
    </tr>
    <tr>
      <td>🌐 <strong>Network</strong></td>
      <td><span style="color: #505050;">Exposed</span></td>
      <td><span style="color: #38761d;">👍 Disabled</span></td>
      <td><span style="color: #38761d;">✌️ Disabled at two levels</span></td>
    </tr>
    <tr>
      <td>🔒 <strong>SELinux</strong></td>
      <td><span style="color: #505050;">Yes (<code class="highlighter-rouge">container_t</code>)</span></td>
      <td><span style="color: #38761d;">👍 Yes (<code class="highlighter-rouge">container_t</code>)</span></td>
      <td><span style="color: #38761d;">👍 Yes (<code class="highlighter-rouge">container_engine_t</code>)</span></td>
    </tr>
    <tr>
      <td>🖥️ <strong>Hardware Virtualization</strong></td>
      <td><span style="color: #505050;">None</span></td>
      <td><span style="color: #990000;">👎 None</span></td>
      <td><span style="color: #990000;">👎 None</span></td>
    </tr>
  </tbody>
</table>

<p>As you can see, the most important protection is that <strong>the document conversion
process no longer has access to the Linux kernel</strong>. Instead, it only has access
to the gVisor kernel (in the Sentry), and must break out of it before it can
access the Linux kernel that it (prior to gVisor integration) had access to.</p>

<p>Additionally, Dangerzone itself configures the two containers to be more secure
with:</p>

<ul>
  <li>Privilege revocation: Removing all privileges and capabilities of the
document conversion process in the <strong>inner container</strong>, and minimizing the
set of capabilities granted to the <strong>outer container</strong> to just <code class="highlighter-rouge">SYS_CHROOT</code>
and no other.</li>
  <li>File modification prevention: Making the <strong>inner container</strong>’s root
filesystem read-only.</li>
  <li>User isolation: Running the <strong>outer container</strong> in a user namespace that
does not include the Dangerzone UI user (available in Linux distributions
with Podman version 4.1 or greater).</li>
  <li>Kernel security settings: Setting the <strong>outer container</strong>’s system call
filter and SELinux label settings.</li>
  <li>Host access prevention: Not using any mounts in either container.</li>
  <li>Network access prevention: Disabling both containers’ ability to use
networking.</li>
</ul>

<figure>
<img alt="Diagram highlighting how gVisor mitigates against bugs and vulnerabilities in the inner container, including exploits which escalate privileges to the outer container." src="/assets/images/2024-09-23-dangerzone-with-gvisor-annotated.svg" />
<figcaption>Explanation of how Dangerzone's latest protections limit its attack surface.</figcaption>
</figure>

<h2 id="conclusion">Conclusion</h2>

<p>Integrating the gVisor project with Dangerzone was very exciting: It’s a good
example of how gVisor can add another line of defense to a project without
requiring application-level changes.</p>

<p>At the same time, the design complexity of the Dangerzone project increased a
bit, mostly to cater to its cross-platform nature, but honestly not that much.
Dangerzone is strongly security-focused, so we believe it’s worth the cost.</p>

<p>We hope that this article demystifies some security aspects of containers, so
that you can use Dangerzone and gVisor with even more confidence. Feel free to
reach out to us with any questions or comments:</p>

<ul>
  <li><a href="https://notmyidea.org">Alexis Métaireau</a></li>
  <li><a href="https://freedom.press/people/alex-p">Alex Pyrgiotis</a></li>
  <li><a href="https://perot.me">Etienne Perot</a></li>
  <li><a href="https://freedom.press/contact/">Freedom of the Press Foundation (FPF)</a></li>
  <li><a href="https://gvisor.dev/community">gVisor community</a></li>
</ul>
