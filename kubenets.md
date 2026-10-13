# Kubernetes core concepts: workloads, networking, storage, security

## 0. Architecture recap

| Plane | Components | Role |
|---|---|---|
| **Control plane** | kube-apiserver, etcd, scheduler, controller-manager, cloud-controller-manager | Stores desired state and reconciles it |
| **Node (data plane)** | kubelet, container runtime (containerd), kube-proxy, CNI plugin | Runs pods and wires up networking |

**Core idea:** Kubernetes is a **declarative, reconciliation-based system**. You submit desired state to the API server, it's stored in etcd, and controllers continuously work to make actual state match it.

---

# PART 1: WORKLOADS

## 1.1 Pod

The smallest deployable unit: one or more containers sharing a network namespace (one IP), storage volumes, and lifecycle. Pods are **ephemeral**, so you rarely create them directly.

- **Init containers** run to completion, in order, before app containers start (migrations, waiting for dependencies).
- **Sidecar containers** run alongside the app (log shippers, proxies). Recent versions support native sidecars (init containers with `restartPolicy: Always`), which fixes startup ordering and Job completion problems. Verify the feature status for your cluster version.

## 1.2 Controllers at a glance

| Resource | Use for | Key behavior |
|---|---|---|
| **ReplicaSet** | Keep N identical pods running | Usually managed by a Deployment |
| **Deployment** | Stateless apps | Rolling updates, rollback, scaling |
| **StatefulSet** | Stateful apps (databases, Kafka) | Stable pod names (`db-0`, `db-1`), stable storage, ordered start and stop |
| **DaemonSet** | One pod per node (log agents, CNI, monitoring) | Runs on every matching node |
| **Job** | Run-to-completion tasks | Retries until success, with `backoffLimit` |
| **CronJob** | Scheduled Jobs | Cron syntax, concurrency policy |

## 1.3 Deployment: the core config

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
  namespace: shop
  labels: { app: orders }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app: orders }          # must match template labels (immutable)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%                       # extra pods allowed during update
      maxUnavailable: 0                   # never drop below desired count
  template:
    metadata:
      labels: { app: orders }
    spec:
      serviceAccountName: orders-sa
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels: { app: orders }
      containers:
      - name: orders
        image: myrepo/orders:1.4.2        # pin a tag or digest, never :latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef: { name: orders-config, key: db_host }
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: { name: orders-secret, key: password }
        resources:
          requests: { cpu: 250m, memory: 256Mi }
          limits:   { memory: 512Mi }
        startupProbe:                     # protects slow-starting apps
          httpGet: { path: /healthz, port: 8080 }
          failureThreshold: 30
          periodSeconds: 5
        readinessProbe:                   # controls traffic
          httpGet: { path: /ready, port: 8080 }
          periodSeconds: 5
        livenessProbe:                    # controls restarts
          httpGet: { path: /healthz, port: 8080 }
          periodSeconds: 10
          failureThreshold: 3
        lifecycle:
          preStop:
            exec: { command: ["sleep", "10"] }   # let endpoints drain before SIGTERM
```

## 1.4 Probes: the three types

| Probe | Question | On failure |
|---|---|---|
| **Startup** | Has the app finished starting? | Container restarted. Disables the other probes until it passes |
| **Readiness** | Can it serve traffic now? | Pod removed from Service endpoints (**not** restarted) |
| **Liveness** | Is it stuck or deadlocked? | Container restarted |

**Common mistake:** a liveness probe that checks downstream dependencies. If the database blips, every pod restarts at once and amplifies the outage. Liveness should check only the process itself.

## 1.5 Resources: requests vs limits and QoS

- **Requests:** what the scheduler reserves. Used for placement decisions.
- **Limits:** the hard cap enforced at runtime. Exceeding **memory** limit means **OOMKilled**. Exceeding **CPU** limit means **throttling**, not killing.

| QoS class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | requests = limits for all containers (CPU and memory) | Evicted last |
| **Burstable** | At least one request or limit set | Middle |
| **BestEffort** | Nothing set | Evicted first |

A common pattern is to set a memory request equal to the limit, and a CPU request without a CPU limit, to avoid throttling. Opinions vary, so be ready to justify your choice.

## 1.6 StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless            # headless Service gives stable DNS per pod
  replicas: 3
  podManagementPolicy: OrderedReady
  selector:
    matchLabels: { app: redis }
  template:
    metadata:
      labels: { app: redis }
    spec:
      containers:
      - name: redis
        image: redis:7
        volumeMounts:
        - { name: data, mountPath: /data }
  volumeClaimTemplates:                  # one PVC per pod, retained when pod is rescheduled
  - metadata: { name: data }
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests: { storage: 20Gi }
```

