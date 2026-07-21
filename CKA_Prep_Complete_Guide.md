# CKA Complete Prep Guide

A full path from zero to exam day: schedule, drills, and hands-on scenarios.

---

## 1. 6-Week Study Schedule

### Week 1 — Cluster Architecture & Installation
| Day | Focus | Task |
|---|---|---|
| 1 | Control plane components | Diagram kube-apiserver, etcd, scheduler, controller-manager, kubelet, kube-proxy — explain what each does |
| 2 | kubeadm | Spin up a 3-node cluster (kubeadm init + join) in a VM/cloud sandbox |
| 3 | Static pods | Create a static pod via `/etc/kubernetes/manifests`, verify it reappears if deleted |
| 4 | RBAC | Create Roles, ClusterRoles, RoleBindings, ServiceAccounts; test with `kubectl auth can-i` |
| 5 | etcd backup/restore | Practice `etcdctl snapshot save` and `snapshot restore` end to end |
| 6 | Cluster upgrade | Upgrade a kubeadm cluster one minor version using the official upgrade docs |
| 7 | Review + quiz yourself | Redo any task from memory, no docs |

### Week 2 — Workloads & Scheduling
| Day | Focus | Task |
|---|---|---|
| 8 | Deployments/ReplicaSets | Create, scale, rolling update, rollback (`kubectl rollout undo`) |
| 9 | DaemonSets/StatefulSets/Jobs/CronJobs | Deploy one of each, understand use cases |
| 10 | ConfigMaps & Secrets | Mount as env vars and as volumes |
| 11 | Resource requests/limits | Set requests/limits, observe OOMKill and throttling |
| 12 | Scheduling — affinity | nodeSelector, nodeAffinity, podAffinity/antiAffinity |
| 13 | Scheduling — taints/tolerations | Taint a node, deploy pods with/without tolerations |
| 14 | Review + quiz yourself | Timed mixed drill (10 tasks, 45 min) |

### Week 3 — Services, Networking & Storage
| Day | Focus | Task |
|---|---|---|
| 15 | Services | ClusterIP, NodePort, LoadBalancer — test connectivity between pods |
| 16 | Ingress | Deploy an ingress controller + Ingress resource, test routing |
| 17 | NetworkPolicies | Write policies to allow/deny traffic between namespaces (practice a LOT here) |
| 18 | CoreDNS | Break DNS resolution intentionally, then fix it |
| 19 | PV/PVC | Create PVs with different reclaim policies, bind PVCs |
| 20 | StorageClasses | Dynamic provisioning with a StorageClass |
| 21 | Review + quiz yourself | Timed mixed drill |

### Week 4 — Troubleshooting Deep Dive (highest exam weight)
| Day | Focus | Task |
|---|---|---|
| 22 | Node troubleshooting | Stop kubelet, diagnose with `journalctl -u kubelet`, fix |
| 23 | Control plane troubleshooting | Break the apiserver static pod manifest, diagnose, fix |
| 24 | App troubleshooting | Misconfigure liveness/readiness probes, fix crash loops |
| 25 | Networking troubleshooting | Break a Service selector mismatch, diagnose, fix |
| 26 | Logs & events | Practice `kubectl logs`, `kubectl describe`, `kubectl get events --sort-by` |
| 27 | Full cluster failure scenario | Simulate 2-3 compounding failures, fix all |
| 28 | Review + quiz yourself | Timed mixed drill |

### Week 5 — Speed & Simulation
- Do the **Killer.sh** simulator session #1 (comes free with registration), full 2 hours, timed.
- Review every missed question — understand *why*, not just the fix.
- Rebuild your `kubectl` muscle memory: aliases, `--dry-run=client -o yaml`, vim shortcuts.
- Repeat weakest domain's practice day.

### Week 6 — Final Polish
- Killer.sh simulator session #2, timed.
- Do 2-3 more timed mixed drills covering all domains.
- Light review only — no new topics. Sleep well before exam day.

---

## 2. Essential Setup (do this before you start any drills)

