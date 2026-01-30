# Deep Dive: Kubernetes Pod Network Lifecycle

*Understanding the Order of Operations in Pod Network Setup and Teardown*

## Introduction

Understanding how Kubernetes sets up and tears down Pod networking is crucial for platform engineers debugging network issues, developers building container platforms, and anyone implementing custom CRI runtimes or CNI plugins. This post provides a detailed walkthrough of the exact order of operations that occur when a Pod is scheduled to a node and when it's deleted, tracing the flow from the Kubelet through the Container Runtime Interface (CRI) to containerd.

We'll examine the specific function calls, the sequence in which they execute, and what happens at each step. By the end of this post, you'll have a clear mental model of the Pod network lifecycle and know exactly where to look in the source code when troubleshooting network-related issues.

## Architecture Overview

Before diving into the detailed flows, let's understand the key components involved:

**Kubelet** - The node agent that manages Pod lifecycle

**Container Runtime Interface (CRI)** - The gRPC API that kubelet uses to communicate with container runtimes

**containerd** - A container runtime that implements the CRI specification

**CNI (Container Network Interface)** - The plugin interface for configuring network interfaces in containers

## Pod Network Setup Flow

When a Pod is scheduled to a node, the kubelet orchestrates a precise sequence of operations to establish the Pod's network connectivity. Let's walk through each step in order.

### Step 1: SyncPod - Kubelet Initiates Pod Synchronization

The kubelet's `SyncPod` function is the entry point for setting up a Pod. This function coordinates all aspects of bringing a Pod to its desired state.

```go
// kubernetes/pkg/kubelet/kuberuntime/kuberuntime_manager.go
func (m *kubeGenericRuntimeManager) SyncPod(...) error
```