Pods get DNS names like `redis-0.redis-headless.<ns>.svc.cluster.local`. PVCs created from `volumeClaimTemplates` are **not deleted** automatically when you delete the StatefulSet (by default).

## 1.7 Job and CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid              # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 3600
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: OnFailure       # Jobs require OnFailure or Never
          containers:
          - name: report
            image: myrepo/report:2.0
```

## 1.8 Scheduling controls

| Mechanism | Purpose |
|---|---|
| **nodeSelector** | Simple label match to a node |
| **Node affinity** | Expressive node rules (required or preferred) |
| **Pod affinity / anti-affinity** | Co-locate with, or spread away from, other pods |
| **Taints and tolerations** | Nodes repel pods unless the pod tolerates the taint (dedicated or GPU nodes) |
| **Topology spread constraints** | Even distribution across zones or nodes |
| **PriorityClass** | Higher-priority pods can preempt lower ones |

```yaml
# Taint a node, then tolerate it:
# kubectl taint nodes node1 dedicated=pci:NoSchedule
tolerations:
- key: dedicated
  operator: Equal
  value: pci
  effect: NoSchedule
```

Note: a toleration only *allows* scheduling on a tainted node. To *force* it, combine it with node affinity or a nodeSelector.

## 1.9 Availability and scaling

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: orders-pdb }
spec:
  minAvailable: 2                        # or maxUnavailable: 1
  selector:
    matchLabels: { app: orders }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders-hpa }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 70 }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # avoid flapping
```

| Autoscaler | Scales | Notes |
|---|---|---|
| **HPA** | Pod replicas | Needs metrics-server. Based on requests, so requests must be set |
| **VPA** | Pod requests and limits | Don't combine with HPA on the same CPU or memory metric |
| **Cluster Autoscaler / Karpenter** | Nodes | Reacts to pending pods |
| **KEDA** | Pod replicas on external events (queue depth, Kafka lag) | Can scale to zero |

**PDBs only protect against voluntary disruptions** (drains, upgrades), not node crashes. A PDB with `minAvailable` equal to the replica count blocks node drains forever.

## 1.10 Rolling update behavior

1. A new ReplicaSet is created.
2. Pods are added to the new RS and removed from the old, within `maxSurge` and `maxUnavailable`.
3. A pod counts as available only after it passes **readiness**. This is why readiness probes matter for zero-downtime deploys.

Rollback: `kubectl rollout undo deployment/orders` (or revert in Git for GitOps).

---

# PART 2: NETWORKING

## 2.1 The Kubernetes network model (the rules)

1. Every pod gets its **own IP**.
2. Pods can talk to all other pods **without NAT**.
3. Agents on a node can talk to all pods on that node.

The **CNI plugin** implements this (AWS VPC CNI, Calico, Cilium, Flannel). On EKS, the VPC CNI gives pods real VPC IPs, so IP exhaustion is a known scaling concern (mitigated with prefix delegation or custom networking).

## 2.2 Services

A Service gives a **stable virtual IP and DNS name** to a changing set of pods, selected by labels.

| Type | Behavior | Typical use |
|---|---|---|
| **ClusterIP** (default) | Internal virtual IP | Service-to-service traffic |
| **NodePort** | Opens a port (30000-32767) on every node | Rarely used directly |
| **LoadBalancer** | Provisions a cloud LB (NLB/ALB on AWS) | External exposure |
| **ExternalName** | DNS CNAME to an external host | Alias to an outside service |
| **Headless** (`clusterIP: None`) | No virtual IP, DNS returns pod IPs | StatefulSets, client-side discovery |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders
  namespace: shop
