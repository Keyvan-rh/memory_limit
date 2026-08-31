# Best Practices for OpenShift Memory Limits

## Why Your Memory Always Looks Full

One of the most common points of confusion in OpenShift is what's often called the **"memory leak illusion."**

The OpenShift console shows a metric called **Working Set Memory** (`container_memory_working_set_bytes`). To understand why your applications look like they are running out of memory when they aren't, you need to understand how Linux handles memory inside a container.

A container's memory usage isn't just the memory your application code is actively using (heap or RSS). It also includes the **Linux Page Cache**. When your application reads or writes files — including writing logs — Linux uses available memory to cache those file pages to speed up future I/O operations. OpenShift counts this active page cache against your container's memory limit.

If your application writes logs continuously, the page cache will grow until it hits your memory limit. At that point, the console shows memory usage near 100%. **However, this is not harmful.** Before the container actually runs out of memory, the Linux kernel will quietly reclaim the page cache to make room for real allocations. Your application only gets killed (`OOMKilled`) if the *actual application memory* (RSS) tries to exceed the limit after all reclaimable cache has been freed.

> **Bottom line:** High working set memory does not mean your application is leaking memory. Do not keep increasing your memory limit just because the console graph looks full.

---

## Best Practices

To prevent false alarms and keep your applications stable without endlessly adding memory, follow these practices.

### 1. Stream Logs to Standard Output

If your application writes logs to local files inside the container, it will inflate the page cache and make your working set memory appear maxed out. Instead, configure your application to stream logs directly to **stdout** and **stderr** so the OpenShift logging stack (e.g., the cluster log forwarder) handles them without inflating your pod's memory metrics.

### 2. Leave a 20-30% Buffer Between Requests and Limits

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

### 3. Match Requests and Limits for Critical Applications (Guaranteed QoS)

If an application is highly critical, set the **Request** and **Limit** to the **same value**. This gives the pod a **Guaranteed** Quality of Service (QoS) class in Kubernetes. Guaranteed pods are the last to be evicted when a node runs low on memory, making them protected from node-level overcommitment.

```yaml
resources:
  requests:
    memory: "1Gi"
  limits:
    memory: "1Gi"      # Same value = Guaranteed QoS
```

### 4. Configure Application Runtimes to Respect Container Limits

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

---

## Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| Memory graph near 100% but no OOMKills | Page cache inflation from file I/O or logging | Stream logs to stdout; the cache is reclaimable |
| Pod `OOMKilled` immediately on start | Runtime ignoring container limit | Set `-XX:MaxRAMPercentage` (Java) or `--max-old-space-size` (Node.js) |
| Pod evicted during node pressure | Request set too low or QoS is Burstable | Raise request, or match request = limit for Guaranteed QoS |

---

## Further Reading

- [OpenShift 4.x: Managing Container Memory](https://docs.openshift.com/container-platform/latest/nodes/clusters/nodes-cluster-resource-configure.html)
- [Kubernetes: Resource Management for Pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Understanding Linux Page Cache](https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html)