```bash
# Aliases — set these up on day 1 and use them from now on
alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# Bash completion
source <(kubectl completion bash)
complete -F __start_kubectl k

# Vim config for fast YAML editing
cat >> ~/.vimrc << 'EOF'
set ts=2 sts=2 sw=2 et
set number
EOF
```

---

## 3. kubectl Speed Drills

Do these until they're automatic — no thinking required.

```bash
# 1. Create a deployment with 3 replicas, image nginx, generate YAML only
k create deployment web --image=nginx --replicas=3 $do > dep.yaml

# 2. Expose it as a NodePort service on port 80
k expose deployment web --port=80 --type=NodePort $do > svc.yaml

# 3. Create a pod with resource limits, no YAML file, straight from CLI
k run limited --image=nginx --dry-run=client -o yaml \
  --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi > pod.yaml

# 4. Generate a ConfigMap from literals
k create configmap app-config --from-literal=ENV=prod --from-literal=DEBUG=false

# 5. Generate a Secret from literals
k create secret generic app-secret --from-literal=password=s3cr3t

# 6. Quickly check what fields a resource supports
k explain pod.spec.containers --recursive | less

# 7. Scale a deployment
k scale deployment web --replicas=5

# 8. Rolling restart + check rollout status
k rollout restart deployment web
k rollout status deployment web

# 9. Undo a bad rollout
k rollout undo deployment web

# 10. Get all resources across all namespaces fast
k get all -A

# 11. Cordon, drain, uncordon a node
k cordon node01
k drain node01 --ignore-daemonsets --delete-emptydir-data
k uncordon node01

# 12. Taint a node and tolerate it
k taint nodes node01 key=value:NoSchedule
# In pod spec: tolerations: [{key: "key", operator: "Equal", value: "value", effect: "NoSchedule"}]
```

**Drill goal:** each command above should take you under 20 seconds to type/produce correctly, with zero syntax lookups by week 4.

---

## 4. Hands-On Practice Scenarios

Set up a real (or KillerCoda/Play-with-K8s) cluster for these — don't just read them.

### Scenario A: Broken kubelet
1. SSH into a worker node.
2. `sudo systemctl stop kubelet`
3. From the control plane, notice the node goes `NotReady`.
4. Diagnose using `kubectl describe node <node>` and `journalctl -u kubelet -f` on the worker.
5. Fix it and confirm the node returns to `Ready`.