spec:
  type: ClusterIP
  selector: { app: orders }
  ports:
  - name: http
    port: 80                 # Service port
    targetPort: 8080         # container port
```

**How it works:** the Service's selector produces **EndpointSlices** listing ready pod IPs. **kube-proxy** (iptables or IPVS mode) or an eBPF-based CNI like Cilium programs rules on each node to load-balance traffic to those endpoints. Only **Ready** pods are included.

## 2.3 DNS

CoreDNS provides names like:

- `orders.shop.svc.cluster.local` (full name)
- `orders.shop` or `orders` (short names work through the pod's search domains)
- `pod-0.headless.shop.svc.cluster.local` (StatefulSet pods)

The default `ndots:5` setting means external lookups may try several search-domain suffixes first, adding latency and DNS load. A common tuning is to use fully qualified names with a trailing dot, lower `ndots`, or run NodeLocal DNSCache.

## 2.4 Ingress and Gateway API

**Ingress** routes external HTTP(S) traffic to Services. It needs an **Ingress controller** (AWS Load Balancer Controller, NGINX, Traefik).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
  namespace: shop
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  tls:
  - hosts: [shop.example.com]
    secretName: shop-tls
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: orders
            port: { number: 80 }
```

**Gateway API** (`Gateway`, `HTTPRoute`, `GatewayClass`) is the newer, more expressive and role-oriented successor: infra teams own Gateways, app teams own Routes. Know that it exists and why it's preferred over annotation-heavy Ingress.

## 2.5 NetworkPolicy

By default, **all pods can talk to all pods**. NetworkPolicy restricts that, but it requires a **CNI that enforces it** (Calico, Cilium, or VPC CNI with network policy support enabled). Without enforcement, the objects are silently ignored.

```yaml
# Step 1: default deny all ingress and egress in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: shop
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Step 2: allow only what's needed
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders-allow
  namespace: shop
spec:
  podSelector:
    matchLabels: { app: orders }
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels: { kubernetes.io/metadata.name: web }
      podSelector:
        matchLabels: { app: frontend }       # same list item = AND
    ports:
    - { protocol: TCP, port: 8080 }
  egress:
  - to:
    - namespaceSelector:
        matchLabels: { kubernetes.io/metadata.name: kube-system }
    ports:
    - { protocol: UDP, port: 53 }            # don't forget DNS!
    - { protocol: TCP, port: 53 }
  - to:
    - podSelector:
        matchLabels: { app: postgres }
    ports:
    - { protocol: TCP, port: 5432 }
```

**Gotchas:**
- Selectors in the **same** `from` item are ANDed. Separate items are ORed.
- After default-deny egress, you must explicitly allow **DNS**, or everything breaks.
- NetworkPolicy is L3/L4 only. For L7 or identity-based rules, use a mesh (Istio) or Cilium.

## 2.6 Traffic flow summary

```
Client → Cloud LB → Ingress controller/Gateway → Service (virtual IP) → kube-proxy/eBPF → Pod
Pod → CoreDNS (name lookup) → Service IP → another Pod
Pod → NAT gateway / egress → Internet
```

---

# PART 3: STORAGE

## 3.1 Volume types

| Type | Lifetime | Use |
|---|---|---|
| **emptyDir** | Pod lifetime | Scratch space, shared data between containers |
| **hostPath** | Node | Node-level access. Risky, avoid in general workloads |
| **configMap / secret** | Object lifetime | Inject config or credentials as files |
| **projected** | Pod | Combine sources, including service account tokens |
| **PersistentVolumeClaim** | Independent of pod | Durable data |

## 3.2 PV, PVC and StorageClass

| Object | Role |
|---|---|
| **PersistentVolume (PV)** | A piece of storage in the cluster (cluster-scoped) |
| **PersistentVolumeClaim (PVC)** | A request for storage by a pod (namespaced) |
| **StorageClass** | Defines *how* to dynamically provision storage (provisioner, type, parameters) |
| **CSI driver** | Plugin that talks to the storage backend (EBS CSI, EFS CSI) |

