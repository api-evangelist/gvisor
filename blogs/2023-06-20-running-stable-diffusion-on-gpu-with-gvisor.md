---
title: "Running Stable Diffusion on GPU with gVisor"
url: "/blog/2023/06/20/gpu-pytorch-stable-diffusion/"
date: "2023-06-20T00:00:00-05:00"
author: "eperot"
feed_url: "https://gvisor.dev/blog/index.xml"
---
<p>gVisor is <a href="https://github.com/google/gvisor/blob/master/g3doc/proposals/nvidia_driver_proxy.md">starting to support GPU</a> workloads. This post
showcases running the <a href="https://stability.ai/blog/stable-diffusion-public-release">Stable Diffusion</a> generative model from <a href="https://stability.ai/">Stability AI</a> to
generate images using a GPU from within gVisor. Both the
<a href="https://github.com/AUTOMATIC1111/stable-diffusion-webui">Automatic1111 Stable Diffusion web UI</a>
and the <a href="https://pytorch.org/">PyTorch</a> code used by Stable Diffusion were run entirely within gVisor
while being able to leverage the NVIDIA GPU.</p>

<!--/excerpt-->

<p><img alt="A sandboxed GPU" src="/assets/images/2023-06-20-sandboxed-gpu.png" title="A sandboxed GPU." />
<span class="attribution"><strong>Sand</strong>boxing a GPU. Generated with Stable Diffusion
v1.5.<br />This picture gets a lot deeper once you realize that GPUs are made out
of sand.</span></p>

<h2 id="disclaimer">Disclaimer</h2>

<p>As of this writing (2023-06), <a href="https://github.com/google/gvisor/blob/master/g3doc/proposals/nvidia_driver_proxy.md">gVisor’s GPU support</a> is not
generalized. Only some PyTorch workloads have been tested on NVIDIA T4, L4,
A100, and H100 GPUs, using the specific driver versions that your runsc version
supports using the command below. Contributions are welcome to expand this set
to support other GPUs and driver versions!</p>

<div class="highlighter-rouge"><div class="highlight"><pre class="highlight"><code># From a cloned gVisor repository:
$ make run TARGETS=runsc ARGS="nvproxy list-supported-drivers"

# From a runsc binary:
$ runsc nvproxy list-supported-drivers
</code></pre></div></div>

<p>Additionally, while gVisor does its best to sandbox the workload, interacting
with the GPU inherently requires running code on GPU hardware, where isolation
is enforced by the GPU driver and hardware itself rather than gVisor. More to
come soon on the value of the protection gVisor provides for GPU workloads.</p>

<p>In a few months, gVisor’s GPU support will have broadened and become
easier-to-use, such that it will not be constrained to the specific sets of
versions used here. In the meantime, this blog stands as an example of what’s
possible today with gVisor’s GPU support.</p>

<p><img alt="Various space suit helmets" src="/assets/images/2023-06-20-spacesuit-helmets.png" title="Various space suit helmets." width="100%" />
<span class="attribution"><strong>A collection of astronaut helmets in various styles</strong>.<br />Other than the helmet in the center, each helmet was generated using Stable Diffusion v1.5.</span></p>

<h2 id="why-even-do-this">Why even do this?</h2>

<p>The recent explosion of machine learning models has led to a large number of new
open-source projects. Much like it is good practice to be careful about running
new software downloaded from the Internet, it is good practice to run new
open-source projects in a sandbox. For projects like the
<a href="https://github.com/AUTOMATIC1111/stable-diffusion-webui">Automatic1111 Stable Diffusion web UI</a>,
which automatically download various models, components, and
<a href="https://github.com/AUTOMATIC1111/stable-diffusion-webui-extensions/blob/master/index.json">extensions</a> from external repositories as
the user enables them in the web UI, this principle applies all the more.</p>

