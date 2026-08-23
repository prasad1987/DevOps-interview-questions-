K8 interview Q

**1. Explain Kubernetes architecture.**

Kubernetes architecture is basically divided into two main parts: Control Plane and Worker Nodes.
The control plane is the brain of Kubernetes.
It makes decisions like where to run containers, when to restart them, and how to maintain the desired state of the cluster.
Inside the control plane there are a few key components:

API Server – this is the entry point. Whenever we run kubectl commands, they go to the API server.
Scheduler – it decides on which node a pod should run based on CPU, memory, and other rules.
Controller Manager – it keeps checking if the cluster state matches what we want. For example, if a pod dies, it creates a new one.

etcd – this is the database of Kubernetes. It stores the entire cluster configuration and state.

Then we have the worker nodes, where our actual applications run.
Each worker node has:

Kubelet – agent that talks to the control plane and ensures containers are running properly.
on each node we need to install kubectl

Pods – it is the smallest unit in the Kubernetes, which runs the containers.

Kube-proxy – handles networking and routing traffic to the right pods.

So in short:
👉 Control plane manages and decides
👉 Worker nodes run the applications

**2) What happens if a container crashes?**

In Kubernetes, the kubelet detects that the container stopped and restarts it based on the pod’s restart policy. If the container keeps failing, Kubernetes marks it as CrashLoopBackOff and keeps retrying with delays while exposing logs so we can debug the issue.
Step-by-step debugging:
If a container crashes, first I check the pod status using kubectl get pods to confirm the failure and see if it’s restarting repeatedly.
Then I check logs using kubectl logs to understand the application error. If logs aren’t enough, I describe the pod with kubectl describe pod to check events like image pull errors, resource limits, or probe failures.
If needed, I check resource usage to see if the container is hitting memory or CPU limits. I also verify environment variables, secrets, and configuration changes that may have caused the crash.
Once the root cause is clear, I fix the issue, redeploy, and monitor the pod to ensure it stabilizes.

kubectl get pods -n <namespace>

kubectl logs <pod-name> -n <namespace>

kubectl logs <pod-name> -c <container-name> -n <namespace>

kubectl logs <pod-name> --previous -n <namespace>

kubectl describe pod <pod-name> -n <namespace>

kubectl top pod <pod-name> -n <namespace>

kubectl get nodes

kubectl describe node <node-name>


**3) What happens when you run kubectl apply?**

When we run kubectl apply, Kubernetes basically reads the YAML file and tries to make the cluster match that configuration.
First, kubectl sends the request to the API Server.
The API server is the entry point for all Kubernetes operations.
Then the API server validates the YAML — it checks if the fields are correct and allowed.
If everything is valid, the configuration gets stored in etcd, which is the cluster’s database.
After that, the control plane components take over:

The Scheduler decides which worker node should run the pod.
The Controller Manager ensures the desired state is maintained.
The Kubelet on the chosen worker node receives instructions and starts the containers using the container runtime.

So in short:
kubectl apply sends config to API server
API server validates and stores it in etcd
Scheduler picks node
Kubelet creates the pod on worker node

When we run kubectl apply, the YAML configuration is sent to the API server. The API server validates it and stores the desired state in etcd. Then the scheduler selects a worker node, and the kubelet on that node creates the pod and starts the containers. Since kubectl apply is declarative, it only updates what has changed instead of recreating everything.

**4) How does scheduler decide node placement?**

The Kubernetes scheduler decides node placement in two main steps:
First, it filters nodes to find which ones are eligible.
It checks things like available CPU and memory, node selectors, taints and tolerations, and whether the node can actually run that pod.
Then, from the remaining nodes, it scores them to pick the best one.
It prefers nodes with balanced resource usage, better locality, or ones that match affinity rules.
So in simple terms, it first finds nodes that can run the pod, and then chooses the one that’s best for it.

**5) What happens if a node dies?**

If a node dies, Kubernetes detects it through the node heartbeat mechanism. When the control plane stops receiving updates from that node, it marks the node as NotReady.
After a short timeout, Kubernetes assumes the pods on that node are lost and reschedules new copies of those pods onto healthy nodes, as long as they’re managed by a controller like a Deployment or ReplicaSet.
So from the user’s perspective, workloads recover automatically, although there may be a short downtime until new pods start running on other nodes.


**6)  How do containers communicate across nodes?**

