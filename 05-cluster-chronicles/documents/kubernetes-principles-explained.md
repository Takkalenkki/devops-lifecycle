# Kubernetes Principles

This documentation explains Kubernetes principles.


## Kubernetes Architecture

Key Components and their roles:

* API Server:
    * The front door to the cluster. All communication (kubectl, internal components, CI/CD tools) goes through it. It validates and processes REST requests, then persists state to etcd.
* etcd:
    * A distributed key-value store that holds the entire cluster state. It's the source of truth. If etcd dies, the cluster loses its mind, which is why production clusters always run etcd with redundancy.
* Controller Manager:
    * Runs control loops (controllers) that watch the cluster state and reconcile it toward the desired state. The Deployment controller, ReplicaSet controller, etc. all live here.
* Scheduler:
    * Watches for newly created pods with no assigned node and selects the best node for them based on resource requests, affinity rules, taints/tolerations, and topology constraints.
* Kubelet:
    * The agent that runs on every node. It receives pod specs from the API server and ensures the containers described in those specs are running and healthy.

## Kubernetes vs VM-based Deployments

|  | Kubernetes | Traditional VMs |
|--|------------|-----------------|
| Density | High - many containers per node | Low - one app per VM is common |
| Startup time | Seconds | Minutes |
| Resource efficiency | Requests/limits allow fine-grained allocation | VMs reserve full resources |
| Self-healing | Built-in (restarts, rescheduling) | Manual or scripted |
| Scaling | HPA/VPA/cluster autoscaler | Usually manual or slow automation |
| Drawbacks | Steep learning curve, complex networking, stateful apps are harder | Simpler mental model, better isolation, easier for legacy apps |

The trade-off is operational complexity for density and agility. VMs still win for workloads requiring strong isolation (compliance, multi-tenancy) or running non-containerizable legacy software.

## Namespaces

Namespaces provide logical partitioning within a cluster. Benefits:

* Isolation:
    * Teams or environments (dev/staging/prod) don't stomp on each other.
* RBAC scope:
    * You can grant a team access to only their namespace.
* Resource quotas:
    * Limit CPU/memory consumption per namespace
* Network policies:
    * Can be scoped to namespaces

For this project, I used `apps`, `ci`, `monitoring`, `logging`, `ingress-nginx` and `cert-manager` to separate the components.

## Deployment vs StatefulSet

|  | Deployment | StatefulSet |
|--|------------|-------------|
| Pod identity | Ephemeral, interchangeable | Stable, sticky (pod-0, pod-1...) |
| Storage | Shared or ephemeral | Each pod gets its own PVC |
| Scaling| Any order | Ordered, sequential |
| Use case | Stateless apps (frontend, APIs) | Databases, message queues, Elasticsearch

App doesn't care which pod handles a request and doesn't need to remember anything locally -> `Deployment`. Pods that have individual identity or need their own persistent disk -> `StatefulSet`.

For that reason, CI/CD pipeline along with monitoring and logging stacks use `StatefulSets`.

## Kubernetes Networking Model

Kubernetes enforces a flat networking model with three rules:
1. Every pod gets its own IP address
2. Pods on different nodes can communicate with each other directly (no NAT)
3. Agents on a node can communicate with all pods on that node

Cross-node pod communication is handled by a CNI plugin (Flannel, Calico, Cilium etc.) which sets up overlay or underlay networking. In Minikube, this is handled automatically.

kube-proxy runs on every node and maintains `iptables` (or IPVS) rules that map `Service` virtual IPs to actual pod IPs. When a pod hits a ClusterIP service, kube-proxy's rules intercept the traffic and forward it to a healthy backend pod. This is how load balandcing works at the kernel level.

## Service Types

| Type | Scope | When to use |
|------|-------|-------------|
| ClusterIP | Internal only | Default. Pod-to-pod communication inside the cluster (e.g. backend -> database) |
| NodePort | Exposes on each node's IP at a static port | Quick testing, simple external access without a load balancer |
| LoadBalancer | Provisions a cloud load balancer | Production external access on cloud providers (GKE, EKS, AKS) |

For this Minikube setup: `ClusterIP` for internal services (backend, Prometheus, Elasticsearch), exposed externally via an Ingress rather that NodePort/LoadBalancer. This setup is closer to a real production pattern and avoids manual port management chaos.

## Persistent Storage

Without persistent storage, pod restarts mean data loss. This matters for Elasticsearch indices, Grafana dashboards, Jenkins job history, and any database.

* PersistentVolume (PV):
    * The actual storage resource (a disk, NFS share, etc.)
* PersistentVolumeClaim (PVC):
    * A request for storage by a pod. Kubernetes binds a PVC to an appropriate PV.

Access modes:
* `ReadWriteOnce (RWO)`:
    * One node can mount it read/write. Fine for most databases.