<p>Additionally, within the machine learning space, tooling for packaging and
distributing models are still nascent. While some models (including Stable
Diffusion) are packaged using the more secure <a href="https://github.com/huggingface/safetensors">safetensors</a> format, <strong>the
majority of models available online today are distributed using the
<a href="https://www.splunk.com/en_us/blog/security/paws-in-the-pickle-jar-risk-vulnerability-in-the-model-sharing-ecosystem.html">Pickle format</a>, which can execute arbitrary Python code</strong> upon deserialization.
As such, even when using trustworthy software, using Pickle-formatted models may
still be risky (<strong>Edited 2024-04-04:
<a href="https://www.wiz.io/blog/wiz-and-hugging-face-address-risks-to-ai-infrastructure">this exact vulnerability vector was found in Hugging Face’s Inference API</a></strong>).
gVisor provides a layer of protection around this process which helps protect
the host machine.</p>

<p>Third, <strong>machine learning applications are typically not I/O heavy</strong>, which
means they tend not to experience a significant performance overhead. The
process of uploading code to the GPU is not a significant number of system
calls, and most communication to/from the GPU happens over shared memory, where
gVisor imposes no overhead. Therefore, the question is not so much “why should I
run this GPU workload in gVisor?” but rather “why not?”.</p>

<p><img alt="Cool astronauts don't look at explosions" src="/assets/images/2023-06-20-turbo.png" title="Cool astronauts don't look at explosions." />
<span class="attribution"><strong>Cool astronauts don’t look at explosions</strong>.
Generated using Stable Diffusion v1.5.</span></p>

<p>Lastly, running GPU workloads in gVisor is pretty cool.</p>

<h2 id="setup">Setup</h2>

<p>We use a Debian virtual machine on GCE. The machine needs to have a GPU and to
have sufficient RAM and disk space to handle Stable Diffusion and its large
model files. The following command creates a VM with 4 vCPUs, 15GiB of RAM, 64GB
of disk space, and an NVIDIA T4 GPU, running Debian 11 (bullseye). Since this is
just an experiment, the VM is set to self-destruct after 6 hours.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>gcloud compute instances create stable-diffusion-testing <span class="se">\</span>
    <span class="nt">--zone</span><span class="o">=</span>us-central1-a <span class="se">\</span>
    <span class="nt">--machine-type</span><span class="o">=</span>n1-standard-4 <span class="se">\</span>
    <span class="nt">--max-run-duration</span><span class="o">=</span>6h <span class="se">\</span>
    <span class="nt">--instance-termination-action</span><span class="o">=</span>DELETE <span class="se">\</span>
    <span class="nt">--maintenance-policy</span> TERMINATE <span class="se">\</span>
    <span class="nt">--accelerator</span><span class="o">=</span><span class="nv">count</span><span class="o">=</span>1,type<span class="o">=</span>nvidia-tesla-t4 <span class="se">\</span>
    <span class="nt">--create-disk</span><span class="o">=</span>auto-delete<span class="o">=</span><span class="nb">yes</span>,boot<span class="o">=</span><span class="nb">yes</span>,device-name<span class="o">=</span>stable-diffusion-testing,image<span class="o">=</span>projects/debian-cloud/global/images/debian-11-bullseye-v20230509,mode<span class="o">=</span>rw,size<span class="o">=</span>64
<span class="nv">$ </span>gcloud compute ssh <span class="nt">--zone</span><span class="o">=</span>us-central1-a stable-diffusion-testing
</code></pre></div></div>

<p>All further commands in this post are performed while SSH’d into the VM. We
first need to install the specific NVIDIA driver version that gVisor is
currently compatible with.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="nb">sudo </span>apt-get <span class="nt">-y</span> upgrade
<span class="nv">$ </span><span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> build-essential linux-headers-<span class="si">$(</span><span class="nb">uname</span> <span class="nt">-r</span><span class="si">)</span>
<span class="nv">$ </span>runsc nvproxy list-supported-drivers
<span class="nv">$ DRIVER_VERSION</span><span class="o">=</span>some-driver-version <span class="c"># Get from your runsc binary.</span>
<span class="nv">$ </span>curl <span class="nt">-fSsl</span> <span class="nt">-O</span> <span class="s2">"https://us.download.nvidia.com/tesla/</span><span class="nv">$DRIVER_VERSION</span><span class="s2">/NVIDIA-Linux-x86_64-</span><span class="nv">$DRIVER_VERSION</span><span class="s2">.run"</span>
<span class="nv">$ </span><span class="nb">sudo </span>sh NVIDIA-Linux-x86_64-<span class="nv">$DRIVER_VERSION</span>.run
</code></pre></div></div>

