# Kubernetes

Kubernetes (K8s) is an open-source container orchestration platform that automates the deployment, scaling, networking, and management of containerized applications across clusters of machines. It abstracts away the underlying infrastructure and provides a declarative, resilient, and self-healing system, ensuring applications run consistently and reliably in any environment—on-premises, cloud, or hybrid.

## Problems Before Kubernetes

- Manual Deployment – Developers had to manually run containers on servers, making scaling and updates error-prone.

- No Self-Healing – If a container crashed, it had to be restarted manually.

- Scaling Issues – Increasing or decreasing the number of containers required scripting or manual intervention.

- Networking Complexity – Containers on different hosts struggled with stable communication and service discovery.

- Load Balancing – Traffic distribution across containers wasn’t automated.

- Environment Drift – Running apps on dev, test, and prod often caused inconsistencies.

- Resource Management – Containers could consume more resources than expected, affecting other workloads.


## Why We Need Kubernetes

- Automation – Deploy, update, and roll back applications automatically.

- Scalability – Scale apps up/down dynamically based on demand.

- Self-Healing – Detects failed containers/pods and restarts or reschedules them automatically.

- Service Discovery & Load Balancing – Provides built-in DNS and load balancing.

- Portability – Run apps consistently across any infrastructure (cloud, on-prem, hybrid).

- Efficient Resource Utilization – Schedules containers optimally across nodes.

- Declarative Management – Define desired state, and Kubernetes maintains it continuously.

👉 In short: Before K8s, managing containers was manual, fragile, and inefficient. With K8s, it became automated, resilient, and scalable.


## Architecture

Kubernetes is a distributed control-plane + data-plane system that makes a cluster of machines behave like a single, declarative, self-healing platform for running containerized workloads — the control plane stores desired state (API + etcd) and controllers continuously reconcile actual state to that desired state while node agents (kubelet, container runtime, kube-proxy) execute workloads.


1) Core control-plane components (what they are and what they do)

- kube-apiserver (API Server)

    - The central management endpoint (REST/gRPC-ish) for the cluster. All reads/writes (kubectl, controllers, kubelets) go through it.

    - AuthN (client certs, token, OIDC), AuthZ (RBAC, ABAC if enabled), Admission (mutating→validating webhooks), audit logging.

    - Persists cluster state to etcd and serves watch streams (long-lived connections) to controllers/kubelets.

    - Default secure port: 6443 (common).

- etcd

    - Consistent, distributed key-value store for all cluster state (object manifests, resourceVersions, leases).

    - Strong consistency; backup/restore is critical. Typically run as an odd-numbered cluster (3/5/7).

    - Ports: client 2379, peer 2380 (typical).

- kube-controller-manager

    - Runs multiple controllers in a single process (replication controller, endpoints controller, namespace controller, service account controller, node controller, garbage collector, etc.).

    - Each controller is a control loop: it watches resources and reconciles current→desired state.

- kube-scheduler

    - Watches for unscheduled Pods and decides a Node for each Pod based on constraints (taints/tolerations, affinities, resource requests/limits, node selectors, custom schedulers).

    - Modern scheduler is plugin-based (preFilter/Filter/Score/Reserve/Permit/Bind phases). Can be extended via scheduler extenders.

- cloud-controller-manager

    - Runs cloud-provider-specific controllers (load balancers, node lifecycle, persistent disks) separated from core controllers so Kubernetes core can be cloud-agnostic.

The API server is the only component that talks directly to etcd; other components talk to the API server.

2) Node / data-plane components

- kubelet

    - Agent on each node. Watches the API server for PodSpecs scheduled to that node.

    - Interacts with the container runtime via the CRI (Container Runtime Interface) to create/delete containers, manage lifecycle, mount volumes, run probes.

    - Reports PodStatus and NodeStatus back to the API server.

    - Enforces lifecycle hooks, liveness/readiness/startup probes.

- container runtime

    - Runs containers — e.g., containerd, CRI-O, (historically Docker via dockershim; dockershim removed).

    - Exposes CRI gRPC API to the kubelet.

    - Handles images, cgroups, namespaces, process isolation.

- kube-proxy

    - Implements Kubernetes Services. It programs network rules on each node so ClusterIP & NodePort traffic gets forwarded to backing Pod endpoints.

    - Modes: iptables, ipvs, and eBPF-based alternatives (e.g., Cilium uses eBPF).

    - Works with kube-apiserver to discover Service/Endpoint changes and updates networking rules.

- CNI plugins

    - Provide cluster pod networking (IP allocation, routing, policy): e.g., Calico, Flannel, Cilium, Weave.

    - Conform to CNI spec; used by kubelet to attach Pod network.



3) Kubernetes API and objects (data model basics)

- API groups & versions: e.g., core/v1 (Pods, Services), apps/v1 (Deployments, StatefulSets), batch/v1 (Jobs), autoscaling/v2.

- Core objects

    - Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob, Service, Ingress (or IngressController), ConfigMap, Secret, PersistentVolume (PV), PersistentVolumeClaim (PVC), StorageClass, Namespace, Node.

- Metadata: labels, annotations, ownerReferences, finalizers, resourceVersion, uid.

- Selectors: labels are used to select groups of objects (e.g., Service selects Pods).


## Pod

A Pod in Kubernetes is the smallest deployable unit.
It represents one or more tightly coupled containers that:

- Run together on the same node (always scheduled together).

- Share the same network namespace (same IP address, hostname, and ports).

- Can share storage volumes for data persistence or communication.

👉 In short: A Pod = wrapper around one or more containers, making them act as a single unit of deployment and management in Kubernetes.


## Pod lifecycle and the exact step-by-step flow when a Pod is created

1. User submits manifest

    - ```kubectl apply -f pod.yaml``` → ```kubectl``` uses kubeconfig to talk to kube-apiserver (authN/authZ).

2. API Server validation & admission

    - Request is authenticated, authorized, admission controllers run (mutating first, e.g., add sidecar, inject defaults; then validating webhooks).

    - Object written to etcd (persisted). A ```resourceVersion``` is assigned.

3. Controllers and scheduler get notified via watch

    - If Pod ```.spec.nodeName``` is unset, scheduler sees unscheduled Pod and selects a node (Filter/Score).

