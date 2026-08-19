# CCS3308 Lab 6 — Checkpoint Answers

## Checkpoint Q1
The control plane is the "brain" of the cluster — it manages the overall 
state and makes decisions (the API Server handles requests, etcd stores 
state, the Scheduler places pods, and the Controller Manager maintains 
the desired state). A worker node is the "muscle" — this is where the 
actual application containers/pods run. On a worker node you'll find 
kubelet (manages pod lifecycle), kube-proxy (networking), and the 
container runtime (Docker). Since this lab uses a single-node cluster, 
the one `minikube` node performs both control-plane and worker roles 
simultaneously.

## Checkpoint Q2
After deleting the pod and reapplying the same manifest, the IP address 
changed (old IP: 10.244.0.4 → new IP: 10.244.0.5). This is because 
Kubernetes Pods are ephemeral — they are not designed to be permanent 
or durable. When a Pod is deleted, it is completely destroyed; this is 
not a "restart," it is a full removal of that Pod object. When the 
manifest is reapplied, Kubernetes creates a brand-new Pod object (even 
though it has the same name), and the cluster's networking layer 
dynamically assigns it a new IP address. This is exactly why 
applications should never hardcode a Pod's IP address — instead, 
Kubernetes provides Services which give a stable, unchanging virtual 
IP/DNS name.

## Checkpoint Q3
Using the control-loop model — Desired State → Controller watches → 
Actual State → Gap Detected → Reconcile — here's what happened when I 
deleted a pod: (1) Desired State: the Deployment spec declared 
replicas: 3. (2) Controller Watches: the ReplicaSet controller was 
continuously watching the actual pod count via the API Server. 
(3) Actual State Changes: deleting the pod dropped the running count 
to 2. (4) Gap Detected: the controller immediately detected desired 
(3) ≠ actual (2). (5) Reconcile: the controller instantly requested a 
new Pod from the API Server, the Scheduler placed it on the node, and 
kubelet started the container — within about 2 seconds the new pod 
reached Running, restoring actual state to match desired state. This 
is the self-healing behaviour that standalone Pods lack, since nothing 
watches over them.

## Checkpoint Q4
Each tier can scale independently because each tier has its own 
separate Deployment, managing its own replicas count independently. 
The frontend Deployment and the database StatefulSet are completely 
separate Kubernetes objects — neither depends on the other. The 
frontend is stateless, so any number of replicas can be added or 
removed freely with no risk of data loss. Running 
`kubectl scale deployment frontend --replicas=10` has zero effect on 
the database, since every Deployment/StatefulSet runs its own 
reconciliation loop independently. This is a key advantage of 
microservices architecture — scaling the tier under heavy traffic 
without disturbing any other tier.

## Checkpoint Q5
Port-forward is a temporary, developer-only debugging tool — it 
creates a direct tunnel to one specific pod, and only works while that 
terminal command keeps running. A Service is a permanent, 
production-grade networking object that persists in the cluster. 
Port-forward points to just one pod (breaks if that pod is deleted); a 
Service automatically load-balances traffic across all matching pods. 
Pods are ephemeral and get new IPs when replaced, but a Service's 
IP/DNS name never changes no matter how many times the underlying pods 
are replaced — this is the core purpose of Services. Port-forward 
requires a terminal command to keep running; a NodePort Service can be 
accessed anytime via the node's IP and assigned port with no command 
needing to stay active.

## Checkpoint Q6
Docker Compose has no built-in rolling update mechanism. Updating an 
image in Compose stops and recreates containers, causing a downtime 
window with no automatic, gradual, health-checked rollout. Compose has 
no revision history — rolling back requires manually editing the old 
image tag back into the compose file. Compose also has no logic to 
pause or fail a rollout based on health checks, so a broken container 
could get deployed straight into production. In Kubernetes, 
`kubectl rollout undo` performs a rollback with a single command, 
instantly, with zero downtime. In short, Kubernetes rolling 
updates/rollbacks are declarative, automated, and safe, while Docker 
Compose's approach is manual, all-or-nothing, and risky.

## Checkpoint Q7
The frontend and API tiers are stateless — every pod replica is 
identical and carries no unique data, so they only need a Deployment: 
pod names are random, no storage is required, and scaling is 
straightforward. The database tier is stateful — each replica holds 
unique, persistent data, so it needs a StatefulSet: (1) Pod naming — 
fixed, predictable names (postgres-0, postgres-1...) instead of 
random ones. (2) Storage — volumeClaimTemplates creates a separate, 
persistent PVC for each pod replica, so even if a pod is deleted and 
recreated, the same PVC reattaches and data stays intact. 
(3) Ordering — StatefulSet pods are created/deleted sequentially, 
unlike Deployment pods which are created/deleted in parallel. Identity, 
ordering, and storage don't matter for stateless tiers, but all three 
are critical for stateful tiers like a database.

## Checkpoint Q8
No, this data would not have survived if postgres had been deployed as 
a plain Deployment without a PersistentVolumeClaim. Without a defined 
storage volume, a container's filesystem is ephemeral and tightly 
bound to that specific container's lifecycle — when a pod is deleted, 
the container's writable layer and any data on it is permanently 
destroyed. A new pod starts from a fresh, empty image with no 
knowledge of previous data. With a StatefulSet, the PVC created via 
volumeClaimTemplates is completely decoupled from the container's 
lifecycle — it is a separate Kubernetes object with its own lifecycle, 
which is why the new postgres-0 pod reattaches to the same PVC and 
sees the same old data. In short: without a PVC, data is tied to the 
pod and is lost when the pod is deleted; with a PVC, data is 
independent of the pod and persists at the storage layer.

## Checkpoint Q9
The broken pod showed the status ImagePullBackOff (initially 
ErrImagePull, then switching to ImagePullBackOff after repeated 
retries). This does not directly match any status in the lecture's Pod 
Status table (Running, Pending, CrashLoopBackOff, OOMKilled) — it is a 
related status not explicitly listed there. CrashLoopBackOff means the 
container successfully starts but keeps crashing repeatedly (an 
application-level failure). ImagePullBackOff means the container never 
even gets to start, because Kubernetes cannot pull the image itself 
from the registry (wrong tag, image doesn't exist, credential issues, 
etc.). Both follow the same underlying pattern — try, fail, wait with 
exponential backoff, retry — which is what the "BackOff" suffix means, 
preventing a broken pod from wasting cluster resources through 
constant aggressive retries. This makes sense because Pending/Running/
CrashLoopBackOff describe the container's runtime lifecycle, while 
ImagePullBackOff fails at an earlier stage — image acquisition — so 
it's technically an image pull error status, not a container runtime 
status.