<!--
The above in a single line, for convenience:
DRIVER_VERSION=some-driver-version; sudo apt-get update && sudo apt-get -y upgrade && sudo apt-get install -y build-essential linux-headers-$(uname -r) && curl -fSsl -O "https://us.download.nvidia.com/tesla/$DRIVER_VERSION/NVIDIA-Linux-x86_64-$DRIVER_VERSION.run" && sudo sh NVIDIA-Linux-x86_64-$DRIVER_VERSION.run
-->

<p>Next, we install Docker, per <a href="https://docs.docker.com/engine/install/debian/">its instructions</a>.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> ca-certificates curl gnupg
<span class="nv">$ </span><span class="nb">sudo install</span> <span class="nt">-m</span> 0755 <span class="nt">-d</span> /etc/apt/keyrings
<span class="nv">$ </span>curl <span class="nt">-fsSL</span> https://download.docker.com/linux/debian/gpg | <span class="nb">sudo </span>gpg <span class="nt">--dearmor</span> <span class="nt">--batch</span> <span class="nt">--yes</span> <span class="nt">-o</span> /etc/apt/keyrings/docker.gpg
<span class="nv">$ </span><span class="nb">sudo chmod </span>a+r /etc/apt/keyrings/docker.gpg
<span class="nv">$ </span><span class="nb">echo</span> <span class="s2">"deb [arch=</span><span class="si">$(</span>dpkg <span class="nt">--print-architecture</span><span class="si">)</span><span class="s2"> signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian </span><span class="si">$(</span><span class="nb">.</span> /etc/os-release <span class="o">&amp;&amp;</span> <span class="nb">echo</span> <span class="s2">"</span><span class="nv">$VERSION_CODENAME</span><span class="s2">"</span><span class="si">)</span><span class="s2"> stable"</span> | <span class="nb">sudo tee</span> /etc/apt/sources.list.d/docker.list <span class="o">&gt;</span> /dev/null
<span class="nv">$ </span><span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> docker-ce docker-ce-cli
</code></pre></div></div>

<!--
The above in a single live, for convenience:
sudo apt-get install -y ca-certificates curl gnupg && sudo install -m 0755 -d /etc/apt/keyrings && curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor --batch --yes -o /etc/apt/keyrings/docker.gpg && sudo chmod a+r /etc/apt/keyrings/docker.gpg && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli
-->

<p>We will also need the <a href="https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/user-guide.html">NVIDIA container toolkit</a>, which enables use of GPUs with
Docker. Per its
<a href="https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html">installation instructions</a>:</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ distribution</span><span class="o">=</span><span class="si">$(</span><span class="nb">.</span> /etc/os-release<span class="p">;</span><span class="nb">echo</span> <span class="nv">$ID$VERSION_ID</span><span class="si">)</span> <span class="o">&amp;&amp;</span> curl <span class="nt">-fsSL</span> https://nvidia.github.io/libnvidia-container/gpgkey | <span class="nb">sudo </span>gpg <span class="nt">--dearmor</span> <span class="nt">-o</span> /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg <span class="o">&amp;&amp;</span> curl <span class="nt">-s</span> <span class="nt">-L</span> https://nvidia.github.io/libnvidia-container/<span class="nv">$distribution</span>/libnvidia-container.list | <span class="nb">sed</span> <span class="s1">'s#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g'</span> | <span class="nb">sudo tee</span> /etc/apt/sources.list.d/nvidia-container-toolkit.list
<span class="nv">$ </span><span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> nvidia-container-toolkit
</code></pre></div></div>