4. Scheduler writes a binding

    - Scheduler updates Pod ```.spec.nodeName``` (a bind operation) via the API server.

5. Kubelet on selected node detects new Pod

    - The node’s kubelet watches the API server and sees the Pod assigned to its node.

6. Kubelet prepares environment

    - Pulls images from registry (using imagePullPolicy), creates network namespace via CNI, sets ```/etc/hosts``` and environment variables, mounts volumes (CSI or in-tree), sets up resource cgroups.

    - Runs ```initContainers``` sequentially if present.

7. Container runtime creates containers

    - Creates container processes, applies seccomp/AppArmor, sets capabilities and resource limits.

8. Networking and service discovery

    - Pod gets an IP from CNI. DNS (CoreDNS) entries are eventually resolvable: ```podIP```, ```svcname.namespace.svc.cluster.local```.

9. Probes and status reporting

    - Kubelet executes liveness/readiness/startup probes. Kubelet updates PodStatus to API server; events are generated.

10. Service endpoints updated

    - Endpoints controller updates Endpoints for Services mapping to this Pod's IP/port.

11. Controller (e.g., ReplicaSet) ensures replicas

    - If a controller created the Pod (eg. Deployment→ReplicaSet→Pod), it continues reconciling replica counts.

12. If Pod fails / dies

    - Kubelet reports failure; controller decides to recreate (depending on ```restartPolicy``` and controller type). Garbage collector cleans up when ownerReferences/finalizers permit.

Important lifecycle details:

Termination: ```kubectl delete``` sets ```deletionTimestamp```; kubelet runs ```preStop``` hook, sends SIGTERM to process, waits grace period (```terminationGracePeriodSeconds```), then SIGKILL.

Finalizers: prevent resource deletion until cleanup (e.g., volumes detached).

Ephemeral containers: for debugging only — injected into a running Pod.


## Controllers & control loops (how reconciliation works)

- **Control loop pattern:**

    - Controller watches resources, computes desired vs current, takes actions via API to converge.
    
    - Idempotent operations and eventual consistency.

- **Primary controllers**

    - ReplicationController / ReplicaSet: ensure N replicas of a Pod template.

    - Deployment: manages ReplicaSets and rolling updates/rollbacks using strategies (RollingUpdate, Recreate).

    - StatefulSet: stable network identities, stable persistent storage using volumeClaimTemplates, ordered startup/upgrade semantics.

    - DaemonSet: run a Pod on each eligible Node.

    - Job / CronJob: run finite workloads; Job ensures Pod completion (parallelism/completions).

    - HorizontalPodAutoscaler (HPA): scales replicas based on metrics (CPU/custom metrics).

- Garbage Collector

    Deletes dependent objects based on ownerReferences and deletion policy.

- Leader election

    Controller-managers can use leader election to run single-instance controllers in HA setups.




## Services & Networking deep-dive

- Kubernetes networking model

    - Every Pod gets its own IP and can communicate with any other Pod IP without NAT (model requirement).

    - CNI implements: IP allocation, routing, network policies.

- Service types

    - ClusterIP (internal virtual IP, default).

    - NodePort (exposes service on a node port range, default 30000–32767).

    - LoadBalancer (cloud LB provisioned by cloud-controller-manager).

    - Headless Service (clusterIP: None) — skip kube-proxy virtual IP; client uses Endpoints directly (useful for StatefulSets).

- Service implementation

    - Kube-proxy programs node-local rules to DNAT to endpoints. In IPVS mode, uses Linux IPVS for load balancing efficiency.

    - Newer approaches: eBPF datapath (Cilium) for high performance and observability.

- DNS

    CoreDNS provides *.svc.cluster.local name resolution. Services also create environment variables in Pods (for older compatibility).

- NetworkPolicy

    Namespaced ingress/egress rules that restrict Pod traffic — default allow unless a deny policy exists (behavior depends on CNI implementation).



## Storage and persistence

- PersistentVolume (PV): cluster resource representing actual storage (NFS, iSCSI, cloud disk).

- PersistentVolumeClaim (PVC): namespaced request for storage (size, access modes).

- StorageClass: defines provisioner and parameters for dynamic provisioning.

    provisioner: CSI driver name.

    reclaimPolicy: Delete (delete backend volume) or Retain (keep).

- CSI (Container Storage Interface)

    - Standard plugin model for storage drivers; supports dynamic provisioning, snapshots, volume expansion.

- Access modes

    - ReadWriteOnce (RWO) — one node can mount RW (most common).

    - ReadOnlyMany (ROX).

    - ReadWriteMany (RWX) — multiple nodes can mount RW (requires appropriate storage backend).

- Volume lifecycle

    - PVC binds to PV when matching; if StorageClass exists, dynamic provisioning creates PVs.



## Scheduling detailed mechanics

- Filtering: node affinity, taints/tolerations, resource requests (CPU/memory), node selectors, topology constraints.

- Scoring: choose the “best” node based on scoring plugins (e.g., balance, binpack).

- Binding: scheduler writes .spec.nodeName (or uses Bind API) to assign Pod.

- Preemption: higher-priority pods can preempt lower priority ones when necessary.

- Taints & tolerations: taints on nodes repel Pods; only Pods with matching tolerations can be scheduled.

- Affinity/Anti-affinity: instruct scheduler to prefer or require co-location or anti-co-location with other pods.


## Security (practical layers)

- Authentication: client certs, bearer tokens, service accounts, OIDC.

- Authorization: RBAC (Roles, ClusterRoles, RoleBinding, ClusterRoleBinding) is most common.

- Admission controllers: e.g., PodSecurity, NodeRestriction, ResourceQuota, mutating webhooks for policy injection (sidecars, defaults).

- Pod Security Standards: restricted, baseline, privileged (policies enforced via PodSecurity admission or OPA/Gatekeeper).

- Secrets: stored in etcd (base64); use KMS/Envelope encryption at rest or external secrets management; consider sealed-secrets.

- Network segmentation: NetworkPolicy and CNI enforcement.

- Runtime security: seccomp, AppArmor, drop Linux capabilities, readOnlyRootFilesystem, avoid privileged containers.





