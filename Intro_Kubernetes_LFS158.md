#Intro to Kubernetes LFS158

## Monolithic: 
Legacy monolithic applications are heavy, complex single-process systems that face severe challenges when migrating to the cloud because they are difficult to scale, demand expensive high-capacity hardware, and require planned downtime for maintenance.

### Key Supporting Points
Cloud Unfriendliness: Like a "1000-ton boulder," a monolith’s tightly coupled architecture makes simple lift-and-shift cloud migration impractical.

Operational Friction: Accumulated code complexity slows down development, compiling, and deployment cycles.

Resource & Scaling Limits: Scaling individual features is impossible; scaling the entire application requires provisioning costly, specialized hardware or manually deploying entire duplicate instances behind expensive load balancers.

High Maintenance Cost: Updates and patches inevitably cause service downtime and require complex, expensive high-availability setups to mitigate.

## Modern Microservices:
Modern microservices break down monolithic applications into small, independent, API-driven components that are easy to deploy, run on cost-effective hardware, scale granularly on demand, and update continuously without service downtime.

### Key Supporting Points
Modular & Functional Division: Applications are composed of loosely coupled, single-purpose services that communicate over network APIs, making them flexible to manage and relocate like "pebbles."

Resource & Cost Efficiency: Services run on lightweight, commodity hardware tailored to their specific needs rather than requiring massive, expensive single servers.

Granular Scalability: Individual components can scale independently—either manually or automatically based on traffic demand—without replicating the entire system.

Zero-Downtime Deployments & Agility: Updates are rolled out seamlessly one service at a time, allowing isolated development teams to release features faster without customer disruptions.

# Container Orchestration: 

### Container:
Containers are an application-centric method to deliver high-performing, scalable applications by providing portable, isolated virtual environments that encapsulate microservices alongside their runtimes, libraries, and dependencies to run without interference across any infrastructure.

**Core Conceptual Components:**
- Primary Purpose: Delivers microservices within lightweight, isolated runtime environments that prevent operational interference from other applications.
- Packaging & Encapsulation: Rather than running code directly, containers execute container images—bundles containing the application code, runtime, system libraries, and required dependencies.
- Portability: Built to run consistently across any underlying platform, including local developer workstations, Virtual Machines, and public cloud environments.

### Container Orchestrator:
Is an automated tool that manages and groups multiple systems into a unified cluster to automate the deployment, scaling, networking, and lifecycle of containers at scale—ensuring high availability, fault tolerance, optimal resource utilization, and zero-downtime updates across production environments.

**Examples:** 
```text
- Amazon Elastic Container Service (ECS) is a hosted service provided by Amazon Web Services (AWS) to run containers at scale on its infrastructure.
- Azure Container Instance (ACI) is a basic container orchestration service provided by Microsoft Azure.
- Azure Service Fabric is an open source container orchestrator provided by Microsoft Azure.
- Kubernetes is an open source orchestration tool, originally started by Google, today part of the Cloud Native Computing Foundation (CNCF) project.
- Nomad is the container and workload orchestrator provided by HashiCorp.
- Docker Swarm is a container orchestrator provided by Docker, Inc. It is part of Docker Engine.
``` 

# Kubernetes: 

Kubernetes provides an extensible, production-ready feature set that automates the entire operational lifecycle of containerized applications—handling scheduling, scaling, networking, storage, updates, and self-healing without requiring manual intervention.
Written in GO lenguage. 
License in Apache Version 2. 

# Kubernetes Architecture: 

![K8S Architecture](../K8S/K8S_Architecture.png)

## Components: 

### Control Plane Node: 
Is the “brain” of a Kubernetes cluster: it decides what should run, where, and keeps everything in the desired state.
- You talk to it through a CLI (kubectl), a web dashboard, or an API.
- If it goes down, your apps can suffer downtime, so you add replica control plane nodes (HA): one is active, the others stay in sync as backup.
- It stores the cluster’s state (config only, not your app data) in a key-value store (etcd).
- That store can live on the control plane nodes (stacked), simpler and replicated with them, or on separate hosts (external), safer, but you must replicate those hosts too, which costs more.

**Control Plane Components:**
* API server:

```text
    All the administrative tasks are coordinated by the kube-apiserver, a central control plane component running on the control plane node.

    Intercepts RESTful calls from users, administrators, developers, operators and external agents, then validates and processes them.

    During processing the API Server reads the Kubernetes cluster's current state from the key-value store, and after a call's execution, the resulting state of the Kubernetes cluster is saved in the key-value store for persistence.

    Is the only control plane component to talk to the key-value store, both to read from and to save Kubernetes cluster state information - acting as a middle interface for any other control plane agent inquiring about the cluster's state.

    It can scale horizontally, but it also supports the addition of custom secondary API Servers, a configuration that transforms the primary API Server into a proxy to all secondary, custom API Servers, routing all incoming RESTful calls to them based on custom defined rules.
```

