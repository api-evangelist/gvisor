---
title: "Multi-Agent gVisor Isolation (MAGI)"
url: "/blog/2026/04/15/magi-multi-agent-gvisor-isolation/"
date: "2026-04-15T00:00:00-05:00"
author: "eperot"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<figure class="img-100pct">
<img alt="Diagram showing the MAGI system: three agents running in gVisor, along with a lot of side-services in gVisor-sandboxed containers. Evangelion style." src="/assets/images/2026-04-15-magi/magi.png" />
<figcaption>Get in the sandbox, Agents.</figcaption>
</figure>

<p><strong>Does gVisor work with OpenClaw?</strong> This question has been asked a lot, so let’s
answer it here and now: <strong>Yes</strong>.</p>

<p>In this post, we will set up a triple-agent system combining
<strong><a href="https://openclaw.ai/">OpenClaw</a></strong>,
<strong><a href="https://github.com/sipeed/picoclaw">PicoClaw</a></strong>, and
<strong><a href="https://hermes-agent.nousresearch.com/">Hermes Agent</a></strong>, each in separate
gVisor sandboxes, all with local inference powered by
<strong><a href="https://ollama.com/">Ollama</a></strong> in a gVisor sandbox using three different
models, convening together in a self-hosted <strong><a href="https://matrix.org">Matrix.org</a></strong>
server (naturally, also running in a gVisor sandbox). Each agent will be given
its own set of capabilities, each of which will be sandboxed. At the end of the
day, you will have a fully self-sovereign triple-agent system that can answer
queries, browse the web, and cogitate with itself.</p>

<p><strong>Does this particular setup make practical sense?</strong> <em>No, but it is cool</em>. More
importantly, it demonstrates the versatility of gVisor at sandboxing basically
any component that an agentic system may need. gVisor’s compatibility has grown
significantly over the last few years, and agent harnesses fit well within what
gVisor is capable of.</p>

<!--/excerpt-->

<p>Let’s go.</p>

<!--* pragma: { seclinter_this_is_fine: true } *-->

<details>

  

    <h3 id="basic-machine-setup-dockergvisornvidia-drivers">Basic machine setup: Docker/gVisor/NVIDIA drivers</h3>

    <p>We will use a <code class="highlighter-rouge">g2-standard-96</code> GCE VM running stock Ubuntu for this, but any
Linux machine with similar GPUs would work. This section describes its basic
setup.</p>

  

  <p>Getting a GCE VM:</p>

  <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>gcloud compute instances create magi <span class="se">\</span>
    <span class="nt">--project</span><span class="o">=</span>eperot-gke-dev <span class="se">\</span>
    <span class="nt">--zone</span><span class="o">=</span>europe-west1-c <span class="se">\</span>
    <span class="nt">--machine-type</span><span class="o">=</span>g2-standard-96 <span class="se">\</span>
    <span class="nt">--maintenance-policy</span><span class="o">=</span>TERMINATE <span class="se">\</span>
    <span class="nt">--accelerator</span><span class="o">=</span><span class="nv">count</span><span class="o">=</span>8,type<span class="o">=</span>nvidia-l4 <span class="se">\</span>
    <span class="nt">--create-disk</span><span class="o">=</span>auto-delete<span class="o">=</span><span class="nb">yes</span>,boot<span class="o">=</span><span class="nb">yes</span>,device-name<span class="o">=</span>magi,image<span class="o">=</span>projects/ubuntu-os-cloud/global/images/ubuntu-2404-noble-amd64-v20260316,mode<span class="o">=</span>rw,size<span class="o">=</span>2048,type<span class="o">=</span>pd-ssd
</code></pre></div>  </div>

  <p>We will be using the following ports:</p>

  <ul>
    <li><code class="highlighter-rouge">8008</code>: Matrix.org server (Synapse)</li>
    <li><code class="highlighter-rouge">8084</code>: Cinny web UI (Matrix.org client)</li>
    <li><code class="highlighter-rouge">11434</code>: Ollama (inference API server)</li>
    <li><code class="highlighter-rouge">18789</code>: OpenClaw gateway web UI</li>
    <li><code class="highlighter-rouge">18790</code>: PicoClaw gateway</li>
    <li><code class="highlighter-rouge">3002</code>: Self-hosted Firecrawl</li>
  </ul>

  <p>If SSHing into a VM, you can forward some of them for convenient access:</p>

  <div class="highlighter-rouge"><div class="highlight"><pre class="highlight"><code>-L 8008:127.0.0.1:8008 -L 8084:127.0.0.1:8084 -L 11434:127.0.0.1:11434 -L 18789:127.0.0.1:18789
</code></pre></div>  </div>

  <p>Setting up the GCE VM (once SSH’d as <code class="highlighter-rouge">root</code>):</p>

  <div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Basics</span>
<span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="nb">sudo </span>apt-get <span class="nt">-y</span> upgrade

<span class="c"># NVIDIA driver</span>
<span class="nv">DRIVER_VERSION</span><span class="o">=</span>590.48.01<span class="p">;</span> <span class="se">\</span>
  <span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> build-essential linux-headers-<span class="si">$(</span><span class="nb">uname</span> <span class="nt">-r</span><span class="si">)</span> <span class="o">&amp;&amp;</span> <span class="se">\</span>
  curl <span class="nt">-fSsl</span> <span class="nt">-O</span> <span class="s2">"https://us.download.nvidia.com/tesla/</span><span class="nv">$DRIVER_VERSION</span><span class="s2">/NVIDIA-Linux-x86_64-</span><span class="nv">$DRIVER_VERSION</span><span class="s2">.run"</span> <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo </span>sh NVIDIA-Linux-x86_64-<span class="nv">$DRIVER_VERSION</span>.run <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">rm </span>NVIDIA-Linux-x86_64-<span class="nv">$DRIVER_VERSION</span>.run

<span class="c"># Docker</span>
<span class="nb">sudo </span>apt update <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo </span>apt <span class="nb">install</span> <span class="nt">-y</span> ca-certificates curl <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo install</span> <span class="nt">-m</span> 0755 <span class="nt">-d</span> /etc/apt/keyrings <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo </span>curl <span class="nt">-fsSL</span> https://download.docker.com/linux/ubuntu/gpg <span class="nt">-o</span> /etc/apt/keyrings/docker.asc <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo chmod </span>a+r /etc/apt/keyrings/docker.asc
<span class="nb">sudo tee</span> /etc/apt/sources.list.d/docker.sources <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh">
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: </span><span class="si">$(</span><span class="nb">.</span> /etc/os-release <span class="o">&amp;&amp;</span> <span class="nb">echo</span> <span class="s2">"</span><span class="k">${</span><span class="nv">UBUNTU_CODENAME</span><span class="k">:-</span><span class="nv">$VERSION_CODENAME</span><span class="k">}</span><span class="s2">"</span><span class="si">)</span><span class="sh">
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
</span><span class="no">EOF
</span><span class="nb">sudo </span>apt update <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo </span>apt <span class="nb">install</span> <span class="nt">-y</span> docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

<span class="c"># NVIDIA container toolkit</span>
<span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> <span class="nt">--no-install-recommends</span> <span class="se">\</span>
  ca-certificates <span class="se">\</span>
  curl <span class="se">\</span>
  gnupg2 <span class="o">&amp;&amp;</span> <span class="se">\</span>
  curl <span class="nt">-fsSL</span> https://nvidia.github.io/libnvidia-container/gpgkey | <span class="nb">sudo </span>gpg <span class="nt">--dearmor</span> <span class="nt">-o</span> /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg <span class="o">&amp;&amp;</span> <span class="se">\</span>
  curl <span class="nt">-s</span> <span class="nt">-L</span> https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | <span class="se">\</span>
    <span class="nb">sed</span> <span class="s1">'s#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g'</span> | <span class="se">\</span>
    <span class="nb">sudo tee</span> /etc/apt/sources.list.d/nvidia-container-toolkit.list <span class="o">&amp;&amp;</span> <span class="se">\</span>
    <span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="se">\</span>
    <span class="nb">export </span><span class="nv">NVIDIA_CONTAINER_TOOLKIT_VERSION</span><span class="o">=</span>1.19.0-1 <span class="o">&amp;&amp;</span> <span class="se">\</span>
    <span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> <span class="se">\</span>
      nvidia-container-toolkit<span class="o">=</span><span class="k">${</span><span class="nv">NVIDIA_CONTAINER_TOOLKIT_VERSION</span><span class="k">}</span> <span class="se">\</span>
      nvidia-container-toolkit-base<span class="o">=</span><span class="k">${</span><span class="nv">NVIDIA_CONTAINER_TOOLKIT_VERSION</span><span class="k">}</span> <span class="se">\</span>
      libnvidia-container-tools<span class="o">=</span><span class="k">${</span><span class="nv">NVIDIA_CONTAINER_TOOLKIT_VERSION</span><span class="k">}</span> <span class="se">\</span>
      libnvidia-container1<span class="o">=</span><span class="k">${</span><span class="nv">NVIDIA_CONTAINER_TOOLKIT_VERSION</span><span class="k">}</span>

