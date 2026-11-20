# **Kubernetes Expert-Level Interview Questions and Answers**

Lead DevOps / Cloud Architect / SRE Level

Docker + Kubernetes + ECS + EKS

The interview statement:

> **Strong understanding of containerization and orchestration (Docker, Kubernetes, ECS/EKS).**

This requirement goes beyond knowing Docker commands or Kubernetes YAML files. At Lead DevOps Engineer or Cloud Architect level, interviewers expect you to demonstrate:

1. Deep understanding of container architecture and runtime.

2. Docker image creation, optimization, security and troubleshooting.

3. Kubernetes architecture and internal components.

4. Advanced workload orchestration and scheduling.

5. Kubernetes networking, storage and security.

6. Production-grade EKS architecture and operations.

7. AWS ECS vs EKS architectural decisions.

8. CI/CD automation and GitOps.

9. Kubernetes monitoring, logging and troubleshooting.

10. High availability, scalability, disaster recovery and cost optimization.

I'll structure this as an interview preparation guide with practical answers, architecture concepts, commands and real-world scenarios.

# **Section 1: Docker and Containerization – Expert-Level Questions**

## **1. What is containerization? How is it different from virtualization?**

**Answer:**

Containerization is an OS-level virtualization technology that allows multiple isolated applications to run on the same operating system kernel.

Each container contains:

* Application code.

* Runtime dependencies.

* Libraries.

* Configuration.

* Required binaries.

Containers share the host operating system kernel.

**Architecture comparison:**

**Virtual Machines**