* `ReadOnlyMany (ROX)`:
    * Many nodes can mount it read-only. Good for shared config/asset distribution.
* `ReadWriteMany (RWX)`:
    * Many nodes can mount it read/write simultaneously. Requires a network filesystem (NFW, CephFS). Needed if multiple pods on different nodes need to share writable storage.

Data loss on rescheduling: If a pod using `RWO` is rescheduled to a different node, the PV may not be available there.

Mitigation strategies:
* Use `NodeAffinity` on the PV to pin it to a node.
* Use a distributed storage solution (Rook/Ceph, Longhorn) that makes data available cluster-wide.
* For `StatefulSet`s, Kubernetes will try to reschedule pods back to the same node where their PV lives.

## Kubernetes Probes

```yaml
livenessProbe:     # Is the container alive? Restart it if this fails
readinessProbe:    # Is the container ready to serve traffic? Remove from Service endpoints if not
startupProbe:      # Give slow-starting containers time before liveness kicks in
```

A common mistake is making `livenessProbe` too aggressive. If it fires before the app is ready, you get a restart loop. `startupProbe` prevents this by disabling liveness/readiness checks until the app signals it's started.

## Resource Requests and Limits

```yaml
resources:
  requests:
    memory: "128Mi"   # Scheduler uses this to find a node with enough room
    cpu: "250m"
  limits:
    memory: "256Mi"   # Hard ceiling
    cpu: "500m"
```

* CPU limit exceeded -> The container is throttled (slowed down), not killed.
* Memory limit exceeded -> The container is OOMKilled (Out-Of-Memory-Killed) immediately. This is visible as `OOMKilled` in `kubectl describe pod`.

Requests should always be set. Without them the scheduler is flying blind and you risk noisy-neighbor problems.

## Init Containers

Init container run to completion before the app containers start.

Use cases:
* Wait for dependency:
    * `until nslookup postgres-service; do sleep2; done` for example ensures that the database is reachable before the app starts.
* Database migrations:
    * Run `flask db upgrade` before the main app boots.
* Config preparation:
    * Fetch secrets from Vault and write them to a shared volume.
* Permission fixing:
    * `chown -R 1000:1000 /data` on a PV before the app mounts it.

They share volumes with the main container but run with a separate image, keeping the app image lean.

## Network Policies

By default, all pods can talk to all pods. Network Policies lock them down:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-metrics-only-prometheus
  namespace: apps
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090
```

This Policy only allows monitoring stack to access `/api/metrics` endpoint of the backend app for Prometheus-friendly metrics. Frontend uses `/api/info` endpoint to display the legacy metrics. This is done to separate the monitoring traffic from the user generated traffic.

## RBAC

RBAC controls who can do what to which resource.

Key objects:
* `Role`/`ClusterRole`:
    * Defines permissions (verbs on resources).
* `RoleBinding`/`ClusterRoleBinding`:
    * Binds a Role to a user/group/ServiceAccount

Example: Give Jenkins' ServiceAccount permission to deploy to the `apps` namespace only:
```yaml
kind: Role
metadata:
  name: jenkins-deploy
  namespace: apps
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

Never give a CI/CD tool `cluster-admin`. Least privilege applies.

## Kubernetes Secrets

Secrets store sensitive data base64-encoded (not encrypted by default; enable encryption at rest in production). Mount them as:
```yaml
# As environment variable
env:
  - name: ELASTIC_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elasticsearch-credentials
        key: password

# As a volume
volumes:
  - name: credentials
    secret:
      secretName: elasticsearch-credentials
```

Volume mounting is generally preferred because env vars can leak through crash logs and child processes.

## Kubernetes Operators

An Operatior is a controller that encodes operational knowledge about a specific application. Instead of manually managing a database cluster's failover, backup, and scaling, the Operator does it by watching custom resources.

It works by:
1. Defining a CustomResourceDefinition (CRD) (e.g. `ElasticsearchCluster`)
2. Running a controller that watches those CRDs and reconciles cluster state

## Minikube Limitations vs Production

| Feature | Minikube | Production |
|---------|----------|------------|
| MultiNode | Limited (possible but clunky) | Native |
| LoadBalancer Services | Needs `minikube tunnel` | Cloud-native |
| Persistent storage | `hostPath` by default | Cloud disks, Ceph, NFS |
| HA control plane | No | Yes (3+ control plane nodes) |
| Resource scale | Single machine | Many nodes |
| Network policies | Depends on CNI addon | Mature CNI options |
| Node autoscaling | No | Cluster Autoscaler |

## Rolling Updates and Rollbacks

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # How many extra pods can exist during update
    maxUnavailable: 0  # How many pods can be down during update