**Source:** [kuberuntime_manager.go#L1394](https://github.com/kubernetes/kubernetes/blob/c8e45a33316e43336dea04c84b7602be04f2ee73/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L1394)

### Step 2: PrepareResource - Dynamic Resource Allocation

Before creating the Pod sandbox, kubelet prepares any dynamic resources that the Pod requires through the Dynamic Resource Allocation (DRA) API.

```go
// Called within SyncPod
PrepareResource(pod, container)
```

**Source:** [kuberuntime_manager.go#L1531](https://github.com/kubernetes/kubernetes/blob/c8e45a33316e43336dea04c84b7602be04f2ee73/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L1531)

### Step 3: RunPodSandbox (CRI) - Create the Pod Sandbox

The kubelet now makes its first CRI RPC call to `RunPodSandbox`. This is where control transitions from kubelet to the container runtime (containerd).

```go
// Kubelet calls the CRI
runtimeService.RunPodSandbox(config, runtimeHandler)
```

**Source:** [kuberuntime_manager.go#L1543](https://github.com/kubernetes/kubernetes/blob/c8e45a33316e43336dea04c84b7602be04f2ee73/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L1543)

#### What happens in containerd's RunPodSandbox:

#### Step 3a: Create Network Namespace

containerd first creates a new network namespace for the Pod. This isolated network environment will host the Pod's network interfaces.

```go
// containerd/internal/cri/server/sandbox_run.go
netNS, err := netns.NewNetNS(id)
```

**Source:** [sandbox_run.go#L207](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_run.go#L207)

#### Step 3b: Setup Pod Network

containerd calls `setupPodNetwork` to configure networking for the Pod. This function orchestrates the network setup steps including bringing up the loopback interface and invoking the CNI plugin.

**Source:** [sandbox_run.go#L265](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_run.go#L265)

#### Step 3c: Bring Up Loopback Interface

Within `setupPodNetwork`, containerd first brings up the loopback interface inside the network namespace. This ensures that localhost (127.0.0.1) is available for inter-process communication within the Pod.

**Source:** [sandbox_run.go#L429](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_run.go#L429)

#### Step 3d: Invoke CNI Plugin (CNI ADD)

This is the critical step where the CNI plugin is invoked to configure the Pod's network. containerd uses the go-cni library, which wraps libcni which is responsible for executing CNI plugin binaries. The CNI ADD command:

- Creates virtual network interfaces (typically a veth pair)
- Assigns IP addresses to the Pod
- Sets up routing rules
- Configures any network policies or additional networking features

```go
// Invoke CNI plugin
result, err := c.cni.Setup(ctx, id, netNSPath, options...)
```

**Source:** [sandbox_run.go#L442](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_run.go#L442)

#### Step 3e: Extract IP Addresses from CNI Result

After the CNI plugin returns, containerd extracts the Pod's IP addresses from the CNI result. It looks for the interface named `eth0` (a hardcoded default in containerd) and retrieves the associated IP addresses.

**Source:** [sandbox_run.go#L457](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_run.go#L457)

### Step 4: PodSandboxStatus (CRI) - Verify Network Setup

After the sandbox is created, kubelet queries the status to verify that networking was set up correctly and to retrieve the Pod's IP addresses.

```go
// Kubelet verifies the sandbox
status, err := runtimeService.PodSandboxStatus(podSandboxID)
```

**Source:** [kuberuntime_manager.go#L1568](https://github.com/kubernetes/kubernetes/blob/c8e45a33316e43336dea04c84b7602be04f2ee73/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L1568)

#### What happens in containerd's PodSandboxStatus:

#### Step 4a: GetIPs - Retrieve Pod IP Addresses

containerd retrieves the Pod's IP addresses from the `IP` and `AdditionalIPs` fields stored on the sandbox object (which were extracted from the CNI result during sandbox creation).

**Source:** [sandbox_status.go#L39](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_status.go#L39)

### Network Ready Condition (Parallel Process)

Separately from Pod creation, the kubelet calls the CRI Status RPC every 5 seconds to check the runtime's health.

**Source:** [kubelet.go#L1871](https://github.com/kubernetes/kubernetes/blob/702e2a3800369a4d7cb1ff04eadb95da38b88aeb/pkg/kubelet/kubelet.go#L1871)

This executes the `updateRuntimeUp` function which calls the CRI Status RPC:

**Source:** [kubelet.go#L3120](https://github.com/kubernetes/kubernetes/blob/702e2a3800369a4d7cb1ff04eadb95da38b88aeb/pkg/kubelet/kubelet.go#L3120)

The response includes the network ready condition. If the network ready condition is false (e.g., CNI plugin not configured), the node transitions to 'not ready' and Pods cannot be scheduled on that node until the condition becomes true.

**Source:** [kubelet.go#L3135](https://github.com/kubernetes/kubernetes/blob/702e2a3800369a4d7cb1ff04eadb95da38b88aeb/pkg/kubelet/kubelet.go#L3135)

containerd sets the network ready condition based on whether a valid CNI configuration is loaded:

**Source:** [status.go#L51](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/status.go#L51)

### Pod Network Setup Summary

To summarize the complete setup flow:

1. kubelet's SyncPod initiates Pod creation
2. PrepareResource allocates any dynamic resources
3. RunPodSandbox CRI call triggers containerd to:
   - a. Create the network namespace
   - b. Call setupPodNetwork which:
      - Brings up the loopback interface
      - Invokes CNI ADD to configure networking
   - c. Extract IP addresses from the CNI result
4. PodSandboxStatus CRI call retrieves Pod IP addresses

Note: The network ready condition is checked separately via the Status RPC every 5 seconds to determine node readiness.

---

## Pod Network Teardown Flow

When a Pod is deleted, the cleanup process follows a careful sequence to properly tear down all network resources. Let's examine each step.

### Step 1: KillPod - Kubelet Initiates Pod Termination

The kubelet's `KillPod` function is the entry point for tearing down a Pod. This coordinates the orderly shutdown of all Pod components.

```go
// kubernetes/pkg/kubelet/kuberuntime/kuberuntime_manager.go
func (m *kubeGenericRuntimeManager) KillPod(...) error
```

**Source:** [kuberuntime_manager.go#L1860](https://github.com/kubernetes/kubernetes/blob/c8e45a33316e43336dea04c84b7602be04f2ee73/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L1860)

### Step 2: Kill Containers - Terminate Application Containers

Before stopping the Pod sandbox, kubelet first stops all application containers running in the Pod. This ensures containers are gracefully terminated before their network is torn down.

```go
// Stop each container in the Pod
killContainersWithSyncResult(pod, runningPod, gracePeriod)
```

**Source:** [kuberuntime_manager.go#L1869](https://github.com/kubernetes/kubernetes/blob/c8e45a33316e43336dea04c84b7602be04f2ee73/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L1869)

### Step 3: StopPodSandbox (CRI) - Tear Down the Pod Sandbox

After all containers are stopped, kubelet makes the `StopPodSandbox` CRI call to clean up the Pod's network and stop the pause container.

```go
// Kubelet calls the CRI
err := runtimeService.StopPodSandbox(podSandboxID)
```

**Source:** [kuberuntime_manager.go#L1879](https://github.com/kubernetes/kubernetes/blob/c8e45a33316e43336dea04c84b7602be04f2ee73/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L1879)

#### What happens in containerd's StopPodSandbox:

#### Step 3a: Teardown Pod Network

containerd calls `teardownPodNetwork` to remove the network from the Pod. This function invokes CNI DEL to clean up the network configuration.

**Source:** [sandbox_stop.go#L118](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_stop.go#L118)

#### Step 3b: Invoke CNI Plugin (CNI DEL)

Within `teardownPodNetwork`, containerd uses the go-cni library, which wraps libcni which is responsible for executing CNI plugin binaries. The CNI DEL command:

- Removes IP address assignments
- Deletes virtual network interfaces
- Cleans up routing rules
- Removes any network policies or additional configurations

```go
// Tear down the network
err := c.cni.Remove(ctx, id, netNSPath, options...)
```

**Source:** [sandbox_stop.go#L170](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_stop.go#L170)

#### Step 3c: Delete Network Namespace

Finally, after the CNI plugin has cleaned up all network resources, containerd deletes the network namespace itself.

```go
// Clean up the network namespace
err := netns.Remove(netNSPath)
```

**Source:** [sandbox_stop.go#L122](https://github.com/containerd/containerd/blob/317286ac00e07bebd7d77925c0fef2af0d620e40/internal/cri/server/sandbox_stop.go#L122)

### Pod Network Teardown Summary

To summarize the complete teardown flow:

1. kubelet's KillPod initiates Pod deletion
2. Kill Containers stops all application containers
3. StopPodSandbox CRI call triggers containerd to:
   - a. Call teardownPodNetwork which invokes CNI DEL to tear down networking
   - b. Delete the network namespace

---

## Why the Order Matters

The sequence of operations in both setup and teardown is carefully designed to ensure proper resource management and avoid race conditions.

### Setup Order Rationale

**Network namespace must be created first** - The CNI plugin needs a target namespace to configure. Creating it first ensures the namespace exists before any network configuration attempts.

**CNI ADD configures the network** - Once the namespace, the CNI plugin can safely create interfaces, assign IPs, and set up routing.

**Status verification confirms readiness** - Checking the sandbox status after creation ensures the network is properly configured before kubelet starts application containers.

### Teardown Order Rationale

**Application containers stop first** - Stopping application containers before network teardown prevents connection errors and allows graceful shutdown with network connectivity still available.

**CNI DEL before namespace deletion** - The CNI plugin needs the namespace to exist in order to remove interfaces and clean up routes. Deleting the namespace first would make cleanup impossible.

**Namespace deleted last** - The namespace is removed only after all network resources within it have been cleaned up, ensuring complete cleanup without orphaned resources.

## Conclusion

The Pod network lifecycle follows a precise, ordered sequence of operations that ensures proper setup and cleanup of network resources. From the kubelet's initial SyncPod call through the CRI interface to containerd's invocation of CNI plugins, each step builds on the previous one.

For setup, the order ensures that namespaces exist before configuration, and that configuration completes before verification. For teardown, the order ensures that applications stop before their network disappears, that network resources are cleaned up while the namespace still exists, and that the namespace is deleted only after it's empty.

This knowledge serves as a foundation for debugging network issues, implementing custom CRI runtimes or CNI plugins, and understanding the inner workings of Kubernetes networking. The source code links provided throughout this post offer starting points for deeper exploration of each component.

Whether you're troubleshooting a production issue, developing platform tooling, or simply curious about how Kubernetes works under the hood, understanding the order of operations in the Pod network lifecycle is an essential part of mastering Kubernetes.

## References

**Kubernetes Source Code:**
- [github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

**containerd Source Code:**
- [github.com/containerd/containerd](https://github.com/containerd/containerd)

**Container Runtime Interface (CRI):**
- [github.com/kubernetes/cri-api](https://github.com/kubernetes/cri-api)

**Container Network Interface (CNI):**
- [github.com/containernetworking/cni](https://github.com/containernetworking/cni)
