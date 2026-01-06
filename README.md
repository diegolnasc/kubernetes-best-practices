# Kubernetes Best Practices 101

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

In most cases, you learn to use platforms to meet the current business need or on standalone projects. The silver lining
is the encouragement of learning and at some point this becomes knowledge, however, hands-on work can lead to cuts in
paths that later cause a series of problems in productive environments. Therefore, the purpose of this guide is to help
with the learning curve, helping to prepare a more stable, reliable and functional environment.

## Documentation

* 📙 [Kubernetes Official Documentation](https://kubernetes.io/docs/home/)
* 📙 [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/docs/how-to)
* 📙 [Amazon Elastic Kubernetes Service (EKS)](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
* 📙 [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/)

## Table of Contents

- [Container](container/container.md)
- [Cluster](#cluster)
	+ [Infrastructure](#infrastructure)
	+ [Cost Optimization](#cost-optimization)
	+ [Namespace](#namespace)
- [Basics](#basics)
    + [Security](#security)
    + [RBAC](#rbac)
    + [Labels](#labels)
    + [Probes](#probes)
    + [Resources](#resources)
    + [Scalability](#scalability)
    + [Pod Disruption Budget](#pod-disruption-budget)
    + [Affinity and Anti-Affinity](#affinity-and-anti-affinity)
    + [Taints and Tolerations](#taints-and-tolerations)
    + [Deployment](#deployment)
    + [Shutdown](#shutdown)
- [Networking](#networking)
    + [Network Policies](#network-policies)
    + [Service Mesh](#service-mesh)
- [Secrets Management](#secrets-management)
- [Observability](#observability)
- [Deployment and Review](#deployment-and-review)
    + [GitOps](#gitops)

---
---

## Cluster
---

#### Infrastructure

I don't intend to go into infrastructure best practices, but we can say that the standard 'paperwork', private VPC,
multiple networks, firewall rules etc. also apply for a kubernetes cluster. The points that need to be highlighted are:

- **Network**: Set aside a network for the cluster and make sure there is enough space for the pods and services. So
  find out how many pods per node you want to use and make calculations in CIDR based on that. It's worth noting that
  each cloud provider can have its own variation and rules, so check the documentation. Practical example:
  The [GCP](https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr#cidr_ranges_for_clusters) reserves
  double the IP for specific ranges based on the maximum pods per node, starting from 8 to 110. So, a direct translation
  is::
	- Subnetwork range (CIDR): Maximum number of nodes.
	- Range for pods (CIDR): Maximum number of pods based on the maximum number of pods per node. Example: A pod CIDR
	  range /19 supports 256 nodes in a configuration of 16 maximum pods per node. Consequently, a subnetwork range (
	  item above) of at least /24 is required.
	- Range for services (CIDR): Maximum number of services based on maximum number of pods per node.

- **Private**: Leave nodes and API restricted and/or inaccessible on the internet. So, use private clusters and, if your
  team is large enough, separate (project/account, private VPC...) them into different environments (development,
  production...).
- **Infrastructure as Code**: Keep all infrastructure versioned and well-documented with tools
  like [Terraform](https://www.terraform.io/), [CloudFormation](https://aws.amazon.com/cloudformation/?nc1=h_ls)
  or [Ansible](https://www.ansible.com/). For deployment management, I particularly think applications deserve a proper
  CD tool.

#### Cost Optimization

- **Cloud**:
	+ Pay attention to the committed use discounts plans.
	+ Choose the right type of machine, it's quite common to have discounts for specific types. For instance, GCP E2
	  types offer you 31% savings compared to the default N1.
	+ Some processes (like batch/job) don't need to be close to the user, so use the region with the most interesting
	  cost. Of course, be wary of transfers between regions and the entire lifecycle of your processes.
	+ For each application deployed, we need 10 more to monitor it. Jokes aside, be aware of the cost of monitoring.
- **Node-pools**:
	+ If you have a robust environment, create specific node-pools according to the characteristics of the applications.
	  A good example is having node-pools high memory, high cpu, and so on. The main purpose is to direct the
	  applications to the correct nodes and use as much resource as possible, as we don't want to have too much resource
	  idle.
	+ Some applications are not as sensitive or don't need to be 24/7 online. If possible, create spot/preemptible node
	  pools and only pay for a small chunk of the instance. It's important to note that there are lots of cool
	  projects ([estafette](https://github.com/estafette/estafette-gke-preemptible-killer)) to play, it's worth taking a
	  look.
	+ Enable auto-scaling to reduce cost at times with fewer users.

#### Namespace

> Use namespace profusely!

Simply put, the namespace is a way to organize objects, products and teams in Kubernetes. Namespaces provide granularity
to separate teams and/or products, in large companies, it's quite common not to know all teams, as well as development
models. Therefore, it's important to isolate and have the freedom to build a fast and secure development flow,
respecting the limits. Of course, it's important to analyze each environment, in a small company, we don't need so much
logical separation, because everyone knows each other and the cost has to make sense with the business.

Here is an example of how to do it (if possible, set quota for each namespace):

```
kubectl create namespace my-first-namespace
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-first-namespace
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 10Gi
    limits.cpu: "20"
    limits.memory: 20Gi
```

## Basics
---

#### Security

Just as we want to separate teams and/or products into namespaces to "walk" freely, we also need to be responsible with
security in the cluster. In other words, we don't want a security breach to happen that spreads all over the cluster,
after all, behind the cluster we have baremetal susceptible to this.

**Pod Security Standards (PSS)**

> ⚠️ **Note**: Pod Security Policy (PSP) was deprecated in Kubernetes 1.21 and removed in 1.25. Use Pod Security Admission (PSA) instead.

Kubernetes provides three built-in [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/):

| Profile | Description |
|---------|-------------|
| **Privileged** | Unrestricted, widest permissions (for system/infrastructure) |
| **Baseline** | Minimal restrictive, prevents known privilege escalations |
| **Restricted** | Heavily restricted, follows hardening best practices |

Apply Pod Security to namespaces using labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

Always ensure:
- Don't run containers with root permission
- Use read-only root filesystems when possible
- Drop all capabilities and add only what's needed
- Set `allowPrivilegeEscalation: false`

For detailed container security practices, see the [Container Security Guide](container/container.md#security).

#### RBAC

Role-Based Access Control (RBAC) is essential for managing who can do what in your cluster. Follow the **principle of least privilege**.

**Key concepts:**
- **Role/ClusterRole**: Defines permissions (verbs on resources)
- **RoleBinding/ClusterRoleBinding**: Assigns roles to users/groups/service accounts

```yaml
# Role: Define what can be done
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: my-namespace
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
# RoleBinding: Define who can do it
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: my-namespace
subjects:
  - kind: ServiceAccount
    name: my-service-account
    namespace: my-namespace
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Best practices:**
- Avoid using `cluster-admin` unless absolutely necessary
- Use namespaced Roles instead of ClusterRoles when possible
- Regularly audit RBAC permissions
- Use service accounts per application, not the default

#### Labels

Build a table with mandatory labels to be used on objects deployed in the cluster. Despite being something simple and
trivial, having descriptive labels helps in the maintenance, visualization and understanding of the resource. Therefore,
create a best practices table with
the [recommended](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) labels plus what
your team understands is necessary.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/created-by: controller-manager
```

#### Probes

Kubernetes provides three types of probes to check the health and readiness of your application:

| Probe | Purpose | Failure Action |
|-------|---------|----------------|
| **Liveness** | Is the container running correctly? | Restart container |
| **Readiness** | Is the container ready to receive traffic? | Remove from service endpoints |
| **Startup** | Has the container finished starting? | Block other probes until success |

**Liveness Probe**

In any environment, it's necessary to develop the application thinking about how to check if the health is good. In
Kubernetes, liveness is responsible for this. The probes constantly check the application's health, in case of failure
the container is restarted and, consequently, stops serving requests.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: liveness
  name: liveness-example
spec:
  containers:
    - name: liveness
      image: gcr.io/google-samples/hello-app:1.0
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 3
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
```

**Readiness Probe**

Like Liveness, the readiness probe is responsible for controlling whether the application is ready to receive requests.
When the return is positive, it means that all the processes necessary for the application to work have already been
carried out and it is ready to receive a request.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3
```

**Startup Probe**

For applications with long cold starts (e.g., legacy apps, ML models), use startup probes to avoid premature restarts:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 30  # 30 * 10s = 5 minutes max startup time
```

> 💡 **Tip**: During startup, liveness and readiness probes are disabled until the startup probe succeeds.

> For more details, check the [probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes): [HTTP](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-http-request), [Command](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command) or [TCP](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-tcp-liveness-probe).

#### Resources

Explicitly set resources on each Pod/Deployment, this makes kubernetes have great node and scale management. In
practice, with well defined features, kubernetes will place applications on correct nodes, as well as control the
scalability of node pools and applications, and prevent applications from being killed.

Defining a resource for an application is not a very simple task, however, with time assertiveness starts to appear. A
good way is to use some load testing application, such as [Locust](https://github.com/locustio/locust), and stress the
application and see how resources are being used. At the same time, it is also useful to use a VPA in recommendation
mode to compare the hints with the defined final value.

One suggestion is to set the requested memory value equal to the limit, as for cpu, we can just set the requested value.
This reason is simple, basically memory is a non-compressible resource!

Here is an example of how to do it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: hello-resource
  name: hello-resource
spec:
  containers:
    - name: hello-resource
      image: gcr.io/google-samples/hello-app:1.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "64Mi"
      livenessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 3
        periodSeconds: 1
```

> 💡 **Tip**: Consider not setting CPU limits. CPU is a compressible resource, and limits can cause unnecessary throttling. See [Stop Using CPU Limits](https://home.robusta.dev/blog/stop-using-cpu-limits).

#### Scalability

Choose the scalability model according to the application's characteristics. In kubernetes, it's very common to use a
Horizontal Pod Autoscaler ([HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)) or
Vertical Pod Autoscaler ([VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)).

For most cases, HPA is used with the trigger based on CPU usage. In this case, a good practice to define the target is:

```math
(CPU-HB - safety)/(CPU-HB + growth)
```

Where:

- **CPU-HB**: CPU high-bound is the usage limit on the pod. In most cases, the limit is 100%, but for node-pools that
  have a considerable percentage of idle resource, we can increase the limit.
- **safety**: We don't want the resource to reach its limit, so we set a safety threshold.
- **growth**: Percentage of traffic growth that we expect in a few minutes.

A practical example is an application where we set the limit at 100% usage for cpu, a safety threshold of 15% with an
expected traffic growth of 45% in 5 minutes:

```math
(1 - 0.15)/(1 + 0.45) = 0.58
```

Here is an example of how to do it:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 58
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

> 💡 **Tip**: Use `behavior` to control scale up/down velocity and avoid flapping.

#### Pod Disruption Budget

A Pod Disruption Budget (PDB) limits how many pods can be voluntarily disrupted at a time. This is essential for maintaining availability during:
- Node drains
- Cluster upgrades
- Voluntary evictions

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  # At least 2 pods must always be available
  minAvailable: 2
  # OR: At most 1 pod can be unavailable at a time
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

**Best practices:**
- Always set PDB for production workloads
- Use `minAvailable` for critical services
- Consider replica count when setting values
- PDB only affects **voluntary** disruptions (not crashes/OOM)

#### Affinity and Anti-Affinity

Control where pods are scheduled based on node labels or other pod locations.

**Node Affinity**: Schedule pods on specific nodes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-type
                operator: In
                values:
                  - gpu
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: region
                operator: In
                values:
                  - us-east-1
  containers:
    - name: gpu-app
      image: my-gpu-app:1.0
```

**Pod Anti-Affinity**: Spread pods across nodes/zones for high availability

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          # Hard requirement: Don't schedule on same node
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: my-app
              topologyKey: kubernetes.io/hostname
          # Soft preference: Try to spread across zones
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: my-app
                topologyKey: topology.kubernetes.io/zone
```

> 💡 **Tip**: For simpler zone spreading, use [Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/).

#### Taints and Tolerations

Taints allow nodes to repel pods. Tolerations allow pods to be scheduled on tainted nodes.

**Common use cases:**
- Dedicated nodes for specific workloads (GPU, high-memory)
- Preventing pods from scheduling on control plane nodes
- Spot/preemptible node pools

```bash
# Add taint to a node
kubectl taint nodes node1 dedicated=gpu:NoSchedule
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
  containers:
    - name: gpu-container
      image: gpu-app:1.0
```

**Taint effects:**
| Effect | Description |
|--------|-------------|
| `NoSchedule` | Pods without toleration won't be scheduled |
| `PreferNoSchedule` | Soft version, scheduler tries to avoid |
| `NoExecute` | Evicts existing pods without toleration |

#### Deployment

Regarding ReplicaSet deployment strategies, there are:

- **RollingUpdate**: Starts new container's before deleting old ones.
	+ Pro: No Downtime.
	+ Cons: Deployment can be time-consuming and there is no traffic control between versions.
- **Recreate**: Remove all old containers and start new versions simultaneously.
    + Pro: Remove previous ~~problematic~~ versions quickly.
    + Cons: Downtime may be relevant depending on the cold start of applications.

![deployment-strategy](images/deployment-strategy.png)

Specifically about the means of deployments, we can highlight:

**Blue-Green**:

A blue/green deployment duplicates the environment with two parallel versions, in other words, two versions will be available. It's a great way to reduce service downtime and ensure all traffic is transferred immediately.

To take advantage of this strategy, you need to use extensions (**recommended**) such as service mesh or knative. However, for small environments, we can also do this manually as this reduces the complexity and again the cost has to make good business sense. The image below shows a way to do this manually, once the versions are online, we just need to switch traffic to the new version (green) with a load balancer/ingress.

![deployment-blue-green-strategy](images/blue-green-deploy.png)

**Canary**:

Canary deployment is a relevant way to test new versions without driving all the traffic right away. The idea is to separate a small part of customers for the new version and gradually increase it until the entire flow is validated or discarded.

As well as blue-green, it is also **highly recommended** to use other solutions such as [Argo Rollouts](https://argoproj.github.io/rollouts/), [Flagger](https://flagger.app/), [Istio](https://istio.io/), or [Linkerd](https://linkerd.io/). However, we can also do this something manually as follows:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-app
spec:
  sessionAffinity: ClientIP # It's important to secure the customer's session.
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: NodePort
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
spec:
  replicas: 9
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: my-app
      version: v1
  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
        - name: my-app
          image: gcr.io/google-samples/hello-app:1.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: my-app
      version: v2
  template:
    metadata:
      labels:
        app: my-app
        version: v2
    spec:
      containers:
        - name: my-app
          image: gcr.io/google-samples/hello-app:2.0
```

In this example, we have a service that exposes two deployment versions (1.0 and 2.0), where the first has 9 instances and the second only 1, so it's expected that a large part of the traffic will be directed to the first version. Anyway, it's important to highlight that in order to guarantee the % of traffic, as well as the automated and smarter implementation, it's necessary to use other solutions like the ones mentioned above. Therefore, the example here is just a solution for specific cases that should not be taken as something definitive and ideal.

#### Shutdown

The kubernetes termination cycle is as follows:

1. **Terminating**: All flow is stopped and the pod state goes into terminating.
2. **PreStop Hook**: A termination alert is sent by command or HTTP request to the container to initiate the termination
   process.
3. **SIGTERM Signal**: A termination event is sent for the purpose of warning that the container will be terminated soon.
4. **GracePeriod**: Kubernetes waits for the grace period defined.
5. **SIGKILL**: Well, the timer has run out and the container will be removed.

Based on the cycle above, we need to ensure that our application is prepared to go through with all events and finish in
a good manner without compromising the user experience. Therefore, it's very important to use the preStop hook, SIGTERM
and grace period so that we don't process any more requests and finish the ones that are in progress.

Here is an example of how to configure:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-terminating
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: lifecycle-terminating
      image: nginx:1.25
      lifecycle:
        preStop:
          exec:
            # Give time for load balancer to update before stopping
            command: ["/bin/sh", "-c", "sleep 10 && nginx -s quit"]
```

> 💡 **Tip**: Add a `sleep` in preStop to allow load balancers to stop sending traffic before the app stops accepting connections.

---

## Networking
---

#### Network Policies

By default, all pods can communicate with all other pods. Network Policies allow you to control traffic flow at the IP address or port level (OSI layer 3 or 4).

**Default deny all ingress**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: my-namespace
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
    - Ingress
```

**Allow specific traffic**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-namespace
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
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 8080
```

> ⚠️ **Note**: Network Policies require a [CNI plugin that supports them](https://kubernetes.io/docs/concepts/services-networking/network-policies/#prerequisites) (e.g., Calico, Cilium, Weave Net).

#### Service Mesh

For advanced traffic management, observability, and security, consider a service mesh:

| Solution | Features | Complexity |
|----------|----------|------------|
| [**Istio**](https://istio.io/) | Full-featured, traffic management, mTLS, observability | High |
| [**Linkerd**](https://linkerd.io/) | Lightweight, easy to use, mTLS, observability | Medium |
| [**Cilium**](https://cilium.io/) | eBPF-based, network policies, observability, service mesh | Medium |

**Benefits:**
- **mTLS**: Automatic encryption between services
- **Observability**: Distributed tracing, metrics, logs
- **Traffic Management**: Canary, A/B testing, fault injection
- **Resiliency**: Retries, timeouts, circuit breakers

---

## Secrets Management
---

Never store secrets in plain text in your manifests or Git repositories!

**Options for secrets management:**

| Solution | Description | Complexity |
|----------|-------------|------------|
| **Kubernetes Secrets** | Built-in, base64 encoded (not encrypted at rest by default) | Low |
| **Sealed Secrets** | Encrypt secrets in Git, decrypt in cluster | Low |
| **External Secrets Operator** | Sync secrets from external providers (AWS, GCP, Vault) | Medium |
| **HashiCorp Vault** | Full-featured secrets management | High |
| **Cloud KMS** | Cloud provider key management (GCP KMS, AWS KMS) | Medium |

**Sealed Secrets example:**

```bash
# Install sealed-secrets controller
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Seal a secret
kubeseal --format=yaml < my-secret.yaml > my-sealed-secret.yaml
```

**External Secrets Operator example:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-external-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-manager
    kind: SecretStore
  target:
    name: my-secret
  data:
    - secretKey: password
      remoteRef:
        key: my-app-password
```

**Best practices:**
- Enable [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) for Kubernetes Secrets
- Use RBAC to limit secret access
- Rotate secrets regularly
- Audit secret access

---

## Observability
---

The three pillars of observability: **Metrics**, **Logs**, and **Traces**.

| Pillar | Tools | Purpose |
|--------|-------|---------|
| **Metrics** | Prometheus, Grafana, Datadog | Quantitative data over time |
| **Logs** | Loki, ELK Stack, Cloud Logging | Event records for debugging |
| **Traces** | Jaeger, Tempo, Zipkin | Request flow across services |

**Prometheus + Grafana stack:**

```yaml
# ServiceMonitor for Prometheus to scrape your app
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

**Application instrumentation:**

- Use [OpenTelemetry](https://opentelemetry.io/) for vendor-agnostic instrumentation
- Expose `/metrics` endpoint in Prometheus format
- Use structured logging (JSON) to stdout/stderr
- Include correlation IDs in logs and traces

**Best practices:**
- Set up alerting for critical metrics
- Create dashboards for key business and technical metrics
- Use log aggregation for centralized debugging
- Implement distributed tracing for microservices

---

## Deployment and Review

---

Develop a strong CI/CD to ensure all mandatory steps are followed, as well as smooth the deployment flow for all teams.
Mandatory features:

- ✅ Only use images from trusted repositories
- ✅ Use the commit SHA as a tag for the image
- ✅ Scan images for vulnerabilities before deployment
- ✅ Use manifests versioned in Git (GitOps)
- ✅ Make sure all the best practices mentioned here are being followed and disseminated among the teams

#### GitOps

GitOps is a way of implementing Continuous Deployment for cloud-native applications. It uses Git as the single source of truth for declarative infrastructure and applications.

**Popular tools:**

| Tool | Description |
|------|-------------|
| [**Argo CD**](https://argoproj.github.io/cd/) | Declarative GitOps CD for Kubernetes |
| [**Flux**](https://fluxcd.io/) | GitOps toolkit for Kubernetes |

**Benefits:**
- Git as single source of truth
- Automated sync between Git and cluster
- Easy rollbacks (git revert)
- Audit trail of all changes
- Self-healing infrastructure

**Argo CD example:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-manifests.git
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Summary Checklist

Before deploying to production, verify:

- [ ] **Security**: Pod Security Standards applied, non-root user, RBAC configured
- [ ] **Resources**: CPU/Memory requests and limits defined
- [ ] **Probes**: Liveness, readiness, and startup (if needed) probes configured
- [ ] **Scalability**: HPA configured with appropriate thresholds
- [ ] **Availability**: PDB defined, anti-affinity configured
- [ ] **Networking**: Network policies in place
- [ ] **Secrets**: Secrets properly managed, not in plain text
- [ ] **Observability**: Metrics, logs, and traces configured
- [ ] **Labels**: Standard labels applied
- [ ] **Shutdown**: Graceful shutdown handling implemented

---

> 📚 **Additional Resources:**
> - [Kubernetes Documentation](https://kubernetes.io/docs/home/)
> - [CNCF Landscape](https://landscape.cncf.io/)
> - [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/overview/)
> - [Production Best Practices Checklist](https://learnk8s.io/production-best-practices)