# Deployment
A Deployment is a declarative controller (API object apps/v1/Deployment) that manages stateless workloads by creating and controlling ReplicaSets, which in turn manage Pods. Deployments provide rolling updates, rollbacks, scaling, revision history, and self-healing for Pods created from a Pod template.



## Core responsibilities

- Maintain the desired number of replicas of a Pod template.

- Manage updates to the Pod template in a controlled way (rolling updates by default).

- Keep revision history (old ReplicaSets) so you can roll back.

- Recreate ReplicaSets and Pods when needed (self-healing).

- Expose status and conditions (progress, available, etc.) for observability.



```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deploy
spec:
  replicas: 3                                   # desired replicas (optional; default 1)
  selector:                                     # REQUIRED: label selector targeting Pods managed by RS
    matchLabels:
      app: my-app
  template:                                     # Pod template (metadata + spec)
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: web
        image: nginx:1.27
  strategy:                                     # update strategy
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 0                            # how long a Pod must be ready before considered available
  revisionHistoryLimit: 10                      # keep up to 10 old ReplicaSets for rollback
  progressDeadlineSeconds: 600                  # time before rollout considered failed
  paused: false                                 # controller will not act if paused
```

Key fields explained:

- replicas: desired number of Pods (controller sets ReplicaSet.spec.replicas to this value).

- selector (matchLabels/matchExpressions): which Pods the ReplicaSet/Deployment should manage. Selector is effectively immutable — you must ensure selectors match the Pod template labels.

- template: the Pod template — metadata (labels/annotations) and spec (containers, volumes, probes, affinity, resources, etc.)

- strategy.type: RollingUpdate (default) or Recreate.

- rollingUpdate.maxSurge: integer or percent — max extra pods above replicas during update.

- rollingUpdate.maxUnavailable: integer or percent — max pods that can be unavailable during update.

- minReadySeconds: a Pod is not considered “available” until it has been ready for this many seconds.

- revisionHistoryLimit: how many old ReplicaSets to keep.

- progressDeadlineSeconds: if rollout doesn’t make progress within this many seconds, Deployment is marked Progressing: False with ProgressDeadlineExceeded.

- paused: boolean to temporarily stop rollout.



## When a deployment is made

1. User Action

    - You run:
        ```
        kubectl apply -f deployment.yaml
        ```

    - kubectl reads your kubeconfig, authenticates with the API Server, and sends a REST request with your Deployment object.

2. API Server Processing

    - The kube-apiserver:

        1. Authenticates (AuthN) and Authorizes (RBAC).

        2. Passes through Admission Controllers:

            - Mutating webhooks (may inject defaults/sidecars).

            - Validating webhooks (validate fields).

        3. Stores the Deployment object in etcd (the source of truth).

    - Responds back to kubectl with “Deployment created.”

3. Deployment Controller Kicks In

    - The Deployment Controller (inside kube-controller-manager) watches Deployments via a watch stream.

    - It notices a new Deployment object and creates a ReplicaSet object with the Pod template from your Deployment spec.

4. ReplicaSet Controller Ensures Pods

    - The ReplicaSet Controller sees the new ReplicaSet and ensures the desired number of Pods exist.

    - It creates Pod objects in the API Server (with .spec but no .spec.nodeName yet).

5. Scheduler Assigns Nodes

    - The kube-scheduler watches for unscheduled Pods (those without .spec.nodeName).

    - It runs scheduling logic:

        - Filters eligible nodes (taints, resources, affinities).

        - Scores and picks the best node.

    - Writes the binding (sets .spec.nodeName on the Pod).

6. Kubelet Executes Pods

    - The kubelet on the chosen node sees the Pod assigned to it.

    It:

    1. Pulls container images (via CRI runtime: containerd, CRI-O).

    2. Sets up networking (via CNI plugin, assigns Pod IP).

    3. Mounts volumes if any.

    4. Starts containers.

    - Runs probes (liveness, readiness) and reports Pod status back to the API Server.

7. Service & Endpoint Updates (if any)

    - If a Service matches the Pods’ labels:

        - The Endpoints Controller adds the Pod IPs to that Service’s endpoint list.

        - kube-proxy updates iptables/ipvs/eBPF rules so traffic to the Service is load-balanced to Pods.

8. Ongoing Reconciliation

    - If a Pod crashes, the ReplicaSet notices the count mismatch and creates a new Pod.

    - If you update the Deployment (e.g., new image), the Deployment controller:

        - Creates a new ReplicaSet with the updated template.

        - Gradually scales down the old ReplicaSet and scales up the new one (rolling update).

    - If you roll back, Deployment controller scales the old ReplicaSet back up.



## How updates work (RollingUpdate strategy — default)

When you change the Pod template (e.g., image), Deployment performs a controlled rollout:

High-level algorithm:

1. Deployment creates a new ReplicaSet (RS-new) whose Pod template matches the new template.

2. It scales RS-new up and scales the old ReplicaSet(s) down while honoring maxSurge and maxUnavailable.

3. It waits for Pods to become available (readiness probe + minReadySeconds) before considering them in the counts that allow more downscaling of old RS.

4. When RS-new reaches desired state and old RSs scaled to zero (or appropriate), rollout completes.

Important parameters:

- maxSurge (int or %) — how many extra Pods above spec.replicas are allowed during rollout. Example: replicas: 4, maxSurge: 1 → at most 5 Pods can exist while updating.

- maxUnavailable (int or %) — how many Pods can be unavailable during the rollout. Example: replicas: 4, maxUnavailable: 1 → at least 3 Pods must be up during update.

Example numeric scenario (clear):

- replicas: 4, maxSurge: 1, maxUnavailable: 1.

    - Desired: 4 running pods.

    - Deployment may spin up 1 new Pod (new RS) resulting in 5 total temporarily.

    - Once new Pod is ready, the Deployment can scale down an old Pod — preserving availability constraints.

Notes:

- maxSurge and maxUnavailable accept percentages or absolute ints. Kubernetes converts percentages into absolute numbers relative to spec.replicas.

- minReadySeconds ensures a Pod must be in Ready state continuously for that many seconds before considered available — useful to avoid flipping traffic to still-warming Pods.