### Scenario B: Broken control plane (static pod)
1. Edit `/etc/kubernetes/manifests/kube-apiserver.yaml` and introduce a typo (e.g., wrong flag).
2. Watch the apiserver static pod fail to come up.
3. Use `crictl ps -a` / `crictl logs` on the control plane node (kubectl won't work since apiserver is down!) to diagnose.
4. Fix the manifest, confirm the cluster recovers.

### Scenario C: etcd disaster recovery
1. Take a snapshot: `ETCDCTL_API=3 etcdctl snapshot save /tmp/backup.db --endpoints=... --cacert=... --cert=... --key=...`
2. Create a test resource (e.g., a namespace with a deployment).
3. "Simulate disaster" — delete the namespace.
4. Restore etcd from the snapshot to a new data directory, update the static pod manifest to point to it.
5. Confirm the deleted namespace/deployment is back.

### Scenario D: RBAC lockdown
1. Create a ServiceAccount `deploy-bot` in namespace `dev`.
2. Create a Role that only allows `get`, `list`, `watch` on pods.
3. Bind it with a RoleBinding.
4. Test: `kubectl auth can-i list pods --as=system:serviceaccount:dev:deploy-bot -n dev` (should be yes)
5. Test: `kubectl auth can-i delete pods --as=system:serviceaccount:dev:deploy-bot -n dev` (should be no)

### Scenario E: NetworkPolicy isolation
1. Create namespaces `frontend` and `backend`, each with a test pod.
2. Confirm pods can reach each other by default.
3. Apply a NetworkPolicy in `backend` that only allows ingress from pods labeled `role: frontend`.
4. Confirm frontend pod (with correct label) can reach backend, and a random pod cannot.

### Scenario F: Storage — PV/PVC/StorageClass
1. Create a PV (hostPath is fine for practice) with `ReclaimPolicy: Retain`.
2. Create a PVC that binds to it.
3. Mount the PVC in a pod, write a file, delete the pod, recreate it — confirm data persists.
4. Delete the PVC, check the PV status (`Released`), and manually reclaim it.

### Scenario G: Crash-looping app
1. Deploy an app with a liveness probe pointing to the wrong port/path.
2. Watch it enter `CrashLoopBackOff`.
3. Use `kubectl describe pod` and `kubectl logs --previous` to diagnose.
4. Fix the probe, confirm the pod stabilizes.

### Scenario H: Compounding failure (mock exam style)
Set up all of these at once in a sandbox, then fix them within 30 minutes:
- A deployment with a bad image tag
- A Service with a selector that doesn't match any pod labels
- A NetworkPolicy that's overly restrictive and blocking legitimate traffic
- A node that's cordoned and shouldn't be

---

## 4b. More Likely Exam Scenarios

These round out topics candidates commonly report seeing. Same rule: build them, don't just read them.

### Scenario I: Multi-container pod (init container + sidecar)
1. Create a pod with an **init container** that writes a file to a shared `emptyDir` volume before the main container starts.
2. Add a second container in the same pod (sidecar) that tails a log file the main container writes.
3. Verify startup order with `kubectl describe pod` (init containers run to completion first) and check both containers' logs with `kubectl logs <pod> -c <container>`.

### Scenario J: Jobs & CronJobs
1. Create a `Job` that runs a pod to completion (e.g., prints something and exits 0). Verify `kubectl get jobs` shows `COMPLETIONS 1/1`.
2. Create a `Job` with `backoffLimit: 3` where the container exits with a non-zero code — watch it retry then fail.
3. Create a `CronJob` running every minute; verify it spawns Jobs on schedule and check `kubectl get cronjob,job,pod`.
4. Suspend the CronJob (`kubectl patch cronjob ... -p '{"spec":{"suspend":true}}'`) and confirm no new jobs spawn.

### Scenario K: ResourceQuota & LimitRange
1. Create a `ResourceQuota` in a namespace capping total pods to 3 and total CPU requests to 1 core.
2. Try creating a 4th pod — confirm it's rejected with a quota-exceeded error.
3. Create a `LimitRange` that sets a default CPU/memory request+limit for containers that don't specify one.
4. Deploy a pod without resource fields and confirm the defaults were applied (`kubectl describe pod`).

### Scenario L: PodDisruptionBudget
1. Deploy a 3-replica Deployment.
2. Create a `PodDisruptionBudget` with `minAvailable: 2`.
3. Attempt `kubectl drain` on the node hosting 2+ of those pods and observe the drain respecting the PDB (it should block/wait until it's safe).

### Scenario M: Certificate management (kubeadm)
1. Check certificate expiration: `kubeadm certs check-expiration`.
2. Renew all certs: `kubeadm certs renew all`.
3. Restart static control-plane pods (kubelet picks up new certs automatically after restart) and confirm cluster still functions with `kubectl get nodes`.

### Scenario N: CSR (CertificateSigningRequest) workflow
1. Generate a private key + CSR for a new user (`openssl req -new ...`).
2. Submit it as a Kubernetes `CertificateSigningRequest` object (base64-encoded in `.spec.request`).
3. Approve it: `kubectl certificate approve <name>`.
4. Extract the issued cert, build a kubeconfig for that user, and bind them to a Role via RoleBinding. Confirm access with `kubectl auth can-i --as=<user>`.

### Scenario O: Multiple scheduling controls together
1. Label two nodes: `disktype=ssd` and `disktype=hdd`.
2. Deploy a pod using `nodeAffinity` (preferred, not required) favoring `ssd` nodes with a weight.
3. Add `podAntiAffinity` so replicas of the same app avoid co-locating on the same node.
4. Scale to 3 replicas and confirm placement matches the rules using `kubectl get pods -o wide`.

### Scenario P: Custom scheduler / manual binding
1. Create a pod with `schedulerName: my-scheduler` (a scheduler that doesn't exist).
2. Confirm it stays `Pending` forever (`kubectl describe pod` shows no scheduler assigned it).
3. Manually bind it to a node by creating a `Binding` object via the API (simulates understanding what a scheduler actually does).

### Scenario Q: Static pod without kubeadm helpers
1. On a worker node, find the kubelet's `--pod-manifest-path` (or `staticPodPath` in the kubelet config file).
2. Drop a raw pod YAML into that directory manually (no `kubectl apply`).
3. Confirm the pod appears in `kubectl get pods` on the control plane with a `-<nodename>` suffix, and that deleting it via `kubectl delete` doesn't actually remove it (only removing the file does).

### Scenario R: Ingress with TLS
1. Generate a self-signed cert/key, store it as a TLS `Secret`.
2. Create an Ingress resource referencing that secret under `spec.tls`.
3. Test HTTPS access to the service through the ingress controller and confirm the cert served matches.

### Scenario S: Headless Service & DNS
1. Create a `StatefulSet` with a headless Service (`clusterIP: None`).
2. From a debug pod, run `nslookup <service-name>` and confirm it returns each pod's individual IP rather than a single VIP.
3. Run `nslookup <pod-name>.<service-name>` to hit a specific StatefulSet replica directly.

### Scenario T: Full "day in the life" mock exam
Give yourself 60 minutes, no notes beyond kubernetes.io docs, and complete all of these against one cluster:
1. Create a namespace and a ResourceQuota in it.
2. Deploy an app with a ConfigMap-mounted config file and a readiness probe.
3. Expose it via ClusterIP Service, then via Ingress.
4. Write a NetworkPolicy restricting access to only a specific namespace.
5. Take an etcd snapshot.
6. Diagnose and fix an intentionally broken node (stopped kubelet).
7. Create a ServiceAccount with a Role scoped to read-only pod access.
8. Scale the deployment and perform a rolling update, then roll it back.

Grade yourself: if you finish all 8 in under 60 minutes with all resources verified working, you're in good shape for the real thing.

---

## 4c. Even More Scenarios

### Scenario U: Topology Spread Constraints
1. Label nodes across two "zones" (`topology.kubernetes.io/zone=zone-a` / `zone-b`).
2. Deploy a Deployment with `topologySpreadConstraints` (`maxSkew: 1`, `whenUnsatisfiable: DoNotSchedule`) keyed on that zone label.
3. Scale to 4 replicas and confirm pods spread evenly across zones (`kubectl get pods -o wide`).
4. Cordon one zone's node and observe scheduling behavior change.

### Scenario V: PriorityClass & preemption
1. Create two `PriorityClass` objects: `high-priority` (value 1000000) and `low-priority` (value 100).
2. Fill a node's capacity with low-priority pods.
3. Deploy a high-priority pod that doesn't fit — confirm a low-priority pod gets evicted/preempted to make room.

### Scenario W: Pod Security Admission
1. Label a namespace with `pod-security.kubernetes.io/enforce=restricted`.
2. Try deploying a pod running as root / with a privileged container — confirm it's rejected.
3. Fix the pod spec (`runAsNonRoot: true`, drop capabilities, etc.) and confirm it's admitted.

### Scenario X: Image pull secrets
1. Create a `docker-registry` type Secret with (fake or real) registry credentials.
2. Reference it in a pod's `imagePullSecrets`.
3. Intentionally use a private image without the secret first — observe `ImagePullBackOff` — then add the secret and confirm it pulls.

### Scenario Y: ServiceAccount token projection
1. Create a pod with a **projected volume** mounting a ServiceAccount token with a custom `audience` and `expirationSeconds`.
2. Exec into the pod and read the token file, decode it (e.g., paste into jwt.io logic manually or `cut -d. -f2 | base64 -d`) to confirm audience/expiry.
3. Contrast with the default auto-mounted token behavior (and how to disable it via `automountServiceAccountToken: false`).

### Scenario Z: Debugging a Pending pod (insufficient resources)
1. Request an absurd amount of CPU/memory in a pod spec (more than any node has).
2. Confirm it stays `Pending` and use `kubectl describe pod` to read the scheduler's `FailedScheduling` event explaining why.
3. Fix the resource request and confirm it schedules.

### Scenario AA: PVC stuck pending (storage troubleshooting)
1. Create a PVC requesting a StorageClass that doesn't exist (typo).
2. Confirm it stays `Pending` — describe it to see the exact error.
3. Fix the StorageClass name and confirm it binds. Also practice: PVC requesting more storage than any PV offers, with no dynamic provisioner available.

### Scenario AB: Kubeconfig & multi-cluster contexts
1. Given two separate kubeconfig files (or contexts), merge them: `KUBECONFIG=file1:file2 kubectl config view --flatten > merged`.
2. List all contexts, switch between clusters (`kubectl config use-context`), and set a default namespace per context.
3. Confirm you can run commands against the *correct* cluster without accidentally targeting the wrong one — this trips people up constantly in the real exam since tasks specify contexts explicitly.

### Scenario AC: kubeadm join with tokens
1. On the control plane, generate a new bootstrap token: `kubeadm token create --print-join-command`.
2. List current tokens: `kubeadm token list`.
3. Use the generated command to join a new (or reset) node to the cluster.
4. Confirm the node appears with `kubectl get nodes` and eventually goes `Ready`.

### Scenario AD: Draining a control-plane node
1. On a single-control-plane cluster, note the control-plane taint (`node-role.kubernetes.io/control-plane:NoSchedule`).
2. Try scheduling a normal pod without tolerations — confirm it never lands there.
3. Drain the control-plane node with `--ignore-daemonsets` and observe what happens to static pods (they aren't evicted — they're managed by the kubelet directly, not the scheduler).

### Scenario AE: Ephemeral containers (live debugging)
1. Deploy a minimal/distroless container with no shell.
2. Attempt `kubectl exec` — confirm it fails.
3. Use `kubectl debug <pod> -it --image=busybox --target=<container>` to attach an ephemeral debug container sharing the target's process namespace, and inspect it from there.

### Scenario AF: Helm basics (if in your exam version's scope — check current curriculum)
1. Add a Helm repo, `helm search repo`, and `helm install` a simple chart (e.g., nginx).
2. List releases: `helm list`.
3. Upgrade the release with a changed value (`helm upgrade --set ...`).
4. Roll it back: `helm rollback <release> <revision>`.

### Scenario AG: kustomize
1. Create a `kustomization.yaml` referencing a base Deployment manifest.
2. Add a patch that changes the replica count and image tag for an "overlay" (e.g., staging vs prod).
3. Build and apply: `kubectl apply -k <dir>`, confirm the patched values took effect over the base.

---

## 5. Exam Day Checklist

- [ ] Test your webcam, mic, and ID ahead of time (PSI/exam proctor requirements)
- [ ] Clear your desk — no extra monitors, phones, notes
- [ ] Open only the allowed browser tab(s): kubernetes.io docs
- [ ] Set up `k=kubectl` alias and `$do` export immediately when the terminal loads
- [ ] Read every question fully before touching the terminal — note the required namespace/context
- [ ] Use `kubectl config use-context <name>` — many questions specify a different cluster/context
- [ ] Flag hard questions and move on — come back at the end
- [ ] Verify your work (`kubectl get`, `kubectl describe`) before moving to the next question
- [ ] Watch the clock — don't burn 20 minutes on a 4% question

---

## 6. Resources
- **Killer.sh** — included free with exam registration, closest thing to real exam difficulty
- **Kubernetes official docs** — practice navigating and searching fast, since you'll use it live
- **KillerCoda / Play with Kubernetes** — free browser-based scratch clusters
- **KodeKloud CKA course** — structured video + labs

---

Good luck — the exam rewards speed and calm troubleshooting more than memorization. Practice the scenarios until they're boring.