In Kubernetes, containers communicate across nodes through the cluster networking model managed by the CNI plugin. Each pod gets its own IP address, and Kubernetes ensures that every pod can reach every other pod directly, even if they’re on different nodes.
When one pod sends traffic to another pod on a different node, the packet first goes through the node’s network interface. The CNI plugin, like Calico or AWS VPC CNI in EKS, handles routing by mapping pod IPs to the correct node and forwarding the traffic across the cluster network.
From the application’s perspective, it feels like everything is on one flat network, but underneath the CNI plugin manages routing, encapsulation, and network policies to make cross-node communication work reliably.

**7 ) How does networking work specifically in AWS EKS?**

In AWS EKS, networking works using the AWS VPC CNI plugin. Unlike some other Kubernetes setups, pods don’t get virtual overlay IPs — instead, each pod gets a real IP address from the VPC subnet.
When a node starts, AWS attaches multiple ENIs and secondary IPs to it. These IPs are then assigned to pods. Because pods use actual VPC IPs, they can communicate with other pods, AWS services, or even on-prem systems directly through the VPC routing tables without NAT.
Security groups, route tables, and NACLs still control traffic, so networking in EKS integrates closely with standard AWS networking rather than creating a completely separate network layer.


**8 ) Explain deployment vs statefulset vs daemonset.**

 
Deployment is used for stateless applications where every pod is identical and interchangeable — like a frontend or backend service. If a pod dies, a new one comes up and works the same way.
StatefulSet is used for stateful applications like databases, where each pod has its own identity, its own storage, and order matters. For example, in a MySQL cluster, the master and replicas need fixed addresses to communicate with each other. StatefulSet gives each pod a stable name like mysql-0, mysql-1 and that name stays the same even after a restart
DaemonSet is used when we want one pod to run on every node, like for monitoring agents, log collectors, or security tools.
a real production examples version
In production, we use Deployments for stateless services like frontend apps, backend APIs, or microservices because they can scale easily and don’t need fixed identities.
StatefulSets are used for things like databases, Kafka, or Elasticsearch where each pod needs stable storage, a fixed name, and ordered startup so the data remains consistent.
DaemonSets are used for tools that must run on every node, like log collectors, monitoring agents, or security scanners, so each node always has one instance running.

 
**9 )why does that actually matter? What problem does it solve compared to random pod names in a Deployment?**

Because each pod gets a stable DNS name tied to that fixed name. So other services or pods always know exactly where to reach a specific replica. Also each pod keeps its own dedicated storage — if mysql-0 restarts, it reconnects to the same data volume, not someone else's. In a Deployment, pods get random names and don't have dedicated storage, which would corrupt a database.

 
**10 ) Explain service types.**

Kubernetes has mainly four service types.

 
ClusterIP is the default and exposes the service only inside the cluster. It’s used when services need to talk to each other internally.

NodePort exposes the service on a port of every node’s IP, so it can be accessed from outside using the node’s IP and port.

LoadBalancer creates an external load balancer from the cloud provider and exposes the service publicly. This is commonly used in production for external traffic.

ExternalName is used to map a Kubernetes service to an external DNS name, so the cluster can access outside services using a Kubernetes-style name.
a real production usage explanation

In production, we usually use ClusterIP for internal communication between microservices, like when a backend service talks to a database or another API.
For public access, we typically use LoadBalancer because it integrates with the cloud provider and gives us a proper external endpoint for users.
NodePort is rarely used in production directly, but it can be useful for testing or when setting up an external load balancer manually.
ExternalName is used when a service inside Kubernetes needs to access something outside the cluster, like a managed database or third-party API, using a consistent service name.

**11. What happens when traffic hits a service?**

 
When traffic hits a Kubernetes Service, it first reaches the service’s virtual IP. Kubernetes then forwards that request to one of the pods behind the service.
This routing is handled by kube-proxy, which uses iptables or IPVS rules on each node to load-balance traffic across the available pods. The service keeps track of healthy pod endpoints, so traffic only goes to running pods.
So from the client’s perspective, they talk to one stable service IP, and Kubernetes automatically distributes the requests to the correct pods in the background.
a step-by-step flow including Ingress and LoadBalancer

 
   When a user sends a request from the internet, it first reaches the cloud LoadBalancer created by Kubernetes.
 The LoadBalancer forwards the traffic to the Ingress controller running inside the cluster.
 The Ingress then checks its routing rules, like domain name or path, and decides which service should handle the request.
The request is sent to that Kubernetes Service, which acts as a stable internal endpoint.
Finally, the service forwards the traffic to one of the healthy pods running the application.
So the flow is: Internet → LoadBalancer → Ingress → Service → Pod.

 
**12) Explain Ingress flow end to end.**