* Scheduler
```text
    Summary: The kube-scheduler decides which node a new workload (like a pod) should run on. It looks at the cluster’s current state and the workload’s requirements, filters out unsuitable nodes, scores the rest, and picks the best one. It then reports its choice to the API Server, which hands the deployment off to other control plane agents.

Key points:

    Role: assigns new pods to nodes, usually worker nodes.
    Inputs (via the API Server): resource usage of each node, plus the workload’s requirements.
    Requirements it considers: user constraints (e.g., disk==ssd label), QoS, data locality, affinity/anti-affinity, taints/tolerations, and cluster topology.
    Process: filter nodes (predicates) → score the candidates (priorities) → pick the best one.
    Output: the decision goes back to the API Server, which triggers the deployment.
    Customizable: scheduling policies, plugins, and profiles are supported.
    Custom schedulers: a workload can name one in its config; if it doesn’t, the default scheduler is used.
    Importance: critical in multi-node clusters, but very simple in single-node clusters used for learning and development.
```

* Controller managers
```text
    Summary: Controller managers run controllers: loops that constantly compare the cluster’s desired state (from object configs) with its current state (from the key-value store, via the API Server). If they don’t match, the controller acts until they do.

Key points:

|   Controllers = watch loops: they run continuously, compare desired vs. current state, and correct any mismatch.
    kube-controller-manager: handles core cluster tasks:
        reacts when nodes become unavailable
        keeps pod counts as expected
        creates endpoints, service accounts, and API access tokens
    cloud-controller-manager: talks to the cloud provider’s infrastructure to:
        handle unavailable nodes
        manage cloud storage volumes
        manage load balancing and routing   
```

* Key-value data store
```text
    Summary: etcd is the strongly consistent, distributed key-value store that holds a Kubernetes cluster’s state. Only the API Server talks to it. Data is append-only and old entries are periodically compacted. For resilience, production clusters should replicate it in HA mode.

Key points:

    What it is: an open source CNCF project, used to persist the cluster’s state.
    Append-only: new data is added, never replaced; obsolete data is compacted periodically to keep the store small.
    Access: only the API Server can communicate with etcd.
    etcdctl: the CLI tool, with snapshot save and restore, especially useful for single-instance clusters (dev and learning).
    Production: stage and production environments should replicate etcd in HA mode to protect the cluster’s configuration data.
    kubeadm default: provisions a stacked etcd, running on the same control plane node and sharing resources with the other components.
```
![Stacked etcd Topology](../K8S/K8S_Stackedetcd_Topology.png) 

![External etcd Topology](../K8S/K8S_ETCD_EXTERN.png)

```text
    Summary: etcd supports HA in both stacked and external topologies. It uses the Raft consensus algorithm, so a group of nodes works as one and survives member failures. One node is the leader and the rest are followers. If the leader fails, the healthy followers elect a new one.

Key points:

    HA support: both stacked and external etcd topologies can run in HA mode.
    Raft: lets a group of machines act as a coherent unit that tolerates the failure of some members.
    Roles: one leader at a time, the rest are followers.
    Failure handling: etcd handles leader elections gracefully, including when the leader itself fails.
    Leader vs. primary/secondary: these are different. No node is favored for the leader role, and no node outranks another. Any node can become leader.
    Leader lifecycle: a leader stays active until it fails, then the healthy followers elect a new one.
```

### Worker Node: 

Is where your applications actually run. Apps are microservices in containers, wrapped in Pods, which the control plane schedules onto worker nodes. Workers provide the compute, memory, storage, and networking Pods need, and handle user traffic directly.

- Purpose: provides the running environment for client applications (microservices in containers).
- Pods: containers are encapsulated in Pods, controlled by the control plane agents.
- Pod = smallest scheduling unit: a logical group of one or more containers scheduled together, and started, stopped, or rescheduled as one unit.
- Resources: worker nodes give Pods the compute, memory, and storage they need, plus networking to talk to each other and the outside world.
- Traffic: in multi-worker clusters, traffic between users and apps goes straight to the worker nodes, not through the control plane node.

**Worker Node Components:**

* Container runtime
```text 
Summary: Kubernetes orchestrates containers but can’t run them itself, so every node (control plane and worker) needs a container runtime to manage container lifecycles. The control plane components are recommended to run as containers, which is why control plane nodes need one too.

Key points:
    Why needed: Kubernetes lacks the ability to directly handle and run containers, so it relies on a runtime on each node.
    Where: on every node, both control plane and worker.
    Control plane too: because the recommendation is to run control plane components as containers.

Supported runtimes:
    CRI-O: lightweight, supports quay.io and Docker Hub registries.
    containerd: simple, robust, and portable.
    Docker Engine: popular and complex platform that uses containerd as its runtime.
    Mirantis Container Runtime: formerly Docker Enterprise Edition.
```

* Node agent - kubelet
```text 
Summary: The kubelet is the agent on every node (control plane and workers) that talks to the control plane. It receives Pod definitions, mainly from the API Server, tells the container runtime to run the containers, and monitors the health and resources of those Pods. It connects to runtimes through the CRI(Container Runtime Interface), using a shim as an abstraction layer.

Key points:

    Where it runs: on each node, both control plane and workers.
    Input: gets Pod definitions, primarily from the API Server.
    Action: interacts with the container runtime to run the Pod’s containers.
    Monitoring: tracks the health and resources of running Pods.
    CRI (Container Runtime Interface): the plugin-based interface kubelet uses to talk to runtimes. It includes protocol buffers, a gRPC API, libraries, and extra specs and tools.
    CRI shim: the app that sits between kubelet and the runtime, making runtimes interchangeable.
```
![Container Runtime INterface](../K8S/K8S_CRI.png)

**Note:**
This diagram shows how the kubelet gets containers to run, step by step, from left to right.