![Deploying VM And Container In Azure](https://images.openai.com/static-rsc-4/cKWp7J4tqd5q_XaeEyeK_BEpez4PSNtu1Bdp5TrvpYeZ4bROPsRH4N00Mff7PfkNbukboW33U9lvepN-0wkdqGXsxTTZMVBrDOvdTrasV0LQ4Q3H7F9_1nfmCyzM7u6IFDn2aaezxzrjvipGFzivfnPXT2Wo252uI1NNjX-ezIo?purpose=inline)

Add to Favorites

Each VM runs its own guest OS and kernel.

**Containers**

![Dockerize YOLO11: Streamline Model Deployment | Ultralytics](https://images.openai.com/static-rsc-4/OURwMeWPgzn4GmWt0t6eMqaVV5NuIOzZlI33pFc4DxluIEAAuNxSe5bn2hKGsV4YMdBiVXlziAfTH_HfkthSSiQSWxQNptROyv7xc6pWoE5YwsHO9lmCV_Amo9RVV0MDdvlkSqkCrNNExfmzMVLjQotCADJoNncJoSUgYfjwFb4?purpose=inline)

Add to Favorites

Containers share the host kernel.

| Parameter      | Virtual Machine            | Container                         |
| -------------- | -------------------------- | --------------------------------- |
| Virtualization | Hardware level             | OS level                          |
| OS             | Separate guest OS          | Shared host kernel                |
| Startup        | Usually minutes or seconds | Usually seconds                   |
| Resource usage | Higher                     | Lower                             |
| Isolation      | Stronger hardware boundary | Kernel-level isolation            |
| Portability    | Good                       | Excellent with compatible runtime |

**Interview follow-up:** How do containers achieve isolation?

Containers use:

* Linux namespaces – process, network, mount, PID, IPC and user isolation.

* Cgroups – CPU, memory and resource limits.

* Linux capabilities – control privileged operations.

* Seccomp – restrict system calls.

* AppArmor/SELinux – mandatory access control.

## **2. Explain Docker architecture.**

**Answer:**

Docker follows a client-server architecture.

![Day 43 —From chroot to clusters: the origin story of containers, Docker, and the Kubernetes era | by Alok Rahul | Feb, 2026 | Medium](https://images.openai.com/static-rsc-4/W5L7vORVGQT_hHx4iPlmZawHsHHhVdx4gCyTxPzDlW9IviyFBjKJOYfol7LqEXXmnzV1dLf-_zBTv1BE2DFrTdyyLDdZYoWYDDpB_e3PHX0UpxOG2PTqJzTXrLAUbobBZ3--ghfHA24lkdGbsS5_E3cZRUsk3dXte-YVMxooQ_E?purpose=inline)

Add to Favorites

Major components:

| Component        | Responsibility                                |
| ---------------- | --------------------------------------------- |
| Docker CLI       | Executes Docker commands                      |
| Docker Daemon    | Manages images, containers and networks       |
| containerd       | Container lifecycle management                |
| runc             | Creates and runs containers using OCI runtime |
| Docker Registry  | Stores Docker images                          |
| Docker Image     | Read-only application template                |
| Docker Container | Running instance of an image                  |

Example:

Bash

```
docker build -t myapp:v1 .
docker run -d -p 8080:80 myapp:v1
docker ps
docker logs <container-id>
docker exec -it <container-id> sh
```

**Expert-level follow-up:** Does Kubernetes require Docker Engine?

No. Kubernetes uses CRI-compatible container runtimes. Modern Kubernetes clusters commonly use containerd or CRI-O. Docker Engine is not required, although Docker-built images can still run on Kubernetes because they use compatible OCI image formats.

## **3. What is the difference between Docker Image and Container?**

**Answer:**

* Image: Immutable, read-only template.

* Container: Running or stopped instance created from an image.

Example:

Bash

```
docker pull nginx:latest

docker run -d \
  --name nginx-container \
  -p 8080:80 nginx:latest
```

Here, `nginx:latest` is an image and `nginx-container` is the container.

Multiple containers can be created from the same image.

## **4. Explain Docker image layers and Dockerfile optimization.**

**Answer:**

Docker images use a layered filesystem. Each instruction that changes the filesystem can create an image layer.

Example:

dockerfile

```
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080

CMD ["python", "app.py"]
```

Optimization techniques:

1. Use lightweight base images.

2. Use multi-stage builds.

3. Minimize unnecessary layers.

4. Copy dependency manifests before application source to improve cache reuse.

5. Use `.dockerignore`.

6. Avoid installing unnecessary packages.

7. Run applications as non-root.

8. Pin base image versions and preferably digests.

**Multi-stage Docker build:**

dockerfile

```
FROM golang:1.24 AS builder

WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o app .

FROM gcr.io/distroless/static-debian12

COPY --from=builder /src/app /app

USER 65532:65532

ENTRYPOINT ["/app"]
```

The final image contains the compiled application rather than the complete build environment.

## **5. What is the difference between CMD and ENTRYPOINT?**

| CMD                                         | ENTRYPOINT                          |
| ------------------------------------------- | ----------------------------------- |
| Provides default command or arguments       | Defines the executable              |
| Easily overridden by command arguments      | Usually remains the main executable |
| Can provide default arguments to ENTRYPOINT | Can receive CMD arguments           |

Example:

dockerfile

```
FROM ubuntu

ENTRYPOINT ["sleep"]

CMD ["100"]
```

Execution:

Bash

```
docker run myimage
```

Runs:

Bash

```
sleep 100
```

Override:

Bash

```
docker run myimage 200
```

Runs:

Bash

```
sleep 200
```

## **6. What is the difference between Docker volumes, bind mounts and tmpfs?**

| Feature     | Volume           | Bind Mount         | tmpfs                    |
| ----------- | ---------------- | ------------------ | ------------------------ |
| Storage     | Docker-managed   | Host directory     | Memory                   |
| Persistence | Yes              | Yes                | No                       |
| Common use  | Application data | Configuration/code | Temporary sensitive data |
| Portability | Better           | Host-dependent     | Runtime-dependent        |

Example:

Bash

```
docker volume create app-data

docker run -d \
  -v app-data:/var/lib/mysql \
  mysql:8
```

## **7. How do you secure Docker containers?**

**Answer:**

I would follow a defense-in-depth approach:

* Use trusted and minimal base images.

* Scan images for vulnerabilities.

* Avoid running containers as root.

* Use read-only root filesystem where possible.

* Drop unnecessary Linux capabilities.

* Avoid privileged containers.

* Store secrets outside the image.

* Use resource limits.

* Sign and verify container images.

* Apply image provenance and SBOM practices.

Example:

Bash

```
docker run -d \
  --read-only \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --memory=512m \
  --cpus=1 \
  nginx
```

## **8. What happens when a Docker container exits unexpectedly?**

**Answer:**

I would troubleshoot in this order:

Bash

```
docker ps -a

docker logs <container-id>

docker inspect <container-id>

docker events
```

Common reasons:

* Application process exits.

* Incorrect CMD or ENTRYPOINT.

* Out-of-memory termination.

* Missing environment variables.

* Application configuration failure.

* Permission issues.

* Dependency connectivity problems.

Check OOM-related termination:

Bash

```
docker inspect <container-id>
```

Look at `State.OOMKilled` and `State.ExitCode`.

# **Section 2: Kubernetes Architecture – Expert-Level Questions**

## **9. Explain Kubernetes architecture in detail.**

**Answer:**

Kubernetes is a container orchestration platform responsible for deploying, scaling, managing and maintaining containerized applications.

It has two major components:

1. Control Plane.

2. Worker Nodes.

![Medium](https://images.openai.com/static-rsc-4/wXnoP9BLBDSYsfDdMIyZxLYfpmG5S1DPiQlyr3zY36fwS1MUQmcaUSwtrgACJVmAv9Dn3tnbY3JD_7C21nsXvtHmgg_Iamc__2uZ9Typufm-x2KwxCAfgEWqXVrwAqCAkVPj1mX-21W9F4e3X-sjv5sMV-NdcgXWCy4G-8x_J1A?purpose=inline)

Add to Favorites

**Control Plane components**

| Component                | Responsibility                                          |
| ------------------------ | ------------------------------------------------------- |
| API Server               | Entry point for Kubernetes API requests                 |
| etcd                     | Distributed key-value store for cluster state           |
| Scheduler                | Assigns Pods to appropriate nodes                       |
| Controller Manager       | Runs controllers to reconcile desired and actual states |
| Cloud Controller Manager | Integrates Kubernetes with cloud provider APIs          |

**Worker Node components**

| Component         | Responsibility                                 |
| ----------------- | ---------------------------------------------- |
| Kubelet           | Ensures Pods are running as specified          |
| Kube-proxy        | Implements Service networking rules where used |
| Container Runtime | Runs containers                                |
| CNI Plugin        | Provides Pod networking                        |
| CSI Driver        | Provides persistent storage integration        |

**Interview follow-up:** What happens when you execute `kubectl apply -f deployment.yaml`?

1. kubectl sends an API request to API Server.

2. API Server authenticates and authorizes the request.

3. Admission controllers validate or mutate the object.

4. The desired state is persisted in etcd.

5. Deployment Controller observes the new Deployment.

6. ReplicaSet Controller creates the required ReplicaSet and Pods.

7. Scheduler assigns Pods to worker nodes.

8. Kubelet receives Pod specifications.

9. Container runtime pulls images and starts containers.

10. CNI configures networking.

11. Readiness probes determine whether Pods can receive traffic.

## **10. Explain Kubernetes API Server and etcd.**

**Answer:**

API Server is the central communication component of Kubernetes.

All cluster operations pass through the API Server.

etcd stores Kubernetes cluster state, including:

* Deployments.

* Pods.

* Services.

* ConfigMaps.

* Secrets.

* Cluster configuration and other API objects.

**Important production considerations:**

* Enable etcd backups.

* Secure etcd communication using TLS.

* Restrict access.

* Monitor etcd latency and disk performance.

* Protect encryption keys.

* Test restoration procedures.

**Scenario:** What happens if etcd becomes unavailable?

Existing containers may continue running, but Kubernetes cannot reliably persist new changes or reconcile cluster state through the API Server. Control-plane operations can become impaired.

## **11. What is the difference between Deployment, ReplicaSet, StatefulSet and DaemonSet?**

| Resource    | Purpose                                 | Example                    |
| ----------- | --------------------------------------- | -------------------------- |
| Deployment  | Stateless applications                  | Web/API services           |
| ReplicaSet  | Maintains Pod replica count             | Managed by Deployment      |
| StatefulSet | Stateful workloads with stable identity | Kafka, databases           |
| DaemonSet   | One Pod per eligible node               | Logging, monitoring agents |
| Job         | Runs finite tasks                       | Database migration         |
| CronJob     | Runs scheduled tasks                    | Backup jobs                |

Example Deployment:

YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

## **12. Explain Kubernetes Pod lifecycle.**

**Answer:**

A Pod can have the following phases:

* Pending.

* Running.

* Succeeded.

* Failed.

* Unknown.

Container states include:

* Waiting.

* Running.

* Terminated.

Important container restart reasons include:

* Error.

* OOMKilled.

* CrashLoopBackOff.

* ImagePullBackOff.

* CreateContainerConfigError.

Useful troubleshooting:

Bash

```
kubectl get pods -A

kubectl describe pod <pod-name> -n <namespace>

kubectl logs <pod-name> -n <namespace>

kubectl logs <pod-name> --previous -n <namespace>

kubectl get events -n <namespace> \
  --sort-by=.metadata.creationTimestamp
```

## **13. What is the difference between Liveness, Readiness and Startup probes?**

**Answer:**

| Probe     | Purpose                                              | Failure behavior                       |
| --------- | ---------------------------------------------------- | -------------------------------------- |
| Liveness  | Determines whether container should be restarted     | Restarts container                     |
| Readiness | Determines whether Pod should receive traffic        | Removes it from ready endpoints        |
| Startup   | Allows slow-starting applications time to initialize | Prevents other probes until successful |

Example:

YAML

```
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

**Expert scenario:** Your application is restarting repeatedly because its database takes 90 seconds to initialize. What would you do?

Use a startup probe with sufficient time for initialization, and configure readiness and liveness probes appropriately. Avoid using liveness probes to check external dependencies such as databases because transient database failures could unnecessarily restart every application replica.

## **14. Explain Kubernetes requests and limits.**

**Answer:**

Requests are the resources Kubernetes uses primarily for scheduling.

Limits define the maximum resources a container can consume.

YAML

```
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

| Resource | Request behavior       | Limit behavior            |
| -------- | ---------------------- | ------------------------- |
| CPU      | Scheduling reservation | CPU throttling            |
| Memory   | Scheduling reservation | OOM termination may occur |

**What are QoS classes?**

1. Guaranteed.

2. Burstable.

3. BestEffort.

Guaranteed Pods have equal CPU and memory requests and limits for every container in the Pod.

BestEffort Pods have neither requests nor limits.

## **15. Explain Kubernetes scheduling.**

**Answer:**

The Kubernetes Scheduler selects a suitable worker node for an unscheduled Pod.

Major scheduling considerations:

* Resource requests.

* Node selectors.

* Node affinity.

* Pod affinity.

* Pod anti-affinity.

* Taints and tolerations.

* Topology spread constraints.

* Volume topology and availability.

Example node selector:

YAML

```
spec:
  nodeSelector:
    workload: production
```

Example toleration:

YAML

```
tolerations:
  - key: dedicated
    operator: Equal
    value: production
    effect: NoSchedule
```

**Expert scenario:** You have three replicas of a critical application and want them distributed across three availability zones.

Use topology spread constraints, with suitable node labels and zone capacity:

YAML

```
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: payment
```

# **Section 3: Kubernetes Networking – Advanced Questions**

## **16. Explain Kubernetes networking architecture.**

**Answer:**

Kubernetes networking provides communication between:

1. Pod to Pod.

2. Pod to Service.

3. Pod to external services.

4. External users to Kubernetes applications.

5. Communication across different worker nodes.

![How to remember Kubernetes Networking in 4 layers | Neel Shah posted on the topic | LinkedIn](https://images.openai.com/static-rsc-4/nCd1A7ARpZsPpixJGyltk3MIu1PLEEIIuGwHpP1qZrI6zg592d9lvvtu0UQsmJngrzjLs0XDIqnoR1BWxumH6ut-fPWmaWqkz5CaP9PcHSGB4Fdkcr7c4-0H4C7y35vClUQ9h6dkt5dMpF5SwTwGE41IbFXgmJi3nNHJM6ZPo4w?purpose=inline)

Add to Favorites

Major components:

| Component          | Purpose                                             |
| ------------------ | --------------------------------------------------- |
| CNI                | Configures Pod networking                           |
| CoreDNS            | Provides cluster DNS                                |
| Service            | Provides stable virtual access to Pods              |
| kube-proxy         | Implements Service forwarding rules, where deployed |
| Ingress Controller | Manages HTTP/HTTPS external access                  |
| NetworkPolicy      | Controls Pod network traffic                        |
| LoadBalancer       | Exposes services externally                         |

Popular CNI plugins:

* AWS VPC CNI.

* Calico.

* Cilium.

* Flannel.

* Weave Net (legacy/deprecated in many environments).

**Expert follow-up:** How does Pod-to-Pod communication work across worker nodes?

The CNI plugin allocates and configures Pod networking. Depending on the implementation, it can use native routing, overlays or other datapath mechanisms. In AWS EKS, the VPC CNI commonly assigns VPC IP addresses to Pods, allowing direct VPC-level routing.

## **17. What is the difference between ClusterIP, NodePort and LoadBalancer?**

**Answer:**

| Service      | Purpose                                    | Accessibility                            |
| ------------ | ------------------------------------------ | ---------------------------------------- |
| ClusterIP    | Internal application communication         | Inside cluster                           |
| NodePort     | Exposes service on node ports              | Node IP and port                         |
| LoadBalancer | Creates external load balancer integration | External/internal based on configuration |
| ExternalName | Maps a service to external DNS             | DNS alias                                |

Example:

YAML

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Check service:

Bash

```
kubectl get svc -A
kubectl describe svc nginx-service
```

## **18. What is the difference between Ingress and Gateway API?**

**Answer:**

Ingress is a Kubernetes API resource for managing HTTP and HTTPS routing into a cluster.

Gateway API is a newer, more expressive API model for managing traffic.

| Ingress                                             | Gateway API                                         |
| --------------------------------------------------- | --------------------------------------------------- |
| Primarily HTTP/HTTPS                                | HTTP, HTTPS, TCP, TLS and other supported protocols |
| Simpler routing                                     | Advanced routing capabilities                       |
| Usually relies on annotations for advanced features | More structured routing resources                   |
| Ingress + Ingress Controller                        | GatewayClass, Gateway, HTTPRoute, etc.              |

Example architecture:

![Kubernetes Gateway API: A Practical Shift from Ingress to Modern Traffic Management | by Bhargav Arisetty | Apr, 2026 | Medium](https://images.openai.com/static-rsc-4/IeZ4v0BjQzqMD9S4aWx9E1gCYNcVC2-TOzwEk9xj4HodN3lpJ6qjm2hyL684k-UKIKldRssX0qohWa4vyT33GF6Xt7dALgGG97D6aJi2whiQ84sJ5KtApm0PY-cvhHFtXeUBL-bqetpRbFGnqMp68l2wpayxIstqHj7CuHUmfNU?purpose=inline)

Add to Favorites

For AWS EKS, AWS Load Balancer Controller can provision ALBs for supported Ingress resources. Gateway API support depends on the selected controller and its version.

## **19. Explain CoreDNS in Kubernetes.**

**Answer:**

CoreDNS provides DNS resolution within Kubernetes.

For example, an application can access another application using:

```
nginx-service.default.svc.cluster.local
```

DNS format:

```
<service-name>.<namespace>.svc.cluster.local
```

Useful commands:

Bash

```
kubectl get pods -n kube-system

kubectl get configmap coredns -n kube-system -o yaml

kubectl logs -n kube-system \
  -l k8s-app=kube-dns
```

**Scenario:** An application cannot connect to another service using its DNS name.

Troubleshooting steps:

1. Check the target Service.

2. Verify Service selectors.

3. Check EndpointSlices.

4. Check CoreDNS Pods.

5. Check Pod DNS configuration.

6. Test DNS using `nslookup` or `dig`.

7. Check NetworkPolicies.

8. Check CNI networking.

## **20. Explain Kubernetes NetworkPolicy.**

**Answer:**

NetworkPolicy controls ingress and egress traffic for selected Pods.

Example: Allow only frontend Pods to communicate with backend Pods.

YAML

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Important considerations:

* NetworkPolicy is namespace-scoped.

* Policies are additive.

* It requires a network plugin that implements NetworkPolicy.

* Standard NetworkPolicy does not provide complete L7 application-aware filtering.

## **21. How do you troubleshoot Kubernetes networking issues?**

**Answer:**

I follow a layered troubleshooting approach.

Bash

```
# 1. Check Pods
kubectl get pods -o wide -n production

# 2. Check Services
kubectl get svc -n production

# 3. Check EndpointSlices
kubectl get endpointslices -n production

# 4. Check DNS
kubectl exec -it <pod> -- nslookup backend-service

# 5. Check connectivity
kubectl exec -it <pod> -- \
  curl -v http://backend-service:8080

# 6. Check NetworkPolicies
kubectl get networkpolicy -n production

# 7. Check events
kubectl get events -n production \
  --sort-by=.metadata.creationTimestamp
```

At infrastructure level, check:

* Security groups.

* Route tables.

* Network ACLs.

* CNI configuration.

* Node-to-node connectivity.

* Load balancer target health.

# **Section 4: Kubernetes Storage and Security**

## **22. Explain PV, PVC and StorageClass.**

**Answer:**

Kubernetes uses PersistentVolumes and PersistentVolumeClaims to provide persistent storage.

| Component    | Responsibility                             |
| ------------ | ------------------------------------------ |
| PV           | Represents available persistent storage    |
| PVC          | Application's request for storage          |
| StorageClass | Defines dynamic provisioning configuration |
| CSI Driver   | Integrates storage systems with Kubernetes |

![Kubernetes Persistent Storage: CSI, Rook & Longhorn](https://images.openai.com/static-rsc-4/o7cQ7Mpi_pSQp1MEWXR0ibQ2pi8cP8PdZ34ZtcSXkPdmVjUNlmwCR6nV3TjZvAwo7s26naXfLrI_eUzX8TCBlUW3QvUP69kwmcmj6gO6BwN4WxjD4QYhp6p9mVSj1RC4BMaXdCfKntl91SOz_VfRIDFCceMHw57exGDYrIFtMMM?purpose=inline)

Add to Favorites

Example PVC:

YAML

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

In AWS EKS:

* EBS CSI Driver: Block storage.

* EFS CSI Driver: Shared file storage.

* FSx CSI Drivers: Specialized file storage options.

## **23. What are Kubernetes access modes?**

**Answer:**

| Access mode             | Meaning                               |
| ----------------------- | ------------------------------------- |
| ReadWriteOnce (RWO)     | Read-write mounting by a single node  |
| ReadOnlyMany (ROX)      | Read-only mounting by multiple nodes  |
| ReadWriteMany (RWX)     | Read-write mounting by multiple nodes |
| ReadWriteOncePod (RWOP) | Read-write mounting by a single Pod   |

Important: RWO means one node, not necessarily one Pod. Support also depends on the storage driver.

## **24. How do you manage secrets in Kubernetes?**

**Answer:**

Kubernetes Secrets store sensitive information such as passwords, API keys and certificates.

Example:

Bash

```
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='StrongPassword'
```

Consume it as an environment variable:

YAML

```
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

Production best practices:

* Enable encryption at rest for Kubernetes Secrets.

* Use RBAC to restrict access.

* Avoid committing secrets to Git.

* Integrate external secret managers.

* Rotate credentials.

* Avoid exposing secret values in logs.

For AWS EKS, commonly used options include:

* AWS Secrets Manager.

* AWS Systems Manager Parameter Store.

* External Secrets Operator.

* Secrets Store CSI Driver.

## **25. Explain Kubernetes RBAC.**

**Answer:**

RBAC stands for Role-Based Access Control.

Four important objects:

* Role.

* ClusterRole.

* RoleBinding.

* ClusterRoleBinding.

Example:

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Bind it to a user or group:

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
  - kind: User
    name: devops-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Expert scenario:** Developers should only be able to deploy applications in their namespace, not access production resources.

I would implement namespace isolation, RBAC, resource quotas, LimitRanges, NetworkPolicies and admission policies.

## **26. What is Pod Security Admission?**

**Answer:**

Pod Security Admission enforces security standards at namespace level.

Three Pod Security Standards:

| Standard   | Description                          |
| ---------- | ------------------------------------ |
| Privileged | Unrestricted                         |
| Baseline   | Prevents known privilege escalations |
| Restricted | Strong security restrictions         |

Example:

Bash

```
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

Additional security practices include:

* Non-root execution.

* Read-only root filesystem.

* Dropping Linux capabilities.

* Seccomp RuntimeDefault.

* Avoiding privileged containers.

* Image vulnerability scanning.

# **Section 5: Kubernetes Deployment, Scaling and Availability**

## **27. Explain rolling updates and rollback in Kubernetes.**

**Answer:**

Kubernetes Deployments support rolling updates without requiring all application Pods to stop simultaneously.

Example:

Bash

```
kubectl set image deployment/myapp \
  myapp=myapp:v2 \
  -n production
```

Check rollout:

Bash

```
kubectl rollout status deployment/myapp

kubectl rollout history deployment/myapp
```

Rollback:

Bash

```
kubectl rollout undo deployment/myapp
```

Rollback to a specific revision:

Bash

```
kubectl rollout undo deployment/myapp --to-revision=2
```

Important Deployment parameters:

YAML

```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

## **28. Explain HPA, VPA and Cluster Autoscaler.**

**Answer:**

These components solve different scaling problems.

| Component          | Scaling responsibility                                                                   |
| ------------------ | ---------------------------------------------------------------------------------------- |
| HPA                | Scales Pod replicas                                                                      |
| VPA                | Recommends or adjusts Pod resource requests                                              |
| Cluster Autoscaler | Scales worker node count                                                                 |
| Karpenter          | Provisions and terminates nodes based on workload requirements in supported environments |

![Mastering Autoscaling in Kubernetes: A Practical Guide](https://images.openai.com/static-rsc-4/1jXr8gt55gEi0fXx9XQgc4_oc6iQgrfQXMtd2aeUexFaCC2QTvck-TnckfG0pJ8jSg5gC9A0SPXk_6kToY6KeQ7DjXaI8jr6UPTkEmknwgt9Vo4dKoAh5aVqAqnjZUmjxtvUglIeVtinVwSFxNd7tUNdJT9pDbujNnLZ3y9SI24?purpose=inline)

Add to Favorites

Example HPA:

YAML

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Expert follow-up:** Can HPA and VPA work together?

Yes, with care. They can interfere when both try to manage CPU or memory requests used by HPA utilization calculations. A common approach is to use VPA in recommendation mode or configure resource responsibilities appropriately.

## **29. How do you achieve high availability in Kubernetes?**

**Answer:**

I would implement HA at multiple levels.

**Control Plane HA**

* Multiple API Server instances.

* Highly available etcd.

* Multiple control-plane nodes where self-managed.

* Reliable API endpoint.

**Worker Node HA**

* Multiple availability zones.

* Node groups distributed across zones.

* Pod anti-affinity.

* Topology spread constraints.

* PodDisruptionBudgets.

**Application HA**

* Multiple replicas.

* Readiness and liveness probes.

* Horizontal autoscaling.

* Graceful shutdown.

* Load balancing.

**Infrastructure HA**

* Highly available storage.

* Redundant network paths.

* Multi-AZ databases.

* Monitoring and automated recovery.

## **30. What is PodDisruptionBudget?**

**Answer:**

PDB ensures that a minimum number of Pods remain available during voluntary disruptions, such as node maintenance.

Example:

YAML

```
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

If an application has three replicas, this policy requires at least two available during supported voluntary disruptions.

Note: PDBs do not prevent involuntary disruptions, such as sudden node failure.

## **31. How do you perform zero-downtime Kubernetes deployment?**

**Answer:**

I would combine:

1. Multiple application replicas.

2. RollingUpdate deployment strategy.

3. Readiness probes.

4. Appropriate termination grace period.

5. Graceful application shutdown.

6. PDB.

7. Backward-compatible database migrations.

8. Traffic management and health checks.

9. Deployment monitoring and rollback.

For critical applications, I may use blue-green or canary deployment strategies through tools such as Argo Rollouts or service mesh traffic management.

## **32. What is the difference between Blue-Green and Canary deployment?**

**Blue-Green**

![Mastering Blue-Green Deployment. Introduction | by Deepal Jain | Medium](https://images.openai.com/static-rsc-4/pFBpZsqseXAT8A6YjfOhfz-MRbnjzPrrDI2GDMrT3ACIPcgCp5HX7zdxOnDytMd8vGML1CGG0DRFxnz1hU_OYE2Kc3NsaW3EJiGD6vNRFkonqEo2L7JvRQ8iL1YKStwc-5LX2pPpOP__CgyB5QUjDd2QdHYyrop22MlpFx1ty7E?purpose=inline)

Add to Favorites

Switch traffic from old environment to new environment.

**Canary**

![Release Strategies in Kubernetes: From Canary Deployments to Progressive Delivery | by Salwan Mohamed | Medium](https://images.openai.com/static-rsc-4/cJv-pV783BplnCagXuIGxMXBqXegytfCgb2d41e4L1IX2b9mRyj1mQ3xopxBN8MN81iTAJ39b2j7Osb6AhkIOuVlX5tNp4hJOVoSgoWRxWZeSjggWFFNZ4gCgGwEMAct72L7vXbZddVK5lE8azgCzPT-izJFcsihlrKBHd54J3A?purpose=inline)

Add to Favorites

Gradually route traffic to the new version.

| Parameter  | Blue-Green                          | Canary                  |
| ---------- | ----------------------------------- | ----------------------- |
| Deployment | Two environments                    | Incremental rollout     |
| Traffic    | Switches between environments       | Gradually shifts        |
| Risk       | Depends on validation and switching | Limits initial exposure |
| Rollback   | Switch back                         | Stop or reverse rollout |

# **Section 6: AWS EKS – Architect-Level Questions**

## **33. What is Amazon EKS? Explain its architecture.**

**Answer:**

Amazon Elastic Kubernetes Service (EKS) is AWS's managed Kubernetes service.

AWS manages the Kubernetes control plane, while customers manage worker compute and application workloads.

![AWS EKS Architecture: Clusters, Nodes, and Networks |  NetApp](https://images.openai.com/static-rsc-4/mtTxJIoa6eDRszb1Kz1fWKmv48WHe5hM8iz0HEsQybmIsnZUDxwvMo9D4pbj0xB8iNbYXr1kUJ6eX0567zdTkCh1cAkE1jY_E2WXhgy5-b9EyLppKb_yOERjhosrSCB3vARK6gi9LcnW78lSThQXjyYnVTcT-KoMkk_cUto72EE?purpose=inline)

Add to Favorites

Major components:

| Component                    | Responsibility                                  |
| ---------------------------- | ----------------------------------------------- |
| EKS Control Plane            | Managed Kubernetes API Server and control plane |
| Managed Node Groups          | AWS-managed EC2 worker node lifecycle           |
| Self-managed Nodes           | Customer-managed worker nodes                   |
| Fargate                      | Serverless compute for supported Pod workloads  |
| AWS VPC CNI                  | Pod networking                                  |
| EBS CSI Driver               | Persistent block storage                        |
| EFS CSI Driver               | Shared file storage                             |
| AWS Load Balancer Controller | Provisions supported ALBs and NLBs              |
| IAM                          | AWS identity and authorization integration      |

## **34. Explain EKS networking.**

**Answer:**

EKS networking is built around the AWS VPC.

Typical architecture:

* VPC across multiple Availability Zones.

* Public and private subnets.

* Private subnets for worker nodes.

* NAT gateways or suitable egress architecture.

* Security groups.

* Route tables.

* VPC CNI for Pod networking.

* Load balancers for external traffic.

![Provision EKS](https://images.openai.com/static-rsc-4/amwcgT7eIt6mzJcFUXL7T_TE55yeMHA5SbTsALH-K7FZK58kTRlw6_266ge5uB7kz25uYU1HWH4JHFZQYDaLQnWbL1zyJDcap7jIgfRJcEOKMEmfOq2sEEX4x1ISFCU4BEH98C6KgjEo2lrL0l1OyBBAt78qibO8HybdsfYBF2s?purpose=inline)

Add to Favorites

**Expert follow-up:** What is AWS VPC CNI?

AWS VPC CNI is the default networking plugin in many EKS configurations. It assigns VPC IP addresses to Pods using network interfaces associated with EC2 instances.

Potential challenges:

* Subnet IP exhaustion.

* ENI/IP allocation limits.

* High Pod density.

* IP consumption during scaling.

Solutions:

* Use appropriately sized subnets.

* Enable VPC CNI prefix delegation where suitable.

* Use custom networking when required.

* Monitor available IP addresses.

* Evaluate supported alternative networking configurations where appropriate.

## **35. How do you expose applications running on EKS?**

**Answer:**

There are several approaches.

| Method                  | Use case                                       |
| ----------------------- | ---------------------------------------------- |
| ClusterIP               | Internal communication                         |
| NodePort                | Node-level exposure                            |
| ALB Ingress             | HTTP/HTTPS applications                        |
| NLB Service             | TCP/UDP or network-level load balancing        |
| API Gateway integration | Managed API exposure architecture              |
| Service mesh            | Advanced service-to-service traffic management |

Typical architecture:

![Application Load Balancer with AWS LoadBalancer Controller and Network LoadBalancer with non ALB… | Sanjal S Eralil](https://images.openai.com/static-rsc-4/8inXLHBxWP5LCVJ7RhQLQqOhiiyABh5wP6oXDOszZ2Sw-tM6eiS2PQ8lbAA1qr1npLoop9-X0uZ2UwnuQJqebMKYhwaiR3s-wfvBCAg8brfBrGum4W6U_w_BcgzUWSVA1EYffWO3Ip-RiXSy9iw5BNIxHz6ELEAVFfzv9AKCEsw?purpose=inline)

Add to Favorites

Request flow:

```
Internet
   |
Route 53
   |
Application Load Balancer
   |
Ingress / Target Groups
   |
Kubernetes Service
   |
Application Pods
```

## **36. What is the difference between EKS Managed Node Groups and Fargate?**

**Answer:**

| Feature              | Managed Node Groups     | Fargate                          |
| -------------------- | ----------------------- | -------------------------------- |
| Compute              | EC2 instances           | Serverless                       |
| Node management      | AWS-managed lifecycle   | No customer-managed worker nodes |
| SSH access           | Available if configured | Not available                    |
| Customization        | High                    | Limited                          |
| DaemonSets           | Supported               | Not supported                    |
| Workload flexibility | High                    | Supported workload types only    |
| Cost model           | EC2-based               | Pod resource-based               |

**When would you choose Fargate?**

* Workloads that benefit from serverless compute.

* Isolated application workloads.

* Reduced infrastructure management.

**When would you choose managed node groups?**

* High-density workloads.

* GPU workloads.

* DaemonSet requirements.

* Greater control over instance types and node configuration.

* Workloads requiring specialized infrastructure.

## **37. Explain EKS authentication and authorization.**

**Answer:**

There are two separate concepts:

**Authentication:** Who are you?

**Authorization:** What are you allowed to do?

EKS commonly integrates with AWS IAM for authentication and Kubernetes RBAC for authorization.

EKS access management supports:

* EKS access entries.

* Access policies.

* Kubernetes group mappings.

* Kubernetes RBAC.

Example:

Bash

```
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name production-cluster

kubectl get nodes
```

**Expert follow-up:** What is the difference between IRSA and EKS Pod Identity?

Both allow workloads to access AWS services without embedding long-lived AWS credentials.

* IRSA: Uses IAM Roles for Service Accounts and an OIDC identity provider.

* EKS Pod Identity: Uses the EKS Pod Identity Agent and associated IAM role configuration.

Use least-privilege IAM policies for both.

## **38. How do you upgrade an EKS cluster?**

**Answer:**

I follow a controlled upgrade procedure:

1. Review the Kubernetes version compatibility.

2. Review deprecated APIs.

3. Upgrade EKS control plane.

4. Upgrade EKS add-ons.

5. Upgrade managed node groups.

6. Upgrade self-managed nodes if present.

7. Validate workloads.

8. Run smoke and integration tests.

9. Monitor application health.

Example:

Bash

```
aws eks update-cluster-version \
  --name production-cluster \
  --kubernetes-version <supported-version>
```

Important components:

* VPC CNI.

* CoreDNS.

* kube-proxy.

* EBS CSI Driver.

* EFS CSI Driver.

* AWS Load Balancer Controller.

Never assume that upgrading the EKS control plane automatically upgrades every node and add-on.

## **39. How do you implement EKS disaster recovery?**

**Answer:**

I would first identify RTO and RPO requirements.

Two common strategies:

**Strategy A: Backup and Restore**

* Back up Kubernetes resources.

* Back up persistent data.

* Store backups in a separate AWS account or region.

* Recreate the cluster using Terraform.

* Restore applications and data.

**Strategy B: Multi-Region DR**

* Primary EKS cluster in one region.

* Standby or active cluster in another region.

* Cross-region data replication.

* Container images available in both regions.

* Replicated infrastructure configuration.

* Route 53 failover or traffic management.

Important technologies:

* Velero.

* AWS Backup where supported for the selected resources.

* Amazon ECR replication.

* Terraform.

* Route 53.

* Cross-region database replication.

A critical consideration: Kubernetes configuration backups alone do not guarantee application recovery. Persistent data, secrets, IAM, networking and dependencies must also be covered.

## **40. How do you monitor EKS?**

**Answer:**

I would implement monitoring at four levels.

| Layer            | Metrics and tools                  |
| ---------------- | ---------------------------------- |
| Infrastructure   | EC2, EBS, VPC, CloudWatch          |
| Kubernetes       | Node, Pod, Deployment metrics      |
| Application      | Latency, errors, throughput        |
| Logs and tracing | Container logs, distributed traces |

Common tools:

* Prometheus.

* Grafana.

* Amazon CloudWatch.

* Amazon Managed Service for Prometheus.

* Amazon Managed Grafana.

* Fluent Bit.

* OpenTelemetry.

* AWS X-Ray.

For production EKS, I would implement:

* Cluster health dashboards.

* Namespace-wise dashboards.

* Application-wise dashboards.

* Node resource utilization.

* Pod restarts.

* Pending Pods.

* CPU throttling.

* Memory pressure.

* API Server health.

* Persistent volume capacity.

* Network performance.

# **Section 7: AWS ECS vs EKS – Architect-Level Questions**

## **41. Explain AWS ECS architecture.**

**Answer:**

Amazon Elastic Container Service (ECS) is AWS's managed container orchestration service.

ECS manages containerized applications using:

* Clusters.

* Services.

* Task Definitions.

* Tasks.

* Container Instances or Fargate compute.

![Deploying a Scalable Node.js Application on AWS ECS Fargate with Docker | by Hina Atif | Medium](https://images.openai.com/static-rsc-4/nrAMI4_TVexc5z1zOHO-H5KkUMMBD2zX08dkFURcI9XgAihmMOACyLlkGFNFyKgS5UanhM3jm_ZBL5a6TV7YLn8rP4kYK0mzB9M3s8OuomtZLmRaWOOlRoLCbsOHQzOVoLuXSNVy-agLW9uYc5tLdgScj4QLamD3wC2oNWVW-6c?purpose=inline)

Add to Favorites

| Component       | Responsibility                        |
| --------------- | ------------------------------------- |
| ECS Cluster     | Logical grouping of resources         |
| Task Definition | Container configuration blueprint     |
| Task            | Running instance of a task definition |
| ECS Service     | Maintains desired task count          |
| EC2 Launch Type | Runs containers on EC2                |
| Fargate         | Serverless container compute          |
| ECR             | Container image registry              |
| ALB             | Routes application traffic            |

## **42. What is the difference between ECS and EKS?**

**Answer:**

This is a very important architect-level interview question.

| Parameter         | ECS                               | EKS                                |
| ----------------- | --------------------------------- | ---------------------------------- |
| Orchestration     | AWS-native                        | Kubernetes                         |
| Complexity        | Lower                             | Higher                             |
| Control plane     | AWS managed                       | AWS managed                        |
| Learning curve    | Easier                            | More complex                       |
| Portability       | AWS-oriented                      | Kubernetes ecosystem portability   |
| Configuration     | Task Definitions                  | Kubernetes YAML                    |
| Networking        | awsvpc and other supported modes  | Kubernetes CNI                     |
| Service discovery | Cloud Map integration             | Kubernetes Services and DNS        |
| Scaling           | ECS Service Auto Scaling          | HPA, VPA, node autoscaling         |
| Ecosystem         | AWS ecosystem                     | Kubernetes ecosystem               |
| Best suited for   | AWS-native container applications | Complex Kubernetes-based platforms |

**How would you decide between ECS and EKS?**

I would evaluate:

1. Application complexity.

2. Kubernetes ecosystem requirements.

3. Multi-cloud requirements.

4. Team Kubernetes expertise.

5. Operational overhead.

6. Scaling requirements.

7. Networking requirements.

8. Cost.

9. Security and compliance.

10. Long-term platform strategy.

My recommendation:

* Choose ECS for simpler AWS-native container workloads where Kubernetes capabilities are unnecessary.

* Choose EKS when Kubernetes APIs, ecosystem integrations, portability, advanced orchestration or an existing Kubernetes operating model are important.

## **43. What is the difference between ECS Task Definition and Kubernetes Deployment?**

**Answer:**

Both define how application containers should run, but they have different responsibilities.

| ECS | Kubernetes | |---|---| | Task Definition | Pod specification and related configuration | | ECS Service | Deployment/ReplicaSet-like workload management | | Task | Running task containing one or more containers | | Service Discovery | Cloud Map or load balancer integration | Service and CoreDNS |

One important distinction: an ECS Task Definition is a reusable configuration blueprint, whereas a Kubernetes Deployment is a controller that manages desired Pod replicas and updates.

## **44. How does ECS scaling work?**

**Answer:**

ECS supports Service Auto Scaling using AWS Application Auto Scaling.

Scaling can be based on:

* CPU utilization.

* Memory utilization.

* Request count per target.

* Custom CloudWatch metrics.

Example:

```
ECS Service
    |
Application Auto Scaling
    |
CloudWatch Metrics
    |
Scale Out / Scale In
    |
Desired Task Count
```

ECS on EC2 also requires sufficient EC2 capacity. Fargate removes the need to manage EC2 worker capacity.

## **45. How do you secure ECS and EKS workloads?**

**Answer:**

I would implement security at multiple levels.

| Security layer   | ECS                 | EKS                                  |
| ---------------- | ------------------- | ------------------------------------ |
| Identity         | Task IAM Role       | Pod Identity / IRSA                  |
| Image security   | ECR scanning        | ECR scanning                         |
| Network security | Security Groups     | Security Groups, NetworkPolicies     |
| Secrets          | Secrets Manager     | Secrets Manager integration          |
| Access           | IAM                 | IAM + RBAC                           |
| Runtime security | Container hardening | Pod Security, runtime controls       |
| Audit            | CloudTrail          | CloudTrail and Kubernetes audit logs |

# **Section 8: Real-World Kubernetes Production Troubleshooting**

These questions are particularly important for Lead DevOps and SRE interviews. Interviewers often provide a production incident and expect you to explain your troubleshooting approach rather than simply give a command.

## **46. A Pod is stuck in Pending state. How would you troubleshoot?**

**Answer:**

First:

Bash

```
kubectl get pods -n production

kubectl describe pod <pod-name> -n production
```

Possible reasons:

| Reason                 | Solution                                  |
| ---------------------- | ----------------------------------------- |
| Insufficient CPU       | Increase node capacity or adjust requests |
| Insufficient memory    | Increase capacity or adjust requests      |
| Node selector mismatch | Verify node labels                        |
| Taints                 | Configure tolerations                     |
| PVC pending            | Check StorageClass and CSI                |
| Node affinity issue    | Review affinity rules                     |
| IP exhaustion          | Check CNI and subnet IP capacity          |

I would also check:

Bash

```
kubectl get nodes

kubectl describe nodes

kubectl get events -n production
```

## **47. A Pod is continuously restarting with CrashLoopBackOff. What would you do?**

**Answer:**

I would investigate:

1. Application logs.

2. Previous container logs.

3. Exit code.

4. OOMKilled status.

5. Application configuration.

6. Environment variables and Secrets.

7. Probe configuration.

8. External dependencies.

Commands:

Bash

```
kubectl logs <pod-name> -n production

kubectl logs <pod-name> \
  --previous -n production

kubectl describe pod <pod-name> -n production
```

Possible causes:

* Application exception.

* Invalid configuration.

* Missing environment variables.

* Memory exhaustion.

* Incorrect startup or liveness probe.

* Database connectivity failure.

## **48. A Kubernetes application is experiencing high latency. How would you investigate?**

**Answer:**

I would investigate the complete request path.

```
User
 |
Load Balancer
 |
Ingress
 |
Service
 |
Application Pod
 |
Database / External API
```

Investigation areas:

* ALB/NLB latency.

* Ingress controller metrics.

* Service endpoint availability.

* Pod CPU and memory.

* CPU throttling.

* Application response time.

* Database query latency.

* Network latency.

* DNS resolution.

* Downstream service dependencies.

Useful commands:

Bash

```
kubectl top nodes

kubectl top pods -A

kubectl get pods -o wide -A

kubectl describe deployment <deployment-name>
```

I would correlate metrics, logs and distributed traces to isolate the bottleneck.

## **49. One worker node becomes NotReady. What happens to the application?**

**Answer:**

Kubernetes detects node health issues through Node heartbeats.

Depending on the condition and configured toleration timings:

* The node becomes NotReady or Unknown.

* Pods may be evicted.

* Controllers attempt to maintain desired replicas.

* Replacement Pods may be scheduled on healthy nodes.

* Persistent storage attachment constraints may delay recovery.

Troubleshooting:

Bash

```
kubectl get nodes

kubectl describe node <node-name>

kubectl get pods -A -o wide
```

Check:

* Kubelet status.

* Container runtime.

* Node CPU and memory.

* Disk pressure.

* Network connectivity.

* CNI health.

* Node IAM and cloud infrastructure.

## **50. Your application is running in three replicas, but only one replica receives traffic. What could be wrong?**

**Answer:**

I would check:

1. Service selector labels.

2. Pod readiness.

3. EndpointSlices.

4. Ingress configuration.

5. Load balancer target registration.

6. Session affinity.

7. Application connection handling.

8. Traffic distribution and long-lived connections.

Commands:

Bash

```
kubectl get pods --show-labels

kubectl get svc

kubectl get endpointslices

kubectl describe svc <service-name>
```

A Service normally distributes connections across eligible endpoints, but it does not guarantee equal request counts. Connection reuse, session affinity and load-balancing implementation can create uneven traffic.

## **51. Pods are experiencing OOMKilled errors. How would you resolve them?**

**Answer:**

OOMKilled indicates that the container was terminated due to memory exhaustion.

I would:

* Check actual memory consumption.

* Inspect application memory leaks.

* Review JVM heap settings if applicable.

* Review memory requests and limits.

* Check node memory pressure.

* Evaluate workload sizing.

* Introduce appropriate autoscaling.

Bash

```
kubectl top pod <pod-name> -n production

kubectl describe pod <pod-name> -n production
```

Example resource configuration:

YAML

```
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"
```

I would not simply increase memory limits without understanding the memory consumption pattern.

## **52. Your Kubernetes cluster is running out of IP addresses. How would you resolve this in EKS?**

**Answer:**

This is a common EKS production problem.

I would investigate:

1. Available IPs in worker node subnets.

2. VPC CNI IP allocation.

3. ENI limits.

4. Prefix delegation configuration.

5. Pod density.

6. Custom networking requirements.

Possible solutions:

* Increase subnet CIDR capacity.

* Add secondary VPC CIDR ranges.

* Create additional suitable subnets.

* Enable prefix delegation where appropriate.

* Optimize Pod density.

* Review CNI configuration.

## **53. A new application deployment has caused production downtime. How would you recover?**

**Answer:**

My immediate priority would be service restoration.

Bash

```
kubectl rollout history deployment/myapp

kubectl rollout undo deployment/myapp

kubectl rollout status deployment/myapp
```

Then:

1. Check application availability.

2. Verify error rate and latency.

3. Identify the deployment change.

4. Review configuration and image differences.

5. Check database migration compatibility.

6. Perform root cause analysis.

7. Add preventive deployment controls.

For future deployments, I would introduce progressive delivery, automated health validation and rollback mechanisms.

# **Section 9: Advanced Architect-Level Scenario Questions**

These questions distinguish a Kubernetes administrator from a Kubernetes platform architect.

## **54. How would you design a production-grade Kubernetes platform for 100 microservices?**

**Suggested answer:**

I would design the platform around these layers:

![실무까지 한 번에 끝내는 DevOps를 위한 Docker & Kubernetes  feat. aws EKS 초격차 패키지 Online. | 패스트캠퍼스](https://images.openai.com/static-rsc-4/Yg4Yox4CkOXmgs7G236daBX9WaRpNXp5W5YqQuXrDdbDGya9wEoJBnpY8z-CZFo8nMVMDDiETkpAMK46po2uKunbIzx6jqo3_rpvFJ8Qej1OqFxMnbcck826_liTNN2pO2Nl7rqOVKr8HOeo5g-A7rinhWa6rz9_D68JrvOdwGs?purpose=inline)

Add to Favorites

[fastcampus.co](https://fastcampus.co.kr/dev_online_devops_kubernetes?utm_source=chatgpt.com)

| Layer                | Technology / Design                        |
| -------------------- | ------------------------------------------ |
| Cloud infrastructure | AWS VPC, EKS, EC2                          |
| Container registry   | Amazon ECR                                 |
| Networking           | VPC CNI, CoreDNS, ALB                      |
| Workload isolation   | Kubernetes namespaces                      |
| Deployment           | Helm, Argo CD                              |
| CI/CD                | Jenkins, GitHub Actions                    |
| Autoscaling          | HPA, Karpenter                             |
| Monitoring           | Prometheus, Grafana                        |
| Logging              | Fluent Bit, Loki or CloudWatch             |
| Tracing              | OpenTelemetry                              |
| Secrets              | AWS Secrets Manager                        |
| Security             | IAM, RBAC, Kyverno or policy engine        |
| Storage              | EBS, EFS                                   |
| DR                   | Multi-AZ, backup and cross-region recovery |

**Architectural decisions:**

* Separate environments using accounts and clusters according to security and isolation requirements.

* Use namespaces for application-level logical separation.

* Implement standard Helm charts.

* Enforce resource requests and limits.

* Use automated policy enforcement.

* Implement centralized observability.

* Introduce GitOps.

* Use progressive deployment.

* Establish platform SLOs and operational runbooks.

## **55. How would you implement multi-tenancy in Kubernetes?**

**Answer:**

I would evaluate isolation at several levels.

**Namespace-level isolation**

* Separate namespaces per team or application.

* RBAC.

* ResourceQuota.

* LimitRange.

* NetworkPolicy.

* Pod Security Admission.

**Cluster-level isolation**

* Dedicated node pools.

* Node affinity and taints.

* Separate ingress controllers where appropriate.

**Strong isolation**

- Dedicated clusters or separate AWS accounts for workloads with stronger security boundaries.

Important point: Kubernetes namespaces alone are not a complete security boundary.

## **56. How would you implement GitOps for Kubernetes?**

**Answer:**

GitOps uses Git as the declarative source of truth for infrastructure and application configuration.

Tools:

* Argo CD.

* Flux CD.

Architecture:

![GitOps on Kubernetes with ArgoCD. Implementing declarative, automated… | by Jakub Jakubowski | Apr, 2026 | Medium](https://images.openai.com/static-rsc-4/wweamSkSnYizZXbTMefdUqb5h8jFofJg4miKnBGMsvxhwjCWeb_UxtyH7k-Iw1GQsgHfbMIIlGZ_ULur9l3GAN2z8vt1RxYIMzkIkzS9lOE4BTwTJ0TsHVYY_AwcntK4_d7F4i5fsBgGDiNv9YzgOI18GGdYSxt8ahe0_NOn0cg?purpose=inline)

Add to Favorites

Workflow:

1. Developer commits application code.

2. CI builds Docker image.

3. CI scans and pushes image to ECR.

4. CI updates Helm values or image version in Git.

5. Argo CD detects the change.

6. Argo CD reconciles the cluster.

7. Deployment health is monitored.

Example:

Bash

```
kubectl get applications -n argocd

argocd app list

argocd app sync production-app
```

Key benefits:

* Version-controlled deployments.

* Auditability.

* Drift detection.

* Automated reconciliation.

* Easier rollback.

* Consistent environment configuration.

## **57. How would you implement Kubernetes security in an enterprise environment?**

**Answer:**

I would follow a layered security model.

| Layer       | Implementation                    |
| ----------- | --------------------------------- |
| Source code | SAST and dependency scanning      |
| Image       | ECR scanning, SBOM, image signing |
| Deployment  | Admission policies                |
| Identity    | IAM, RBAC, least privilege        |
| Network     | NetworkPolicy, security groups    |
| Runtime     | Non-root, seccomp, capabilities   |
| Secrets     | Secrets Manager                   |
| Audit       | CloudTrail, Kubernetes audit logs |
| Compliance  | CIS Kubernetes Benchmark          |
| Monitoring  | Runtime threat detection          |

I would also establish security gates in the CI/CD pipeline and periodically validate compliance.

## **58. How would you optimize Kubernetes infrastructure cost?**

**Answer:**

I would focus on five areas.

1. Resource optimization.

2. Compute optimization.

3. Autoscaling.

4. Storage optimization.

5. Workload scheduling.

Practical measures:

* Right-size CPU and memory requests.

* Remove unused workloads.

* Use HPA and Karpenter.

* Use Spot Instances for fault-tolerant workloads.

* Select suitable EC2 instance families.

* Optimize storage classes and volumes.

* Reduce unnecessary cross-AZ data transfer.

* Use namespace quotas.

* Monitor cost by team and application.

Useful tools:

* Kubecost.

* AWS Cost Explorer.

* AWS Compute Optimizer.

* Prometheus and Grafana.

## **59. How would you design a Kubernetes platform supporting multiple environments?**

**Answer:**

I would consider three approaches:

| Approach        | Description                                              |
| --------------- | -------------------------------------------------------- |
| Namespace-based | Multiple environments in one cluster                     |
| Cluster-based   | Dedicated cluster per environment                        |
| Account-based   | Dedicated AWS account and cluster for stronger isolation |

Example:

```
AWS Organization
   |
   +--- Development Account
   |       |
   |       +--- EKS Development
   |
   +--- QA Account
   |       |
   |       +--- EKS QA
   |
   +--- Production Account
           |
           +--- EKS Production
```

For enterprise environments, I generally prefer separate AWS accounts for production isolation and governance, with the cluster strategy determined by availability, security, workload and cost requirements.

## **60. How would you design Kubernetes disaster recovery with RTO of 30 minutes and RPO of 5 minutes?**

**Answer:**

First, I would clarify which applications, databases and dependencies must meet these targets.

A possible design:

* Primary EKS cluster in Region A.

* Warm standby EKS cluster in Region B.

* Terraform-based infrastructure provisioning.

* ECR cross-region image replication.

* GitOps-based application deployment.

* Frequent application data replication.

* Database cross-region replication.

* Automated DNS failover.

* Tested recovery runbooks.

![Automating Disaster Recovery (DR) for EKS Across AWS Regions. | by Sandeepkomal | Medium](https://images.openai.com/static-rsc-4/5K7bhkGKOSvPj24sinDfGHq6hRYGZITpxg2iNQ7wZWWh-rYmfgnZfZSkYiP55q_Q3WeR4FV5BxoCAMZRTBb-NHIaWC3glBbNC0VdojVa5sI4psxEyV8K0xEx0rboKhKs45W0czWD8XAGDObFF2UnBo5RlGSyi1PjZkG9OC__uGw?purpose=inline)

Add to Favorites

I would validate the RTO/RPO through regular disaster recovery exercises rather than assume that backups or replication automatically guarantee them.

# **Section 10: Additional Expert-Level Questions**

These are frequently used to test whether candidates understand Kubernetes internals beyond basic operations.

## **61. What is the difference between CNI, CSI and CRI?**

| Interface | Full form                   | Responsibility                |
| --------- | --------------------------- | ----------------------------- |
| CNI       | Container Network Interface | Container networking          |
| CSI       | Container Storage Interface | Persistent storage            |
| CRI       | Container Runtime Interface | Container runtime integration |

Remember:

* CNI = Networking.

* CSI = Storage.

* CRI = Runtime.

## **62. What is the difference between Helm and Kustomize?**

**Answer:**

Both are used for Kubernetes configuration management.

| Helm                        | Kustomize                          |
| --------------------------- | ---------------------------------- |
| Package manager             | Configuration customization        |
| Uses Helm charts            | Uses base and overlays             |
| Supports templates          | Uses declarative patches           |
| Supports chart dependencies | Supports configuration composition |
| Release management          | No native release management       |

Example Helm:

Bash

```
helm install myapp ./mychart \
  -n production

helm upgrade myapp ./mychart \
  -n production

helm rollback myapp 1 \
  -n production
```

Example Kustomize:

Bash

```
kubectl apply -k overlays/production
```

## **63. What is a Service Mesh? Why would you use Istio?**

**Answer:**

A Service Mesh manages communication between microservices.

![\[Architecture\] MSA : Istio 개념 잡기 - Azderica](https://images.openai.com/static-rsc-4/5pIp2UrNDIEcWQwm-FE5ew5UmmpCMDQoZqwMcQwJVRExhi8dSOEbTgOYcY8aJyfer2fs4sjYaFG5b26bO1WN9TheFP78EdzlmZg1XW8PhWlO_GA1P0Y684np5EgUq53kmkrNZiin7W4lwUcOsRRrUgZnwZ1cHlmtRDyWlwnZThA?purpose=inline)

Add to Favorites

Common capabilities:

* Service-to-service communication.

* Mutual TLS.

* Traffic management.

* Retries and timeouts.

* Circuit breaking.

* Observability.

* Traffic splitting.

Popular options:

* Istio.

* Linkerd.

* Cilium service mesh.

I would introduce a service mesh when the organization needs advanced service-to-service security, traffic management and observability, while considering its operational overhead.

## **64. What happens when a Kubernetes Deployment is deleted?**

**Answer:**

When a Deployment is deleted through the normal cascading deletion mechanism:

1. Deployment object is deleted.

2. Its ReplicaSet is deleted.

3. The ReplicaSet-managed Pods are deleted.

4. Associated containers terminate.

Services, ConfigMaps, Secrets and PVCs are not automatically deleted merely because they are referenced by the Deployment.

## **65. Explain Kubernetes admission controllers.**

**Answer:**

Admission controllers intercept API requests after authentication and authorization but before persistence.

Two important types:

* Mutating admission: Changes an incoming object.

* Validating admission: Accepts or rejects an object.

Examples:

* NamespaceLifecycle.

* ResourceQuota.

* PodSecurity.

* MutatingAdmissionWebhook.

* ValidatingAdmissionWebhook.

Enterprise policy tools include Kyverno and OPA Gatekeeper.

Example policy requirements:

* Only approved container registries.

* No privileged containers.

* Mandatory resource requests and limits.

* Mandatory labels.

* Disallow `latest` image tags.

# **Section 11: Important Kubernetes Commands for Practical Interviews**

You should be comfortable executing these commands without referring to documentation.

| Requirement               | Command                                                       |
| ------------------------- | ------------------------------------------------------------- |
| List all Pods             | `kubectl get pods -A`                                         |
| List nodes                | `kubectl get nodes -o wide`                                   |
| Pod details               | `kubectl describe pod POD`                                    |
| Pod logs                  | `kubectl logs POD`                                            |
| Previous container logs   | `kubectl logs POD --previous`                                 |
| Execute into container    | `kubectl exec -it POD -- sh`                                  |
| Pod resource utilization  | `kubectl top pods -A`                                         |
| Node resource utilization | `kubectl top nodes`                                           |
| Create Deployment         | `kubectl create deployment nginx --image=nginx`               |
| Scale Deployment          | `kubectl scale deployment nginx --replicas=5`                 |
| Update image              | `kubectl set image deployment/nginx nginx=nginx:1.27`         |
| Rollback Deployment       | `kubectl rollout undo deployment/nginx`                       |
| Check rollout             | `kubectl rollout status deployment/nginx`                     |
| List Services             | `kubectl get svc -A`                                          |
| List Ingress              | `kubectl get ingress -A`                                      |
| List NetworkPolicies      | `kubectl get networkpolicy -A`                                |
| List PVC                  | `kubectl get pvc -A`                                          |
| List StorageClasses       | `kubectl get storageclass`                                    |
| List ConfigMaps           | `kubectl get configmap -A`                                    |
| List Secrets              | `kubectl get secrets -A`                                      |
| List events               | `kubectl get events -A --sort-by=.metadata.creationTimestamp` |
| Check API resources       | `kubectl api-resources`                                       |
| Check cluster information | `kubectl cluster-info`                                        |

# **Section 12: Hands-on Practical Preparation**

For an expert-level interview, I recommend completing these 10 practical exercises.

## **Kubernetes hands-on checklist**

Use these exercises as a practical interview readiness tracker.

**Progress**

# **0****/****10**

**1****.** **Docker**

Create optimized multi-stage Docker images and implement container security.

**2****.** **Kubernetes deployment**

Deploy three microservices using Deployment, Service, ConfigMap and Secret.

**3****.** **Networking**

Configure Ingress, CoreDNS and NetworkPolicy for frontend-to-backend communication.

**4****.** **Storage**

Configure EBS CSI, StorageClass, PVC and a StatefulSet.

**5****.** **Autoscaling**

Implement HPA and configure Karpenter or Cluster Autoscaler.

**6****.** **Security**

Configure RBAC, IAM integration, Pod Security Admission and external secrets.

**7****.** **Monitoring**

Install Prometheus and Grafana and create namespace-wise dashboards.

**8****.** **CI/CD**

Build a Jenkins pipeline to build, scan, push and deploy images to EKS.

**9****.** **GitOps**

Implement Argo CD with Helm and environment-specific overlays.

**10****.** **Disaster recovery**

Back up and restore Kubernetes resources and application persistent data.

# **Section 13: Recommended preparation resources**

These are useful official resources for expert-level revision.

![Kubernetes Logo Png | Kubernetes Logo Vector  - freepng](https://images.openai.com/static-rsc-4/DlwnvfIwR7-nYgYdMKS3S1y9EFWc2tbWNgokHxUN7wanK4l0jgxArwrNFL-u2OJEK12w91M3I2_EAr1Vtmc3hj0Emg3MHhzghZdtmb7GRkaOK0S0sYz49ZQxSaSrdGPdtIwRssmjvzPj-mPZ61OahTeoLwT9rcWaFvbhrTjL2I0?purpose=inline)

Add to Favorites

**Kubernetes Documentation**

Architecture, workloads, networking, storage and security.

[Official Kubernetes documentation](https://kubernetes.io/docs/?utm_source=chatgpt.com)

![Docker Logo PNG, AI, EPS, CDR, PDF, SVG - iconLogoVector](https://images.openai.com/static-rsc-4/x_Ge8Om-lmdDyESi_E1do4drGQT9DSkX52vTn2ewhosAAWrXUQb75wMc1ir1acbib9ghrUC265cv3GoWGy3jQpD5Eo8uaFw2K4vVUP6L8ro7xoeh077PcgfTkj0A9dAagm-ASFMY2S_Q_IAVAVpZCEojLDWXxsXMojeGIp5mFLE?purpose=inline)

Add to Favorites

**Docker Documentation**

Dockerfiles, image building and container security.

[Official Docker documentation](https://docs.docker.com/?utm_source=chatgpt.com)

![Free High-Quality Amazon EKS Logo for Creative Design](https://images.openai.com/static-rsc-4/Or04aWW_S6Y8QP12Tpysowjs5H9xNIfIoUsmXmvXwhfdlDyIWS0alY3VTsThAXaJg4qtUdzLyz94i6YfXPubs3qOLgJhm-TK30A0cOWV6iHv3QfWUkel7f0UY90cm-3f1HjL8goBOgC4_6of--688sUXF4gstLAhGxNnzVWd7Tk?purpose=inline)

Add to Favorites

**Amazon EKS**

EKS architecture, networking, security and operations.

[AWS EKS documentation](https://docs.aws.amazon.com/eks/?utm_source=chatgpt.com)

![Cloud Icons | Amazon Elastic Container Service](https://images.openai.com/static-rsc-4/w1qdMQfvOlNZyVcZMy4NlCLj8-i3GcHhK9gtcukzN95vBe0ITrNeshxJW9VhWVjfMwan7KpEb08EsiF_v7NBtZ8YdzKmRkeK5YiOofdUAofo-dQmgXiGpaaumIcSM3-qrvxKoF-3kKF8j5WKwOapuuaaLYxRV0o69tBrWFJ6Q9M?purpose=inline)

Add to Favorites

**Amazon ECS**

ECS architecture, Fargate, task definitions and scaling.

[AWS ECS documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html?utm_source=chatgpt.com)

![Kubernetes PNG Logo — Download Free Transparent Logo](https://images.openai.com/static-rsc-4/xsltUvZs55BJFWvEIuhcsBe4GpnKMbQfDP7au1Vqu9IzACOtu4gKlk2WJ9nNwhE2lzj7mDPVy-JVnRtTg4RX-ccY6FCE82mkHfZk6ickAweOzRNiI6cHDUKq9Z4uBYfxVFRplLMuVJLvUdj1U0QN1v2X7xfwqcgZNaDGmSHweDM?purpose=inline)

Add to Favorites

**KillerCoda**

Hands-on Kubernetes terminal exercises.

[Practice Kubernetes scenarios](https://killercoda.com/?utm_source=chatgpt.com)

# **Section 14: My recommended interview preparation priority**

For a Lead DevOps Engineer / Cloud Architect interview, I would prioritize the following topics:

| Priority | Topic                                      | Importance |
| -------- | ------------------------------------------ | ---------- |
| 1        | Kubernetes architecture and internals      | Very high  |
| 2        | Docker architecture and image optimization | Very high  |
| 3        | Kubernetes networking and CNI              | Very high  |
| 4        | EKS architecture and VPC networking        | Very high  |
| 5        | Production troubleshooting                 | Very high  |
| 6        | HPA, VPA and Karpenter                     | High       |
| 7        | Kubernetes security and RBAC               | High       |
| 8        | CI/CD and GitOps                           | High       |
| 9        | ECS vs EKS architecture                    | High       |
| 10       | DR, HA and cost optimization               | High       |

## **Suggested 20-hour study plan**

Share

Chart options

20-hour Kubernetes interview preparation

Recommended time allocation by subject.



**My final recommendation:** Don't prepare only theoretical Kubernetes questions. At your target Lead DevOps / Architect level, you should be able to explain how you designed, deployed, secured, monitored, scaled and recovered a production Kubernetes platform.

The next useful step would be to build a **100-question Kubernetes Expert Interview Q&A collection**, with detailed answers divided into Docker (15), Kubernetes architecture (25), networking and storage (15), EKS/ECS (20), security and monitoring (10), and production troubleshooting and architecture scenarios (15).


20-hour Kubernetes interview preparation

Recommended time allocation by subject.

topic	hours
Kubernetes architecture	3
Docker and containerization	2
Networking and storage	3
EKS and ECS	3
Troubleshooting	3
Security and scaling	2
CI/CD and GitOps	2
Mock interview	2
