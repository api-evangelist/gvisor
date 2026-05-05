---
title: "Threat Detection in gVisor"
url: "/blog/2022/08/01/threat-detection/"
date: "2022-08-31T00:00:00-05:00"
author: "fvoznika"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<p>gVisor helps users secure their infrastructure by running containers in a
dedicated kernel that is isolated from the host. But wouldn’t it be nice if you
could tell when someone attempts to break out? Or get an early warning that your
web server might have been compromised? Now you can do it with gVisor! We are
pleased to announce support for <strong>runtime monitoring</strong>. Runtime monitoring
provides the ability for an external process to observe application behavior and
detect threats at runtime. Using this mechanism, gVisor users can watch actions
performed by the container and generate alerts when something unexpected occurs.</p>

<!--/excerpt-->

<p>A monitoring process can connect to the gVisor sandbox and receive a stream of
actions that the application is performing. The monitoring process decides what
actions are allowed and what steps to take based on policies for the given
application. gVisor communicates with the monitoring process via a simple
protocol based on
<a href="https://developers.google.com/protocol-buffers">Protocol Buffers</a>, which is the
basis for <a href="https://grpc.io/">gRPC</a> and is well supported in several languages.
The monitoring process runs isolated from the application inside the sandbox for
security reasons, and can be shared among all sandboxes running on the same
machine to save resources. Trace points can be individually configured when
creating a tracing session to capture only what’s needed.</p>

<p>Let’s go over a simple example of a web server that gets compromised while being
monitored. The web server can execute files from <code class="highlighter-rouge">/bin</code>, read files from <code class="highlighter-rouge">/etc</code>
and <code class="highlighter-rouge">/html</code> directories, create files under <code class="highlighter-rouge">/tmp</code>, etc. All these actions are
reported to a monitoring process which analyzes them and deems them normal
application behavior. Now suppose that an attacker takes control over the web
server and starts executing code inside the container. The attacker writes a
script under <code class="highlighter-rouge">/tmp</code> and, in an attempt to make it executable, runs <code class="highlighter-rouge">chmod u+x
/tmp/exploit.sh</code>. The monitoring process determines that making a file
executable is not expected in the normal web server execution and raises an
alert to the security team for investigation. Additionally, it can also decide
to kill the container and stop the attacker from making more progress.</p>

<h2 id="falco">Falco</h2>

<p><a href="https://falco.org/">Falco</a> is an Open Source Cloud Native Security monitor that
detects threats at runtime by observing the behavior of your applications and
containers. Falco
<a href="https://falco.org/blog/falco-0-32-1/">supports monitoring applications running inside gVisor</a>.
All the Falco rules and tooling work seamlessly with gVisor. You can use
<a href="https://gvisor.dev/docs/tutorials/falco/">this tutorial</a> to learn how to
configure Falco and gVisor together. More information can be found on the
<a href="https://falco.org/blog/intro-gvisor-falco/">Falco blog</a>.</p>

<h2 id="whats-next">What’s next?</h2>

<p>We’re looking for more projects to take advantage of the runtime monitoring
system and the visibility that it provides into the sandbox. There are a few
unique capabilities provided by the system that makes it easy to monitor
applications inside gVisor, like resolving file descriptors to full paths,
providing container ID with traces, separating processes that were exec’ed into
the container, internal procfs state access, and many more.</p>

<p>If you would like to explore it further, there is a
<a href="https://docs.google.com/document/d/1RQQKzeFpO-zOoBHZLA-tr5Ed_bvAOLDqgGgKhqUff2A">design document</a>
and
<a href="https://github.com/google/gvisor/tree/master/pkg/sentry/seccheck/README.md">documentation</a>
with more details about the configuration and communication protocol. In
addition, the <a href="https://gvisor.dev/docs/tutorials/falco/">tutorial using Falco</a>
is a great way to see it in action.</p>

<p>We would like to thank <a href="https://github.com/LucaGuerra">Luca Guerra</a>,
<a href="https://github.com/loresuso">Lorenzo Susini</a>, and the Falco team for their
support while building this feature.</p>