Kubelet (blue): the agent on the node. Inside it is a gRPC client, the part that sends requests (“run this container”, “stop that one”).
CRI protobuf (the arrow): the “language” the two sides use to talk. It’s the Container Runtime Interface (CRI), so both sides understand the same messages.
CRI shim (orange): a translator in the middle. It has a gRPC server that receives the kubelet’s requests and converts them into something the specific runtime understands.
Container runtime (green): the software that actually does the work (containerd, CRI-O, etc.).
Containers (white boxes on the right): the runtime creates and manages many containers, which is why one runtime has several arrows pointing to them.

Why the shim matters: the kubelet always speaks the same CRI language, so you can swap one runtime for another without changing the kubelet. Only the shim in the middle changes.

A simple analogy is a restaurant: the kubelet is the waiter (takes the order), the shim is the translator, and the runtime is the kitchen that cooks (runs) the containers.

* kubelet - CRI shims
```text 
Summary: Originally, the kubelet had Docker (then rkt) support hardcoded into its source code. Kubernetes later introduced the CRI, a standard interface, so any CRI-compatible runtime can be used without recompiling Kubernetes. Shims are the adapters that connect each runtime to the CRI. Since v1.24, dockershim was removed, and cri-dockerd replaces it for Docker Engine and MCR.

Key points:

    Before the CRI: the kubelet supported only Docker first, then rkt, through an interface built directly into its source code. That wasn’t meant to last.
    The CRI: a standardized, decoupled approach. Any runtime that implements it can be used without recompiling Kubernetes.
    Shims: CRI adapters, specific to each supported runtime.
    Examples:
        cri-containerd: lets containerd create and manage containers directly at the kubelet’s request.
        CRI-O: lets Kubernetes use any OCI-compatible runtime, such as runC.
        dockershim: the default before v1.24, it managed containers by invoking Docker Engine and its internal containerd.
    dockershim removal: from v1.24 it is no longer maintained and its code was removed from the kubelet.
    cri-dockerd: the replacement adapter, maintained by Docker, Inc. and Mirantis. It keeps Docker Engine (and Mirantis Container Runtime) usable with Kubernetes through the standard CRI method.
```

* Proxy - kube-proxy
```text 
Summary: The kube-proxy is the network agent on every node (control plane and workers). It keeps the node’s networking rules up to date, hides the details of Pod networking, and forwards connection requests to the right containers in Pods. It works together with the node’s iptables.

Key points:

    Where it runs: on each node, both control plane and workers.
    Role: dynamically updates and maintains all networking rules on the node.
    Abstraction: hides Pod networking details and forwards connection requests to the Pods’ containers.
    Traffic types: handles TCP, UDP, and SCTP stream forwarding, or random forwarding across a set of Pod backends for an application.
    Rules source: implements the forwarding rules users define through Service API objects.
    iptables: kube-proxy works with the node’s iptables, a Linux firewall utility managed through the CLI tool of the same name and pre-installed on many Linux distributions.
```

* Add-ons
```text 
Summary: Add-ons are extra cluster features that Kubernetes doesn’t provide natively, so they’re implemented through 3rd-party plugins and services.

Key points:

DNS: a cluster DNS server that assigns DNS records to Kubernetes objects and resources.
Dashboard: a general-purpose web UI for managing the cluster.
Monitoring: collects cluster-level container metrics and saves them to a central data store.
Logging: collects cluster-level container logs and saves them to a central log store for analysis.
Device Plugins: let a node advertise hardware resources (GPU, FPGA, high-performance NIC) to application Pods.
```

### Networking Challenges

Microservices are decoupled, so they depend heavily on networking to work together like the old monolithic apps did. Kubernetes has to solve four distinct networking challenges, using the cluster itself plus its plugins.

**Why it matters:** decoupled microservices rely on networking to mimic the tight coupling of monoliths, and networking is hard to understand and implement.

**The 4 challenges:**
* Container-to-Container: communication inside a Pod.
```text
Network namespace: the container runtime uses the host OS kernel’s virtualization features to create an isolated network space for each container it starts. On Linux, this is called a network namespace.
Sharing: a network namespace can be shared across containers, or with the host OS.
Pause container: when a Pod starts, the runtime first creates a special infrastructure container, the Pause container, whose only purpose is to create the Pod’s network namespace.
Result: all other containers started in the Pod share the Pause container’s network namespace, so they can all talk to each other via localhost.
```

* Pod-to-Pod: on the same node and across different nodes.
```text
Summary: Pods land on nodes unpredictably, yet every Pod must be able to reach every other Pod in the cluster without NAT. Kubernetes achieves this with the “IP-per-Pod” model: each Pod gets its own unique IP, like a VM on a network. Containers plug into this model through the CNI, and CNI plugins (mostly 3rd-party SDN solutions) implement the actual networking.

Key points:

    The challenge: Pods are scheduled on nodes in a nearly unpredictable way, but must communicate with all other Pods regardless of their host node.
    No NAT: this must work without Network Address Translation, a fundamental requirement of any Kubernetes networking implementation.
    IP-per-Pod model: Pods are treated like VMs on a network, each with its own network interface and a unique IP address. This keeps complexity low and lets Pods talk to each other like VMs on the same network.
    Containers inside a Pod: they share the Pod’s network namespace, so they must coordinate port assignments (like apps on a VM) and talk to each other via localhost.
    CNI (Container Network Interface): a set of specifications and libraries that lets plugins configure networking for containers, integrating them into the Kubernetes network model.

    CNI plugins: a few are core plugins, but most are 3rd-party Software Defined Networking (SDN) solutions that implement the Kubernetes networking model.
        some also support Network Policies.
        examples: Flannel, Weave, Calico, Cilium.
Note:
The container runtime offloads the IP assignment to CNI, which connects to the underlying configured plugin, such as Bridge or MACvlan, to get the IP address. Once the IP address is given by the respective plugin, CNI forwards it back to the requested container runtime.

```
* Service-to-Pod: within the same namespace and across namespaces.