When a user makes a request to a domain, it first reaches the external LoadBalancer created for the Ingress controller.
 
That LoadBalancer forwards the request to the Ingress controller pod inside the cluster.
The Ingress controller reads the Ingress rules, which define routing based on hostnames or paths, and decides which Kubernetes Service should receive the request.
The request is then sent to that Service, which forwards it to one of the healthy pods running the application.
 
So the full flow is: User → DNS → LoadBalancer → Ingress Controller → Service → Pod.


**13 ) what is HPA in Kubernetes:**

 
HPA stands for Horizontal Pod Autoscaler. It automatically scales the number of pod replicas up or down based on real time metrics like CPU and memory usage.
Here is how it works:
 
 First we install the metrics server in the cluster. It collects CPU and memory usage data from all pods and nodes.
 HPA continuously watches these metrics.
 We define thresholds in the HPA config — like scale up if CPU usage exceeds 70%.
 We also set a minimum and maximum replica count — for example minimum 2 pods and maximum 10 pods.
 If traffic increases and CPU crosses the threshold, HPA automatically creates new pod replicas.
 When traffic reduces, HPA scales down the pods to save resources.
 
This way our application handles traffic spikes automatically without manual intervention.
a real production scenario explanation
In production, we had a backend API that handled user requests, and traffic used to spike during peak hours. We configured HPA to monitor CPU usage with a target of around 60%.
 
When traffic increased, pod CPU usage crossed the threshold, so HPA automatically increased the number of pods to handle the load. Once traffic dropped later, HPA gradually scaled the pods back down to save resources.
 
This helped us handle traffic spikes without manual intervention and kept the application responsive while optimizing infrastructure usage.

**14. Diffrence B/w Application load balancer(ALB) and Network Load Balancer(NLB)**

 
“ALB works at Layer 7 and is used for HTTP/HTTPS traffic. It supports intelligent routing like path-based and host-based routing, which is ideal for microservices architectures.
NLB works at Layer 4 and handles TCP/UDP traffic with very high performance and low latency, but it cannot inspect HTTP requests.”
ALB → for Ingress / web traffic
NLB → for exposing services requiring TCP performance
 

 **15) How will you check the threshold of the pods. And prepare for auto scaling**
 
To check pod thresholds, I monitor CPU and memory usage using kubectl top pods, which requires the metrics server. Based on the resource usage, I define CPU and memory requests in the deployment. Then I configure a Horizontal Pod Autoscaler that scales pods automatically when resource utilization crosses a defined threshold, for example 70% CPU usage. I verify autoscaling using kubectl get hpa and kubectl describe hpa.

 
**16) Explain how you implemented auto-scaling on EKS. How does HPA work?**
 
On EKS, we use two levels of auto-scaling:
HPA (Horizontal Pod Autoscaler): Watches metrics like CPU/memory utilisation (via Metrics Server). When CPU on a pod exceeds, say, 70%, HPA creates more pod replicas. When load drops, it scales back. I defined HPA in YAML specifying minReplicas, maxReplicas, and the target CPU percentage.
Cluster Autoscaler: When HPA wants to create pods but there's no node capacity, Cluster Autoscaler triggers AWS Auto Scaling Groups to add EC2 nodes. When nodes are underutilised, it drains and terminates them to save cost.
In our retail app project, we configured HPA targeting 60% CPU. During load tests, it scaled from 2 to 8 replicas in about 90 seconds, then scaled back as load dropped — with zero downtime."

 
**17) A pod is in CrashLoopBackOff state. How do you debug it?**

CrashLoopBackOff means the container starts, crashes, and Kubernetes keeps restarting it with exponential backoff. My debugging steps:
Step 1: kubectl describe pod <pod-name> — Check Events section for OOMKilled (out of memory), image pull errors, or failed health checks.
Step 2: kubectl logs <pod-name> --previous — The --previous flag shows logs from the crashed container, not the new one.
Step 3: Check resource limits — if CPU/memory limits are too low, the container might be getting killed by OOM killer.
Step 4: Check liveness probe configuration — if the probe timeout is too short or the path is wrong, K8s kills the pod thinking it's unhealthy.
Step 5: kubectl exec -it <pod-name> -- /bin/sh to get a shell if the pod starts momentarily.
Common root causes I've seen: wrong env variables (DB connection string), misconfigured secrets, or app failing to start because of a dependency not being ready.

 
**18)What are Kubernetes Namespaces and why do you use them?**