```

This ensures zero-downtime deployments. Rollback:
```sh
kubectl rollout undo deployment/backend
kubectl rollout undo deployment/backend --to-revision=3  # specific version
kubectl rollout history deployment/backend               # see history
```

A failed rollout (pods not becoming ready) will stall and not proceed. `readinessProbe` is the safety gate here.

## Troubleshooting

CrashLoopBackOff:
```sh
kubectl describe pod <pod>        # check Events section for clues
kubectl logs <pod> --previous     # logs from the crashed container
kubectl logs <pod> -c <container> # specific container in multi-container pod
```

Look for:
* Bad env vars
* Missing secrets/ConfigMaps
* OOMKilled
* Failed liveness probe
* Bad entrypoint

Pending pods:
```sh
kubectl describe pod <pod>  # "Insufficient cpu/memory" or "no nodes available"
kubectl get nodes           # are nodes Ready?
kubectl describe node <node> # check Allocatable vs Requests
```

Common causes:
* Resource request too high
* All nodes tainted
* PVC not bound
* Image pull failing

## HPA

Horizontal Pod Autoscaler (HPA) automatically scales pod replicas based on observed metrics (CPU, memory, or custom metrics).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: apps
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

The implemented HPA scales the frontend pods up to 10 replicas if the CPU utilization is above 50% of the resource limit.

Note that the HPA requires the Metrics Server addon to function. This can be enabled with:
```sh
minikube addons enable metrics-server
```

After enabling, run a load-generator pod:
```sh
# Basic load-generator pod with 50 concurrent workers and 100 req/s for two minutes
kubectl run load-generator --image=williamyeh/hey --restart=Never -- -z 120s -c 50 -q 100 http://frontend.apps.svc.cluster.local

# Monitor with:
# 1.
kubectl get hpa -n apps frontend-hpa -w
# 2.
kubectl get pods -n apps -w
```

You should start seeing `TARGETS` climb above 50% and then `REPLICAS` increase. After the load stops, HPA will scale back down after about 5 minutes.

It is recommended to set `replicas` in frontend `Deployment` manifest to the same as HPA `minReplicas` to prevent them from clashing.

## Jenkins Image Scanning

Integrated Trivy (lightweight) into the pipeline (Jenkinsfile):
```groovy
// Backend image scanning
stage('Scan Backend') {
    when {
        anyOf {
            changeset "metrics-app/backend/**"
            expression { params.TEST }
        }
    }
    steps {
        container('trivy') {
            sh """
            trivy image \
                --exit-code ${env.BRANCH_NAME == 'main' ? '1' : '0'} \
                --severity HIGH,CRITICAL \
                --format table \
                ${REGISTRY}/metrics-backend:${IMAGE_TAG}
            """
        }
    }
}
```

`--exit-code ${env.BRANCH_NAME == 'main' ? '1' : '0'}` causes the build to fail if HIGH or CRITICAL vulnerabilities are found in the `main` branch.

Jenkins pipeline stages:
1. Checkout SCM
2. Retrieve the Registry IP dynamically
3. Compute Tag for the app versioning
4. Build Docker images and run integrated tests
5. Push to registry
6. Scan images
7. `kubectl set image` call for the recently built images
8. Verify rollout with `kubectl rollout status`

If any of the stages fail, the pipeline is halted and the app is not updated to prevent deploying a broken/insecure application.

## Additional Technologies

### Local container registry

A container registry is deployed inside the cluster so the CI/CD pipeline can push and pull images without an external dependency.

Minikube's built-in registry runs on a `NodePort` and requires configuring Docker to treat it as an insecure registry, since it doesn't have a valid TLS certificate by default. The `scripts/start.sh` handles this with the `--insecure-registry` flag passed to Minikube on startup.

In production this would be replaced by a managed registry (ECR, GCR, Artifact Registry) with image signing and access controls.

### Helm for Third-Party Deployments

Helm is used to deploy third-party components (cert-manager, kube-prometheus-stack, Fluent Bit, Elasticsearch Exporter) rather than managing raw manifests manually.

Benefits over raw manifests:
* Versioned, reproducible deployments (`helm upgrade --install` is idempotent)
* Values files allow environment-specific overrides without forking upstream manifests
* Rollback support (`helm rollback <release>`)
* Dependency management (a chart can pull in its own sub-charts)

First-party app manifests stay as plain YAML. Helm adds overhead for simple, owned resources, but pays off for complex third-party software with many moving parts.

### TLS certificates with `cert-manager`

Helm Chart for creating and managing TLS certificates is used. These certificates are automatically rotated and managed by `cert-manager`, enhancing the security.

### Scripts

Three scripts are used to automate the cluster creation and deployment. These scripts are run in order:

1. Start Minikube with necessary additional flags (`scripts/start.sh`)
2. Create necessary secrets, either from an env-file or prompted interactively (`scripts/create-secrets.sh`)
3. Deploy the cluster automatically by applying the manifests and using Helm charts (`scripts/deploy.sh`)

If the deployed cluster is stopped and needs to be started again (`minikube stop`), only the first script is necessary to get the cluster up.