# Making Landlock Transparent: From Code Changes to OCI and Kubernetes Integration  

### Why Landlock Matters  

Container security has traditionally relied on seccomp filters, AppArmor, or SELinux profiles to restrict what applications can do. These mechanisms are powerful, but they can also be difficult to configure and often lack fine-grained controls.  

Other modern tools exist as well:  

- **seccomp** filters restrict syscalls but can’t reason about higher-level resources (e.g., “only bind to port 80”).  
- **AppArmor / SELinux** provide mandatory access controls, but profiles can be complex to author, distro-specific, and often require privileged setup.  
- **eBPF-based policies** (used by projects like Cilium or Falco) can monitor and sometimes block behavior at runtime, but they are typically enforced outside the process, rely on privileged agents, and don’t always provide the same *per-process* guarantees that kernel-integrated LSMs do.  

**Landlock** is different. It’s a Linux Security Module (LSM) designed for *unprivileged, fine-grained sandboxing*. Developers (or runtimes) can define rules like:  

- This process can only read `/data` but not `/etc`.  
- This process can only connect or bind to specific network ports.  
- This process can only create files under `/tmp`.  

Unlike eBPF-based monitoring or seccomp filtering, Landlock attaches directly to the process itself in the kernel, enforcing least-privilege policies at the object level (filesystem paths, ports). That makes it a natural fit for transparent container enforcement.  

The challenge today is that **using Landlock usually requires modifying application code** to call its APIs directly. That limits adoption and makes it harder for developers and operators to take advantage of its protections.  

My work aims to fix that. By adding Landlock support to the **OCI runtime specification** and integrating it into **runc**, we can make Landlock transparent to users. That means container policies can be declared once (in OCI config or Kubernetes manifests), and enforced automatically by the runtime and kernel — without requiring application changes.  

---

### A Practical Example: Binding to the Right Port  

Imagine you’re writing a simple web server. You know your app should **only ever bind to port 80**.  

Now suppose you import a dependency that contains malicious code. It tries to bind to **port 100**, opening a backdoor listener.  

On a regular Linux system, nothing prevents this — the kernel will happily let the process open port 100. You might never even realize it’s running.  

With Landlock integrated into runc, you can declare a simple policy:  

> *“This process may only bind to port 80.”*  

When the malicious dependency attempts to bind to port 100, the kernel denies it instantly. The server keeps running as expected on port 80, and the exploit attempt is silently blocked.  

This enforcement happens at the kernel level, so there’s no dependency on application logic or library code. Developers don’t need to wire in Landlock APIs — they just describe the allowed behavior once, and the runtime enforces it.  

---

### What Happens on Denial: `bind(2)` Returns `EPERM`  

When Landlock blocks a forbidden network operation, the syscall fails with **`-1` and `errno = EPERM`** (Operation not permitted).  

**strace view**  
```
bind(3, {sa_family=AF_INET, sin_port=htons(100), sin_addr=inet_addr("0.0.0.0")}, 16) = -1 EPERM (Operation not permitted)
```

**Go example**  
```go
ln, err := net.Listen("tcp", ":100")
if err != nil {
    // Typically prints: "listen tcp :100: bind: operation not permitted"
    var sysErr *os.SyscallError
    if errors.As(err, &sysErr) {
        if errno, ok := sysErr.Err.(syscall.Errno); ok && errno == syscall.EPERM {
            log.Println("bind denied by Landlock policy (EPERM)")
        }
    }
}
```

This is a clean, kernel-level signal you can surface in logs or tests. If an app unexpectedly gets `EPERM` on `bind(2)`, that’s a strong hint it attempted something outside the declared Landlock policy.  

---

### Extending to Kubernetes  

This approach becomes even more powerful when combined with Kubernetes.  

In Kubernetes, developers already declare which ports a container should expose in the **Pod spec**:  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
  - name: web
    image: myapp:latest
    ports:
    - containerPort: 80
```  

Here the intent is explicit: the container should only bind to port 80.  

With Landlock integrated into the OCI spec and runc, those Pod declarations can be passed all the way down to the runtime. runc then builds and applies the Landlock ruleset:  

- ✅ Allowed: bind to port 80  
- ❌ Denied: bind to any other port  

This creates a strong link between **developer intent (Pod manifest)** and **kernel-enforced behavior (Landlock)**. Even if a dependency misbehaves, the kernel prevents it from stepping outside the declared boundaries.  

---

### Why Transparent Integration Matters  

Making Landlock part of the OCI spec and runc provides several benefits:  

- **No code changes**: Developers don’t need to modify their applications to gain Landlock protections.  
- **Declarative security**: Sandboxing rules become part of the container configuration, just like seccomp or cgroups.  
- **Kubernetes-native**: Pod specs naturally flow down into runtime enforcement.  
- **Least privilege by default**: Applications only get the resources they actually need.  

Compared to eBPF monitoring or seccomp filters, Landlock’s LSM approach ensures the enforcement is both *in-kernel* and *process-local*. It can’t be bypassed by removing a privileged agent or forgetting a profile.  

---

### What’s Next  

The long-term goal is to standardize Landlock support across the OCI ecosystem so all runtimes can enforce it consistently. By bridging developer declarations (like Pod ports) with kernel-level enforcement, we close a critical gap: ensuring that what developers *say* an application should do is what it *can* do.  

It’s also worth noting that **Landlock itself is under active development**. Each new kernel release expands its capabilities:  

- Initial support focused on filesystem restrictions.  
- Later ABIs introduced **network controls** (e.g., restricting bind/connect).  
- Future work is exploring additional domains like IPC and refinements to existing controls.  

This means the protections available through the OCI spec and runc integration will only get stronger over time, without requiring users to change how they configure containers. As the kernel improves, the container ecosystem automatically benefits.  

Landlock is already part of the Linux kernel. By making it part of OCI and runc, we make it part of everyday container security and ensure that its ongoing improvements become immediately useful to developers and operators everywhere.  