<p>Of course, we also need to <a href="https://gvisor.dev/docs/user_guide/install/">install gVisor</a> itself.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> apt-transport-https ca-certificates curl gnupg
<span class="nv">$ </span>curl <span class="nt">-fsSL</span> https://gvisor.dev/archive.key | <span class="nb">sudo </span>gpg <span class="nt">--dearmor</span> <span class="nt">-o</span> /usr/share/keyrings/gvisor-archive-keyring.gpg
<span class="nv">$ </span><span class="nb">echo</span> <span class="s2">"deb [arch=</span><span class="si">$(</span>dpkg <span class="nt">--print-architecture</span><span class="si">)</span><span class="s2"> signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main"</span> | <span class="nb">sudo tee</span> /etc/apt/sources.list.d/gvisor.list <span class="o">&gt;</span> /dev/null
<span class="nv">$ </span><span class="nb">sudo </span>apt-get update <span class="o">&amp;&amp;</span> <span class="nb">sudo </span>apt-get <span class="nb">install</span> <span class="nt">-y</span> runsc

＃ As gVisor does not yet <span class="nb">enable </span>GPU support by default, we need to <span class="nb">set </span>the flags
＃ that will <span class="nb">enable </span>it:
<span class="nv">$ </span><span class="nb">sudo </span>runsc <span class="nb">install</span> <span class="nt">--</span> <span class="nt">--nvproxy</span><span class="o">=</span><span class="nb">true</span> <span class="nt">--nvproxy-docker</span><span class="o">=</span><span class="nb">true</span>

<span class="nv">$ </span><span class="nb">sudo </span>systemctl restart docker
</code></pre></div></div>

<p>Now, let’s make sure everything works by running commands that involve more and
more of what we just set up.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code>＃ Check that the NVIDIA drivers are installed, with the right version, and with
＃ a supported GPU attached
<span class="nv">$ </span><span class="nb">sudo </span>nvidia-smi <span class="nt">-L</span>
GPU 0: Tesla T4 <span class="o">(</span>UUID: GPU-6a96a2af-2271-5627-34c5-91dcb4f408aa<span class="o">)</span>
<span class="nv">$ </span><span class="nb">sudo cat</span> /proc/driver/nvidia/version
NVRM version: NVIDIA UNIX x86_64 Kernel Module  DRIVER_VERSION  Wed Nov 30 06:39:21 UTC 2022

＃ Check that Docker works.
<span class="nv">$ </span><span class="nb">sudo </span>docker version
＃ <span class="o">[</span>...]
Server: Docker Engine - Community
 Engine:
  Version:          24.0.2
＃ <span class="o">[</span>...]

＃ Check that gVisor works.
<span class="nv">$ </span><span class="nb">sudo </span>docker run <span class="nt">--rm</span> <span class="nt">--runtime</span><span class="o">=</span>runsc debian:latest dmesg | <span class="nb">head</span> <span class="nt">-1</span>
<span class="o">[</span>    0.000000] Starting gVisor...

＃ Check that Docker GPU support <span class="o">(</span>without gVisor<span class="o">)</span> works.
<span class="nv">$ </span><span class="nb">sudo </span>docker run <span class="nt">--rm</span> <span class="nt">--gpus</span><span class="o">=</span>all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi <span class="nt">-L</span>
GPU 0: Tesla T4 <span class="o">(</span>UUID: GPU-6a96a2af-2271-5627-34c5-91dcb4f408aa<span class="o">)</span>

＃ Check that gVisor works with the GPU.
<span class="nv">$ </span><span class="nb">sudo </span>docker run <span class="nt">--rm</span> <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="nt">--gpus</span><span class="o">=</span>all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi <span class="nt">-L</span>
GPU 0: Tesla T4 <span class="o">(</span>UUID: GPU-6a96a2af-2271-5627-34c5-91dcb4f408aa<span class="o">)</span>
</code></pre></div></div>

<p>We’re all set! Now we can actually get Stable Diffusion running.</p>

<p>We used the following <code class="highlighter-rouge">Dockerfile</code> to run Stable Diffusion and its web UI within
a GPU-enabled Docker container.</p>

<div class="language-dockerfile highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">FROM</span><span class="s"> python:3.10</span>