### Recreate strategy

- **strategy.type: Recreate** — Deployment deletes all old Pods (scales old RS to 0) before creating new ones.

- Use this if you cannot run old/new versions concurrently (breaking changes) or when order matters.

- This causes downtime unless the app handles it.



### Revisioning, history & rollback

* Each time Deployment template changes, it creates a new ReplicaSet with an incremented revision. The revision number is stored as an annotation (deployment.kubernetes.io/revision) on ReplicaSets and Pods.

* revisionHistoryLimit controls how many old ReplicaSets are kept. Old RSs beyond that are garbage collected.

* Rollback:

    * ```kubectl rollout history deployment/<d>``` shows revisions.

    * ```kubectl rollout undo deployment/<d>``` will rollback to previous revision (or --to-revision=<n>).

    * Rollback works by taking the old RS’s Pod template and updating the Deployment template to match it, triggering a new rollout.


### Under-the-hood: Deployment ↔ ReplicaSet interaction

- Deployment manages ReplicaSets. ReplicaSet manages Pods.

- On update, Deployment creates a new ReplicaSet (name like <deploy-name>-<hash>). The pod-template-hash label distinguishes Pods of different ReplicaSets.

- The Deployment controller scales ReplicaSets up/down; it never directly creates Pods — ReplicaSets do.

- Selector immutability: once set, spec.selector should not change; mismatches between selector and pod template labels will produce unexpected behavior (pods might not be adopted).





### Quick cheatsheet (commands)

```
kubectl apply -f deploy.yaml

kubectl get deploy

kubectl describe deploy webapp

kubectl rollout status deploy/webapp

kubectl rollout history deploy/webapp

kubectl rollout undo deploy/webapp

kubectl set image deploy/webapp web=myimage:tag

kubectl rollout pause|resume deploy/webapp
```

# Sharding in Kubernetes

Sharding = splitting data, workloads, or control responsibilities into separate partitions (shards) so each shard handles a subset of traffic/data/resources. In Kubernetes, “sharding” is a design/operational pattern used at several layers to improve scalability, isolation, performance, and multi-tenancy. Below I’ll unpack the concept, show the common forms, how you implement them on Kubernetes, trade-offs, failure modes, and best practices.


# Replicaset

A ReplicaSet is a Kubernetes controller whose job is to ensure that a specified number of identical Pods are always running.

It is lower-level than a Deployment:

- ReplicaSets manage Pods directly.

- Deployments manage ReplicaSets (and through them, Pods).

👉 Think of ReplicaSet as the engine that keeps N Pods alive.



### Core Responsibilities

1. Maintain desired Pod count (spec.replicas).

    - If Pods die → RS creates new ones.

    - If too many Pods exist → RS deletes extra ones.

2. Match Pods via selectors (spec.selector) → Only Pods with matching labels are managed.

3. Adoption/Orphaning

    - RS can adopt matching Pods not already owned.

    - If a Deployment is deleted with orphan policy, Pods can stay alive without RS control.



### API Shape (Important Fields)


```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3                   # desired number of Pods
  selector:                     # which Pods this RS manages
    matchLabels:
      app: my-app
  template:                     # Pod template used if RS creates Pods
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: web
        image: nginx:1.27
```


Key fields:

- ```.spec.replicas``` → number of Pods desired (default: 1).

- ```.spec.selector``` → label selector that identifies Pods RS should manage. Immutable after creation.

-  ```.spec.template``` → Pod template RS uses to create new Pods.

- ```.status``` → reports observed replicas, available replicas, ready replicas, fullyLabeledReplicas.


## ReplicaSet Lifecycle (Step-by-Step)

1. Created via Deployment or directly

    - If via Deployment: Deployment controller creates RS and sets .spec.replicas.

    - If standalone: User applies RS YAML.

2. Stored in etcd via API server.

3. ReplicaSet Controller (inside kube-controller-manager) sees new RS.

    - Watches RS objects.

    - Reconciles desired .spec.replicas vs actual Pods.

4. If Pods < replicas → creates Pods using .spec.template.

5. If Pods > replicas → deletes extra Pods (chosen randomly, but avoids deleting unhealthy ones first).

6. Scheduler assigns Pods to nodes (since RS-created Pods have no .spec.nodeName).

7. Kubelet starts containers → Pod running.

8. Controller continuously reconciles: ensures actual matches desired.


### Differences: ReplicaSet vs ReplicationController

- ReplicationController was the older primitive.

- ReplicaSet = newer + supports set-based selectors (In, NotIn, Exists).

- ReplicaSet is the recommended controller.

### How ReplicaSets are used by Deployments

- Deployment = higher-level manager that creates RS.

- On rollout:

    - Deployment creates a new RS for the new Pod template.

    - Gradually scales new RS up, old RS down.

- RS themselves don’t handle updates/rollbacks → that’s Deployment’s job.

👉 You rarely create RS directly. Instead, you use Deployments, which handle RS lifecycle.




# Storage in k8s

## StatefulSet

A StatefulSet is a Kubernetes controller (like Deployment/ReplicaSet) designed to manage stateful applications.

It provides guarantees about the identity, ordering, and storage of Pods.


While Deployments/ReplicaSets are for stateless apps, StatefulSets are for stateful apps where Pod identity and persistence matter.



## Key Features of StatefulSet

1. Stable Pod Identity

    - Pods get predictable names (<statefulset-name>-0, -1, -2 …).

    - Even if rescheduled, Pod keeps its ordinal index.

2. Stable Network Identity

    - Each Pod gets a stable DNS name (pod-name.service-name.namespace.svc.cluster.local).

    - This is crucial for clustered systems (e.g., Kafka brokers, MongoDB nodes).

3. Stable Storage (Persistent Volumes)

    - Each Pod gets its own PersistentVolumeClaim (PVC).

    - Volumes are not deleted when Pods restart or reschedule — ensuring persistent data.

3. Ordered Deployment & Scaling

    - Pods start in order (0 → 1 → 2 …).

    - Pods are terminated in reverse order.

    - Useful when some Pods depend on others (e.g., master before replicas).

4. Ordered Rolling Updates

    - Updates Pods one by one (respecting order).

    - Ensures cluster remains consistent during upgrades.



