# Common Kubernetes Pod Errors — Complete Reference with Solutions

---

## Quick Diagnostic Sequence (Apply to ALL errors)

```bash
kubectl get pods -n <ns> -o wide          # 1. What's the status?
kubectl describe pod <name> -n <ns>       # 2. Read Events (the #1 clue)
kubectl logs <pod> -n <ns> --previous     # 3. Previous container logs
kubectl logs <pod> -n <ns> -f             # 4. Live logs
kubectl exec -it <pod> -- bash            # 5. Shell in (if running)
kubectl get events --sort-by=.lastTimestamp  # 6. Cluster-wide events
```

---

## 1. CrashLoopBackOff

**What it means:** Container starts → exits → kubelet restarts it → exits again. K8s backs off exponentially (10s, 20s, 40s… up to 5 min max) between restarts.

**It's a SYMPTOM, not a cause.** The real cause is in the exit code + logs.

| Exit Code | Meaning | Typical Cause | Fix |
|---|---|---|---|
| **0** | Clean exit | Process finished (not a server); entrypoint runs a script that exits | Keep main process in foreground; use a Job if one-shot |
| **1** | Generic app error | Unhandled exception, missing env var, failed DB connection, bad config | `kubectl logs --previous` → fix the error shown |
| **2** | Misuse of shell/built-in | Bad shell syntax in entrypoint, wrong `sh -c` quoting | Fix the command/args in the spec |
| **126** | Found but not executable | Binary lacks `+x`, wrong architecture (ARM binary on x86) | Fix permissions: `chmod +x`; rebuild for correct arch |
| **127** | Command not found | Typo in `command`/`args`; binary missing from slim image | Fix the path; install the tool in the image |
| **137** | SIGKILL (128+9) | OOMKilled (see #3 below) OR liveness probe kill after grace period exceeded | Check `Reason: OOMKilled` in describe; if not OOM → add `startupProbe` |
| **139** | SIGSEGV (128+11) | Segmentation fault; native library mismatch, wrong CPU arch | Rebuild image; check `uname -m` matches image arch |
| **143** | SIGTERM (128+15) | Graceful shutdown signal; liveness probe failure; rolling update | Usually normal during rollouts; if persistent → fix liveness probe |
| **134** | SIGABRT (128+6) | Uncaught C++ exception, assertion failure, glibc abort | Check app logs; fix the assertion/exception |

**Diagnosis:**
```bash
kubectl describe pod <name>
# Look at:
#   Last State: Terminated
#     Reason:    OOMKilled / Error
#     Exit Code: 137 / 1 / 127 ...
#   Events:      Liveness probe failed / Back-off restarting

kubectl logs <pod> --previous -c <container>   # The crashed container's output
```

**Common root causes & fixes:**

| Root Cause | Signal | Fix |
|---|---|---|
| Missing env var / config | Exit 1; log says "env var X not set" or "file not found" | Add the env var, ConfigMap key, or Secret |
| Wrong entrypoint / command | Exit 126/127; immediate crash | Fix `command`/`args` in spec |
| Port already in use | Log: "Address already in use" | Check for port conflicts (sidecar, init container) |
| DB / dependency not ready | Log: "connection refused to postgres:5432" | Add `startupProbe`; use init container to wait; check Service DNS |
| Liveness probe too aggressive | Events: "Liveness probe failed"; app looks healthy | Add `startupProbe`; increase `initialDelaySeconds` / `failureThreshold` |
| File permission on volume mount | Exit 126; "Permission denied" | Set `securityContext.fsGroup`; check PVC access mode |

---

## 2. OOMKilled (Exit Code 137)

**What it means:** The container's memory usage exceeded `resources.limits.memory`. The Linux kernel's OOM killer sent **SIGKILL** (cannot be caught or handled). No application error in logs — the process just vanishes.

**Exit code:** 137 = 128 + 9 (SIGKILL)

**Diagnosis:**
```bash
kubectl describe pod <name>
# Look for:
#   Last State: Terminated
#     Reason:    OOMKilled
#     Exit Code: 137

kubectl top pod <name> -n <ns>    # Current memory usage
```

**Causes & Solutions:**

| Cause | How to confirm | Fix |
|---|---|---|
| Memory limit too low | `kubectl top pod` shows usage near limit; app works fine after restart | Increase `resources.limits.memory` (e.g., 512Mi → 1Gi) |
| Memory leak in app | Memory climbs steadily over hours/days; OOM recurs at increasing intervals | Fix the leak in code; set JVM heap (`-Xmx`) below the K8s limit |
| Node-level MemoryPressure | Multiple Pods on same node killed; `kubectl describe node` shows `MemoryPressure: True` | Add nodes; reduce Pod requests; set limits on all Pods (BestEffort killed first) |
| Burst spike (one-off) | Single OOM event, doesn't recur | Usually acceptable; set limit with 20–30% headroom above peak |
| No memory limit set (BestEffort QoS) | Pod has no `limits.memory`; killed during node pressure | Always set memory limits (prevents unbounded growth) |

**Best Practices:**
```yaml
resources:
  requests:
    memory: "256Mi"    # Scheduling guarantee
  limits:
    memory: "512Mi"    # Hard cap (OOM kill if exceeded)
```
- Set JVM heap: `-Xmx384m` when limit is `512Mi` (leave room for metaspace, thread stacks)
- Monitor: `kubectl top pod`, Prometheus `container_memory_working_set_bytes`
- Alert when usage > 80% of limit

**Critical distinction:**
- **Exit 137 + Reason: OOMKilled** → Memory limit exceeded (container cgroup OOM)
- **Exit 137 + NO OOMKilled reason** → SIGKILL from another source (liveness probe kill after grace period, node eviction, manual `kill -9`)

---

## 3. ImagePullBackOff / ErrImagePull

**What it means:** kubelet cannot download the container image. The container **never starts**.

| Status | Meaning |
|---|---|
| **ErrImagePull** | First pull attempt failed |
| **ImagePullBackOff** | Subsequent retries also failed (backoff applied) |

**Diagnosis:**
```bash
kubectl describe pod <name>
# Events will say:
#   Warning  Failed  Failed to pull image "myapp:v1.2": rpc error: ...
#   Warning  Failed  Error: ImagePullBackOff
```

**Causes & Solutions:**

| Cause | Event message | Fix |
|---|---|---|
| **Wrong image name or tag** | `manifest unknown` / `not found` | Fix the typo; verify tag exists: `docker pull <image>` locally |
| **Private registry, no credentials** | `unauthorized` / `authentication required` / `denied` | Create `imagePullSecret` and reference it in Pod spec |
| **Registry unreachable** | `dial tcp <registry>:443: i/o timeout` | Check network/firewall/security groups; verify DNS resolves registry |
| **Rate limiting (Docker Hub)** | `toomanyrequests: rate limit exceeded` | Use a private registry mirror; or use `imagePullPolicy: IfNotPresent` with pre-loaded images |
| **Wrong architecture** | `exec format error` / `no matching manifest for linux/arm64` | Build multi-arch image; match `node.kubernetes.io/arch` |
| **Corrupted image / digest mismatch** | `invalid checksum` / `manifest validation failed` | Re-push the image; pin to digest (`@sha256:...`) |

**Fix: Create imagePullSecret:**
```bash
kubectl create secret dockerregistry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

# In Pod spec:
spec:
  imagePullSecrets:
    - name: my-registry-secret
```

**For AWS ECR (EKS):**
```bash
# Use IRSA (IAM Roles for Service Accounts) — no secret needed
# Or:
aws ecr get-login-password --region ap-south-1 | \
  kubectl create secret dockerregistry ecr-secret \
  --docker-server=<account>.dkr.ecr.ap-south-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-south-1)
```

---

## 4. Pod Stuck in Pending

**What it means:** Pod is accepted by the API server but the **scheduler cannot place it** on any node. No container is created.

**Diagnosis:**
```bash
kubectl describe pod <name>
# Events from default-scheduler:
#   Warning  FailedScheduling  0/6 nodes are available:
#     3 Insufficient memory, 3 node(s) had untolerated taint {dedicated: gpu}
```

**Causes & Solutions:**

| Cause | Event message | Fix |
|---|---|---|
| **Insufficient CPU/Memory** | `Insufficient cpu` / `Insufficient memory` | Lower `resources.requests`; add nodes; scale up node group |
| **Untolerated taint** | `node(s) had untolerated taint {key: value}` | Add matching `tolerations` to Pod spec; or remove taint from node |
| **Node selector / affinity mismatch** | `node(s) didn't match Pod's node affinity/selector` | Fix labels on nodes or relax the selector/affinity |
| **Pod anti-affinity conflict** | `node(s) didn't satisfy existing pods' anti-affinity` | Relax anti-affinity (change `required` → `preferred`) |
| **Topology spread constraint** | `node(s) didn't satisfy topology spread` | Increase `maxSkew` or add nodes in missing zones |
| **Unbound PVC** | `pod has unbound immediate PersistentVolumeClaims` | Check PVC: `kubectl get pvc`; ensure StorageClass exists and can provision |
| **ResourceQuota exceeded** | `exceeded quota: compute-resources` | Increase quota or reduce requests |
| **No matching nodes at all** | `0/0 nodes are available` | Check if nodes exist: `kubectl get nodes` |

**Quick fix examples:**
```yaml
# Add toleration for a taint
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

# Relax anti-affinity
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:  # was required
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels: {app: api}
          topologyKey: kubernetes.io/hostname
```

---

## 5. CreateContainerConfigError

**What it means:** The container **cannot start** because a referenced ConfigMap, Secret, or volume is missing or invalid.

**Diagnosis:**
```bash
kubectl describe pod <name>
# Events:
#   Warning  FailedCreatePodContainer  Error: mountVolume.SetUp failed
#   Warning  Failed  spec.containers[0].env[2]: ConfigMap "my-config" not found
#   Warning  Failed  spec.containers[0].env[3]: Secrets "my-secret" not found
```

**Causes & Solutions:**

| Cause | Event message | Fix |
|---|---|---|
| **ConfigMap not found** | `ConfigMap "name" not found` | Create the ConfigMap in the **same namespace** as the Pod |
| **Secret not found** | `Secrets "name" not found` | Create the Secret; check namespace |
| **Key missing in ConfigMap/Secret** | `key "MY_KEY" not found in ConfigMap` | Add the key: `kubectl create configmap cm --from-literal=MY_KEY=value` |
| **Volume mount path invalid** | `mountVolume.SetUp failed` / `no such file or directory` | Fix `volumeMounts.mountPath` and `volumes` spec |
| **imagePullSecret not found** | `secret "name" not found` | Create the secret (see ImagePullBackOff above) |

**Classic ordering bug:** Deployment applied before ConfigMap/Secret exists.
```bash
# Always apply config first:
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
```

---

## 6. RunContainerError

**What it means:** The container runtime (containerd/CRI-O) accepted the container spec but **failed to actually start the process**.

**Diagnosis:**
```bash
kubectl describe pod <name>
# Events:
#   Warning  Failed  Error: failed to create containerd task: OCI runtime create failed:
#     error running hook: ...
#   Warning  Failed  containerd: failed to mount ...: permission denied
```

**Causes & Solutions:**

| Cause | Event message | Fix |
|---|---|---|
| **Entrypoint not executable** | `permission denied` / `exec format error` | `chmod +x` in Dockerfile; check arch |
| **Volume mounted over binary path** | `no such file or directory` for the entrypoint | Fix `mountPath` — don't mount over `/app` if binary is there |
| **cgroup / seccomp issue** | `failed to create containerd task` | Check `securityContext`; remove overly restrictive settings |
| **Missing shared library** | `error while loading shared libraries: libX.so` | Install the library in the image (base image too slim) |
| **GPU / device not available** | `failed to initialize NVML` / `no such device` | Check node has GPU; add `nvidia.com/gpu` resource |

---

## 7. ContainerCreating (Stuck)

**What it means:** Pod is scheduled to a node but the kubelet **cannot create the container**. Stays in this state indefinitely.

**Diagnosis:**
```bash
kubectl describe pod <name>
# Events:
#   Warning  FailedMount  MountVolume.SetUp failed for volume "my-pvc"
#   Warning  FailedAttachVolume  Multi-Attach error for volume "vol-xxx"
```

**Causes & Solutions:**

| Cause | Event message | Fix |
|---|---|---|
| **PVC not bound** | `persistentvolumeclaim "name" not found` / PVC status: Pending | Check StorageClass; `kubectl get pvc`; fix provisioner |
| **Volume attach conflict** | `Multi-Attach error for volume "vol-xxx"` | EBS volume still attached to old node; wait for detach or force-detach |
| **Volume mount failure** | `MountVolume.SetUp failed` / `permission denied` | Check access mode (RWX needs NFS); check `fsGroup` |
| **CNI issue** | `failed to set up sandbox` / `CNI plugin not found` | Check CNI DaemonSet; `kubectl logs -n kube-system <cni-pod>` |
| **Insufficient ephemeral storage** | `node has insufficient free ephemeral-storage` | Clean up logs/images on node; increase disk |

---

## 8. Evicted

**What it means:** The **kubelet evicted** the Pod because the node ran low on resources. The Pod is terminated and (for Deployments) rescheduled elsewhere.

**Diagnosis:**
```bash
kubectl get pods -n <ns>
# NAME    READY   STATUS   RESTARTS   AGE
# my-pod  0/1     Evicted  0          2h

kubectl describe pod <name>
#   Reason:  Evicted
#   Message: The node was low on resource: [memory].
```

**Causes & Solutions:**

| Cause | Message | Fix |
|---|---|---|
| **Node memory pressure** | `low on resource: [memory]` | Add nodes; reduce Pod memory requests; set limits on all Pods |
| **Node disk pressure** | `low on resource: [ephemeral-storage]` | Clean up images/logs; increase node disk; set `emptyDir.sizeLimit` |
| **Node PID pressure** | `low on resource: [pids]` | Reduce processes per Pod; increase node PID limit |
| **Node NotReady timeout** | `node was unreachable for more than X seconds` | Fix node networking; check kubelet |

**Prevention:**
```yaml
# Always set requests to prevent overcommit:
resources:
  requests:
    memory: "256Mi"
    ephemeral-storage: "1Gi"
  limits:
    memory: "512Mi"
```
- Monitor: `kubelet_evictions_total`, node conditions (`MemoryPressure`, `DiskPressure`)
- Set `PodDisruptionBudget` to limit concurrent evictions

---

## 9. Node NotReady

**What it means:** The node stopped communicating with the control plane. All Pods on it are at risk (evicted after `node-monitor-grace-period`, default 40s).

**Diagnosis:**
```bash
kubectl get nodes
# NAME    STATUS     ROLES    AGE   VERSION
# node-1  NotReady   <none>   10d   v1.30

kubectl describe node node-1
# Conditions:
#   Ready:              False
#     Reason:           KubeletNotReady
#     Message:          container runtime network not ready
```

**Causes & Solutions:**

| Cause | Condition/Message | Fix |
|---|---|---|
| **kubelet crashed** | `KubeletNotReady` | `systemctl restart kubelet` on the node |
| **Container runtime down** | `container runtime is down` | Restart containerd/CRI-O |
| **Network failure** | `KubeletNotReachable` | Check VPC peering, security groups, ENI |
| **Disk pressure (kubelet can't write)** | `KubeletHasSufficientDisk: False` | Clean up `/var/lib/containerd`, `/var/log` |
| **Memory pressure** | `KubeletHasSufficientMemory: False` | Free memory; add swap (not recommended); scale out |
| **Cloud VM terminated** | Node gone from cloud console | Replace the node (ASG should auto-replace) |
| **CNI broken** | `container runtime network not ready` | Restart CNI DaemonSet; check CNI logs |

**Recovery:**
```bash
kubectl cordon node-1              # Mark unschedulable
kubectl drain node-1 --ignore-daemonsets
# Fix the issue (restart kubelet, fix network, clean disk)
kubectl uncordon node-1
```

---

## 10. Liveness / Readiness / Startup Probe Failures

**What it means:** The kubelet's health checks are failing, causing restarts (liveness) or traffic removal (readiness).

| Probe | Failure action | Impact |
|---|---|---|
| **Liveness** | Container is **killed and restarted** | Pod restarts (CrashLoopBackOff if repeated) |
| **Readiness** | Pod removed from Service endpoints | No traffic (Pod stays running) |
| **Startup** | Container is killed (if it fails before succeeding once) | Prevents liveness from killing slow-starting apps |

**Diagnosis:**
```bash
kubectl describe pod <name>
# Events:
#   Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 503
#   Warning  Unhealthy  Readiness probe failed: connection refused
```

**Causes & Solutions:**

| Cause | Signal | Fix |
|---|---|---|
| **App boots slower than probe allows** | Liveness fails for first 30–60s, then succeeds | Add `startupProbe` (e.g., `failureThreshold: 30, periodSeconds: 5` = 150s to start) |
| **Wrong port or path** | `connection refused` / `404` | Fix `port` and `path` in probe spec to match the app |
| **Probe too aggressive** | `timeoutSeconds: 1` fails under CPU throttling | Increase `timeoutSeconds` to 3–5 |
| **App genuinely unhealthy** | Persistent 5xx from health endpoint | Fix the application bug |
| **Probe checks external dependency** | Health endpoint returns 503 because DB is down | Make liveness check only the process (not dependencies); use readiness for dependency checks |

**Recommended probe configuration:**
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30      # 30 × 5s = 150s to start
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

---

## 11. CreateContainerError / InvalidImageName

**What it means:** The image reference is syntactically invalid or the container spec has an error.

| Cause | Fix |
|---|---|
| Image name has invalid characters | Fix the image string (no spaces, valid registry format) |
| `imagePullPolicy: Always` but no network | Change to `IfNotPresent` or fix network |
| Container port not a number | Fix `containerPort` to a valid integer (1–65535) |

---

## 12. Terminating (Stuck)

**What it means:** Pod deletion was requested but the Pod won't go away (usually stuck in `Terminating` for minutes).

**Causes & Solutions:**

| Cause | Signal | Fix |
|---|---|---|
| **Finalizer blocking deletion** | `kubectl get pod -o yaml` shows `finalizers` | Remove finalizer: `kubectl patch pod <name> -p '{"metadata":{"finalizers":null}}'` |
| **Grace period timeout** | `terminationGracePeriodSeconds` too long | Wait it out; or `kubectl delete pod <name> --grace-period=0 --force` |
| **Node NotReady** | Pod on a dead node | Wait for node controller to clean up (default 5 min); or force-delete |
| **PVC deletion blocked** | Volume can't be detached | Check volume attachment; force-detach in cloud console |

**Force delete (last resort):**
```bash
kubectl delete pod <name> --grace-period=0 --force
```

---

## 13. Init:CrashLoopBackOff / Init:Error

**What it means:** An **init container** is failing. The main container won't start until all init containers complete successfully.

**Diagnosis:**
```bash
kubectl describe pod <name>
#   Init:CrashLoopBackOff
#   Init Container: wait-for-db
#   Last State: Terminated, Exit Code: 1

kubectl logs <pod> -c wait-for-db --previous
```

**Common causes:**
- Init container tries to connect to a service that doesn't exist yet
- Wrong image or command in init container
- Missing ConfigMap/Secret referenced by init container

**Fix:** Same as CrashLoopBackOff (see #1) but target the init container name with `-c <init-container-name>`.

---

## 14. DNS Resolution Failures (NXDOMAIN)

**What it means:** Pods can't resolve Service names (`*.svc.cluster.local`). All service-to-service calls fail.

**Diagnosis:**
```bash
kubectl exec -it <pod> -- nslookup my-service.default.svc.cluster.local
# Server:  10.96.0.10
# Address: 10.96.0.10:53
# ** server can't find my-service.default.svc.cluster.local: NXDOMAIN
```

**Causes & Solutions:**

| Cause | Signal | Fix |
|---|---|---|
| **CoreDNS down** | All Pods fail DNS; `kubectl get pods -n kube-system` shows coredns not running | Restart CoreDNS; check its logs |
| **Service has no endpoints** | `kubectl get endpoints <svc>` shows empty | Check Service `selector` matches Pod labels |
| **Wrong namespace** | Pod in `ns1` tries to reach Service in `ns2` without FQDN | Use FQDN: `svc.ns2.svc.cluster.local` |
| **`/etc/resolv.conf` misconfigured** | Custom `dnsConfig` broke resolution | Remove custom `dnsConfig` or fix it |
| **NetworkPolicy blocking DNS** | UDP 53 to CoreDNS blocked | Add NetworkPolicy allowing egress to CoreDNS Pod |

---

## 15. FailedMount / FailedAttachVolume (Storage)

**What it means:** A PersistentVolume cannot be attached or mounted to the Pod.

**Diagnosis:**
```bash
kubectl describe pod <name>
# Events:
#   Warning  FailedAttachVolume  Multi-Attach error for volume "vol-0abc123"
#   Warning  FailedMount  Unable to attach or mount volumes: unattached volumes: my-pvc

kubectl get pvc -n <ns>
# NAME    STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc  Pending                                      gp2            5m
```

**Causes & Solutions:**

| Cause | Signal | Fix |
|---|---|---|
| **PVC Pending (no PV available)** | PVC status: Pending | Check StorageClass exists; check provisioner IAM permissions; check volume quota |
| **Multi-Attach error** | `Multi-Attach error for volume` | Volume is `ReadWriteOnce` but requested by 2 nodes; or old node's detach is stuck |
| **Wrong access mode** | `volume does not support ReadWriteMany` | Use NFS/CephFS for RWX; or use `ReadWriteOnce` |
| **Volume in different AZ** | `volume is in zone us-east-1a, node is in us-east-1b` | Use zone-aware storage; or use `topology` in StorageClass |
| **Provisioner quota exceeded** | `QuotaExceeded` in events | Increase cloud quota (EBS volumes per account) |

---

## 16. NetworkPolicy Blocking Traffic

**What it means:** A NetworkPolicy was applied that **silently drops** traffic. No error is generated — connections just **time out**.

**Diagnosis:**
```bash
# From the source Pod:
kubectl exec -it <source-pod> -- curl -v http://<target-svc>:80/
# * Connected to target-svc (10.96.x.x) port 80
# * (connection hangs... times out after 30s)

# Check NetworkPolicies:
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <ns>
```

**Causes & Solutions:**

| Cause | Signal | Fix |
|---|---|---|
| **Default-deny policy without allow rule** | All egress/ingress blocked after policy applied | Add an `allow` rule for the specific traffic |
| **Namespace selector too narrow** | Traffic from specific namespace blocked | Broaden `from`/`to` namespaceSelector |
| **Port not included** | Specific port blocked, others work | Add the port to the policy |

**Quick test:**
```yaml
# Temporary: remove all NetworkPolicies to confirm they're the cause
kubectl delete networkpolicy --all -n <ns>
# If traffic flows again → add back rules one by one
```

---

## 17. ResourceQuota / LimitRange Violations

**What it means:** Pod creation is rejected because it exceeds namespace-level quotas.

**Diagnosis:**
```bash
kubectl describe pod <name>
# Events:
#   Warning  FailedCreate  Error creating: pods "my-pod" is forbidden:
#     exceeded quota: compute-resources, requested: memory=1Gi, used: memory=3Gi, limited: memory=4Gi
```

**Fix:**
```bash
kubectl get resourcequota -n <ns> -o yaml
# Increase the quota:
kubectl patch resourcequota compute-resources -n <ns> \
  -p '{"spec":{"hard":{"memory":"8Gi"}}}'
```

---

## 18. Exit Code 143 (SIGTERM) — Not Always a Problem

**Context matters:**

| Scenario | Normal? | Action |
|---|---|---|
| During rolling update (`kubectl rollout`) | ✅ Yes | No action — old Pod is being replaced |
| During scale-down | ✅ Yes | No action |
| During node drain | ✅ Yes | No action |
| Repeatedly, without any rollout | ❌ No | Check: liveness probe? `terminationGracePeriodSeconds` too short? External `kill -9`? |

---

## Master Cheat Sheet

| Status / Reason | Exit Code | First Command | Most Likely Fix |
|---|---|---|---|
| **CrashLoopBackOff** | 1 | `kubectl logs --previous` | Fix app error / config |
| **CrashLoopBackOff** | 126/127 | `kubectl describe pod` | Fix entrypoint / permissions |
| **OOMKilled** | 137 | `kubectl describe pod` | Increase memory limit |
| **ImagePullBackOff** | — | `kubectl describe pod` (Events) | Fix image tag / add pull secret |
| **Pending** | — | `kubectl describe pod` (Events) | Add resources / fix taints / fix PVC |
| **CreateContainerConfigError** | — | `kubectl describe pod` (Events) | Create missing ConfigMap/Secret |
| **RunContainerError** | — | `kubectl describe pod` (Events) | Fix entrypoint / volume / arch |
| **ContainerCreating** | — | `kubectl describe pod` (Events) | Fix PVC / CNI / storage |
| **Evicted** | — | `kubectl describe pod` | Add node capacity / set limits |
| **Node NotReady** | — | `kubectl describe node` | Fix kubelet / network / disk |
| **Probe failures** | 137/143 | `kubectl describe pod` (Events) | Add startupProbe / fix probe config |
| **Terminating (stuck)** | — | `kubectl get pod -o yaml` (finalizers) | Remove finalizer / force delete |
| **Init:CrashLoopBackOff** | varies | `kubectl logs -c <init> --previous` | Fix init container |
| **DNS failures** | — | `kubectl exec -- nslookup svc` | Fix CoreDNS / endpoints / NetworkPolicy |
| **FailedMount/Attach** | — | `kubectl describe pod` + `kubectl get pvc` | Fix StorageClass / access mode / AZ |
| **NetworkPolicy block** | — | `kubectl get networkpolicy -A` | Add allow rule |
| **ResourceQuota exceeded** | — | `kubectl get resourcequota` | Increase quota |

---

## Pro Tips

```bash
# Find ALL problematic Pods across the cluster
kubectl get pods -A | grep -v Running | grep -v Completed

# Find Pods with high restart counts
kubectl get pods -A -o json | jq -r '.items[] | select(.status.containerStatuses[0].restartCount > 3) | "\(.metadata.namespace)/\(.metadata.name): \(.status.containerStatuses[0].restartCount) restarts"'

# Check node conditions (pressure, not ready)
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason

# Debug a stuck Pod (ephemeral debug container)
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>

# Check if it's a scheduling issue
kubectl describe pod <name> | grep -A 10 "Events:"

# Check Service endpoints (empty = no ready Pods)
kubectl get endpoints <service-name> -n <ns>
```