* External-to-Service: clients outside the cluster accessing applications inside it.
```text
 Kubernetes enables external accessibility through Services, complex encapsulations of network routing rule definitions stored in iptables on cluster nodes and implemented by kube-proxy agents. By exposing services to the external world with the aid of kube-proxy, applications become accessible from outside the cluster over a virtual IP address and a dedicated port number.
```

# Kubernetes Configuration

## Kubernetes Installation Types

| # | Type | Control Plane | etcd | Workers | Typical Use |
|---|------|---------------|------|---------|-------------|
| 1 | All-in-One Single-Node | 1 (same node as workers) | Same node | Same node | Learning, dev, testing (not for production) |
| 2 | Single-CP + Multi-Worker | 1 | Stacked | Multiple | Small/test clusters |
| 3 | Single-CP + Single-Node etcd + Multi-Worker | 1 | External (1 instance) | Multiple | Small clusters with etcd isolated |
| 4 | Multi-CP + Multi-Worker (HA) | Multiple (HA) | Stacked, HA etcd cluster | Multiple | HA setups |
| 5 | Multi-CP + Multi-Node etcd + Multi-Worker | Multiple (HA) | External, HA etcd cluster | Multiple | Production (most advanced) |

### Key Concepts
- **Stacked etcd:** etcd runs on the same node as the control plane. Simpler, but control plane and etcd fail together.
- **External etcd:** etcd runs on separate nodes. More resilient and complex, with more hardware to manage.
- **HA (High Availability):** multiple control plane nodes remove the single point of failure.

### Quick Summary
1. **All-in-One:** everything on one node. Only for learning and testing.
2. **Single-CP, stacked etcd:** one control plane managing many workers.
3. **Single-CP, external etcd:** like #2, but etcd lives outside the control plane.
4. **Multi-CP, stacked etcd (HA):** several control planes, each with its own etcd, forming an HA etcd cluster.
5. **Multi-CP, external etcd (HA):** several control planes plus a separate HA etcd cluster. Recommended for production.

### LAB Worked:
- https://github.com/kelseyhightower/kubernetes-the-hard-way

## Minikube

**Minikube** is a tool that allows users to easily run a local, all-in-one or multi-node Kubernetes cluster on their workstations (Linux, macOS, or Windows). It uses isolation software (like a Type-2 Hypervisor or Container Runtime) to keep cluster components separate from the host system, leaving no permanent traces when removed.

---

## Key Points

* **Purpose & Cross-Platform:** Runs local Kubernetes clusters on Linux, macOS, and Windows.
* **Isolation & Drivers:** 
  * Requires a Type-2 Hypervisor or Container Runtime to isolate cluster components from the host OS.
  * Isolation software can be manually specified using the `--driver` flag, or Minikube will auto-select a preferred default.
  * Can also be run on bare-metal (risk of permanent host changes) or inside a dedicated Virtual Machine.
* **Underlying Tech:** Built on Docker's `libmachine` library.
* **Cluster Configuration:** 
  * Supports single-node (all-in-one) and multi-node clusters.
  * Users define node counts, CPU, memory, and disk space allocations.
* **Resource Recommendations (per learning node):**
  * **CPU:** Minimum 2 cores / vCPUs.
  * **RAM:** Minimum 2 GB (4–8 GB recommended).
  * **Disk Space:** 20+ GB.
  * *Note:* Performance is ultimately limited by the host workstation's physical resources.
* **Bootstrapping:** After provisioning nodes via the hypervisor/runtime, Minikube uses `kubeadm` to bootstrap the Kubernetes cluster components.
* **Requirements:** Nodes need internet access for software updates, container image downloads, and client accessibility.

## System Requirements for Running Minikube

* **Hardware Virtualization:** VT-x/AMD-v virtualization may need to be enabled in system BIOS/UEFI for certain hypervisors.
* **Kubernetes CLI (`kubectl`):** 
  * Required to access and manage clusters.
  * Can be accessed directly via `minikube kubectl` or installed as a standalone binary.
* **Hypervisors & Container Runtimes (by OS):**
  * **Default Driver Preference (Linux):** Docker $\rightarrow$ KVM2 $\rightarrow$ Podman $\rightarrow$ VMware $\rightarrow$ VirtualBox.
  * **Linux:** Docker, Podman, VirtualBox, KVM2, QEMU.
  * **macOS:** Docker, Podman, VirtualBox, HyperKit, VMware Fusion, Parallels, QEMU.
  * **Windows:** Docker, Podman, VirtualBox, Hyper-V, VMware Workstation, QEMU.
  * **Bare-Metal Option (`--driver=none`):** Linux-only feature that runs Kubernetes directly on the host OS without a VM; requires Docker installed (recommended for advanced users).