| Feature           | Deployment / ReplicaSet                  | StatefulSet                              |
|-------------------|-------------------------------------------|------------------------------------------|
| Pod identity      | Random                                    | Stable, ordinal (-0, -1, …)              |
| Network identity  | Ephemeral Pod IP                          | Stable DNS hostname                      |
| Storage           | Shared / ephemeral                        | Per-Pod dedicated PVC                    |
| Scaling order     | Parallel (any order)                      | Ordered (0,1,2 →)                        |
| Rolling updates   | Parallel / batched                        | Ordered, one-by-one                      |
| Use case          | Stateless apps (e.g., web servers)        | Stateful apps (DBs, queues, clustered)   |



## StatefulSet API (important fields)

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql                # Headless Service for DNS
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:                         # Pod template
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:             # Each Pod gets its own PVC
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
      storageClassName: standard
```



### Key Fields Explained:

- serviceName → must be a headless Service (clusterIP: None) → provides stable DNS.

- replicas → number of Pods (e.g., mysql-0, mysql-1, mysql-2).

- template → Pod spec (like Deployment).

- volumeClaimTemplates → auto-generates PVCs for each Pod (mysql-data-mysql-0, etc.).



## Lifecycle of a StatefulSet

1. User creates StatefulSet object (kubectl apply).

2. API Server validates & stores in etcd.

3. StatefulSet Controller (inside kube-controller-manager) sees the new object.

4. Controller creates Pods one-by-one:

    - Creates pod-0 → waits until it’s Ready.

    - Then creates pod-1 → waits → then pod-2…

5. If a Pod crashes → Controller recreates it with the same name + PVC.

6. Scaling down → removes Pods in reverse order (e.g., pod-2, then pod-1).



## Volumes (Pod-scoped)

A Volume in a Pod spec is a directory that containers can mount. It is created for the lifetime of the Pod. (Examples: emptyDir, hostPath, configMap, secret, nfs, CSI volumes, etc.) 
Kubernetes

- Volume types:

    - emptyDir — ephemeral, lives as long as Pod.

    - hostPath — maps host filesystem into Pod (not portable, security risk).

    - configMap, secret, downwardAPI — inject small config/metadata.

    - nfs, cephfs, glusterfs — network file systems (can be RWX if backend supports).

    - CSI volumes — use CSI driver (recommended for most production backends).

- VolumeMode (in PV/PVC): Filesystem (default) or Block. Block mode mounts a raw block device into the container (no filesystem); useful for databases or special use-cases.



## PersistentVolume (PV) — cluster resource

PV is a cluster-scoped representation of storage (backed by cloud disk, NFS, iSCSI, Ceph, etc.). Admins can create PVs statically or dynamic provisioning can create them. PVs declare capacity, accessModes, persistentVolumeReclaimPolicy, and volume-specific config. 
Kubernetes

- Important PV fields:

    - **spec.capacity.storage** — size (e.g., 10Gi).

    - **spec.accessModes** — ReadWriteOnce (RWO), ReadOnlyMany (ROX), ReadWriteMany (RWX). Semantics depend on backend.

    - **spec.persistentVolumeReclaimPolicy** — Retain (preserve data), Delete (delete backend volume), Recycle (deprecated).

    - **spec.storageClassName** — which StorageClass this PV belongs to (optional).

    - **spec.nodeAffinity** — restricts which nodes the PV can be used on (important for local volumes).

- Lifecycle: PV exists independent of Pods and can survive Pod deletion (useful for stateful workloads).


```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-1
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 10.0.0.25
    path: /exports/data
  storageClassName: slow-nfs

```


What are Access Modes?

Access Modes define how a PersistentVolume (PV) can be mounted by Pods.

They describe read/write permissions and node sharing rules.

Declared on both PersistentVolume (PV) and PersistentVolumeClaim (PVC).

### Types of Access Modes
1. ReadWriteOnce (RWO)

    - Volume can be mounted as read-write by a single node.

    - Multiple Pods on the same node can use it at the same time.

    - Example backends: AWS EBS, GCP Persistent Disk, Azure Disk.

    ✅ Good for databases (e.g., MySQL, Postgres).

2. ReadWriteOncePod (RWOP) (K8s 1.22+)

    - Stricter version of RWO.

    - Volume can be mounted as read-write by only one Pod in the cluster.

    - Even if multiple Pods are on the same node, only one gets access.

    ✅ Ensures strong single-writer guarantee (e.g., etcd).

3. ReadOnlyMany (ROX)

    - Volume can be mounted read-only by many nodes.

    - Useful for distributing data that doesn’t change (e.g., config files, shared reference data).

    Example backends: NFS, CephFS.

4. ReadWriteMany (RWX)

    - Volume can be mounted read-write by many nodes simultaneously.

    - Required for apps that need shared, concurrent access (e.g., CMS, WordPress with shared media, logs).

    Example backends: NFS, CephFS, GlusterFS, Azure Files, EFS (AWS).

5. ReadWriteOnce (Block) / ReadOnlyMany (Block) (rare)

    - When using block storage mode (not filesystem).

    - Pod gets direct raw block device.



## PersistentVolumeClaim (PVC) — user request and binding

- PVC is a namespaced object requesting storage: a size, accessModes, and optionally a storageClassName. The PVC controller matches & binds a PVC to a suitable PV (static) or triggers dynamic provisioning (if StorageClass present). 
Kubernetes

- PVC fields:

    - spec.resources.requests.storage — requested capacity (e.g., 10Gi).

    - spec.accessModes — what the Pod requires (RWO/RWX/ROX).

    - spec.storageClassName — which StorageClass to use (or "" for no dynamic provisioning).

- Binding rules:

    - A PV binds to only one PVC (claimRef) unless the PV is RWX and backend supports multiple mounts (still bound once).

    - Matching based on storageClassName (if present), accessModes compatibility, capacity (PV >= PVC request), and other attributes (labels/selector on PV).

- Status: Pending until bound → Bound. After bound, Pod mounting waits until PVC is Bound.


## StorageClass & Dynamic Provisioning — how PVs are created

StorageClass is the “blueprint” for dynamically provisioning PVs. It names the provisioner (a CSI driver or in-tree plugin), plus driver parameters (disk type, replication, fsType), reclaimPolicy, allowVolumeExpansion, mountOptions, and volumeBindingMode. 
Kubernetes

Important StorageClass fields:

- provisioner — e.g., kubernetes.io/aws-ebs or CSI driver name like ebs.csi.aws.com.

- parameters — provider-specific options (IOPS, encryption, fsType).

- reclaimPolicy — Delete or Retain.

- allowVolumeExpansion — boolean allowing PVC resize.

- mountOptions — mount flags.

- volumeBindingMode:

    - Immediate — provision/bind at PVC creation time.

    - WaitForFirstConsumer — delay binding until a Pod using the PVC is scheduled; this allows the scheduler to pick a node and provision volume in the correct zone/topology. This is critical for multi-zone clusters and local volumes.


```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```


## Dynamic provisioning flow — step-by-step (what actually happens)

1. User creates PVC with ```storageClassName```.

2. external-provisioner (a sidecar/controller for the CSI driver) sees the PVC and calls the CSI ```CreateVolume``` RPC to the storage system (creates cloud disk, etc.). The external-provisioner is needed because Kubernetes core controller cannot call CSI drivers directly. 

3. CSI driver returns volume handle; the provisioner creates a PV object bound to the PVC.

4. PVC status becomes ```Bound```.

5. When a Pod using that PVC is scheduled, Attach/Detach and Publish/Mount steps happen (see next section). If ```volumeBindingMode=WaitForFirstConsumer```, step 2 is delayed until scheduler selects a node — allowing topology-aware provisioning.


## What is a DaemonSet?

A DaemonSet is a Kubernetes controller that ensures a copy of a Pod runs on every (or selected) Node in the cluster.

👉 Unlike a Deployment (which schedules Pods anywhere to satisfy replica count), a DaemonSet guarantees Pod presence per Node.

### Use Cases

- Cluster storage daemons → e.g., Ceph, GlusterFS.

- Node monitoring/logging agents → e.g., Prometheus node-exporter, Fluentd, Datadog Agent.

- Networking agents → e.g., CNI plugins, kube-proxy, service mesh sidecars.

- Security/infra agents → e.g., Falco, antivirus Daemons.

Basically: “one Pod per Node” workloads.


FILE EXAMPLE
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
```




