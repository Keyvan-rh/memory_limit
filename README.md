# Best Practices for OpenShift Memory Limits

[![Try the Simulator](https://img.shields.io/badge/Interactive_Demo-Try_the_Memory_Simulator-blue?style=for-the-badge&logo=kubernetes&logoColor=white)](https://keyvan-rh.github.io/memory_limit/k8s-simulator.html)

> Drag the sliders to see how Linux page cache, application memory (RSS), and container limits interact — and when `OOMKilled` actually happens.

## Why Your Memory Always Looks Full

One of the most common points of confusion in OpenShift is what's often called the **"memory leak illusion."**

The OpenShift console shows a metric called **Working Set Memory** (`container_memory_working_set_bytes`). To understand why your applications look like they are running out of memory when they aren't, you need to understand how Linux handles memory inside a container.

A container's memory usage isn't just the memory your application code is actively using (heap or RSS). It also includes the **Linux Page Cache**. When your application reads or writes files — including writing logs — Linux uses available memory to cache those file pages to speed up future I/O operations. OpenShift counts this active page cache against your container's memory limit.

If your application writes logs continuously, the page cache will grow until it hits your memory limit. At that point, the console shows memory usage near 100%. **However, this is not harmful.** Before the container actually runs out of memory, the Linux kernel will quietly reclaim the page cache to make room for real allocations. Your application only gets killed (`OOMKilled`) if the *actual application memory* (RSS) tries to exceed the limit after all reclaimable cache has been freed.

> **Bottom line:** High working set memory does not mean your application is leaking memory. Do not keep increasing your memory limit just because the console graph looks full.

### Other Hidden Memory Consumers

Page cache is the most common reason your memory looks full, but it isn't the only one. The following also count against your container's cgroup memory limit even though they are not your application's heap:

| Consumer | Why It Uses Memory | Reclaimable? |
|---|---|---|
| **Kernel slab cache** (dentry/inode cache) | Linux caches filesystem metadata (directory entries, inodes) to speed up path lookups. In cgroup v2 (default in OpenShift 4.12+), this is charged to your container. | Yes — the kernel reclaims it under pressure. |
| **tmpfs / emptyDir `medium: Memory`** | An `emptyDir` volume with `medium: Memory` is backed by tmpfs, which stores data directly in RAM. Every byte written to it counts against your memory limit. | No — it stays until files are deleted or the pod is terminated. |
| **Shared memory (`/dev/shm`)** | Applications that use POSIX shared memory (e.g., PostgreSQL, NGINX with shared zones) write to `/dev/shm`, which is also a tmpfs mount. | No — same as tmpfs above. |
| **Socket buffers** | TCP/UDP send and receive buffers are charged to the cgroup. High-throughput networking can add tens of megabytes. | Partially — buffers are freed when connections close. |
| **Memory-mapped files** | Files opened with `mmap()` (common in databases and search engines) are paged into memory and counted in working set, similar to page cache. | Yes — the kernel can evict clean pages. |

> **Key takeaway:** If your memory graph is high but there are no OOMKills, check whether you are using tmpfs volumes or `/dev/shm` before increasing your limit. Unlike page cache, tmpfs memory is **not reclaimable** — it is a real consumer.

---

## Best Practices

To prevent false alarms and keep your applications stable without endlessly adding memory, follow these practices.

### 1. Monitor RSS, Not Working Set

The default OpenShift console graph shows **Working Set Memory**, which includes reclaimable cache and makes your app look like it's out of memory. For alerting and capacity planning, use **RSS** (Resident Set Size) instead — it measures only the memory your application is actually using.

In a Prometheus/OpenShift Monitoring query:

```promql
container_memory_rss{container="your-app", namespace="your-ns"}
```

Compare it with the limit to see actual headroom:

```promql
container_memory_rss{container="your-app"} / container_spec_memory_limit_bytes{container="your-app"}
```

> If RSS is at 40% of your limit but working set is at 95%, your app is fine — the rest is reclaimable cache.

### 2. Stream Logs to Standard Output

If your application writes logs to local files inside the container, it will inflate the page cache and make your working set memory appear maxed out. Instead, configure your application to stream logs directly to **stdout** and **stderr** so the OpenShift logging stack (e.g., the cluster log forwarder) handles them without inflating your pod's memory metrics.

### 3. Leave a 20-30% Buffer Between Requests and Limits

Never set your memory limit to exactly what your application uses at peak.

| Field       | How to Set It                                                                                           |
|-------------|---------------------------------------------------------------------------------------------------------|
| **Request** | Set to your expected **average** usage. This tells the scheduler how much memory to reserve when placing the pod on a node. |
| **Limit**   | Set **20% to 30% higher** than your expected **peak** usage. This leaves room for the page cache and off-heap memory so the kernel doesn't have to constantly reclaim memory. |

Example:

```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "768Mi"    # ~50% higher than request, ~25% above expected peak
```

### 4. Match Requests and Limits for Critical Applications (Guaranteed QoS)

If an application is highly critical, set the **Request** and **Limit** to the **same value**. This gives the pod a **Guaranteed** Quality of Service (QoS) class in Kubernetes. Guaranteed pods are the last to be evicted when a node runs low on memory, making them protected from node-level overcommitment.

```yaml
resources:
  requests:
    memory: "1Gi"
  limits:
    memory: "1Gi"      # Same value = Guaranteed QoS
```

### 5. Avoid `emptyDir` with `medium: Memory` Unless Necessary

An `emptyDir` volume with `medium: Memory` is backed by tmpfs — every byte written to it comes directly from your container's memory limit and is **not reclaimable**. If your pod writes temporary files to such a volume, it can silently consume hundreds of megabytes.

```yaml
volumes:
  - name: temp
    emptyDir:
      medium: Memory       # This eats from your memory limit!
      sizeLimit: "100Mi"   # Always set a sizeLimit to cap it
```

If you don't need in-memory speed, use the default `emptyDir` (backed by disk) instead:

```yaml
volumes:
  - name: temp
    emptyDir: {}           # Backed by disk, does not consume memory limit
```

### 6. Configure Application Runtimes to Respect Container Limits

Many runtimes look at the **physical memory of the worker node**, not the container's cgroup limit. If a pod with a 2 GiB limit lands on a 64 GiB node, a misconfigured runtime will try to allocate far more than 2 GiB, resulting in an immediate `OOMKill`.

**Java (JDK 11+):**
Modern JVMs enable `-XX:+UseContainerSupport` by default, which reads the cgroup memory limit. Use `-XX:MaxRAMPercentage` to control how much of the container limit the heap can consume:

```bash
java -XX:MaxRAMPercentage=75.0 -jar app.jar
```

> For JDK 8, ensure you are on **8u191 or later** where container support was backported, or explicitly set `-Xmx`.

**Node.js:**
Set `--max-old-space-size` in megabytes to cap the V8 heap:

```bash
node --max-old-space-size=384 server.js
```

### 7. Use the Vertical Pod Autoscaler (VPA) to Right-Size

Instead of guessing requests and limits, let the **Vertical Pod Autoscaler** observe your application's actual memory usage over time and recommend (or automatically apply) right-sized values. This is available in OpenShift under **Workloads > VPA** or via the `VerticalPodAutoscaler` custom resource.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"      # "Off" = recommendations only, no auto-apply
```

> Start with `updateMode: "Off"` to review recommendations before letting VPA resize your pods automatically.

---

## Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| Memory graph near 100% but no OOMKills | Page cache inflation from file I/O or logging | Stream logs to stdout; check RSS instead of working set |
| Pod `OOMKilled` immediately on start | Runtime ignoring container limit | Set `-XX:MaxRAMPercentage` (Java) or `--max-old-space-size` (Node.js) |
| Pod evicted during node pressure | Request set too low or QoS is Burstable | Raise request, or match request = limit for Guaranteed QoS |
| Memory grows steadily, never reclaimed | tmpfs `emptyDir` or `/dev/shm` usage | Switch to disk-backed `emptyDir` or set a `sizeLimit` |
| Memory spikes during high traffic | TCP/UDP socket buffers under load | Account for networking overhead in your limit buffer |
| Not sure what the right limit should be | Guessing instead of measuring | Deploy VPA in `Off` mode and review its recommendations |

---

## Further Reading

- [OpenShift 4.x: Managing Container Memory](https://docs.openshift.com/container-platform/latest/nodes/clusters/nodes-cluster-resource-configure.html)
- [Kubernetes: Resource Management for Pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Understanding Linux Page Cache](https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html)