<span class="c"># gVisor</span>
<span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> <span class="se">\</span>
    apt-transport-https <span class="se">\</span>
    ca-certificates <span class="se">\</span>
    curl <span class="se">\</span>
    gnupg <span class="o">&amp;&amp;</span> <span class="se">\</span>
  curl <span class="nt">-fsSL</span> https://gvisor.dev/archive.key | <span class="nb">sudo </span>gpg <span class="nt">--dearmor</span> <span class="nt">-o</span> /usr/share/keyrings/gvisor-archive-keyring.gpg <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">echo</span> <span class="s2">"deb [arch=</span><span class="si">$(</span>dpkg <span class="nt">--print-architecture</span><span class="si">)</span><span class="s2"> signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main"</span> | <span class="nb">sudo tee</span> /etc/apt/sources.list.d/gvisor.list <span class="o">&gt;</span> /dev/null <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> runsc <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo </span>runsc <span class="nb">install</span> <span class="nt">--</span> <span class="nt">--nvproxy</span><span class="o">=</span><span class="nb">true</span> <span class="nt">--nvproxy-allowed-driver-capabilities</span><span class="o">=</span>all <span class="nt">--net-raw</span><span class="o">=</span><span class="nb">true</span> <span class="nt">--allow-packet-socket-write</span><span class="o">=</span><span class="nb">true</span> <span class="nt">--host-uds</span><span class="o">=</span>all <span class="nt">--debug-log</span><span class="o">=</span>/tmp/runsc/ <span class="o">&amp;&amp;</span> <span class="se">\</span>
  <span class="nb">sudo </span>systemctl restart docker
</code></pre></div>  </div>

  <p>Verifying everything works:</p>

  <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>nvidia-smi
<span class="nv">$ </span>docker run <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="nt">--gpus</span><span class="o">=</span>all <span class="nt">--rm</span> ubuntu:latest sh <span class="nt">-c</span> <span class="s1">'ls -al /dev/nvidia*'</span>
</code></pre></div>  </div>

</details>

<section class="sticky-section">

  <h2 id="self-hosted-matrixorg-server--cinny-web-frontend-setup">Self-hosted Matrix.org server + Cinny web frontend setup</h2>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'Synapse' and 'Cinny' containers blinking." src="/assets/images/2026-04-15-magi/synapse.blink.gif" />
<figcaption>Setting up Synapse and Cinny.</figcaption>
</figure>

    <div class="section-content">

      <p>Let’s set up the <strong>Matrix.org server</strong> for communication, and the <strong>Cinny</strong> web
client that we humans can use to communicate with it.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Generate homeserver.yaml</span>
<span class="nv">$ </span>docker run <span class="nt">-it</span> <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="nt">--rm</span> <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="nb">type</span><span class="o">=</span>volume,src<span class="o">=</span>synapse-data,dst<span class="o">=</span>/data <span class="se">\</span>
    <span class="nt">-e</span> <span class="nv">SYNAPSE_SERVER_NAME</span><span class="o">=</span>magi <span class="se">\</span>
    <span class="nt">-e</span> <span class="nv">SYNAPSE_REPORT_STATS</span><span class="o">=</span>no <span class="se">\</span>
    matrixdotorg/synapse:latest generate

<span class="c"># Run server</span>
<span class="nv">$ </span>docker run <span class="nt">--detach</span> <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="nt">--restart</span><span class="o">=</span>always <span class="nt">--name</span><span class="o">=</span>synapse <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="nb">type</span><span class="o">=</span>volume,src<span class="o">=</span>synapse-data,dst<span class="o">=</span>/data <span class="se">\</span>
    <span class="nt">-p</span> 8008:8008 <span class="se">\</span>
    matrixdotorg/synapse:latest

<span class="c"># Create admin user</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> synapse register_new_matrix_user <span class="se">\</span>
    <span class="nt">-c</span> /data/homeserver.yaml <span class="se">\</span>
    <span class="nt">--user</span> gendo <span class="nt">--password</span> yui <span class="nt">--admin</span>

<span class="c"># Run cinny (Matrix client)</span>
<span class="nv">$ </span>docker run <span class="nt">-it</span> <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="nt">--restart</span><span class="o">=</span>always <span class="nt">--name</span><span class="o">=</span>cinny <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>synapse:synapse <span class="se">\</span>
    <span class="nt">-p</span> 8084:80 <span class="se">\</span>
    ghcr.io/cinnyapp/cinny:latest

<span class="c"># Access Cinny web UI at http://localhost:8084</span>
<span class="c"># Log in as:</span>
<span class="c">#   Homeserver: http://127.0.0.1:8008</span>
<span class="c">#   Username: gendo</span>
<span class="c">#   Password: yui</span>
</code></pre></div>      </div>

    </div>

  </div>

</section>

<section class="sticky-section">

  <h2 id="self-hosted-inference-server-ollama">Self-hosted inference server: Ollama</h2>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'Ollama' and 'NVIDIA GPU' boxes blinking." src="/assets/images/2026-04-15-magi/ollama.blink.gif" />
<figcaption>Setting up Ollama for GPU inference.</figcaption>
</figure>

    <div class="section-content">

      <p>Setting up <strong>Ollama</strong>, the GPU-enabled inference server and the brain of it all.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>docker run <span class="nt">--detach</span> <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="nt">--restart</span><span class="o">=</span>always <span class="nt">--name</span><span class="o">=</span>ollama <span class="se">\</span>
    <span class="nt">--gpus</span><span class="o">=</span>all <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="nb">type</span><span class="o">=</span>volume,src<span class="o">=</span>ollama-data,dst<span class="o">=</span>/root <span class="se">\</span>
    <span class="nt">-p</span> 11434:11434 <span class="se">\</span>
    ollama/ollama:0.20.0

<span class="c"># Pull and load some models.</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> ollama sh <span class="nt">-c</span> <span class="s1">'ollama pull qwen3.5:27b-q4_K_M   &amp;&amp; ollama run --keepalive=9001h qwen3.5:27b-q4_K_M     Say hello.'</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> ollama sh <span class="nt">-c</span> <span class="s1">'ollama pull glm-4.7-flash:q4_K_M &amp;&amp; ollama run --keepalive=9001h glm-4.7-flash:q4_K_M   Say hello.'</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> ollama sh <span class="nt">-c</span> <span class="s1">'ollama pull gpt-oss:20b          &amp;&amp; ollama run --keepalive=9001h gemma4:26b-a4b-it-q8_0 Say hello.'</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> ollama sh <span class="nt">-c</span> <span class="s1">'ollama pull gpt-oss:20b          &amp;&amp; ollama run --keepalive=9001h nomic-embed-text:137m-v1.5-fp16 ""'</span>

<span class="c"># Make sure they all fit together in VRAM, otherwise you'll get bad performance.</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> ollama ollama ps
NAME                      ID              SIZE     PROCESSOR    CONTEXT    UNTIL
gemma4:26b-a4b-it-q8_0    6bfaf9a8cb37    89 GB    100% GPU     262144     12 months from now
glm-4.7-flash:q4_K_M      d1a8a26252f1    40 GB    100% GPU     202752     12 months from now
qwen3.5:27b-q4_K_M        7653528ba5cb    44 GB    100% GPU     262144     12 months from now
</code></pre></div>      </div>

    </div>

  </div>

</section>

<section class="sticky-section">

  <h2 id="containerized-openclaw-setup-with-browser-use">Containerized OpenClaw setup with Browser Use</h2>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'OpenClaw' and 'Chrome' containers blinking." src="/assets/images/2026-04-15-magi/openclaw.blink.gif" />
<figcaption>Setting up OpenClaw and Chrome browser.</figcaption>
</figure>

    <div class="section-content">

      <p>Now let’s set up <strong>OpenClaw</strong> and hook it up to a web browser for fully-local
Browser Use.</p>

      <p>We will use the official <code class="highlighter-rouge">ghcr.io/openclaw/openclaw</code> OpenClaw container image,
but we will also modify it to install the Google Chrome, as per
<a href="https://docs.openclaw.ai/tools/browser-linux-troubleshooting#solution-1-install-google-chrome-recommended">recommended in the OpenClaw docs</a>.
This will allow the agent to use a web browser, all running in gVisor.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">export </span><span class="nv">MELCHIOR</span><span class="o">=</span><span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/agents/melchior-1"</span><span class="p">;</span> <span class="nb">mkdir</span> <span class="nt">-p</span> <span class="s2">"</span><span class="nv">$MELCHIOR</span><span class="s2">"</span>
<span class="nv">$ </span><span class="nb">cat</span> <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh"> &gt; "</span><span class="nv">$MELCHIOR</span><span class="sh">/Dockerfile"
FROM ghcr.io/openclaw/openclaw:2026.4.2

USER 0:0
RUN export DEBIAN_FRONTEND=noninteractive; apt update -y &amp;&amp; </span><span class="se">\</span><span class="sh">
    apt install -y wget chromium libvulkan1 &amp;&amp; </span><span class="se">\</span><span class="sh">
    wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb &amp;&amp; </span><span class="se">\</span><span class="sh">
    dpkg -i google-chrome-stable_current_amd64.deb &amp;&amp; </span><span class="se">\</span><span class="sh">
    rm google-chrome-stable_current_amd64.deb &amp;&amp; </span><span class="se">\</span><span class="sh">
    apt --fix-broken install -y
</span><span class="no">EOF

