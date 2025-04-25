
# How Istio ztunnel Accesses Pod Network Namespaces Without Mounting `/run/netns`

## The Mystery

When exploring **Istio's ztunnel**, I noticed something odd, the pod **didn’t** mount common paths like `/proc` or `/run/netns` and it was able to access other pod network namespaces. Usually, containerized environments mount these paths to access network namespaces. So how was ztunnel doing it? I want to dive into specifically on how ztunnel is able to create sockets in the pod workload network namespaces. The redirects and how the iptables rules are created are out of scope for this blog. 

---

## Digging Deeper: Unix Domain Sockets and SCM_RIGHTS

While investigating, I found something interesting: **UnixRights** — Go's abstraction over the **SCM_RIGHTS** mechanism — being used over a Unix domain socket.

In Linux, SCM_RIGHTS is the underlying kernel-level mechanism that allows one process to transfer a file descriptor to another over a Unix domain socket using a socket control message. In Go, the syscall.UnixRights() function provides a convenient way to construct the necessary socket control message for using SCM_RIGHTS. Thus, while Go developers typically call UnixRights, it ultimately leverages the Linux SCM_RIGHTS system capability.

> **SCM_RIGHTS** lets a process send a *file descriptor* (FD) to another process over a Unix domain socket.

This is extremely powerful because:
- A file descriptor can point to *anything*: a file, a socket, or even a network namespace.
- The receiving process can then use that FD as if it had opened it itself — *without* needing filesystem access.

In ztunnel’s case:
- A privileged component (istio-cni-node) opens the pod’s network namespace.
- It sends the network namespace FD to ztunnel using SCM_RIGHTS.
- ztunnel receives the FD over a Unix domain socket and calls `setns(fd, CLONE_NEWNET)` to enter that pod's network namespace.

No `/run/netns`, no `/proc`, no special mounts. Just a clean FD handoff.

Interestingly, SCM_RIGHTS is not unique to ztunnel. For example, **Envoy** uses SCM_RIGHTS during its **hot restart** process to transfer open listening sockets from an old Envoy process to a new one — allowing zero downtime restarts without dropping client connections. SCM_RIGHTS is also used in **runc** the `OCI container runtime` however I won't go into additional detail here. 

This works because in Linux, **file descriptors** are **process-scoped references** to **file descriptions**, which live in the kernel.
The **file description** contains the metadata about the open resource (like a socket or a file) and points to an **inode** when appropriate.
When you send an FD using SCM_RIGHTS, the receiving process gets a **new FD** that points to the **same file description** — and therefore, the same underlying resource.

Thus, SCM_RIGHTS allows safe, efficient sharing of open resources across cooperating processes without re-opening or duplicating the underlying file or socket.

### Visual Diagram

```text
Process A
  ↳ File Descriptor (e.g., 3)
    ↳ File Description
      ↳ Inode

[SCM_RIGHTS transfer]

Process B
  ↳ New File Descriptor (e.g., 5)
    ↳ Same File Description
      ↳ Same Inode
```

---

## How Istio-cni-node Hands Off the Pod Network Namespace

During pod creation, the `istio-cni-node` DaemonSet plays a crucial role.

Here's what happens:
- The container runtime such as `containerd` triggers the CNI plugins with CNI ADD command.
- The `istio-cni` meta-plugin — inserted into the CNI chain — is executed.
- It sends a message containing the network namespace path (e.g., `/proc/<pid>/ns/net`) to the local node's `/var/run/istio-cni/pluginevent.sock`.
- `istio-cni-node` receives this message, opens the network namespace file descriptor based on the path, and sends the FD to ztunnel using SCM_RIGHTS.

Now ztunnel has access to the pod network namespace, without needing direct filesystem access.

### Visual Diagram:

```text
[pod creation]
   ↓
[istio-cni plugin]
   ↓ (sends network namespace path)
[istio-cni-node]
   ↓ (opens /proc/<pid>/ns/net and sends FD via SCM_RIGHTS)
[ztunnel]
   ↓ (calls setns(fd, CLONE_NEWNET))
[inside pod netns]
```

---

## Simple Go Example: Sending and Receiving a File Descriptor

### Sender (sending an FD with metadata):