**Dynamic provisioning flow:** pod references PVC, PVC names a StorageClass, the CSI provisioner creates the volume (e.g., an EBS disk), a PV is created and bound to the PVC, and the kubelet attaches and mounts it to the pod.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: <kms-key-arn>
volumeBindingMode: WaitForFirstConsumer   # provision in the same AZ as the pod
reclaimPolicy: Delete                      # Retain for critical data
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: orders-data
  namespace: shop
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources:
    requests: { storage: 50Gi }
---
apiVersion: v1
kind: Pod
metadata: { name: demo }
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - { name: data, mountPath: /data }
  volumes:
  - name: data
    persistentVolumeClaim: { claimName: orders-data }
```

## 3.3 Access modes

| Mode | Meaning | AWS example |
|---|---|---|
| **ReadWriteOnce (RWO)** | Read-write by one **node** | EBS |
| **ReadOnlyMany (ROX)** | Read-only by many nodes | EFS, others |
| **ReadWriteMany (RWX)** | Read-write by many nodes | EFS, FSx |
| **ReadWriteOncePod (RWOP)** | Read-write by a single **pod** | CSI drivers that support it |

## 3.4 Key behaviors interviewers love

| Topic | Detail |
|---|---|
| **volumeBindingMode** | `WaitForFirstConsumer` delays provisioning until the pod is scheduled. **EBS volumes are AZ-bound**, so `Immediate` can create a volume in a zone where the pod can't run, leaving it stuck Pending |
| **reclaimPolicy** | `Delete` removes the backend volume when the PVC is deleted. `Retain` keeps it for manual recovery |
| **Volume expansion** | Needs `allowVolumeExpansion: true`. You can grow, never shrink |
| **Snapshots** | `VolumeSnapshot` and `VolumeSnapshotClass` via the CSI snapshot controller, used for backup and cloning |
| **StatefulSet storage** | `volumeClaimTemplates` give each replica its own PVC, which stays bound to that identity |
| **Node failure with EBS** | The volume must detach from the dead node before reattaching elsewhere, which can delay recovery. Multi-attach has restrictions |
| **Backups** | PVs are not backups. Use snapshots plus a tool like Velero or AWS Backup, and test restores |
| **Ephemeral storage** | Container writable layers and emptyDir count toward `ephemeral-storage`, and exceeding limits can evict the pod |

## 3.5 ConfigMap and Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: orders-config, namespace: shop }
data:
  db_host: postgres.shop.svc
  app.properties: |
    log.level=info
---
apiVersion: v1
kind: Secret
metadata: { name: orders-secret, namespace: shop }
type: Opaque
stringData:
  password: change-me      # in practice, never commit this; use External Secrets / Sealed Secrets
```

- Mounted as **volumes**, ConfigMap and Secret updates propagate to running pods eventually. As **environment variables**, they do **not** update until the pod restarts.
- Secrets are **base64-encoded, not encrypted** by default. Enable envelope encryption of etcd with KMS, restrict RBAC on secrets, and prefer external secret stores (AWS Secrets Manager via External Secrets Operator or the Secrets Store CSI driver).
- Using `subPath` mounts means updates are not propagated.

---

# PART 4: SECURITY

## 4.1 The request path: authentication, authorization, admission

```
kubectl / client → API server
   1. Authentication  (who are you?)      certs, OIDC, tokens, IAM (EKS)
   2. Authorization   (are you allowed?)  RBAC
   3. Admission       (is the object valid/allowed?)  mutating → validating (PSA, Kyverno/OPA)
   4. Persist to etcd
```

## 4.2 RBAC

Four objects: **Role** (namespaced permissions), **ClusterRole** (cluster-wide permissions), and the bindings that attach them to subjects (**RoleBinding**, **ClusterRoleBinding**). Permissions are **additive only**, with no deny rules.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-reader, namespace: shop }
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: read-pods, namespace: shop }
subjects:
- kind: Group
  name: shop-developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Test with `kubectl auth can-i list pods -n shop --as=jane`.

