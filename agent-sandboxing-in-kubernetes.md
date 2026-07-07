# Agent Sandboxing Layers in Kubernetes

## NVIDIA OpenShell

OpenShell is a safe, private runtime for autonomous AI agents. It wraps each agent session in a sandboxed container governed by declarative YAML policies that control filesystem access, network egress, process capabilities, and inference routing. A lightweight gateway acts as the control plane, managing sandbox lifecycle, while a policy engine intercepts every outbound connection -- allowing, denying, or rerouting it through a privacy-aware inference router. Policies for filesystem and process constraints are locked at creation; network and inference policies can be hot-reloaded at runtime. OpenShell supports Docker, Podman, MicroVM, and Kubernetes as compute backends.

## Agent-Sandbox (Kubernetes SIG Apps)

Agent-Sandbox is a Kubernetes-native project under SIG Apps that introduces a `Sandbox` Custom Resource Definition (CRD) and controller. It provides a declarative API for managing isolated, stateful, singleton pods with stable identity and persistent storage -- a lightweight single-container VM experience built on Kubernetes primitives. The controller handles pod lifecycle including creation, scheduled deletion, pausing, and hibernation. Extension CRDs (`SandboxTemplate`, `SandboxClaim`, `SandboxWarmPool`) enable templated creation and pre-warmed pools for fast allocation. Agent-Sandbox is runtime-agnostic and explicitly supports plugging in stronger isolation backends like gVisor or Kata Containers.

## Kata Containers

Kata Containers is an open-source container runtime that provides VM-level isolation while maintaining the developer experience and performance profile of standard containers. Each container (or pod) runs inside its own lightweight virtual machine with a dedicated kernel, using hardware virtualization extensions (Intel VT-x, AMD-V) as a second layer of defense. It is OCI-compatible and integrates with the Kubernetes CRI interface via containerd, supporting hypervisors including QEMU, Cloud-Hypervisor, and Firecracker. Kata provides isolation of network, I/O, and memory without requiring nested VMs.

---

## How Each Layer Sandboxes an Agent

![Sandbox Layers Overview](sandbox-layers-overview.png)

<details>
<summary>Mermaid source</summary>

```mermaid
graph TB
    User{User}

    subgraph Kubelet["Kubelet"]
        subgraph KataMicroVM["Kata MicroVM"]
            subgraph Sandbox["Sandbox"]
                Agent[Agent]
                Executor[OpenShell Executor]
                Executor -->|Executes| Agent
                Agent <-->|I/O| Executor
            end
        end
    end

    subgraph Infra["Cluster Services"]
        Router[Sandbox Router]
        GIE["GIE\n(Kubernetes API\nGateway Extension)"]
        vLLM1[vLLM / llm-d / etc.]
        vLLM2[vLLM / llm-d / etc.]
        vLLM3[vLLM / llm-d / etc.]
    end

    User <--> Router
    User <--> GIE
    Router <--> Executor
    Router <--> GIE
    GIE <--> vLLM1

    style User fill:#e94560,color:#fff,stroke:#e94560
    style Kubelet fill:#1a1a2e,color:#fff
    style KataMicroVM fill:#16213e,color:#fff
    style Sandbox fill:#2f9e44,color:#fff,stroke:#2f9e44,stroke-dasharray:5 5
    style Agent fill:#0f3460,color:#fff
    style Executor fill:#0f3460,color:#fff
    style Infra fill:#1a1a2e,color:#fff
    style Router fill:#16213e,color:#fff
    style GIE fill:#16213e,color:#fff
    style vLLM1 fill:#533483,color:#fff
    style vLLM2 fill:#533483,color:#fff
    style vLLM3 fill:#533483,color:#fff
```

</details>

### Layered Defense Model

![Layered defense model](layered-defense.png)

<details>
<summary>Mermaid source</summary>