* **Internet Connectivity:**
  * **First Run:** Required to download packages, dependencies, updates, and initial cluster images.
  * **Subsequent Runs:** Only required when pulling new external container images or handling client accessibility; existing images are reused from the local runtime cache offline.

# Minikube + Docker on an Ubuntu Azure VM
 
> Session notes reconstructed from the `history` output of an SSH session.
> **Caveat:** `history` only records commands, not their output. Anything described as "likely" or "expected" is inferred from the command order, not from seen output.
 
## Environment
 
| Item | Value |
|---|---|
| Host | `LNXMINIKUBEKAM` (Azure VM) |
| User | `noremak` |
| OS | Ubuntu (release checked with `cat /etc/*release` and `lsb_release -a`) |
| Architecture | `uname -m` (the Minikube binary used is `linux-amd64`, so x86_64) |
| Container runtime | Docker Engine (`docker-ce`) from Docker's official apt repo |
| Kubernetes tooling | Minikube (latest release binary) + `kubectl` (snap) |
| Minikube driver | `docker` |
 
## Goal
 
Get a single-node local Kubernetes cluster (Minikube) running on an Ubuntu VM in Azure, using Docker as the driver.
 
---
 
## Summary of what was done
 
1. **Inspected the system** (OS version, CPU architecture, virtualization flags).
2. **Updated the OS packages** and installed build tools + kernel headers.
3. **Installed Docker Engine** from Docker's official apt repository, enabled the service, and tested with `hello-world`.
4. **Installed Minikube** (binary into `/usr/local/bin`) and **kubectl** (snap).
5. **First cluster start attempts** ran into a permissions problem (the user was not in the `docker` group), so `sudo` was used as a workaround.
6. **Fixed it properly**: added the user to the `docker` group, logged out and back in, verified Docker works without `sudo`.
7. **Started Minikube as the normal user** with `--driver=docker` and verified nodes and pods.
---
 
## Step by step
 
### 1. System inspection
 
```bash
pwd
cat /etc/*release          # distro info
lsb_release -a             # Ubuntu version and codename
uname -m                   # CPU architecture (x86_64 / aarch64)
grep -E -m 1 'vmx|svm' /proc/cpuinfo   # does the CPU expose virtualization extensions?
```
 
**Why:** the architecture decides which binaries to download (`amd64` vs `arm64`). The `vmx|svm` check shows whether hardware virtualization (Intel VT-x / AMD-V) is visible inside the VM. This matters for VM-based Minikube drivers (KVM, VirtualBox), and on Azure it depends on the VM size supporting nested virtualization.
 
**Note:** with the **`docker` driver, hardware virtualization is not required**, since Minikube runs as a container, not a VM.
 
### 2. Update the OS and install build tools
 
```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) gcc make perl
test -e /lib/modules/$(uname -r)/build && echo "Kernel headers available"
```
 
**Why:** `build-essential`, `gcc`, `make`, `perl` and the kernel headers for the running kernel (`uname -r`) are what you need to compile kernel modules (typical for VirtualBox or guest-driver setups). The `test -e ...` line confirms the headers matching the running kernel are present.
 
**Note:** this step is probably **not needed** for Minikube with the Docker driver. It is harmless, but it can be skipped next time.
 

### 3. Install Docker Engine (official repo, deb822 format)
 
```bash
# Prerequisites
sudo apt update
sudo apt install -y ca-certificates curl
 
# Keyring directory + Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
 
# Add the repository
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
 
# Install
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
 
# Start now + enable at boot
sudo systemctl enable --now docker
 
# Smoke test
sudo docker run hello-world
```
 
**What each piece does:**
 
- `/etc/apt/keyrings/docker.asc`: Docker's GPG key, so apt can verify the packages are authentic.
- `docker.sources`: the repo definition in the newer **deb822** format (`.sources` file) instead of the old one-line `.list` format.
- `$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")`: resolves to the Ubuntu codename (for example `jammy`, `noble`) so apt picks the right repo suite.
- `$(dpkg --print-architecture)`: resolves to `amd64`/`arm64`.
- `Signed-By`: ties this repo to that specific key only.
- `enable --now`: starts the service immediately **and** makes it start on boot.
- `hello-world`: pulls a tiny image and prints a message. Confirms the daemon, image pull, and container execution all work.

**Packages installed:**
 
| Package | Purpose |
|---|---|
| `docker-ce` | Docker Engine (daemon) |
| `docker-ce-cli` | `docker` command |
| `containerd.io` | Container runtime used by Docker |
| `docker-buildx-plugin` | `docker buildx` (advanced builds) |
| `docker-compose-plugin` | `docker compose` (v2) |
 
### 4. Install Minikube and kubectl
 
```bash
# Download latest Minikube binary
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
 
# Install into PATH (owner root, mode 0755)
sudo install minikube-linux-amd64 /usr/local/bin/minikube
 
# Clean up the downloaded file
rm minikube-linux-amd64
 
# kubectl via snap
sudo snap install kubectl --classic
```
 
**Notes:**
 
- `install` copies the file **and** sets permissions, so no separate `chmod +x` is needed.
- History #33 repeats `sudo install minikube-linux-amd64 ...` **after** the file was deleted with `rm` (#32). That second install most likely failed with "No such file", but it did not matter because the first install (#31) had already put `minikube` in `/usr/local/bin`. **Lesson:** run `rm` only when you are done, and do not repeat the install.
- `--classic` is required for the kubectl snap because it needs unrestricted filesystem access (for example `~/.kube/config`).
### 5. First start attempts (the permissions problem)
 