**Best practices:** least privilege, avoid wildcards and `cluster-admin`, bind to groups (from your IdP) not individuals, be careful granting `create pods`, `secrets` access, `escalate`, `bind`, or `impersonate`, since these allow privilege escalation.

On **EKS**, IAM identities map into Kubernetes through **access entries** (or the older `aws-auth` ConfigMap), and Kubernetes RBAC then decides what they can do.

## 4.3 ServiceAccounts and workload identity

Pods authenticate to the API server (and to AWS) using a **ServiceAccount**. Modern clusters use **short-lived, projected, audience-bound tokens**, not the old long-lived secret tokens.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-sa
  namespace: shop
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/orders-irsa   # IRSA
automountServiceAccountToken: false      # disable unless the pod actually calls the API
```

On EKS, **IRSA** or **EKS Pod Identity** gives each ServiceAccount its own IAM role, so pods get least-privilege AWS access instead of inheriting the node role. The default ServiceAccount should have no meaningful permissions.

## 4.4 Pod security: securityContext

```yaml
spec:
  securityContext:                       # pod level
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
    seccompProfile: { type: RuntimeDefault }
  containers:
  - name: orders
    image: myrepo/orders:1.4.2
    securityContext:                     # container level
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

| Setting | Why |
|---|---|
| `runAsNonRoot` | Limits damage from a container breakout |
| `allowPrivilegeEscalation: false` | Blocks setuid-style escalation |
| `readOnlyRootFilesystem` | Prevents tampering and malware drops (mount emptyDir where writes are needed) |
| `drop: ALL` capabilities | Removes Linux capabilities not required |
| `seccompProfile` | Restricts system calls |
| Avoid `privileged: true`, `hostNetwork`, `hostPID`, `hostPath` | Each weakens isolation sharply |

## 4.5 Pod Security Admission (PSA)

PodSecurityPolicy was **removed in v1.25**. It was replaced by built-in **Pod Security Admission**, which enforces three standards per namespace through labels:

| Level | Meaning |
|---|---|
| **privileged** | Unrestricted |
| **baseline** | Blocks known privilege escalations (no privileged containers, hostPath, etc.) |
| **restricted** | Hardened best practice (non-root, drop capabilities, seccomp) |

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shop
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

For finer policy (image registries, required labels, resource limits, signed images), use **Kyverno**, **OPA Gatekeeper**, or built-in **ValidatingAdmissionPolicy** (CEL-based).

```yaml
# Kyverno example: disallow :latest tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: disallow-latest-tag }
spec:
  validationFailureAction: Enforce
  rules:
  - name: no-latest
    match:
      any:
      - resources: { kinds: [Pod] }
    validate:
      message: "Image tag ':latest' is not allowed."
      pattern:
        spec:
          containers:
          - image: "!*:latest"
```

## 4.6 Defense in depth: the layers

| Layer | Controls |
|---|---|
| **Cluster / API** | Private API endpoint, audit logging, RBAC, OIDC SSO, regular upgrades |
| **etcd / secrets** | KMS envelope encryption, restricted access to secrets, external secret store |
| **Network** | NetworkPolicy default-deny, mesh mTLS, egress control, private subnets |
| **Workload** | securityContext, PSA restricted, resource limits, non-root, read-only FS |
| **Supply chain** | Trusted registries, image scanning (ECR, Trivy), image signing (cosign) with admission verification, SBOMs, pinned digests |
| **Runtime** | Falco or GuardDuty runtime monitoring, seccomp/AppArmor, node hardening |
| **Nodes** | Minimal OS (Bottlerocket), IMDSv2 with hop limit 1 to protect node credentials, patching |
| **Observability** | Audit logs, control plane logs to CloudWatch, alerts on exec into pods or RBAC changes |

---

# PART 5: TROUBLESHOOTING CHEAT SHEET