＃ Set of dependencies that are needed to make this work.
<span class="k">RUN </span>apt-get update <span class="o">&amp;&amp;</span> apt-get <span class="nb">install</span> <span class="nt">-y</span> git wget build-essential <span class="se">\
</span>        nghttp2 libnghttp2-dev libssl-dev ffmpeg libsm6 libxext6
＃ Clone the project at the revision used for this test.
<span class="k">RUN </span>git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git <span class="o">&amp;&amp;</span> <span class="se">\
</span>    <span class="nb">cd</span> /stable-diffusion-webui <span class="o">&amp;&amp;</span> <span class="se">\
</span>    git checkout baf6946e06249c5af9851c60171692c44ef633e0
＃ We don't want the build step to start the server.
<span class="k">RUN </span><span class="nb">sed</span> <span class="nt">-i</span> <span class="s1">'/start()/d'</span> /stable-diffusion-webui/launch.py
＃ Install some pip packages.
＃ Note that this command will run as part of the Docker build process,
＃ which is *not* sandboxed by gVisor.
<span class="k">RUN </span><span class="nb">cd</span> /stable-diffusion-webui <span class="o">&amp;&amp;</span> <span class="nv">COMMANDLINE_ARGS</span><span class="o">=</span><span class="nt">--skip-torch-cuda-test</span> python launch.py
<span class="k">WORKDIR</span><span class="s"> /stable-diffusion-webui</span>
＃ This causes the web UI to use the Gradio service to create a public URL.
＃ Do not use this if you plan on leaving the container running long-term.
<span class="k">ENV</span><span class="s"> COMMANDLINE_ARGS=--share</span>
＃ Start the webui app.
<span class="k">CMD</span><span class="s"> ["python", "webui.py"]</span>
</code></pre></div></div>

<p>We build the image and create a container with it using the <code class="highlighter-rouge">docker</code>
command-line.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">cat</span> <span class="o">&gt;</span> Dockerfile
<span class="o">(</span>... Paste the above contents...<span class="o">)</span>
^D
<span class="nv">$ </span><span class="nb">sudo </span>docker build <span class="nt">--tag</span><span class="o">=</span>sdui <span class="nb">.</span>
</code></pre></div></div>

<p>Finally, we can start the Stable Diffusion web UI. Note that it will take a long
time to start, as it has to download all the models from the Internet. To keep
this post simple, we didn’t set up any kind of volume that would enable data
persistence, so it will do this every time the container starts.</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>docker run <span class="nt">--runtime</span><span class="o">=</span>runsc <span class="nt">--gpus</span><span class="o">=</span>all <span class="nt">--name</span><span class="o">=</span>sdui <span class="nt">--detach</span> sdui

＃ Follow the logs:
<span class="nv">$ </span><span class="nb">sudo </span>docker logs <span class="nt">-f</span> sdui
＃ <span class="o">[</span>...]
Calculating sha256 <span class="k">for</span> /stable-diffusion-webui/models/Stable-diffusion/v1-5-pruned-emaonly.safetensors: Running on <span class="nb">local </span>URL:  http://127.0.0.1:7860
Running on public URL: https://4446d982b4129a66d7.gradio.live

This share <span class="nb">link </span>expires <span class="k">in </span>72 hours.
＃ <span class="o">[</span>...]
</code></pre></div></div>

<p>We’re all set! Now we can browse to the Gradio URL shown in the logs and start
generating pictures, all within the secure confines of gVisor.</p>

<p><img alt="Stable Diffusion Web UI" src="/assets/images/2023-06-20-stable-diffusion-web-ui.png" title="Stable Diffusion UI." width="100%" />
<span class="attribution"><strong>Stable Diffusion Web UI screenshot.</strong> Inner image
generated with Stable Diffusion v1.5.</span></p>

<p>Happy sandboxing!</p>

<p><img alt="Astronaut showing thumbs up" src="/assets/images/2023-06-20-astronaut-thumbs-up.png" title="Astronaut showing thumbs up." />
<span class="attribution"><strong>Happy sandboxing!</strong> Generated with Stable Diffusion
v1.5.</span></p>