### Key Features

1. One Pod per Node (by default)

- DaemonSet controller ensures every Node (existing or new) runs one Pod.

2. Node targeting

- Can restrict Pods to specific Nodes with:

    - nodeSelector

    - nodeAffinity

    - taints and tolerations

3. Pod scheduling

- Uses the default scheduler (since Kubernetes 1.12).

- Respects node constraints (e.g., only GPU nodes).

4. Rolling updates

- Default strategy = RollingUpdate (one node at a time).

- Supports OnDelete strategy (only update Pods when manually deleted).

5. No ReplicaSets

- Unlike Deployments, DaemonSets directly manage Pods.

- You don’t scale them with replicas — they scale automatically with nodes.




### How DaemonSets Work (Step by Step)

1. You create a DaemonSet object via API.

2. API server stores it in etcd.

3. DaemonSet controller (inside kube-controller-manager) watches DaemonSets.

4. For each eligible Node:

- If Pod missing → creates Pod.

- If Node added → schedules new Pod.

- If Node removed → deletes Pod.

5. Scheduler assigns DaemonSet Pods to Nodes (respects affinity/tolerations).

6. Kubelet starts Pod → Pod runs.

7. DaemonSet continuously reconciles: ensures Pods exist on all matching Nodes.



# Netwroking


## Why do we need Services?

- Pods are ephemeral: they can die, restart, get rescheduled, get new IPs.

- Clients need a stable way to reach a group of Pods.

- Service = stable virtual IP + DNS name that load-balances traffic to healthy Pods behind it.




## Service Basics

- ClusterIP: Virtual IP allocated inside the cluster.

- Selectors: Service selects Pods using labels (spec.selector).

- Endpoints: Service maps to Pods’ IP:Port via the Endpoints object (or EndpointSlice).

- kube-proxy: Programs networking rules (iptables/ipvs/eBPF) on Nodes to forward traffic to endpoints.




## Types of Services

1) ClusterIP (default)

- Provides an internal virtual IP.

- Only reachable inside cluster.

- Used for service-to-service communication.


Example

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80         # Service port
    targetPort: 8080 # Pod port
```

👉 Pods labeled app=my-app are endpoints.


2. NodePort

- Exposes Service on each Node’s IP at a static port (default range: 30000–32767).

- Accessible outside cluster via <NodeIP>:<NodePort>.

- Typically used for development/testing or as backend for an external load balancer.

```
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
```


3. LoadBalancer

- Provisioned when running on cloud providers (AWS ELB, GCP LB, Azure LB, etc.).

- Creates an external load balancer that routes to NodePorts.

- Simplest way to expose apps to the internet in cloud environments.

4. ExternalName

- Maps Service to an external DNS name.

- Doesn’t create Endpoints or proxy traffic.

- Clients inside cluster resolve Service name → external DNS.

5. Headless Service

- Special case: clusterIP: None.

- No virtual IP allocated.

- DNS queries resolve to individual Pod IPs (via Endpoints).

- Useful for StatefulSets (e.g., mysql-0.mysql.default.svc.cluster.local).


Example
```
spec:
  clusterIP: None
  selector:
    app: my-db