| Symptom | Likely cause | First commands |
|---|---|---|
| **Pending** | Insufficient resources, taints, unbound PVC, affinity unsatisfiable | `kubectl describe pod` (Events section) |
| **ImagePullBackOff** | Wrong image or tag, missing pull secret, registry auth or network | `kubectl describe pod` |
| **CrashLoopBackOff** | App crash, bad config, failing liveness probe, missing secret | `kubectl logs --previous`, `describe pod` |
| **OOMKilled** | Memory limit too low or a leak | `kubectl describe pod` (Last State, exit code 137) |
| **Running but not Ready** | Readiness probe failing | `describe pod`, test the probe endpoint |
| **Service unreachable** | Selector mismatch, no ready endpoints, wrong targetPort, NetworkPolicy | `kubectl get endpointslices`, `kubectl get svc -o wide` |
| **DNS failures** | CoreDNS down, NetworkPolicy blocking port 53, ndots issues | `kubectl exec ... -- nslookup`, check CoreDNS pods and logs |
| **Node NotReady** | Kubelet, disk or memory pressure, network, CNI problem | `kubectl describe node`, node logs |
| **PVC Pending** | No matching StorageClass, AZ mismatch, CSI driver or IAM issue | `kubectl describe pvc`, CSI controller logs |
| **Evicted** | Node pressure (memory, disk, ephemeral storage) | `kubectl describe pod`, node conditions |
| **Forbidden errors** | RBAC missing permissions | `kubectl auth can-i`, review Role bindings |

```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -c <container> --previous
kubectl exec -it <pod> -- sh
kubectl debug -it <pod> --image=busybox --target=<container>   # ephemeral debug container
kubectl top pod / kubectl top node
kubectl get endpointslices -n <ns>
kubectl rollout status|history|undo deployment/<name>
```

---

# PART 6: INTERVIEW QUESTIONS AND MODEL ANSWERS

## Workloads

**Q1. Deployment vs StatefulSet vs DaemonSet?**
Deployment: interchangeable stateless pods with rolling updates. StatefulSet: stable identity, ordered operations, and per-pod persistent storage for stateful systems. DaemonSet: exactly one pod per node (or matching nodes) for node-level agents.

**Q2. Liveness vs readiness vs startup probe?**
Liveness restarts a stuck container, readiness gates traffic by adding or removing the pod from Service endpoints, and startup gives slow apps time to boot before the others apply. A liveness probe should never depend on external dependencies.

**Q3. Requests vs limits? What happens when exceeded?**
Requests drive scheduling, limits cap usage. Over the memory limit the container is OOMKilled. Over the CPU limit it's throttled.

**Q4. Explain QoS classes.**
Guaranteed, Burstable, BestEffort, determined by requests and limits. They set eviction order under node pressure.

**Q5. How does a rolling update achieve zero downtime?**
`maxUnavailable: 0` with `maxSurge`, readiness probes gating traffic, `preStop` sleep and graceful shutdown handling SIGTERM, and a PDB to protect against concurrent disruptions.

**Q6. What does a PodDisruptionBudget protect against?**
Voluntary disruptions like drains and upgrades, not involuntary ones like node crashes. Too strict a PDB can block node upgrades.

**Q7. A pod is Pending. How do you debug?**
`kubectl describe pod` and read the Events. Check resource requests against node capacity, taints and tolerations, nodeSelector and affinity, PVC binding, quota limits, and Cluster Autoscaler or Karpenter activity.

**Q8. CrashLoopBackOff: how do you troubleshoot?**
Check `logs --previous` and `describe pod` for the exit code (137 means OOM or SIGKILL, 1 is an app error), validate config and secrets, check whether the liveness probe is too aggressive, and use an ephemeral debug container if needed.

**Q9. HPA isn't scaling. Why?**
metrics-server missing or unhealthy, no CPU requests set (utilization is computed against requests), `maxReplicas` reached, no capacity to schedule new pods, a custom metric unavailable, or stabilization windows delaying the decision.

**Q10. How do taints and tolerations differ from node affinity?**
Taints repel pods from nodes unless tolerated, with the node as the decision-maker. Affinity attracts pods to nodes, with the pod as the decision-maker. Dedicated nodes typically use both.

## Networking

**Q11. How does a Service route traffic?**
The selector builds EndpointSlices of ready pods, and kube-proxy (iptables or IPVS) or an eBPF dataplane load-balances from the virtual IP to a pod IP.