```bash
minikube status
minikube start                          # as normal user, before being in the docker group
sudo minikube start --driver=docker     # workaround with sudo
sudo minikube status
sudo minikube kubectl -- get nodes
```
 
**What happened (inferred):** the plain `minikube start` ran as a normal user who could not talk to the Docker socket (`/var/run/docker.sock`), which is why the next step was to retry with `sudo`. Running Minikube with `sudo` works around it, but it is **not a good practice**:
 
- The cluster state is stored under **root's** home (`/root/.minikube`, `/root/.kube`), not under your user's.
- Your normal-user `minikube` and `kubectl` then do not see that cluster.
- Files created by root cause ownership issues later.
`sudo minikube kubectl -- get nodes` shows the bundled kubectl: `minikube kubectl -- <args>` runs a kubectl version matching the cluster, and `--` separates minikube's flags from kubectl's.
 
### 6. Proper fix: add the user to the `docker` group
 
```bash
sudo usermod -aG docker noremak
exit                       # log out so the new group membership applies
# (reconnect via SSH)
id -nG                     # confirm "docker" is in the list
docker version             # works without sudo
docker run --rm hello-world
sudo systemctl status docker --no-pager
```
 
**Why:**
 
- `-aG` = **a**ppend to **G**roup. Without `-a`, `usermod -G` would *replace* all your supplementary groups.
- Group changes **do not apply to the current session**, so you must log out and back in (`exit`, then reconnect). Alternative without logging out: `newgrp docker`.
- `id -nG` lists your groups by name. It should now include `docker`.
- `--rm` removes the container once it exits (no leftover containers).
- `--no-pager` prints the status straight to the terminal instead of opening `less`.
**Security note:** membership in the `docker` group is effectively **root-equivalent** on that machine. Fine for a personal lab VM, think twice on shared or production hosts.
 
### 7. Start Minikube as the normal user and verify
 
```bash
minikube start --driver=docker
minikube status
kubectl get nodes
kubectl get pods -A
```
 
**Expected results (not shown in history):**
 
- `minikube status`: `host: Running`, `kubelet: Running`, `apiserver: Running`, `kubeconfig: Configured`.
- `kubectl get nodes`: one node named `minikube`, status `Ready`, role `control-plane`.
- `kubectl get pods -A`: system pods in `kube-system` (coredns, etcd, kube-apiserver, kube-controller-manager, kube-proxy, kube-scheduler, storage-provisioner), all `Running`. `-A` = all namespaces.
---
 
## Commands cheat sheet
 
```bash
# Docker
sudo systemctl status docker --no-pager
docker ps -a
docker images
 
# Minikube lifecycle
minikube start --driver=docker
minikube status
minikube stop
minikube delete                  # destroys the cluster
minikube delete --all --purge    # wipes all profiles and cached data
 
# Minikube extras
minikube dashboard
minikube addons list
minikube ip
minikube ssh                     # shell inside the minikube node container
 
# Kubernetes
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
kubectl cluster-info
kubectl config current-context   # should say "minikube"
```
 
## Troubleshooting notes
 