```go
package main

import (
	"net"
	"os"
	"syscall"
)

func main() {
	conn, _ := net.Dial("unix", "/tmp/ns.sock")
	file, _ := os.Open("/run/netns/cni-xxxxxxxxxx")
	defer file.Close()
	rights := syscall.UnixRights(int(file.Fd()))

	// Send a meaningful message, such as CNI command and pod metadata
	message := []byte("CNI_ADD:pod-name=myapp-pod,namespace=default")
	conn.(*net.UnixConn).WriteMsgUnix(message, rights, nil)
}
```

### Receiver (receiving the FD and parsing metadata):

```go
package main

import (
	"fmt"
	"net"
	"os"
	"strings"
	"syscall"
)

func main() {
	ln, _ := net.Listen("unix", "/tmp/ns.sock")
	conn, _ := ln.Accept()
	defer conn.Close()

	oob := make([]byte, 1024)
	buf := make([]byte, 1024)
	n, oobn, _, _, _ := conn.(*net.UnixConn).ReadMsgUnix(buf, oob)

	message := string(buf[:n])
	fmt.Printf("Received message: %s\n", message)

	commandFields := strings.SplitN(message, ":", 2)
	if len(commandFields) != 2 {
		fmt.Println("Invalid message format")
		return
	}

	cniCommand := commandFields[0]
	metadata := commandFields[1]
	fmt.Printf("CNI Command: %s\n", cniCommand)
	fmt.Printf("Metadata: %s\n", metadata)

	scms, _ := syscall.ParseSocketControlMessage(oob[:oobn])
	fds, _ := syscall.ParseUnixRights(&scms[0])

	origNetns, _ := os.Open("/proc/self/ns/net")
	defer origNetns.Close()

	syscall.Setns(fds[0], syscall.CLONE_NEWNET)
	fmt.Println("Switched to new namespace")
	// this is just an example, ztunnel does not do this. 
	switch cniCommand {
	case "CNI_ADD":
		fmt.Println("Handling pod ADD operation...")
	case "CNI_DEL":
		fmt.Println("Handling pod DELETE operation...")
	default:
		fmt.Println("Unknown command")
	}

	/* You could do anything in the network namespace, create a veth from pod to root netns. 
	Create a socket, add iptables rules like ztunnel, etc. 
	*/

	syscall.Setns(int(origNetns.Fd()), syscall.CLONE_NEWNET)
	fmt.Println("Switched back to original namespace")
}
```

---

## A Quick Note About setns and Threads

The setns syscall is what makes namespace switching possible. But:

> setns **only moves the calling thread** into the new namespace — **not** the entire process.

In multi-threaded programs, this is critical to understand:
- Only the thread that calls setns is affected.
- Other threads remain in their original namespaces.

This subtle behavior shapes why SCM_RIGHTS ztunnel is written in **Rust**.

---

## Why Rust Matters for Namespace Switching

Languages like **Go** use **goroutines**, which can be rescheduled between OS threads.

- In Go, a goroutine could switch threads after calling setns, leading to undefined behavior.
- Go offers no easy way to pin a goroutine to a thread safely across arbitrary operations.

**Rust**, however, uses **native OS threads** by default.

In ztunnel:
- The thread can safely setns into the pod's namespace.
- All operations occur on the same thread without scheduling surprises.

Thus, **Rust provides safe, predictable namespace handling**, essential for a service mesh data plane.

| Aspect | Go (goroutines) | Rust (native threads) |
|:------|:-----------------|:----------------------|
| Thread behavior | Goroutines can move between threads | Threads stay pinned |
| setns effect | Dangerous unless pinned manually | Safe and predictable |
| Namespace switching | Complicated and error-prone | Straightforward |

Note: you can pin Goroutines to a specific thread however this is undesirable and should be short lived. 

---

## Conclusion

When I started investigating how ztunnel accessed pod network namespaces without traditional mounts, I didn't expect to stumble into the fascinating world of **socket control messages**. Under the hood, SCM_RIGHTS and setns are the secret ingredients — with **Rust's native threading model** making it all feasible and reliable. In complex systems like service meshes, it’s often the low-level details — like a single system call — that shape the high-level architecture. It makes you wonder where else this approach can be applied such as more broadly in the container networking and Kubernetes space. 

---