**Q12. ClusterIP vs NodePort vs LoadBalancer vs Headless?**
Internal virtual IP; a port on every node; a cloud load balancer; no virtual IP, DNS returns pod IPs directly (for StatefulSets and client-side balancing).

**Q13. Service has no endpoints. What do you check?**
Selector and pod labels match, pods are Ready, port names and numbers match `targetPort`, and the pods are in the same namespace.

**Q14. How does DNS work in Kubernetes?**
CoreDNS resolves service and pod names. Pods get a `resolv.conf` with search domains and `ndots:5`. Slow external lookups often come from the search-domain expansion.

**Q15. Ingress vs Service type LoadBalancer vs Gateway API?**
LoadBalancer creates one L4 LB per Service. Ingress consolidates L7 routing for many Services behind one entry point via a controller. Gateway API is the more expressive, role-oriented successor.

**Q16. How do NetworkPolicies work, and what do they require?**
They're allow-list rules selected by labels, additive, and enforced only if the CNI supports them. A default-deny policy plus explicit allows (including DNS egress) is the standard pattern.

**Q17. Pod-to-pod communication across nodes on EKS?**
The VPC CNI assigns VPC IP addresses to pods via ENIs, so pod traffic routes natively through the VPC with no overlay. This means IP planning matters.

**Q18. kube-proxy iptables vs IPVS vs eBPF?**
iptables rules scale poorly with very large service counts. IPVS uses hash tables for better scalability. eBPF (Cilium) can replace kube-proxy with lower overhead and richer observability.

## Storage

**Q19. PV vs PVC vs StorageClass?**
PV is the actual storage resource, PVC is a pod's request for it, and StorageClass defines how to provision it dynamically.

**Q20. Why use `WaitForFirstConsumer`?**
Zonal volumes like EBS must be created in the AZ where the pod runs. Delayed binding lets the scheduler choose the node first, avoiding unschedulable pods.

**Q21. RWO vs RWX, and when do you pick EFS over EBS?**
EBS is RWO, single-AZ, and high performance for databases. EFS is RWX, multi-AZ, and shared, for content or shared state, with different latency and cost trade-offs.

**Q22. What happens to the data when a PVC is deleted?**
It depends on `reclaimPolicy`: Delete removes the underlying volume, Retain keeps it for manual cleanup. For critical data, use Retain plus snapshots.

**Q23. How do StatefulSets handle storage?**
`volumeClaimTemplates` create a dedicated PVC per replica, and the pod keeps re-attaching to its own volume after rescheduling.

**Q24. How do you back up persistent data?**
CSI VolumeSnapshots, Velero for cluster resources and volumes, AWS Backup, and application-level backups for databases. Regularly test restores.

**Q25. A pod can't mount its volume after a node failure. Why?**
The EBS volume may still be attached to the failed node, or the pod landed in a different AZ than the volume. Check attach and detach events, the volume's AZ, and the CSI controller.

## Security

**Q26. Explain the API server request flow.**
Authentication, then authorization (RBAC), then admission (mutating, then validating), then persistence in etcd.

**Q27. Role vs ClusterRole, RoleBinding vs ClusterRoleBinding?**
Namespaced versus cluster-wide permissions, and the bindings attach them to subjects. A RoleBinding can reference a ClusterRole to reuse a permission set within one namespace.

**Q28. Are Kubernetes Secrets secure?**
By default they are only base64-encoded in etcd. Harden them with KMS envelope encryption, tight RBAC, an external secret manager, audit logging, and avoiding secrets in env vars where possible.

**Q29. What replaced PodSecurityPolicy?**
Pod Security Admission, with privileged, baseline, and restricted levels applied through namespace labels, plus policy engines such as Kyverno, OPA Gatekeeper, and ValidatingAdmissionPolicy for finer rules.

**Q30. How do you give a pod access to AWS services securely?**
IRSA or EKS Pod Identity, with a dedicated least-privilege IAM role per ServiceAccount, instead of using the node's instance role. Also enforce IMDSv2 to limit credential theft.

**Q31. How do you secure the supply chain?**
Trusted base images, vulnerability scanning in CI and ECR, image signing and admission verification, pinned digests, SBOMs, and a restricted registry allow-list.