```




## How Services Work (Internals)

1. Service object creation

    - User applies Service manifest.

    - API server stores in etcd.

    - Endpoints Controller monitors Pods matching spec.selector → creates Endpoints / EndpointSlices.

2. kube-proxy programming

    - kube-proxy runs on each Node.

    - Watches Services & Endpoints via API.

    - Programs kernel rules:

        - iptables mode → creates DNAT rules mapping ServiceIP:Port → PodIP:Port.

        - ipvs mode → creates IPVS virtual servers with endpoints as backends.

        - eBPF mode (Cilium, newer) → attaches BPF programs for faster, programmable packet forwarding.

3. Traffic flow

- Client connects to ServiceIP:Port.

- Node kernel (iptables/ipvs/eBPF) rewrites packet to a backend PodIP.

- Pod receives traffic as if it was directly targeted.


### Service Discovery

- Each Service gets a DNS entry (<svc>.<namespace>.svc.cluster.local).

- Example: my-service.default.svc.cluster.local.

- CoreDNS resolves Service name → ClusterIP.

- For headless service (clusterIP: None), DNS returns Pod IPs.

### Ports in Service

- port: Port exposed by Service (cluster-wide stable).

- targetPort: Pod’s container port (can be number or name).

- nodePort: Port allocated on each Node (if type=NodePort/LoadBalancer).



# Networking

1. Kubernetes Networking Model

The Kubernetes networking model is based on simplicity:

1. Every Pod gets its own IP address.

- No NAT between Pods.

- Containers inside a Pod share the same network namespace (same IP, loopback).

2. Every Pod can communicate with every other Pod in the cluster, without NAT.

3. Services get stable virtual IPs (ClusterIPs) to provide stable access to Pod groups.

👉 Unlike Docker (where containers are NATed), Kubernetes enforces a flat network model.


2. Pod-to-Pod Networking

- Each Pod gets an IP from a cluster-wide Pod CIDR.

- When a Pod starts:

    - Kubelet calls CNI plugin to allocate Pod IP.

    - CNI sets up veth pair → one end in Pod, one in Node network.

    - Routes configured so Pod can reach other Pods.

CNI Plugin Options:

    - Flannel (overlay, VXLAN) → simple, easy.

    - Calico (routing, eBPF, network policy) → scalable, high performance.

    - Weave Net (mesh, encryption).

    - Cilium (eBPF-based, advanced observability & security).

    - Canal (Calico + Flannel).

3. Pod-to-Service Networking

- Service IP = ClusterIP (virtual IP).

- kube-proxy on each Node installs rules:

    - iptables/ipvs/eBPF rules forward traffic from ClusterIP → Pod IPs.

    - Load-balances across healthy Pods.

- Clients inside cluster use DNS (my-svc.my-namespace.svc.cluster.local).

4. Node-to-Pod Networking

- Each Node routes to Pod CIDRs of other Nodes.

- How this works depends on CNI plugin:

    - Overlay networks (Flannel VXLAN) → encapsulates Pod traffic in UDP tunnels.

    - Routing-based (Calico) → configures BGP routes between Nodes.

    - Cloud CNI (AWS, GCP, Azure) → integrates with VPC networking.




5. kube-proxy Internals

kube-proxy runs on every Node, responsible for Service load balancing.

Modes:

1. iptables (default)

    - Rules DNAT Service IP → Pod IP.

    - Simple but limited performance in huge clusters.

2. ipvs

    - Kernel IPVS load balancer.

    - Scales better, supports algorithms (rr, lc, wlc).

3. eBPF (via Cilium, kube-proxy replacement)

    - Fastest, programmable packet filtering.

    - Lower latency, higher throughput.

👉 Each Service creates kernel rules mapping ClusterIP:Port → Pod IP:Port (endpoints).


6. Service Networking

Service Types in networking terms:

- ClusterIP → Internal-only virtual IP.

- NodePort → Opens static port on every Node (30000–32767).

- LoadBalancer → Cloud load balancer routes → NodePort → Pods.

- Headless (```clusterIP: None```) → No virtual IP; DNS returns Pod IPs directly.

- ExternalName → DNS alias to external service.



7. DNS in Kubernetes

- CoreDNS (default cluster DNS).

- Every Service gets DNS:
```
<svc>.<namespace>.svc.cluster.local
```

- Headless Services → DNS A records for each Pod.

- Pods also get DNS:
```
<pod-ip>.<namespace>.pod.cluster.local
```



8. NetworkPolicy (firewalls between Pods)

- Kubernetes has no built-in firewalling by default. All Pods can talk to all Pods.

- NetworkPolicy objects restrict traffic. Supported by CNIs like Calico, Cilium, Weave.

Example: allow traffic to app=db only from app=backend Pods:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend

```


# Packet Flow Examples

1) Inside the same Pod — exact mechanics


    - Network namespace rules

    - All containers in a Pod share one network namespace (same Linux netns). That means:

        - They share the same IP address (the Pod IP).

        - They share the same loopback device (lo) and same set of network interfaces.

        - Ports are global within the Pod: two processes/containers cannot bind the same TCP port on the same IP (port conflict).

    - Because of that, containers can communicate:

        - By TCP/UDP over localhost (127.0.0.1) or the Pod IP.

        - By Unix domain sockets placed on a shared volume (very fast, avoids kernel network stack).

        - By shared filesystem (emptyDir, hostPath) for file-based IPC.

    **Typical flows**

    - Container A curl http://localhost:8080 → kernel routes to process listening on 127.0.0.1:8080 inside that same netns (could be process started by Container B).

    - If the server listens on 0.0.0.0:8080, it receives packets from localhost or external pods (same behavior).



2) Between Pods on the same Node — exact mechanics & packet flow

Namespaces & veth pairs

- Each Pod has its own network namespace and one virtual ethernet pair (veth): one end in pod netns, other end in host netns (usually attached to some bridge or CNI dataplane). The Pod IP lives on the pod-side veth.

### Typical packet path (Pod A → Pod B on same node)

1. Container A sends packet to Pod B IP: ```src=PodA_IP:portA → dst=PodB_IP:portB.```

2. Packet goes out Pod A’s veth into the host network namespace.

3. The CNI dataplane (bridge, vxlan driver, or host routing) forwards the packet to the host-side of Pod B’s veth.

    - If the CNI uses a simple Linux bridge (e.g., cni0) the kernel L2 switch forwards it.

    - If the CNI uses a routed model (Calico), kernel routing rules / BGP ensure correct next-hop.

    - If the CNI uses an overlay, it may encapsulate decapsulate — but for same-node overlay, encapsulation may be skipped.

4. Packet enters Pod B netns via Pod B veth, kernel delivers to container B process listening on that destination port.

5. No SNAT is applied by default — Pod B sees Pod A’s real Pod IP as the source (unless policy or special CNI changes it).

**What about Services (ClusterIP) on same node?**