"Namespaces provide virtual clusters within a physical cluster — they're used to isolate resources and apply policies per team or environment.
 
Why we use them:
1. Environment isolation: Separate namespaces for dev, staging, and production on the same cluster — each with its own resource quotas.
2. Team isolation: Different teams own different namespaces with RBAC policies so team-A can't accidentally touch team-B's pods.
3. Resource quotas: Limit CPU/memory per namespace so one team can't starve others.
4. Network policies: Restrict traffic between namespaces for security.
 
In our EKS setup, we have namespaces like production, staging, monitoring (for Prometheus/Grafana), and argocd — each with appropriate RBAC.

 
**19 ) What is Helm and how have you used it?**

"Helm is the package manager for Kubernetes — it lets you define, install, and upgrade applications using charts, which are templated K8s YAML files.
 
Instead of maintaining separate YAML files per environment, Helm lets you have one chart with a values.yaml per environment (values-dev.yaml, values-prod.yaml) where you override things like replica count, image tag, and resource limits.
 
How I use it: I use Helm charts for deploying applications like Prometheus, Grafana, and ingress-nginx. For our own microservices, I've created custom Helm charts that ArgoCD deploys by referencing the chart from a Git repo. This way environment-specific config is cleanly separated from the chart template.
 
Key commands I use: helm install, helm upgrade --install, helm rollback, helm template (to debug generated YAML before applying).

 
**20) What is Kubernetes and why use it over plain Docker?**

Kubernetes is a container orchestration tool. Docker alone is fine for running containers on a single machine, but in production you need to manage hundreds of containers across multiple machines. Kubernetes does that automatically — it handles deployment, scaling, self-healing, load balancing, and rolling updates. That's why teams use it over plain Docker.
 

 
**21) How does auto-healing work in Kubernetes?**


Each node has a component called kubelet. It constantly monitors the containers running on that node and reports their status to the API server. If a container or pod goes down, the controller manager detects that the actual state doesn't match the desired state. It then creates a new pod, and the scheduler decides which node to place it on. That's how Kubernetes automatically heals itself."
Key thing to remember:
kubelet ==> Health monitoring on each node
API Server ==> Central communication hub
Controller Manager ==> Detects desired vs actual state mismatch
Scheduler ==> Decides which node gets the new podkube-proxyHandles network rules
 

**22. A pod is in Pending state and it has been like that for 10 minutes. How would you troubleshoot it? What are the possible reasons?**

 
"A Pending pod means the scheduler cannot find a suitable node to place it. Here is how I troubleshoot:
Step 1 — I run kubectl describe pod -n and look at the events section. It usually tells me the exact reason — either insufficient resources, node selector mismatch, or waiting for PVC.
Step 2 — I check node resources using kubectl top nodes to see if any node has available CPU and memory.
Step 3 — I check for common causes:
Insufficient resources — pod requested more CPU or memory than any node has available. I either increase node resources or reduce pod requests.
Node selector or affinity mismatch — the pod requires specific labels but no node has them. I check the pod spec and either add labels to nodes or remove the requirement.
PVC not bound — I check kubectl get pvc to see if the persistent volume claim exists and is bound.
Taints on nodes — I check kubectl describe node and see if the pod has matching tolerations.
Step 4 — Once I fix the issue, the scheduler automatically places the pod."
| `kubectl describe pod <name> -n <ns>` | Check pending reason |
| `kubectl top nodes` | Check node resource availability |
| `kubectl get pvc -n <ns>` | Check if PVCs exist and are bound |
| `kubectl describe node <node-name>` | Check taints and resource allocation |
 

**23. ClusterIP vs NodePort vs LoadBalancer**

 
There are three main service types:
ClusterIP — This is the default service type. It gives the service an internal IP address that only works inside the cluster. Pods communicate with each other using this IP. Example: A backend pod talks to a database pod using the database service's ClusterIP.
NodePort — This exposes the service on a port on every node in the cluster. You can access it from outside the cluster using any node's IP and a port between 30000 and 32767. Example: If you want to access a web app from your laptop without a load balancer, you use NodePort.
LoadBalancer — This creates an external load balancer managed by your cloud provider like AWS or GCP. It distributes external traffic across all nodes and is production-ready. Example: A production web app that needs high availability and automatic traffic distribution.
So use ClusterIP for internal communication, NodePort for testing or small apps, and LoadBalancer for production apps that need external access.
Quick Comparison:
 