**Q32. How would you harden a cluster for PCI?**
Private API endpoint, dedicated CDE cluster or node groups, default-deny NetworkPolicies, mesh mTLS, PSA restricted, KMS-encrypted secrets, audit logs retained 12 months, image scanning and signing, MFA-backed SSO, and automated compliance checks.

**Q33. What are the risks of `privileged: true` or mounting the Docker/containerd socket?**
Near-complete host access, so a container compromise becomes a node compromise and potentially a cluster compromise.

## Scenario and design questions

**Q34. Your application is returning 502/503 intermittently during deployments. What do you investigate?**
Readiness probe accuracy, `maxUnavailable` settings, missing `preStop` or graceful shutdown (connections dropped when the pod terminates), load balancer deregistration delay versus `terminationGracePeriodSeconds`, and PDBs.

**Q35. Walk me through what happens when you run `kubectl apply -f deployment.yaml`.**
kubectl authenticates and sends the request, the API server authorizes it, runs admission, and persists to etcd. The Deployment controller creates a ReplicaSet, which creates Pods. The scheduler binds each Pod to a node, the kubelet pulls images through the container runtime, the CNI sets up networking, and probes decide when it's Ready and added to Service endpoints.

**Q36. How would you design a multi-tenant cluster?**
Namespaces per tenant, ResourceQuotas and LimitRanges, RBAC per team, NetworkPolicy isolation, PSA, separate node pools for sensitive tenants, and policy engines. For stronger isolation, separate clusters or accounts.

**Q37. How do you upgrade a production cluster safely?**
Scan for deprecated APIs, review release notes and add-on compatibility, upgrade non-prod first, upgrade the control plane, then add-ons, then nodes with PDB-aware rolling replacement (or blue/green node groups), watch SLO dashboards throughout, and keep a rollback path.

**Q38. How do you handle a node under memory pressure?**
Check `kubectl describe node` conditions and eviction events, identify top consumers, verify requests and limits reflect reality, and look for BestEffort pods. Longer-term, right-size resources and set memory limits to prevent noisy neighbors.

**Q39. How would you reduce Kubernetes costs?**
Right-size requests from actual usage (VPA recommendations), use HPA and Karpenter for consolidation, spot instances for fault-tolerant workloads, bin-packing, and remove idle resources. Add cost allocation through labels.

**Q40. Pods are fine but users can't reach the app. How do you trace it?**
Work outward-in: DNS record, load balancer health, Ingress or Gateway rules, Service selector and endpoints, NetworkPolicy, pod readiness, and application logs. Use `curl` from a debug pod at each hop.

---

# PART 7: QUICK CHEAT SHEET

| Area | Remember |
|---|---|
| **Workloads** | Deployment (stateless), StatefulSet (stable ID and storage), DaemonSet (per node), Job/CronJob (batch). Probes and resource settings drive reliability |
| **Scheduling** | Requests for placement, taints/tolerations, affinity, topology spread, PDBs |
| **Networking** | Flat pod network via CNI, Service gives a stable VIP, EndpointSlices track ready pods, CoreDNS for names, NetworkPolicy needs an enforcing CNI |
| **Storage** | PVC to StorageClass to CSI to PV. `WaitForFirstConsumer` for zonal disks, reclaim policy, RWO vs RWX, snapshots are not a full backup strategy |
| **Security** | AuthN, then RBAC, then admission. ServiceAccounts with IRSA/Pod Identity, PSA restricted, securityContext, default-deny network, KMS-encrypted secrets, signed images |
| **Debugging** | `describe`, `logs --previous`, events, endpoints, `kubectl debug`, `auth can-i` |

**Interview tip:** for Kubernetes questions, structure answers as *what it is, how it works internally, a failure mode you've seen, and how you'd debug or harden it*. For a Staff-level role, add a trade-off: for example, "I'd avoid CPU limits for latency-sensitive services because of throttling, but I'd enforce namespace quotas to keep multi-tenant clusters safe."

If you'd like, I can build a hands-on lab set (broken-pod scenarios to debug), a mock Kubernetes interview with follow-up probing, or a deep dive on one area such as EKS networking and the VPC CNI.