- If client hits a Service IP:

    - kube-proxy (iptables/ipvs) on the node intercepts and DNATs the traffic to one of the backend Pod IPs (maybe on same node or remote).

    - For iptables mode, the kernel performs NAT (DNAT) and forwards; conntrack tracks the session.

Commands to inspect on the node

(on the host node)
```
# list veth pairs and check which belong to Pod
ip link show
# inspect packet forwarding / bridge
brctl show   # if bridge CNI
# iptables rules for services
sudo iptables -t nat -L -n -v
# ipvs backends (if ipvs mode)
sudo ipvsadm -Ln
# watch veth traffic with tcpdump
sudo tcpdump -i <veth-host-end> -nn
```
Example test

1. kubectl run -it --rm client --image=appropriate/curl -- bash

2. From client pod: curl http://<podB-ip>:8080 (should respond).

3. On node, tcpdump -i any host <podB-ip> to observe unencapsulated packets.

**Performance & latency**

- Same-node Pod-to-Pod is fast (no cross-node hop). Overlay CNIs may still cause slight overhead if they encapsulate packets even on same node, but many optimized CNIs avoid encapsulation for same-node traffic.

- No kube-proxy involvement when talking directly to Pod IP (only used for Service IPs).

3) Between Pods on different Nodes — exact mechanics & packet flow
Two general models for cross-node Pod networking

    1. Routed model (no encapsulation) — nodes exchange routing information (BGP) so each node knows how to reach every Pod IP. Example: Calico in routing mode. Packets are routed natively in the underlay network.

    2. Overlay model (encapsulation) — CNI encapsulates Pod packets (VXLAN, Geneve) and sends them to the destination node IP: the destination node decapsulates and delivers to Pod. Example: Flannel VXLAN, some Calico modes.

Which model is used depends on the CNI and its config.

**Step-by-step packet flow (Pod A on Node1 → Pod B on Node2)**

1. Container A sends packet src=PodA_IP → dst=PodB_IP.

2. Packet leaves Pod A veth → host namespace on Node1.

3. CNI dataplane decides next hop:

    - Routed mode: kernel routing forwards to Node2’s IP (L3), maybe via the cluster network; no encapsulation.

    - Overlay mode: packet is encapsulated in outer IP with src=Node1_IP, dst=Node2_IP and sent over the physical network to Node2.

4. At Node2, if encapsulated, decapsulate; underlying system then forwards packet into Pod B’s veth. Pod B receives dst=PodB_IP.

5. Pod B sees original src=PodA_IP (no SNAT performed by default).

6. Reply goes back the same way.

**Where Services intervene**

- If Pod A connects to a Service ClusterIP instead of PodB IP:

    - kube-proxy on Node1 DNATs packet to chosen backend (if backend on Node2, packet is DNATed to PodB_IP and then forwarded across nodes). conntrack keeps state so return traffic is routed back correctly.

    - IPVS mode may perform load balancing in kernel with less overhead.


**NetworkPolicy & cross-node**

NetworkPolicy enforcement occurs at the dataplane on each node. If policy denies traffic, packets are dropped at node-level before delivery. Implementation differs by CNI.

# Ingress

## hat is Ingress?

- A Kubernetes Ingress is an API object that manages external HTTP/HTTPS access to Services inside the cluster.

- It defines L7 (Layer 7) routing rules (hostnames, paths) instead of just exposing ports like Services do.

- Needs an Ingress Controller to work (Ingress itself is just config).

👉 Service = L4 (TCP/UDP), Ingress = L7 (HTTP/HTTPS).


## Why Ingress?

- Without Ingress:

    - Every app exposed externally needs its own LoadBalancer Service (expensive in cloud) or NodePort (not production-friendly).

    - Hard to do host/path-based routing.

- With Ingress:

    - Single external load balancer → routes traffic to multiple Services.

    - Central point for TLS termination, rewrites, rate limiting, auth, etc.


### Ingress Architecture

1. Ingress Resource (YAML object) → defines rules (host, path, service).

2. Ingress Controller → software that implements the rules (e.g., NGINX, Traefik, HAProxy, Istio Gateway).

3. External Load Balancer / NodePort → exposes the controller Pod to the outside world.

4. CoreDNS → resolves app domains (myapp.com) to the ingress controller’s IP.

Flow:
```
User → DNS (myapp.com) → LoadBalancer IP → Ingress Controller Pod → Ingress rules → Service → Pod
```

### Ingress Resource Anatomy

Example basic Ingress:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /shop
        pathType: Prefix
        backend:
          service:
            name: shop-service
            port:
              number: 80
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
```

- Important Fields

    - ```rules```: list of routing rules.

    - ```host```: domain name (e.g., myapp.com).

    - ```paths```: URL path matching rules.

    - ```pathType```:

        - ```Prefix``` → match based on URL prefix.

        - ```Exact``` → match full path exactly.

        - ```ImplementationSpecific``` → behavior depends on controller.

    - ```backend```: the Service name + port to route traffic.


## Ingress in EKS

1. Why Ingress in EKS?

- By default in Kubernetes, if you want to expose services externally you use Service type=LoadBalancer.

- In AWS this means: every Service = one Classic Load Balancer (CLB) or Network Load Balancer (NLB). Expensive & limited.

- Ingress in EKS lets you:

    - Use a single AWS Application Load Balancer (ALB) to route traffic to multiple Services (cheaper + simpler).

    - Get L7 routing (host/path based).

    - Integrate with AWS IAM, WAF, Cognito, SSL certs from ACM.



2. Components in EKS Ingress Architecture

- Ingress Resource (YAML) → defines routing rules.

- Ingress Controller (must be installed):

    - In AWS EKS, the recommended one is AWS Load Balancer Controller (replaces the old ALB Ingress Controller).

    - Runs as Pods in your cluster, watching for Ingress resources.

- AWS ALB → provisioned automatically by controller when Ingress is applied.

- Target Groups → created per Service backend (Pods behind NodePort or IP mode).

- Security Groups → automatically attached to ALB.

- Route53 (optional) → DNS maps domain → ALB DNS name.
```
Flow:

Client (myapp.com) → Route53 → ALB DNS → ALB Listener rules (Ingress) → TargetGroup → Service → Pod
```