```mermaid
graph LR
    A[AI Agent] -->|1| B[OpenShell Policy<br/>Filesystem · Network · Process · Inference]
    B -->|2| C[Agent-Sandbox CRD<br/>No defense — orchestration only]
    C -->|3| D[Kata Containers<br/>Dedicated VM · Guest Kernel · HW Isolation]
    D -->|4| E[Kubernetes Node<br/>Host Kernel]

    style A fill:#e94560,color:#fff
    style B fill:#0f3460,color:#fff
    style C fill:#16213e,color:#fff,stroke:#f5a623,stroke-width:2px,stroke-dasharray:5 5
    style D fill:#533483,color:#fff
    style E fill:#1a1a2e,color:#fff
```

</details>

| Layer | Scope | Defense? | Mechanism |
|---|---|---|---|
| **OpenShell** | Application | Yes | YAML policies enforcing L7 network rules, filesystem ACLs, process restrictions, and inference routing |
| **Agent-Sandbox** | Orchestration | **No** | Provides no isolation or security enforcement of its own. It manages pod lifecycle and delegates all defense to the runtime (Kata/gVisor) and policy layer (OpenShell) |
| **Kata Containers** | Runtime | Yes | Hardware-virtualized micro-VM with dedicated kernel, providing memory, I/O, and network isolation via VT-x/AMD-V |

---

## Beyond Defense: Other Major Features

### OpenShell

OpenShell is primarily a defense layer. Its core value proposition -- policy-enforced filesystem, network, process, and inference controls -- is entirely security-focused. The privacy router (stripping caller credentials, injecting backend credentials for inference) is also a security feature. There are no major non-defense features to call out separately.

### Agent-Sandbox

While Agent-Sandbox provides no security isolation on its own, it is the orchestration backbone for agent workloads:

- **Warm Pools (`SandboxWarmPool`)** -- Pre-provisions a pool of ready-to-go sandboxes so that new agent sessions can be claimed instantly instead of waiting for pod scheduling, image pull, and container startup. Critical for latency-sensitive agent deployments at scale.
- **Hibernation & Pause** -- Sandboxes can be suspended to persistent storage and resumed later, freeing cluster resources without losing agent state.
- **Stable Identity & Persistent Storage** -- Each sandbox gets a stable hostname and optional PVC-backed storage that survives restarts, giving agents a durable workspace.
- **Templated Creation (`SandboxTemplate`, `SandboxClaim`)** -- Standardizes sandbox configuration across teams and enables self-service provisioning through claims against warm pools.
- **Lifecycle Management** -- Scheduled deletion, automatic cleanup, and pause/resume give operators control over sandbox sprawl.

### Kata Containers

Beyond VM-level isolation, Kata provides capabilities that standard container runtimes cannot:

- **Peer Pods** -- Via the Cloud API Adaptor, Kata can spawn its micro-VMs as actual cloud provider instances (AWS, Azure, GCP, IBM Cloud) rather than local VMs on the worker node. This eliminates the need for bare-metal nodes or nested virtualization, and enables confidential computing (SEV, TDX) on cloud VMs.
- **VFIO Device Passthrough** -- Because Kata runs a real VM, it can pass through PCI devices (GPUs, FPGAs, SmartNICs) directly to the guest via VFIO, giving the agent near-native hardware access. Standard containers share the host kernel and cannot achieve the same level of direct device assignment.
- **Multiple Hypervisor Backends** -- Choose between QEMU (full-featured), Cloud-Hypervisor (optimized for cloud workloads), or Firecracker (minimal, fast boot) depending on the tradeoff between feature set and startup latency.
- **Confidential Containers** -- Integration with the confidential-containers project enables hardware-attested TEEs (Trusted Execution Environments), ensuring that neither the cloud provider nor the host operator can inspect the agent's memory or code.
- **VSOCKs** -- Direct VM-to-host communication via virtio-vsock, eliminating per-pod proxy overhead and saving ~4.5 MB of RAM per pod in high-density deployments.