</span><span class="nv">$ </span>docker build <span class="nt">-t</span> openclaw:melchior-1 <span class="s2">"</span><span class="nv">$MELCHIOR</span><span class="s2">"</span>
</code></pre></div>      </div>

      <p>Note that the resulting image runs as root. This is not a security risk; “root”
in a gVisor sandbox doesn’t imply any root-like level access on the host.</p>

      <p>Let’s create a Matrix account for it and seed its configuration:</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">mkdir</span> <span class="nt">-p</span> <span class="s2">"</span><span class="nv">$MELCHIOR</span><span class="s2">/config"</span> <span class="s2">"</span><span class="nv">$MELCHIOR</span><span class="s2">/home"</span>

<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> synapse register_new_matrix_user <span class="se">\</span>
    <span class="nt">-c</span> /data/homeserver.yaml <span class="se">\</span>
    <span class="nt">--user</span> melchior <span class="nt">--password</span> akagi <span class="nt">--no-admin</span>

<span class="nv">$ </span><span class="nb">cat</span> <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh"> &gt; "</span><span class="nv">$MELCHIOR</span><span class="sh">/config/openclaw.json"
{
  "auth": {
    "profiles": {
      "ollama:default": {
        "provider": "ollama",
        "mode": "api_key"
      }
    }
  },
  "agents": {
    "defaults": {
      "models": {
        "ollama/gemma4:26b-a4b-it-q8_0": {}
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "ollama": {
        "baseUrl": "http://ollama:11434",
        "api": "ollama",
        "apiKey": "OLLAMA_API_KEY",
        "models": [
          {
            "id": "gemma4:26b-a4b-it-q8_0",
            "name": "gemma4:26b-a4b-it-q8_0",
            "reasoning": true,
            "input": [
              "text"
            ],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 262144,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "channels": {
    "matrix": {
      "enabled": true,
      "homeserver": "http://synapse:8008",
      "userId": "@melchior:magi",
      "password": "akagi",
      "deviceName": "Melchior",
      "allowPrivateNetwork": true,
      "encryption": false,
      "groupPolicy": "open",
      "autoJoin": "always",
      "dm": {
        "policy": "open",
        "allowFrom": [
          "*"
        ]
      }
    }
  },
  "gateway": {
    "mode": "local",
    "controlUi": {
      "dangerouslyDisableDeviceAuth": true,
      "dangerouslyAllowHostHeaderOriginFallback": true
    }
  },
  "skills": {
    "install": {
      "nodeManager": "npm"
    }
  },
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  },
  "tools": {
    "web": {
      "search": {
        "enabled": true,
        "provider": "duckduckgo"
      },
      "fetch": {
        "enabled": true
      }
    }
  },
  "plugins": {
    "entries": {
      "matrix": {
        "enabled": true
      },
      "browser": {
        "enabled": true
      }
    }
  }
}
</span><span class="no">EOF
</span></code></pre></div>      </div>

      <p>Note: for the purpose of simplifying demo setup, the above configuration
disables authentication, allows the bot to auto-join all Matrix channels it is
invited to, etc. For real deployments, do not use these settings.</p>

      <p>Let’s run it!</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">export </span><span class="nv">MELCHIOR</span><span class="o">=</span><span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/agents/melchior-1"</span><span class="p">;</span> docker run <span class="nt">--detach</span> <span class="se">\</span>
    <span class="nt">--name</span><span class="o">=</span>melchior <span class="se">\</span>
    <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="se">\</span>
    <span class="nt">--restart</span><span class="o">=</span>always <span class="se">\</span>
    <span class="nt">--env</span><span class="o">=</span><span class="nv">OPENCLAW_GATEWAY_TOKEN</span><span class="o">=</span><span class="s2">"dummy-token-for-sandbox"</span> <span class="se">\</span>
    <span class="nt">--env</span><span class="o">=</span><span class="nv">OPENCLAW_CONFIG_PATH</span><span class="o">=</span><span class="s2">"/etc/openclaw/openclaw.json"</span> <span class="se">\</span>
    <span class="nt">-p</span> 18789:18789 <span class="se">\</span>
    <span class="nt">--env</span><span class="o">=</span><span class="nv">HOME</span><span class="o">=</span>/home/node <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>synapse:synapse <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>ollama:ollama <span class="se">\</span>
    <span class="nt">-v</span> <span class="s2">"</span><span class="nv">$MELCHIOR</span><span class="s2">/home"</span>:/home/node/.openclaw <span class="se">\</span>
    <span class="nt">-v</span> <span class="s2">"</span><span class="nv">$MELCHIOR</span><span class="s2">/config"</span>:/etc/openclaw <span class="se">\</span>
    openclaw:melchior-1 <span class="se">\</span>
    node <span class="se">\</span>
        dist/index.js <span class="se">\</span>
        gateway <span class="se">\</span>
           <span class="nt">--bind</span><span class="o">=</span>lan <span class="se">\</span>
           <span class="nt">--port</span><span class="o">=</span>18789 <span class="se">\</span>
           <span class="nt">--allow-unconfigured</span> <span class="se">\</span>
           <span class="nt">--verbose</span>
</code></pre></div>      </div>

      <p>Run <code class="highlighter-rouge">docker exec -it melchior openclaw configure</code> for further interactive
configuration.</p>

    </div>

  </div>

</section>

<p>You can now go to <code class="highlighter-rouge">http://127.0.0.1:18789/?token=dummy-token-for-sandbox</code> and
talk to your OpenClaw instance!</p>

<figure class="img-100pct">
<div class="double-border-glow">
<img alt="The OpenClaw gateway web UI displaying a chat with the dmesg output, confirming that it is running in gVisor." src="/assets/images/2026-04-15-magi/openclaw-ui.png" />
</div>
<figcaption>OpenClaw web UI running in gVisor. The <code>dmesg</code> output is characteristic of gVisor.</figcaption>
</figure>

<h3 id="browser-use">Browser Use</h3>

<p>The <code class="highlighter-rouge">Dockerfile</code> we built earlier contains the Google Chrome web browser, which
<a href="/blog/2026/04/23/scaling-agentic-rl-sandboxes-to-the-millions-with-gvisor-at-tencent/">OpenClaw knows how to use</a>. You can ask it to open websites and take
screenshots. Here is the gVisor website rendered in Chrome-in-gVisor by
OpenClaw:</p>

<figure>
<div class="double-border-glow">
<img alt="gVisor website rendered by Chrome running in gVisor." src="/assets/images/2026-04-15-magi/gvisor-website.png" />
</div>
<figcaption>gVisor website rendered by Chrome in gVisor, orchestrated by OpenClaw.<br /><em>Funnily enough, the OpenClaw web interface didn't provide the means for OpenClaw to display this image directly.</em><br /><em>OpenClaw autonomously solved this problem by uploading this picture to a temporary image hosting service and responding with the uploaded image URL.</em></figcaption>
</figure>

<p>Now let’s bring the other two brains to life.</p>

<section class="sticky-section">

  <h2 id="containerized-picoclaw-with-web-and-github-skills">Containerized PicoClaw with web and GitHub skills</h2>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'PicoClaw' container blinking." src="/assets/images/2026-04-15-magi/picoclaw.blink.gif" />
<figcaption>Setting up PicoClaw.</figcaption>
</figure>

    <div class="section-content">

      <p>Moving on to PicoClaw, the minimal agent.</p>

      <p>We will use the
<a href="https://hub.docker.com/r/sipeed/picoclaw">PicoClaw Docker image</a>, and enable a
few skills for GitHub interaction with the
<a href="https://github.com/google/gvisor">gVisor repository</a>.</p>

      <p>Note that while this demo was on a x86-64 VM, PicoClaw has also been confirmed
to work in <strong>gVisor on arm64 on a Raspberry Pi 4 Model B</strong>.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">export </span><span class="nv">BALTHASAR</span><span class="o">=</span><span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/agents/balthasar-2"</span><span class="p">;</span> <span class="nb">mkdir</span> <span class="nt">-p</span> <span class="s2">"</span><span class="nv">$BALTHASAR</span><span class="s2">/picoclaw"</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> synapse register_new_matrix_user <span class="se">\</span>
    <span class="nt">-c</span> /data/homeserver.yaml <span class="se">\</span>
    <span class="nt">--user</span> balthasar <span class="nt">--password</span> ritsuko <span class="nt">--no-admin</span>
<span class="nv">$ matrix_token</span><span class="o">=</span><span class="s2">"</span><span class="si">$(</span>curl <span class="nt">-X</span> POST <span class="nt">-H</span> <span class="s2">"Content-Type: application/json"</span> <span class="se">\</span>
    <span class="s2">"http://127.0.0.1:8008/_matrix/client/v3/login"</span> <span class="se">\</span>
    <span class="nt">-d</span> <span class="se">\</span>
    <span class="s1">'{"type": "m.login.password", "user": "balthasar", "password": "ritsuko"}'</span> | <span class="se">\</span>
    jq <span class="nt">-r</span> .access_token<span class="si">)</span><span class="s2">"</span>
<span class="nv">$ </span><span class="nb">cat</span> <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh"> &gt; "</span><span class="nv">$BALTHASAR</span><span class="sh">/picoclaw/config.json"
{
  "model_list": [
    {
      "model_name": "glm-4.7-flash",
      "model": "ollama/glm-4.7-flash:q4_K_M",
      "api_base": "http://ollama:11434/v1"
    }
  ],
  "agents": {
    "defaults": {
      "model_name": "glm-4.7-flash"
    }
  },
  "gateway": {
    "host": "0.0.0.0",
    "port": 18790
  },
  "channels": {
    "matrix": {
      "enabled": true,
      "homeserver": "http://synapse:8008",
      "user_id": "@balthasar:magi",
      "access_token": "</span><span class="k">${</span><span class="nv">matrix_token</span><span class="k">}</span><span class="sh">",
      "join_on_invite": true,
      "allow_from": []
    }
  }
}
</span><span class="no">EOF
</span><span class="nv">$ </span>docker run <span class="nt">-it</span> <span class="se">\</span>
    <span class="nt">--name</span><span class="o">=</span>balthasar <span class="se">\</span>
    <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="se">\</span>
    <span class="nt">--restart</span><span class="o">=</span>always <span class="se">\</span>
    <span class="nt">-v</span> <span class="s2">"</span><span class="nv">$BALTHASAR</span><span class="s2">/picoclaw:/root/.picoclaw"</span> <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>synapse:synapse <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>ollama:ollama <span class="se">\</span>
    <span class="nt">--entrypoint</span><span class="o">=</span>/usr/local/bin/picoclaw <span class="se">\</span>
    sipeed/picoclaw:latest gateway
</code></pre></div>      </div>

      <p>PicoClaw should start, although it does not have a lot of functionality out of
the box. Let’s enable some skills:</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">cp</span> <span class="s2">"</span><span class="nv">$BALTHASAR</span><span class="s2">/picoclaw/config.json"</span> <span class="s2">"</span><span class="nv">$BALTHASAR</span><span class="s2">/picoclaw/config.json.bak"</span> <span class="o">&amp;&amp;</span> <span class="se">\</span>
  jq <span class="s1">'.tools.web.enabled = true |
      .tools.web.prefer_native = true |
      .tools.exec.enabled = true |
      .tools.exec.allow_remote = true |
      .tools.skills.enabled = true |
      .tools.skills.github = {
        "enabled": true,
        "token": "YOUR_GITHUB_TOKEN_HERE",
        "timeout": 30,
        "max_results": 5
      } |
      .tools.skills.max_concurrent_searches = 5
      | .tools.skills.search_cache = {
        "max_size": 100,
        "ttl_seconds": 300
      } |
      .tools.web_fetch.enabled = true'</span> <span class="se">\</span>
      &lt; <span class="s2">"</span><span class="nv">$BALTHASAR</span><span class="s2">/picoclaw/config.json.bak"</span> <span class="se">\</span>
      <span class="o">&gt;</span> <span class="s2">"</span><span class="nv">$BALTHASAR</span><span class="s2">/picoclaw/config.json"</span>

<span class="c"># Restart PicoClaw to apply config changes.</span>
<span class="nv">$ </span>docker restart balthasar

<span class="c"># You can re-attach to an interactive CLI for PicoClaw with:</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> balthasar picoclaw agent
</code></pre></div>      </div>

      <p>Now we can ask it to interact with GitHub.</p>

      <figure class="img-100pct">
<div class="double-border-glow">
<img alt="PicoClaw starting up and being tasked with looking up the top trending GitHub repositories that day." src="/assets/images/2026-04-15-magi/picoclaw-1.png" />
</div>

<figcaption>PicoClaw being tasked with looking up the current trending GitHub
repositories.</figcaption> </figure>

      <p>Funnily enough, the top GitHub repository today is Hermes Agent, which we will
install next. For now, let’s review a small gVisor PR:</p>

      <figure class="img-100pct">
<div class="double-border-glow">
<img alt="PicoClaw being tasked with explaining and reviewing a gVisor pull request." src="/assets/images/2026-04-15-magi/picoclaw-2.png" />
</div>
<figcaption>PicoClaw being tasked with explaining and reviewing [gVisor pull request #12911](https://github.com/google/gvisor/pull/12911).<br />Which was later reviewed by a human as well.</figcaption>
</figure>

    </div>

  </div>

</section>

<section class="sticky-section">

  <h2 id="modularized--sandboxed-hermes-agent-setup">Modularized &amp; sandboxed Hermes Agent setup</h2>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'Hermes Agent' container blinking." src="/assets/images/2026-04-15-magi/hermes-agent.blink.gif" />
<figcaption>Setting up Hermes Agent.</figcaption>
</figure>

    <div class="section-content">

      <p>Finally, let’s set up <strong>Hermes Agent</strong>, and fully load it with sandboxed
<strong>Browser Use</strong>, sandboxed <strong>web crawling</strong>, and sandboxed <strong>code execution</strong>.</p>

      <p>We will use
<a href="https://hermes-agent.nousresearch.com/docs/user-guide/docker">Hermes Agent’s official Docker image</a>:
<code class="highlighter-rouge">nousresearch/hermes-agent</code>, expanded with the dependencies needed to perform
local text-to-speech and Matrix.org integration, all running in gVisor.
Additionally, for extra security, we will do the following:</p>

      <ul>
        <li>Run <a href="https://github.com/jo-inc/camofox-browser">Camofox Browser</a> in a
separate gVisor container, for browser use.</li>
        <li>Run
<a href="https://github.com/firecrawl/firecrawl/blob/main/SELF_HOST.md">self-hosted Firecrawl</a>
in a separate gVisor container, for agentic search.</li>
        <li>Run <a href="/docs/tutorials/docker-in-gvisor/">Docker-in-gVisor</a> in a separate
container, for Hermes Agent to execute arbitrary code safely.</li>
      </ul>

      <p>Note that the <code class="highlighter-rouge">--net-raw=true --allow-packet-socket-write=true</code> runsc flags are
<a href="/docs/tutorials/docker-in-gvisor/">required for Docker to work in gVisor</a>. For
this reason, we need to install a secondary runtime for the Docker-in-gVisor
container, and enable host UDS (<code class="highlighter-rouge">--host-uds=all</code>) so that the Docker daemon
socket file can be exported out of that sandbox into the Hermes Agent sandbox.</p>

    </div>

  </div>

</section>

<figure class="img-100pct">
<div class="double-border-glow">
<img alt="Hermes Agent running in gVisor." src="/assets/images/2026-04-15-magi/hermes-agent-in-gvisor.png" />
</div>
<figcaption>Hermes Agent running in gVisor.</figcaption>
</figure>

<section class="sticky-section">

  <h3 id="setting-up-docker-in-gvisor-for-code-execution">Setting up Docker-in-gVisor for code execution</h3>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'Docker' box blinking." src="/assets/images/2026-04-15-magi/docker.blink.gif" />
<figcaption>Setting up Docker-in-gVisor for code execution.</figcaption>
</figure>

    <div class="section-content">

      <p><strong>gVisor is capable of
<a href="https://gvisor.dev/docs/tutorials/docker-in-gvisor/">running Docker inside of itself</a></strong>.
Since Hermes Agent has
<a href="https://hermes-agent.nousresearch.com/docs/user-guide/configuration#docker-backend">Docker as a code execution backend</a>,
we will use this to spawn a separate Docker-in-gVisor container which Hermes
Agent can use to run code safely.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">export </span><span class="nv">CASPER</span><span class="o">=</span><span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/agents/casper-3"</span>
<span class="nv">$ </span>runsc <span class="nb">install</span> <span class="nt">--runtime</span><span class="o">=</span>docker-in-gvisor <span class="nt">--</span> <span class="nt">--net-raw</span><span class="o">=</span><span class="nb">true</span> <span class="nt">--allow-packet-socket-write</span><span class="o">=</span><span class="nb">true</span> <span class="nt">--host-uds</span><span class="o">=</span>all

<span class="c"># Reload *host* dockerd configuration to make it notice the new runtime we just added.</span>
<span class="nv">$ </span><span class="nb">kill</span> <span class="nt">-HUP</span> <span class="s2">"</span><span class="si">$(</span>pidof dockerd<span class="si">)</span><span class="s2">"</span>

<span class="c"># Run Docker-in-gVisor container.</span>
<span class="c"># Note: The `--cap-add=all` flag does *not* grant the container any</span>
<span class="c"># capabilities on the host. It only enables the sandboxed workload to use</span>
<span class="c"># elevated privileges **within the sandbox**.</span>
<span class="c"># This is necessary to be able to run `dockerd` inside a container.</span>
<span class="nv">$ </span><span class="nb">mkdir</span> <span class="nt">-p</span> <span class="s2">"</span><span class="nv">$CASPER</span><span class="s2">/docker-run"</span><span class="p">;</span> docker run <span class="nt">--detach</span> <span class="se">\</span>
    <span class="nt">--name</span><span class="o">=</span>hermes-exec <span class="se">\</span>
    <span class="nt">--runtime</span><span class="o">=</span>docker-in-gvisor <span class="se">\</span>
    <span class="nt">--restart</span><span class="o">=</span>always <span class="se">\</span>
    <span class="nt">--cap-add</span><span class="o">=</span>all <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="s2">"type=bind,src=</span><span class="nv">$CASPER</span><span class="s2">/docker-run,dst=/var/run"</span> <span class="se">\</span>
    us-central1-docker.pkg.dev/gvisor-presubmit/gvisor-presubmit-images/basic/docker_x86_64

<span class="c"># Verify that we can talk to the `dockerd` server running in gVisor.</span>
<span class="c"># We need --security-opt=seccomp=unconfined here, because otherwise</span>
<span class="c"># Docker's default seccomp profile would block the `syslog(2)` syscall that</span>
<span class="c"># the `dmesg` process uses to read the kernel logs (which here is actually</span>
<span class="c"># reading the gVisor kernel logs). This is not a security problem, since we</span>
<span class="c"># are still all running in gVisor.</span>
<span class="nv">$ DOCKER_HOST</span><span class="o">=</span><span class="s2">"unix://</span><span class="nv">$CASPER</span><span class="s2">/docker-run/docker.sock"</span> docker run <span class="se">\</span>
    <span class="nt">--rm</span> <span class="se">\</span>
    <span class="nt">--security-opt</span><span class="o">=</span><span class="nv">seccomp</span><span class="o">=</span>unconfined <span class="se">\</span>
    debian:latest <span class="se">\</span>
    dmesg
<span class="c"># [...]</span>
<span class="o">[</span>    0.000000] Starting gVisor...
<span class="o">[</span>    0.429798] DeFUSEing fork bombs...
<span class="o">[</span>    0.782957] Adversarially training Redcode AI...
<span class="c"># [...]</span>
</code></pre></div>      </div>

    </div>

  </div>

</section>

<section class="sticky-section">

  <h3 id="building-camofox-docker-image-in-docker-in-gvisor">Building Camofox Docker image in Docker-in-gVisor</h3>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'Camofox' container blinking." src="/assets/images/2026-04-15-magi/camofox.blink.gif" />
<figcaption>Setting up Camofox Browser.</figcaption>
</figure>

    <div class="section-content">

      <p><a href="https://github.com/jo-inc/camofox-browser">Camofox</a> is a Firefox-based web
browser for agentic browsing. Let’s run it in its own sandboxed container.</p>

      <p>Camofox comes with an image that also contains <code class="highlighter-rouge">Xvfb</code> to simulate an X11 display
server, and <code class="highlighter-rouge">yt-dlp</code> for YouTube video extraction, all working in gVisor. Let’s
build it.</p>

      <p>The Camofox project doesn’t provide pre-built Docker images, so we need to build
it ourselves. But wait! Camofox may or may not be a fishy project. What if it
contains malicious code?</p>

      <p><strong>Have no fear, gVisor is here!</strong> We can simply build the image inside gVisor.
Let’s spin up an ephemeral Docker-in-gVisor container, run the Camofox Docker
image build process within, extract the image out, and import it into the host
<code class="highlighter-rouge">dockerd</code>’s local image repository.</p>

      <figure class="follow-along">
<img alt="I heard you like containers so we put Docker build in Docker in gVisor in Docker." src="/assets/images/2026-04-15-magi/turtles.jpg" />
<figcaption>It's containers all the way down.</figcaption>
</figure>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Start Docker-in-gVisor with large-enough /var/lib/docker tmpfs</span>
<span class="nv">$ </span><span class="nb">mkdir</span> <span class="nt">-p</span> /tmp/docker-tmp <span class="o">&amp;&amp;</span> docker run <span class="nt">--detach</span> <span class="se">\</span>
    <span class="nt">--name</span><span class="o">=</span>docker-tmp <span class="se">\</span>
    <span class="nt">--runtime</span><span class="o">=</span>docker-in-gvisor <span class="se">\</span>
    <span class="nt">--restart</span><span class="o">=</span>always <span class="se">\</span>
    <span class="nt">--cap-add</span><span class="o">=</span>all <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="s2">"type=bind,src=/tmp/docker-tmp,dst=/tmp/docker-tmp"</span> <span class="se">\</span>
    <span class="nt">-e</span> <span class="nv">DOCKER_TMPFS_SIZE</span><span class="o">=</span>8G <span class="se">\</span>
    us-central1-docker.pkg.dev/gvisor-presubmit/gvisor-presubmit-images/basic/docker_x86_64

<span class="c"># Build image within the in-gVisor Docker.</span>
<span class="c"># The `make` command will run `docker build` in-sandbox.</span>
<span class="nv">$ </span>docker <span class="nb">exec </span>docker-tmp sh <span class="nt">-c</span> <span class="s1">'true &amp;&amp; \
    apt update -y &amp;&amp; \
    apt install -y git build-essential &amp;&amp; \
    git clone https://github.com/jo-inc/camofox-browser.git &amp;&amp; \
    cd camofox-browser &amp;&amp; \
    make'</span>

<span class="c"># Extract the image out of the container and import as host Docker image.</span>
<span class="c"># The `docker save` command dumps the image to stdout, which gets piped</span>
<span class="c"># to the out-of-sandbox `docker load` command.</span>
<span class="nv">$ </span>docker <span class="nb">exec </span>docker-tmp docker save camofox-browser | docker load
Loaded image: camofox-browser:135.0.1-x86_64

<span class="c"># You now have the image on the host Docker:</span>
<span class="nv">$ </span>docker images | <span class="nb">grep </span>camofox
camofox-browser:135.0.1-x86_64      80c072259479      4.6GB      2.27GB

<span class="c"># Clean up.</span>
<span class="nv">$ </span>docker <span class="nb">rm</span> <span class="nt">-f</span> docker-tmp
</code></pre></div>      </div>

      <p>Now that we have our Camofox image, let’s run it:</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>docker run <span class="nt">--detach</span> <span class="se">\</span>
    <span class="nt">--name</span><span class="o">=</span>camofox <span class="se">\</span>
    <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="se">\</span>
    <span class="nt">--restart</span><span class="o">=</span>always <span class="se">\</span>
    camofox-browser:135.0.1-x86_64

<span class="c"># Camofox binds on port 3000 by default; we don't need to expose it</span>
<span class="c"># to the host though, as we will use inter-container networking.</span>
<span class="c"># Nonetheless, let's make sure it works:</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-e</span> <span class="nv">DEBIAN_FRONTEND</span><span class="o">=</span>noninteractive camofox sh <span class="nt">-c</span> <span class="s1">'true &amp;&amp; \
    apt update -y &gt;/dev/null &amp;&amp; \
    apt install -y curl jq &gt;/dev/null &amp;&amp; \
    tabId="$(curl -q -X POST http://127.0.0.1:3000/tabs -H "Content-Type: application/json" -d "{\"userId\": \"me\", \"sessionKey\": \"task\", \"url\": \"https://gvisor.dev\"}" | jq -r .tabId)" &amp;&amp; \
    curl -q --output - "http://127.0.0.1:3000/tabs/${tabId}/screenshot?userId=me"
  '</span> <span class="o">&gt;</span> /tmp/screenshot.png
<span class="nv">$ </span>file /tmp/screenshot.png
/tmp/screenshot.png: PNG image data, 1280 x 720, 8-bit/color RGBA, non-interlaced
</code></pre></div>      </div>

    </div>

  </div>

</section>

<section class="sticky-section">

  <h3 id="running-self-hosted-firecrawl-in-gvisor">Running self-hosted Firecrawl in gVisor</h3>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'Firecrawl', 'Redis', 'RabbitMQ', 'Playwright', and 'PostgreSQL' containers blinking." src="/assets/images/2026-04-15-magi/firecrawl.blink.gif" />
<figcaption>Setting up the Firecrawl stack.</figcaption>
</figure>

    <div class="section-content">

      <p>We will use the
<a href="https://github.com/firecrawl/firecrawl/blob/main/docker-compose.yaml">Firecrawl <code class="highlighter-rouge">docker-compose.yaml</code> template</a>,
simply modified to run all containers in gVisor. Because
<a href="https://github.com/google/gvisor/issues/7469">the way <code class="highlighter-rouge">docker-compose</code> sets up DNS</a>
is incompatible with gVisor’s per-container network stack, we need to use
pre-assigned IPs rather than container hostnames in the <code class="highlighter-rouge">docker-compose</code> file.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">export </span><span class="nv">CASPER</span><span class="o">=</span><span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/agents/casper-3"</span><span class="p">;</span> git clone https://github.com/firecrawl/firecrawl.git <span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/agents/casper-3/firecrawl"</span>
<span class="nv">$ </span><span class="nb">cat</span> <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh"> &gt; "</span><span class="nv">$CASPER</span><span class="sh">/firecrawl/.env"
PORT=3002
HOST=0.0.0.0
OLLAMA_BASE_URL=http://172.17.0.1:11434/api
MODEL_NAME=qwen3.5:27b-q4_K_M
MODEL_EMBEDDING_NAME=nomic-embed-text:137m-v1.5-fp16
BULL_AUTH_KEY=CHANGEME
</span><span class="no">EOF
</span><span class="nv">$ </span>git apply <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh">
diff --git a/docker-compose.yaml b/docker-compose.yaml
index 46829cafb..819f9cc87 100644
--- a/docker-compose.yaml
+++ b/docker-compose.yaml
@@ -10,8 +10,6 @@ x-common-service: &amp;common-service
     nofile:
       soft: 65535
       hard: 65535
-  networks:
-    - backend
   extra_hosts:
     - "host.docker.internal:host-gateway"
   logging:
@@ -22,13 +20,13 @@ x-common-service: &amp;common-service
       compress: "true"

 x-common-env: &amp;common-env
-  REDIS_URL: </span><span class="se">\$</span><span class="sh">{REDIS_URL:-redis://redis:6379}
-  REDIS_RATE_LIMIT_URL: </span><span class="se">\$</span><span class="sh">{REDIS_URL:-redis://redis:6379}
-  PLAYWRIGHT_MICROSERVICE_URL: </span><span class="se">\$</span><span class="sh">{PLAYWRIGHT_MICROSERVICE_URL:-http://playwright-service:3000/scrape}
+  REDIS_URL: </span><span class="se">\$</span><span class="sh">{REDIS_URL:-redis://172.16.0.30:6379}
+  REDIS_RATE_LIMIT_URL: </span><span class="se">\$</span><span class="sh">{REDIS_URL:-redis://172.16.0.30:6379}
+  PLAYWRIGHT_MICROSERVICE_URL: </span><span class="se">\$</span><span class="sh">{PLAYWRIGHT_MICROSERVICE_URL:-http://172.16.0.20:3000/scrape}
   POSTGRES_USER: </span><span class="se">\$</span><span class="sh">{POSTGRES_USER:-postgres}
   POSTGRES_PASSWORD: "</span><span class="se">\$</span><span class="sh">{POSTGRES_PASSWORD:-postgres}"
   POSTGRES_DB: </span><span class="se">\$</span><span class="sh">{POSTGRES_DB:-postgres}
-  POSTGRES_HOST: </span><span class="se">\$</span><span class="sh">{POSTGRES_HOST:-nuq-postgres}
+  POSTGRES_HOST: </span><span class="se">\$</span><span class="sh">{POSTGRES_HOST:-172.16.0.50}
   POSTGRES_PORT: </span><span class="se">\$</span><span class="sh">{POSTGRES_PORT:-5432}
   USE_DB_AUTHENTICATION: </span><span class="se">\$</span><span class="sh">{USE_DB_AUTHENTICATION:-false}
   NUM_WORKERS_PER_QUEUE: </span><span class="se">\$</span><span class="sh">{NUM_WORKERS_PER_QUEUE:-8}
@@ -58,6 +56,10 @@ x-common-env: &amp;common-env

 services:
   playwright-service:
+    runtime: "runsc"
+    networks:
+      backend:
+        ipv4_address: 172.16.0.20
     # NOTE: If you don't want to build the service locally,
     # comment out the build: statement and uncomment the image: statement
     # image: ghcr.io/firecrawl/playwright-service:latest
@@ -71,8 +73,6 @@ services:
       BLOCK_MEDIA: </span><span class="se">\$</span><span class="sh">{BLOCK_MEDIA}
       # Configure maximum concurrent pages for Playwright browser instances
       MAX_CONCURRENT_PAGES: </span><span class="se">\$</span><span class="sh">{CRAWL_CONCURRENT_REQUESTS:-10}
-    networks:
-      - backend
     # Resource limits for Docker Compose (not Swarm)
     cpus: 2.0
     mem_limit: 4G
@@ -88,13 +88,17 @@ services:

   api:
     &lt;&lt;: *common-service
+    runtime: "runsc"
+    networks:
+      backend:
+        ipv4_address: 172.16.0.10
     environment:
       &lt;&lt;: *common-env
       HOST: "0.0.0.0"
       PORT: </span><span class="se">\$</span><span class="sh">{INTERNAL_PORT:-3002}
       EXTRACT_WORKER_PORT: </span><span class="se">\$</span><span class="sh">{EXTRACT_WORKER_PORT:-3004}
       WORKER_PORT: </span><span class="se">\$</span><span class="sh">{WORKER_PORT:-3005}
-      NUQ_RABBITMQ_URL: amqp://rabbitmq:5672
+      NUQ_RABBITMQ_URL: amqp://172.16.0.40:5672
       ENV: local
     depends_on:
       redis:
@@ -113,6 +117,7 @@ services:
     memswap_limit: 8G

   redis:
+    runtime: "runsc"
     # NOTE: If you want to use Valkey (open source) instead of Redis (source available),
     # uncomment the Valkey statement and comment out the Redis statement.
     # Using Valkey with Firecrawl is untested and not guaranteed to work. Use with caution.
@@ -120,7 +125,8 @@ services:
     # image: valkey/valkey:alpine

     networks:
-      - backend
+      backend:
+        ipv4_address: 172.16.0.30
     command: redis-server --bind 0.0.0.0
     logging:
       driver: "json-file"
@@ -130,9 +136,11 @@ services:
         compress: "true"

   rabbitmq:
+    runtime: "runsc"
     image: rabbitmq:3-management
     networks:
-      - backend
+      backend:
+        ipv4_address: 172.16.0.40
     command: rabbitmq-server
     healthcheck:
       test: ["CMD", "rabbitmq-diagnostics", "-q", "check_running"]
@@ -148,6 +156,7 @@ services:
         compress: "true"

   nuq-postgres:
+    runtime: "runsc"
     # NOTE: If you don't want to build the image locally,
     # comment out the build: statement and uncomment the image: statement
     # image: ghcr.io/firecrawl/nuq-postgres:latest
@@ -157,7 +166,8 @@ services:
       POSTGRES_PASSWORD: </span><span class="se">\$</span><span class="sh">{POSTGRES_PASSWORD:-postgres}
       POSTGRES_DB: </span><span class="se">\$</span><span class="sh">{POSTGRES_DB:-postgres}
     networks:
-      - backend
+      backend:
+        ipv4_address: 172.16.0.50
     logging:
       driver: "json-file"
       options:
@@ -168,3 +178,8 @@ services:
 networks:
   backend:
     driver: bridge
+    ipam:
+      config:
+        - gateway: 172.16.0.1
+          subnet: 172.16.0.0/16
+      driver: default
</span><span class="no">EOF

</span><span class="c"># Run.</span>
<span class="nv">$ </span><span class="o">(</span> <span class="nb">cd</span> <span class="s2">"</span><span class="nv">$CASPER</span><span class="s2">/firecrawl"</span><span class="p">;</span> docker compose build <span class="o">&amp;&amp;</span> docker compose up <span class="o">)</span>

<span class="c"># Make sure it works:</span>
<span class="nv">$ </span>curl <span class="nt">-X</span> POST http://localhost:3002/v1/crawl <span class="se">\</span>
    <span class="nt">-H</span> <span class="s1">'Content-Type: application/json'</span> <span class="se">\</span>
    <span class="nt">-d</span> <span class="s1">'{
      "url": "https://firecrawl.dev"
    }'</span>
<span class="o">{</span><span class="s2">"success"</span>:true,<span class="s2">"id"</span>:<span class="s2">"019d7a78-e77a-70af-9f49-8e03421dad32"</span>,<span class="s2">"url"</span>:<span class="s2">"http://localhost:3002/v1/crawl/019d7a78-e77a-70af-9f49-8e03421dad32"</span><span class="o">}</span>
</code></pre></div>      </div>

      <p>This brings up all the following applications in separate gVisor containers on
their own inter-container network:</p>

      <ul>
        <li><strong>Redis</strong> for key/value storage.</li>
        <li><strong>RabbitMQ</strong> for message queuing.</li>
        <li><strong>Playwright</strong> for browser automation.</li>
        <li><strong>PostgreSQL</strong> for long-term storage.</li>
        <li><strong>Firecrawl</strong> as main API endpoint for Hermes Agent to interact with.</li>
      </ul>

    </div>

  </div>

</section>

<section class="sticky-section">

  <h3 id="putting-it-all-together">Putting it all together</h3>

  <div class="sticky-section-body">

    <figure class="follow-along">
<img alt="Diagram showing the MAGI system with the 'Hermes Agent' container blinking." src="/assets/images/2026-04-15-magi/hermes-agent.blink.gif" />
<figcaption>Setting up Hermes Agent and connecting it.</figcaption>
</figure>

    <div class="section-content">

      <p>Let’s put the pieces together for the Hermes Agent container.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">export </span><span class="nv">CASPER</span><span class="o">=</span><span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/agents/casper-3"</span><span class="p">;</span> <span class="nb">mkdir</span> <span class="nt">-p</span> <span class="s2">"</span><span class="nv">$CASPER</span><span class="s2">"</span>

<span class="c"># Register Matrix user.</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> synapse register_new_matrix_user <span class="se">\</span>
    <span class="nt">-c</span> /data/homeserver.yaml <span class="se">\</span>
    <span class="nt">--user</span> casper <span class="nt">--password</span> naoko <span class="nt">--no-admin</span>

<span class="c"># Hermes requires a non-root user for its home directory.</span>
<span class="nv">$ </span>groupadd <span class="nt">--gid</span><span class="o">=</span>10337 hermes <span class="o">&amp;&amp;</span> <span class="se">\</span>
    useradd <span class="nt">--home-dir</span><span class="o">=</span>/dev/null <span class="nt">--no-create-home</span> <span class="nt">--shell</span><span class="o">=</span><span class="s2">"</span><span class="si">$(</span>which nologin<span class="si">)</span><span class="s2">"</span> <span class="se">\</span>
      <span class="nt">--uid</span><span class="o">=</span>10337 <span class="nt">--gid</span><span class="o">=</span>10337 hermes

<span class="c"># Build Docker image with extra packages.</span>
<span class="nv">$ </span><span class="nb">cat</span> <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh"> &gt; "</span><span class="nv">$CASPER</span><span class="sh">/Dockerfile"
FROM nousresearch/hermes-agent:v2026.4.13

# Install basic packages.
RUN export DEBIAN_FRONTEND=noninteractive; apt update -y &amp;&amp; </span><span class="se">\</span><span class="sh">
    apt install -y sudo wget curl git build-essential python3-pip

# Install dependencies for Hermes Agent's Matrix.org support.
RUN export DEBIAN_FRONTEND=noninteractive; apt update -y &amp;&amp; </span><span class="se">\</span><span class="sh">
    apt install -y libolm-dev &amp;&amp; </span><span class="se">\</span><span class="sh">
    python3 -m pip config set global.break-system-packages true &amp;&amp; </span><span class="se">\</span><span class="sh">
    pip install 'matrix-nio' 'mautrix[encryption]'

# Install espeak-ng and NeuTTS model for local text-to-speech capabilities.
RUN export DEBIAN_FRONTEND=noninteractive; apt update -y &amp;&amp; </span><span class="se">\</span><span class="sh">
    apt install -y espeak-ng &amp;&amp; </span><span class="se">\</span><span class="sh">
    pip install 'neutts[all]'

# Install Docker; not required for dockerd since that's running in a separate
# container, but Hermes Agent still needs the Docker **client** CLI.
RUN export DEBIAN_FRONTEND=noninteractive; apt update -y &amp;&amp; </span><span class="se">\</span><span class="sh">
    apt install -y docker.io
</span><span class="no">EOF

</span><span class="nv">$ </span>docker build <span class="nt">-t</span> hermes-agent:casper-3 <span class="s2">"</span><span class="nv">$CASPER</span><span class="s2">"</span>
</code></pre></div>      </div>

      <p>As Hermes Agent does not easily support non-interactive configuration, we need
to configure it manually. Let’s run it for interactive configuration purposes:</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">export </span><span class="nv">CASPER</span><span class="o">=</span><span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/agents/casper-3"</span><span class="p">;</span> <span class="se">\</span>
    <span class="nb">mkdir</span> <span class="s2">"</span><span class="nv">$CASPER</span><span class="s2">/home"</span> <span class="o">&amp;&amp;</span> <span class="nb">chown </span>hermes:hermes <span class="s2">"</span><span class="nv">$CASPER</span><span class="s2">/home"</span>
<span class="nv">$ </span>docker run <span class="nt">-it</span> <span class="se">\</span>
    <span class="nt">--name</span><span class="o">=</span>casper <span class="se">\</span>
    <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="se">\</span>
    <span class="nt">--restart</span><span class="o">=</span>always <span class="se">\</span>
    <span class="nt">--shm-size</span><span class="o">=</span>1g <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>synapse:synapse <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>ollama:ollama <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>camofox:camofox <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="s2">"type=bind,src=</span><span class="nv">$CASPER</span><span class="s2">/home,dst=/opt/data"</span> <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="s2">"type=bind,src=</span><span class="nv">$CASPER</span><span class="s2">/docker-run,dst=/docker-run"</span> <span class="se">\</span>
    <span class="nt">-e</span> <span class="nv">HERMES_UID</span><span class="o">=</span><span class="s2">"</span><span class="si">$(</span><span class="nb">id</span> <span class="nt">-u</span> hermes<span class="si">)</span><span class="s2">"</span> <span class="se">\</span>
    <span class="nt">-e</span> <span class="nv">HERMES_GID</span><span class="o">=</span><span class="s2">"</span><span class="si">$(</span><span class="nb">id</span> <span class="nt">-g</span> hermes<span class="si">)</span><span class="s2">"</span> <span class="se">\</span>
    <span class="nt">-e</span> <span class="nv">DOCKER_HOST</span><span class="o">=</span><span class="s2">"unix:///docker-run/docker.sock"</span> <span class="se">\</span>
    hermes-agent:casper-3 setup
</code></pre></div>      </div>

      <figure class="img-100pct">
<div class="double-border-glow">
<video loop="" src="/assets/images/2026-04-15-magi/hermes-agent-setup.webm"></video>
</div>

<figcaption>Going through Hermes Agent's interactive setup process in
gVisor.</figcaption> </figure>

      <details>

        

          <h4 id="interactive-setup-instructions">Interactive setup instructions</h4>

          <p>Expand this section for a text version of the screen recording above.</p>

        

        <ul>
          <li>Choose <code class="highlighter-rouge">Full setup</code></li>
          <li>Inference Provider: <code class="highlighter-rouge">More providers</code> → <code class="highlighter-rouge">Custom endpoint</code></li>
          <li>API base URL: <code class="highlighter-rouge">http://ollama:11434/v1</code></li>
          <li>API key: (leave empty)</li>
          <li>Select model: <code class="highlighter-rouge">qwen3.5:27b-q4_K_M</code></li>
          <li>Context length in tokens: <code class="highlighter-rouge">262144</code> (per the
<a href="https://huggingface.co/Qwen/Qwen3.5-27B">Qwen3.7-27B model card</a>)</li>
          <li>Select TTS provider: <code class="highlighter-rouge">NeuTTS</code> (local on-device)</li>
          <li>Terminal Backend: <code class="highlighter-rouge">Docker</code></li>
          <li>Docker image: (leave default)</li>
          <li>Container Resource Settings: Up to you</li>
          <li>Max iterations / Tool progress mode/ […] / Inactivity timeout: Up to you</li>
          <li>Select platforms: <code class="highlighter-rouge">Matrix</code></li>
          <li>Homeserver URL: <code class="highlighter-rouge">http://synapse:8008</code></li>
          <li>Access token: (leave empty)</li>
          <li>User ID: <code class="highlighter-rouge">@casper:magi</code></li>
          <li>Password: <code class="highlighter-rouge">naoko</code></li>
          <li>Enable end-to-end encryption (E2EE): Up to you</li>
          <li>Allowed user IDs: <code class="highlighter-rouge">@gendo:magi</code></li>
          <li>Home room ID: (leave empty)</li>
          <li>Install gateway as systemd service: No, as this isn’t relevant for a
containerized install.</li>
          <li>Tools: Feel free to configure.</li>
          <li>Browser provider: <code class="highlighter-rouge">Camofox</code></li>
          <li>Camofox server URL: <code class="highlighter-rouge">http://camofox:3000</code></li>
          <li>Image generation FAL API key: (leave empty unless you have one)</li>
          <li>TTS provider: Skip</li>
          <li>Search provider: <code class="highlighter-rouge">Self-hosted Firecrawl</code></li>
          <li>Firecrawl instance URL: <code class="highlighter-rouge">http://172.17.0.1:3002</code></li>
        </ul>

      </details>

      <p>You can verify that Hermes Agent’s “terminal” backend is the Docker-in-gVisor by
running <code class="highlighter-rouge">htop</code> in the <code class="highlighter-rouge">hermes-exec</code> container.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> hermes-exec sh <span class="nt">-c</span> <span class="s1">'apt update -y &amp;&amp; apt install -y htop'</span>

<span class="c"># Watch this command while asking Hermes Agent to run `curl https://gvisor.dev`:</span>
<span class="nv">$ </span>docker <span class="nb">exec</span> <span class="nt">-it</span> hermes-exec htop
</code></pre></div>      </div>

      <p>To make Hermes Agent actually join the Matrix room, you need to restart the
container in gateway mode.</p>

      <div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>docker <span class="nb">rm</span> <span class="nt">-f</span> casper<span class="p">;</span> docker run <span class="nt">--detach</span> <span class="se">\</span>
    <span class="nt">--name</span><span class="o">=</span>casper <span class="se">\</span>
    <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="se">\</span>
    <span class="nt">--restart</span><span class="o">=</span>always <span class="se">\</span>
    <span class="nt">--shm-size</span><span class="o">=</span>1g <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>synapse:synapse <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>ollama:ollama <span class="se">\</span>
    <span class="nt">--link</span><span class="o">=</span>camofox:camofox <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="s2">"type=bind,src=</span><span class="nv">$CASPER</span><span class="s2">/home,dst=/opt/data"</span> <span class="se">\</span>
    <span class="nt">--mount</span><span class="o">=</span><span class="s2">"type=bind,src=</span><span class="nv">$CASPER</span><span class="s2">/docker-run,dst=/docker-run"</span> <span class="se">\</span>
    <span class="nt">-e</span> <span class="nv">DOCKER_HOST</span><span class="o">=</span><span class="s2">"unix:///docker-run/docker.sock"</span> <span class="se">\</span>
    hermes-agent:casper-3 gateway
</code></pre></div>      </div>

      <p>Now invite the bot to your Matrix room and send <code class="highlighter-rouge">/sethome</code> on the main channel.</p>

      <p>You now have Hermes Agent running in gVisor. To recap, Hermes Agent has:</p>

      <ul>
        <li><strong>Hermes Agent</strong> running in its own gVisor container</li>
        <li><strong><code class="highlighter-rouge">dockerd</code></strong> running in a separate gVisor container, for subcommand
execution</li>
        <li><strong>Camofox Browser</strong> running with a virtual display (<strong><code class="highlighter-rouge">Xvfb</code></strong>) for browser
use, in its own gVisor container</li>
        <li>Self-hosted <strong>Firecrawl</strong> for agentic search, in its own set of gVisor
containers.</li>
        <li><strong>NeuTTS</strong> for text-to-speech capabilities in Hermes Agent, evaluated within
gVisor.</li>
        <li><strong>Ollama</strong> for inference and <strong>Matrix.org</strong> for communication, same as the
other agents.</li>
      </ul>

    </div>

  </div>

</section>

<section class="sticky-section">

  <h3 id="putting-these-agents-in-a-room">Putting these agents in a room</h3>

  <p>You can now ask your 3 agents to do your bidding and get various perspectives.</p>

  <figure class="img-100pct">
<div class="double-border-glow">
<img alt="All three agents together in a Matrix.org room displayed in the Cinny web UI, with each agent fetching the gVisor homepage and confirming that they are each running in gVisor." src="/assets/images/2026-04-15-magi/magi-three-way.png" />
</div>
<figcaption>The three agents fetching the gVisor homepage and verifying that they are running in gVisor.<br />Note: Hermes Agent cannot call <code>dmesg</code>, due to the default system call filter applied to the Docker container that its code execution tool runs in.<br />However, the <code>4.4.0</code> kernel version is characteristic of gVisor.</figcaption>
</figure>

</section>

<section class="sticky-section">

  <h2 id="sandboxing-agents-what-actually-makes-sense">Sandboxing agents: What actually makes sense?</h2>

  <p>The setup described in this blog post is a contrived example of agent
sandboxing, where every part of the stack is mutually sandboxed from one
another. In closer-to-real-world settings, not all of these components are
untrusted, some of them will run remotely, others may be delegated to
off-machine APIs, etc. So what would a more practical setup look like?</p>

  <p>At a high level, an autonomous agent stack looks like this:</p>

  <ul>
    <li>A <strong>core daemon</strong> (written in good old regular code, e.g. TypeScript for
OpenClaw), typically listening on a TCP port. This daemon is responsible
for:
      <ul>
        <li>Receiving user requests via a communications plugin (e.g. Signal,
Mattermost…)</li>
        <li>Running inference API calls</li>
        <li>Dispatching tool calls</li>
        <li>Running the control loop necessary to make forward progress on long-term
tasks, using inference and tool calls</li>
        <li>Running cron-like tasks and
<a href="https://docs.openclaw.ai/gateway/heartbeat">heartbeats</a> to keep the
agent autonomous</li>
      </ul>
    </li>
    <li>A pretty <strong>web interface</strong> (sometimes part of the core daemon, sometimes
separate)</li>
    <li>A <strong>plugin ecosystem</strong>, adding new tools, communication channels, etc. to
the agent</li>
    <li>A database of <strong>skills and general knowledge</strong> (memory) that the agent can
evolve over time as they learn from its mistakes, or learn more about their
raison d’être and the user they are dealing with.</li>
    <li>A <strong>policy engine</strong> that can decide on the security policies needed for any
action the agent would like to take (tool call, API call, credential access,
etc.).</li>
  </ul>

  <p>When you send a message to such an agent, it ends up running a control loop to
handle your query. This control loop will initially run inference, then very
likely follow this up by a sequence of tool calls and further inference
requests, until a satisfying conclusion is reached. These tool calls can
include:</p>

  <ul>
    <li><strong>Data lookups</strong> on the web</li>
    <li><strong>API requests</strong> to external services, often requiring sensitive credentials
to “act as” the user</li>
    <li><strong>Browser use</strong>, sometimes with similar credential needs</li>
    <li><strong>Code snippet</strong> executions</li>
    <li><strong>Memory</strong> reads and writes, database-like</li>
    <li><strong>Introspection requests</strong>, where the agent can modify its own configuration
or skill database, sometimes fixing its own setup/configuration issues
rather than requiring a human to get it unstuck.</li>
  </ul>

  <p>Where does sandboxing fit in?</p>

  <ul>
    <li><strong>Sandboxing individual tools</strong>: Most tool calls don’t do anything fancy.
They just make web requests and are not expected to have side-effects. They
have no business reading local files or modifying the agent’s own
configuration. Sandboxing these tools allows for defense-in-depth.
      <ul>
        <li>Concrete example: One can craft malicious <code class="highlighter-rouge">.mov</code> videos which can refer
to arbitrary file paths on the host. What if your agent gets tricked
into converting a video that tries to embed a subtitle file pointing to
<code class="highlighter-rouge">/etc/shadow</code>? Sandbox your tool calls and avoid this problem.</li>
      </ul>
    </li>
    <li><strong>Sandboxing subsystems</strong>: Some agent functionality may depend on
long-running daemons which themselves don’t need system-wide access. This
can be important for network-exposed or network-accessing subsystems.
      <ul>
        <li>Concrete example: If using Signal as communications layer, the
<a href="https://github.com/AsamK/signal-cli"><code class="highlighter-rouge">signal-cli</code> daemon</a> can run in a
sandbox for defense-in-depth.</li>
        <li>Similarly, in the examples above, we sandbox <code class="highlighter-rouge">dockerd</code> and Camofox
browser in separate containers.</li>
      </ul>
    </li>
    <li><strong>Sandbox the core daemon</strong>: The need for the agent to be able to <strong>change
its own environment</strong> to debug or update itself is a very powerful feature.
To do so, the agent requires effectively root control over its own core code
and configuration. Therefore, <strong>sandboxing the entire agent’s core daemon</strong>
makes sense: the agent can leverage its own intelligence to make itself
better, while still being confined to a box. That box is useful because:
      <ul>
        <li>Destructive changes can be <strong>rolled back</strong>.</li>
        <li>The agent’s <strong>policy engine can live outside</strong> the core sandbox. This
prevents the agent from changing the policy engine’s policies
maliciously.</li>
        <li>Relatedly, sensitive <strong>credentials can live outside</strong> the core sandbox.
This ensures that all credential use is mediated through components the
agent can’t modify. This includes API keys, crypto wallet keys for
agentic commerce, and user-authenticated browser sessions.</li>
      </ul>
    </li>
  </ul>

  <p><em>Other parts of the stack typically run fully-trusted code with little to no
need for sandboxing. For example, the memory subsystem may be a local vector
lookup or similar database, with no internet connectivity and no need to run
arbitrary code. Thus, similar to the
<a href="/docs/user_guide/production/">gVisor production guide</a>, it does not need to be
sandboxed.</em></p>

  <p>We see some of these ideas being implemented across the ecosystem:</p>

  <ul>
    <li>OpenClaw supports agent-level containerization via
<a href="https://docs.openclaw.ai/install/docker">Docker</a> and
<a href="https://docs.openclaw.ai/install/podman">Podman</a>.</li>
    <li>NemoClaw uses <a href="https://github.com/NVIDIA/OpenShell">OpenShell</a> to ensure
tool calls have initially-restricted access which can then be widened as
needed by the tool.</li>
    <li>Hermes Agent implements
<a href="https://hermes-agent.nousresearch.com/docs/user-guide/checkpoints-and-rollback">checkpoints and rollbacks</a>
to protect against destructive operations.</li>
    <li><a href="https://www.ironclaw.com/">IronClaw</a> segregates API keys out of the agent’s
core sandbox and injects them at egress time.</li>
  </ul>

  <p>Security practices for these tools are rapidly evolving, and gVisor has a role
to play.</p>

</section>

<section class="sticky-section">

  <h2 id="should-i-use-gvisor-to-sandbox-my-agent">Should I use gVisor to sandbox my agent?</h2>

  <p>gVisor dramatically <strong>reduces the attack surface</strong> for sandbox escapes. It does
so by reimplementing a large portion of Linux in userspace, preventing the
sandboxed application from attacking the host kernel. Read
<a href="https://gvisor.dev/docs/architecture_guide/intro/">more about gVisor’s security architecture</a>.</p>

  <p>For autonomous agents, you don’t just need a strong sandbox, you also need
<strong>strong policies around <em>when</em> and <em>what</em> to sandbox</strong>. As a sandboxing
technology, gVisor does not help you with these decisions. gVisor only
<strong>enhances the level of security of the sandboxing capabilities that the agent
already has</strong>. Thus, <strong>gVisor is <em>necessary</em>, but not <em>sufficient</em></strong>.</p>

  <p>gVisor’s capabilities are also uniquely well-suited to agentic workloads:</p>

  <ul>
    <li>Sandboxes <strong>start and stop in milliseconds</strong>, critical to keeping these
systems responsive and minimizing time between inference calls.</li>
    <li>Thanks to its process-like model (not a virtual machine), gVisor can achieve
<strong>superior density</strong>, i.e. more sandboxes running concurrently on the same
host.</li>
    <li>gVisor supports <strong>checkpoint/restore</strong>, making slow-to-initialize repetitive
actions quick to replay, and checkpoints/rollbacks can be done seamlessly
without sandboxed-workload-specific support.</li>
  </ul>

  <p>One current drawback of gVisor is its relative difficulty to integrate within
existing applications that have such sandboxing needs. For example, this is one
reason why the above demo does not sandbox Hermes Agent tool calls in
<strong>separate</strong> gVisor instances. This is being worked on. Watch this space!</p>

</section>

<figure class="img-100pct">
<img alt="Diagram showing the MAGI system: three agents running in gVisor, along with a lot of side-services in gVisor-sandboxed containers. Evangelion style." src="/assets/images/2026-04-15-magi/magi.gif" />
<figcaption><em>*cogitation intensifies*</em></figcaption>
</figure>

<!--* pragma: { seclinter_this_is_fine: false } *-->