ClusterIP ==> Inside cluster only ==> Internal communication ==> Free
NodePort ==> Outside cluster ==> Testing, small apps ==> Free
LoadBalancer ==> Outside cluster ==> Production apps ==> Paid
 

**24. ConfigMap vs Secret:**

 
"Both ConfigMap and Secret are used to separate configuration from application code, but they serve different purposes.
ConfigMap is used to store non-sensitive configuration data like app settings, environment names, port numbers, or config files. For example, storing the database host URL or the app environment like staging or production.
Secret is used to store sensitive data like passwords, API keys, JWT tokens, and database credentials. Secrets are base64 encoded so they are not stored in plain text.
 
We use them instead of hardcoding values in deployment YAML files so that we don't accidentally push sensitive data to Git.
We can use them inside a pod in two ways:
As environment variables — the pod reads the value as an env variable at runtime
As mounted files — the value is mounted as a file inside the container"
Quick Comparison:
 
Feature  ConfigMap  Secret
Data type Non-sensitive Sensitive
Encoding Plain text Base64 encoded
Example App port, env name, config file Password, API key, JWT token
Usage Env variable or mounted file Env variable or mounted file

**25. Taints and Tolerations**


Taints and Tolerations are used to control which pods can be scheduled on which nodes.
A Taint is applied on a node — it tells the scheduler to reject all pods that don't have a matching toleration. A Toleration is applied on a pod — it gives the pod permission to be scheduled on a tainted node.
A real world example — if you have dedicated high memory nodes for databases, you taint those nodes so that only database pods with matching tolerations can run there. Regular application pods cannot use those nodes. This helps in resource management and workload isolation.


**26 Liveness vs Readiness Probe**

Liveness probe checks if app is alive. If it fails, Kubernetes restarts the container. Use case — app is stuck in deadlock.
Readiness probe checks if app is ready for traffic. If it fails, Kubernetes removes pod from endpoints but does not restart it. Use case — app is still loading data on startup.


**27. What is Kubernetes and how does it work?**

Senior answer:

Kubernetes is a container orchestration platform used to automate deployment, scaling, networking, and lifecycle management of containerized applications.

Internally it follows a control plane and worker node architecture.

The control plane contains components like:

* API Server → entry point for all requests
* etcd → stores cluster state
* Scheduler → decides pod placement
* Controller Manager → maintains desired state

Worker nodes run:

* kubelet
* kube-proxy
* container runtime

Flow:

When I deploy YAML, request goes to API server → stored in etcd → scheduler selects node → kubelet creates containers → controller continuously ensures desired and actual state match.

In production on EKS, I manage deployments through Helm and CI/CD pipelines where Kubernetes automatically handles rolling updates and self-healing.

==========================

1. Job

A Kubernetes Job is used to run a task until it completes successfully.
It creates one or more Pods and ensures the required number of successful completions.
Example: database migration, batch processing, or one-time data processing.

2. CronJob

A CronJob runs Jobs automatically according to a defined schedule, similar to Linux cron.
For example, 0 2 * * * can run a backup every day at 2 AM.
CronJob → Job → Pod.

3. NodeSelector

NodeSelector is used to schedule a Pod only on nodes having specific labels.
It is simple and provides basic node selection.
Example: schedule a Pod only on nodes labeled disktype=ssd.

4. Node Affinity

Node Affinity is an advanced version of NodeSelector.
It supports complex conditions and both required and preferred scheduling rules.
It is useful when Pods need to be placed on specific types of nodes based on business or infrastructure requirements.

5. Taint

A taint is applied to a Kubernetes node to prevent Pods from being scheduled on it.
It tells the scheduler that only Pods with a matching toleration should run there.
Example: reserving specific nodes only for database workloads.

6. Toleration

A toleration is configured on a Pod to allow it to run on a node with a matching taint.
It does not force the Pod onto that node; it only allows scheduling there.
For dedicated workloads, we commonly use Taint + Toleration + Node Affinity.

7. Cordon

Cordon marks a node as unschedulable.
New Pods will not be scheduled on that node, but existing Pods continue running normally.
It is commonly used before node maintenance or upgrades.

8. Drain

Drain safely removes workloads from a node by evicting its Pods and preventing new Pods from being scheduled there.
Pods managed by Deployments/ReplicaSets can be recreated on other suitable nodes.
Before draining production nodes, check PDB, replicas, capacity, DaemonSets, and StatefulSets/PVs.

9. Uncordon

Uncordon makes a previously cordoned node schedulable again.
New Pods can be scheduled onto the node after uncordoning.
It does not automatically move existing Pods back to that node.