| Symptom | Likely cause | Fix |
|---|---|---|
| `permission denied ... /var/run/docker.sock` | User not in `docker` group, or session not refreshed | `sudo usermod -aG docker $USER`, then log out and back in (or `newgrp docker`) |
| `minikube start` fails mentioning root | Running with `sudo` and the docker driver | Run as a normal user after fixing group membership |
| `kubectl` shows a different or no cluster | The cluster was created with `sudo` (root's kubeconfig) | Start Minikube as the normal user; delete the root one (see below) |
| `Invalid operation udpate` | Typo | `sudo apt update` |
| `install: cannot stat 'minikube-linux-amd64'` | File already removed with `rm` | Not an issue if the binary is already in `/usr/local/bin` |
 
### Possible leftover to clean up
 
Because Minikube was started with `sudo` earlier (#37), a **root-owned cluster** may still exist alongside the one created later by the normal user. To check and remove it:
 
```bash
sudo minikube status
sudo minikube delete        # removes the root-owned cluster
minikube status             # confirm the user-owned cluster is still fine
```
 
## Clean reproducible version (what to run next time)
 
```bash
# 1. Update OS
sudo apt update && sudo apt upgrade -y
 
# 2. Docker (official repo)
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
 
# 3. Run Docker without sudo (do this BEFORE starting Minikube)
sudo usermod -aG docker $USER
newgrp docker            # or log out and back in
 
# 4. Minikube + kubectl
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
sudo snap install kubectl --classic
 
# 5. Start and verify (as normal user, never with sudo)
minikube start --driver=docker
minikube status
kubectl get nodes
kubectl get pods -A
```
 
**Key differences vs. the original session:** skipped the unnecessary build tools/kernel headers, no typo, no duplicate install, and the `docker` group fix is done **before** the first `minikube start`, so `sudo minikube` is never needed.
 
## Key takeaways
 
- With the **Docker driver**, Minikube does not need hardware virtualization, kernel headers, or build tools.
- **Never run `minikube` with `sudo`** when using the docker driver. Fix the `docker` group membership instead.
- Group changes need a **new login session** to take effect.
- Docker's official apt repo (deb822 `.sources` file + GPG key in `/etc/apt/keyrings`) is the recommended install method over Ubuntu's `docker.io` package.
- `kubectl` and `minikube` read cluster config from the **current user's** home, so root and normal user see different clusters.
 

## Lab: Minikube multi-profile practice (Docker driver) on Ubuntu 22.04 / Azure VM

**Host:** `LNXMINIKUBEKAM` | **User:** `noremak` | **Minikube:** v1.39.0 | **Date:** 2026-10-15

### Goal

Verify the Minikube + Docker install, then practice running **two clusters side by side** (profiles), switching between them, stopping/starting them, and cleaning up.

### Environment checked

```bash
nproc          # 4 CPUs
free -h        # 15 GiB RAM total, ~14 GiB available
df -h /        # 29 GB disk, 23 GB free
sudo systemctl status docker --no-pager   # active (running), enabled, socket-activated
```

**Why:** each Minikube node is a Docker container that reserves CPU and RAM. Checking VM capacity *before* creating a second cluster avoids out-of-memory failures.

---

## Timeline of actions

### 1. Default cluster (`minikube` profile)

```bash
minikube start --driver=docker
minikube status
kubectl get nodes
kubectl get pods -A
```

| Item | Value |
|---|---|
| Kubernetes | v1.37.0 (this Minikube's default) |
| Runtime inside node | **containerd 2.3.4** |
| Resources (defaults) | 2 CPUs, 3900 MB RAM |
| CNI | kindnet |
| Node IP | 192.168.49.2 |
| Addons | storage-provisioner, default-storageclass |
| Downloaded | k8s preload (~348 MiB) + kicbase image (~508 MiB) |

All control-plane pods were `Running` in `kube-system`: coredns, etcd, kindnet, kube-apiserver, kube-controller-manager, kube-proxy, kube-scheduler, storage-provisioner.

**Key concept:** `--driver=docker` means the *node* runs as a Docker container. The *Kubernetes runtime inside that node* is a separate choice, and here it is containerd, not Docker.

### 2. Second cluster attempt, failed (unsupported version)

```bash
minikube start --driver=docker --cpus=4 --memory=8g \
  --kubernetes-version="1.27.12" -p largedock
```

```
Exiting due to K8S_OLD_UNSUPPORTED: Kubernetes 1.27.12 is not supported by this release of minikube
```

**Why it failed:** each Minikube release supports a *range* of Kubernetes versions. For v1.39.0 the floor is **v1.28.0**. It exited at validation, so nothing was created. `--force` would bypass the check, but the combination is untested.

**Fix, step 1: list supported versions**

```bash
minikube config defaults kubernetes-version   # range v1.28.0 ... v1.37.0 (plus rc/beta/alpha)
```

### 3. Second cluster (`largedock` profile), success

```bash
minikube start --driver=docker --cpus=4 --memory=8g \
  --kubernetes-version=v1.28.15 -p largedock
```

| Item | Value |
|---|---|
| Profile | `largedock` |
| Kubernetes | v1.28.15 (oldest supported line) |
| Resources | 4 CPUs, 8192 MB RAM |
| Node IP | 192.168.58.2 (separate Docker network from the first cluster) |

**Why `-p`:** a *profile* is an independent cluster with its own container, IP, config, and kubectl context. Without `-p` every command targets the profile named `minikube`.

### 4. Inspecting and switching profiles

```bash
minikube profile list          # table of all profiles, status, version, active profile/context
minikube profile minikube      # set the default active profile
minikube node list             # nodes of the active profile
minikube node list -p largedock
minikube ip                    # IP of the active profile
minikube -p largedock ip       # IP of a specific profile
```

Two separate "active" markers exist in `profile list`:

- **ACTIVE PROFILE:** which profile Minikube commands target by default (changed with `minikube profile <name>`).
- **ACTIVE KUBECONTEXT:** which cluster `kubectl` talks to. `minikube start` switches this to the cluster just started, which is why the two markers pointed at different profiles for a while.

### 5. Stop / start lifecycle

```bash
minikube stop -p largedock      # powers off the node container
minikube start -p largedock     # restarts it
minikube stop                   # default profile
minikube start
```

**Observed behavior on restart:**

- "Using the docker driver based on **existing profile**" and "Restarting existing docker container". No new cluster is created.
- Kubernetes version, CPUs, and memory are **remembered**, so the original flags were not needed again.
- Minikube printed a hint that Kubernetes 1.37.0 is available for `largedock`. Upgrading in place would need `--kubernetes-version=v1.37.0`.

### 6. Bash completion (partially done)

```bash
sudo apt install bash-completion   # already installed (1:2.11-5ubuntu1)
source /etc/bash_completion
minikube completion bash           # only PRINTED the script, did not enable it
```

**Important:** `minikube completion bash` just writes the script to the terminal. To actually enable tab-completion:

```bash
# current shell only
source <(minikube completion bash)

# persistent
echo 'source <(minikube completion bash)' >> ~/.bashrc

# same idea for kubectl
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

### 7. Cleanup

```bash
minikube delete                 # deletes the "minikube" profile
minikube delete -p largedock    # deletes the "largedock" profile
```

Each delete removed the node container and `/home/noremak/.minikube/machines/<profile>`. **Both clusters are gone; the VM is back to Docker + Minikube + kubectl only.**

Downloaded images and caches (kicbase, k8s preloads) should remain under `~/.minikube/cache` and in Docker, so recreating a cluster is faster. For a full wipe: `minikube delete --all --purge`.

---

## Mistakes made (and the right command)

| What I typed | What happened | Correct |
|---|---|---|
| `--kubernetes-version="1.27.12"` | Version below supported floor | Pick from `minikube config defaults kubernetes-version` |
| `minikube profile minibox` | Profile not found | Check names with `minikube profile list` |
| `minikube stop -p minkube` | Typo, profile not found | `minikube stop -p minikube` (or just `minikube stop`) |
| `sudo systemctl minikube` | "Unknown command verb": minikube is not a systemd service | `minikube status` / `minikube stop` / `minikube start` |
| `largedock ip` | `command not found`: a profile name is not a command | `minikube -p largedock ip` |

---

## Key takeaways

1. **Profiles (`-p <name>`)** give you multiple isolated clusters on one VM. Always pass `-p` for non-default ones.
2. **Resource planning:** two clusters here reserved about 12 GB of the VM's 15 GiB RAM, and `--cpus=4` equals all of the VM's CPUs. Stop one cluster when not using it.
3. **Version support is bound to the Minikube release.** Use `minikube config defaults kubernetes-version` before pinning a version.
4. **kubectl version skew:** the snap kubectl is **v1.35.9**, which is outside the supported ±1 minor range for both v1.37.0 and v1.28.15 (Minikube warned about it). Use the matching client with `minikube kubectl -- get pods -A`, or `minikube -p largedock kubectl -- ...`.
5. **`minikube stop` keeps the cluster; `minikube delete` destroys it.**
6. **Driver vs runtime:** driver = `docker` (how the node is hosted); runtime = `containerd` (what runs pods inside the node).

## Open items / to verify

- During both starts Minikube printed **"Using Docker driver with root privileges"** even though the prompt is `noremak`. Cluster state lived under `/home/noremak/.minikube` (confirmed by the delete output), so it looks user-owned, but worth checking with `id` and `ls -la ~/.minikube`.
- `apt` reported **74 packages not upgraded**, so only `sudo apt update` (refresh the index) was ever run, not `sudo apt upgrade`. Run `sudo apt upgrade -y` if a full OS update is wanted.
- Enable persistent shell completion (section 6).
- Optional: re-create a cluster and practice deploying a workload (`kubectl create deployment`, `kubectl expose`, `minikube service`).


# Accessing Minikube
 
Any healthy running Kubernetes cluster (Minikube included) can be accessed in three ways. These methods apply to **all** Kubernetes clusters.
 
| Method | Tool | Best for |
|---|---|---|
| CLI | [`kubectl`](https://kubernetes.io/docs/reference/kubectl/overview/) | Flexible, scriptable, automation |
| Web UI | [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) | Users less comfortable with the CLI |
| API | Direct HTTP calls to the API Server | Programmatic access, integrations |
 
## Command Line Interface (CLI)
 
- `kubectl` is the Kubernetes CLI client used to manage cluster resources and applications.
- It integrates easily with other systems: use it standalone or inside scripts and automation tools.
- Once credentials and cluster access points are configured, it can be used remotely from anywhere.
- It is the main tool for deploying applications and managing/configuring resources.
## Web-based User Interface (Web UI)
 
- The Kubernetes Dashboard lets you manage resources and containerized applications from a browser.
- It is less flexible than `kubectl`, but preferred by users who are less proficient with the CLI.
## APIs
 
- The **API Server** is the main component of the control plane and exposes the Kubernetes APIs.
- The CLI and the Dashboard both talk to the API Server (running on the control plane node) to modify cluster state.
- Agents and users with the required credentials can reach it through its endpoints, either directly or via the CLI/Dashboard.
### HTTP API directory tree
 
```text
/
├── /healthz
├── /metrics
├── /api
│   └── /api/v1
│       ├── /api/v1/pods
│       ├── /api/v1/nodes
│       ├── /api/v1/services
│       └── ...
├── /apis
│   ├── /apis/apps
│   │   ├── /apis/apps/v1
│   │   │   ├── /apis/apps/v1/Deployment
│   │   │   ├── /apis/apps/v1/DaemonSet
│   │   │   └── /apis/apps/v1/StatefulSet
│   │   └── /apis/apps/v1beta1
│   └── /apis/...
└── ...
```
 
The tree splits into three independent group types:
 
### 1. Core group (`/api/v1`)
 
Holds the fundamental objects: Pods, Services, Nodes, Namespaces, ConfigMaps, Secrets, etc.
 
### 2. Named group (`/apis/$NAME/$VERSION`)
 
The API version signals the level of stability and support:
 
| Level | Meaning | Example |
|---|---|---|
| Alpha | May be dropped at any time, without notice | `/apis/batch/v2alpha1` |
| Beta | Well tested, but object semantics may change incompatibly in later beta/stable releases | `/apis/certificates.k8s.io/v1beta1` |
| Stable | Appears in released software for many subsequent versions | `/apis/networking.k8s.io/v1` |
 
### 3. System-wide
 
Cluster-wide endpoints such as `/healthz`, `/logs`, `/metrics`, `/ui`, etc.
 
## Key takeaway

You can reach the API Server by calling its endpoints directly, through the CLI (`kubectl`), or through the Dashboard UI.


 

