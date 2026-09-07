# DSO202 — Assignment 1: Three-Tier Task Tracker on Kubernetes

Cluster: kind, 1 control-plane + 2 workers (config: `cluster/kind-cluster.yaml`)
Namespace: `dso202-assignment-01`
Images used: `rynorbu11/dso202-frontend:1.0`, `rynorbu11/dso202-backend:1.0`, `rynorbu11/dso202-db:1.0`
(Note: the originally-issued `sarojsanyasi/dso202-*` images were published `linux/arm64`-only
and would not pull on this `amd64` Windows host — `no matching manifest for linux/amd64`.
A same-spec `amd64` rebuild from a classmate/tutor, `rynorbu11/dso202-*:1.0`, was confirmed via
`docker image inspect --format "{{.Os}}/{{.Architecture}}"` to be `linux/amd64` for all three
tiers and used instead. This should be flagged to the module tutor as an architecture-publishing
gap affecting any student on an amd64 machine.)

---

## Task 1 — Architecture Note

**Control-plane components involved in scheduling and running the three Pods:**
- **kube-apiserver** — receives every `kubectl apply`/`create` request (namespace, ConfigMap,
  Secret, PVC, Deployments, Services) and validates/persists the desired state.
- **etcd** — the backing store where that desired state (all objects above) is durably written.
- **kube-scheduler** — watches for unscheduled Pods (the three Deployments' Pods) and assigns
  each to a Node. In this cluster the `control-plane` node carries a
  `node-role.kubernetes.io/control-plane:NoSchedule` taint, so the scheduler places all three
  application Pods only on `worker-node-1` / `worker-node-2` (confirmed: the database Pod landed
  on `worker-node-1`, the backend Pod on `worker-node-2`, the frontend Pod on `worker-node-1`).
- **kube-controller-manager** — runs the Deployment/ReplicaSet controllers; this is what
  recreated the backend Pod automatically after it was manually deleted in Task 7c.

**Node components on whichever worker a Pod lands on:**
- **kubelet** — pulls the container image, starts/monitors the container, reports Pod status
  back to the API server.
- **kube-proxy** — programs the Service virtual IPs/DNS-backed routing so that `db-svc`,
  `backend-svc`, and `frontend-svc` route traffic to the correct Pod IP.
- **containerd** — the container runtime actually running each image.

**Kubernetes objects used per tier, and why:**
- **Database** → `Deployment` (1 replica) + `PersistentVolumeClaim` + headless `Service`
  (`clusterIP: None`). Needs stable, durable storage across restarts; does not need
  load-balancing or an external IP, and per the assignment's constraints must never be
  reachable outside the namespace.
- **Backend** → `Deployment` + `ClusterIP` `Service`. Stateless REST API; only needs to be
  reachable from the frontend, inside the cluster.
- **Frontend** → `Deployment` + `NodePort` `Service`. Stateless static UI; needs to be reachable
  from outside the cluster (the browser), hence the only tier permitted a NodePort.

---

## Task 2 — Secret Encoding Caveat

Confirmed directly: `kubectl get secret app-secret -o yaml` returns `data.DB_PASSWORD:
cGFzc3dvcmQxMjM=` etc. — this is **base64, not encryption**. Anyone with read access to this
Secret object, or to etcd directly, can trivially decode it (`echo cGFzc3dvcmQxMjM= | base64
-d` → `password123`). Encryption at rest requires a cluster-level `EncryptionConfiguration`,
which is out of scope for this assignment.

---

## Task 6 — ResourceQuota / LimitRange Justification

| Deployment | CPU request | CPU limit | Memory request | Memory limit | Reasoning |
|---|---|---|---|---|---|
| database  | 250m | 500m | 128Mi | 256Mi | Postgres idles most of the time for a single test table; small headroom kept for writes |
| backend   | 250m | 500m | 128Mi | 256Mi | Simple REST API, low per-request cost, single replica |
| frontend  | 250m | 500m | 128Mi | 256Mi | Static nginx serving, negligible CPU/memory footprint |

These per-container values come from the namespace `LimitRange`
(`defaultRequest: 250m/128Mi`, `default limit: 500m/256Mi`), confirmed via
`kubectl describe limitrange dso202-limitrange` and visible on every Pod's
`kubernetes.io/limit-ranger` annotation in `kubectl describe pod`.

The namespace `ResourceQuota` (`requests.cpu: 1500m`, `requests.memory: 768Mi`,
`limits.cpu: 3`, `limits.memory: 1536Mi`, `pods: 10`) leaves roughly 2x headroom above the sum
of the three Deployments' requests, so a rolling update (which briefly runs an old + new Pod
side-by-side) does not get blocked by the quota. Confirmed via
`kubectl describe resourcequota dso202-quota` showing `Used: 0` against these hard caps prior
to deploying any workload, then successfully absorbing all three Deployments afterward.

---

## Task 7 — Verification Evidence

### 7a — Full CRUD cycle (via curl through a port-forwarded backend — see "Known Limitation" below)

```
$ curl -X POST http://localhost:8081/api/tasks -H "Content-Type: application/json" -d "{\"title\":\"Write assignment README\"}"
{"id":4,"title":"Write assignment README","description":null,"status":"pending","created_at":"2026-09-07T08:52:39.973Z"}

$ curl http://localhost:8081/api/tasks
[... tasks 1-5 listed, including seed data (id 1-3) from the database image's init script ...]

$ curl -X PUT http://localhost:8081/api/tasks/1 -H "Content-Type: application/json" -d "{\"status\":\"done\"}"
{"id":1,"title":"Set up kind cluster","description":"Bring up the local cluster from Practical 1","status":"done","created_at":"2026-09-07T08:40:24.942Z"}

$ curl -X DELETE http://localhost:8081/api/tasks/1

$ curl http://localhost:8081/api/tasks
[task id 1 no longer present; ids 2,3,4,5 remain]
```

Create, read, update, and delete all confirmed working end-to-end through the backend, database,
and their Service wiring.

### 7b — Service DNS resolution (from inside the frontend Pod)

```
$ kubectl exec -it deploy/frontend-deployment -- sh -c "curl -s http://backend-svc:8080/api/status"
{"status":"ok","db":"connected"}
```

Confirms cluster DNS resolves the `backend-svc` Service name correctly from inside the cluster.

### 7c — Self-healing and data persistence

```
$ kubectl get pods -l tier=backend
NAME                                  READY   STATUS    RESTARTS   AGE
backend-deployment-55f8f9b46b-g87pj   1/1     Running   0          13m

$ kubectl delete pod backend-deployment-55f8f9b46b-g87pj
pod "backend-deployment-55f8f9b46b-g87pj" deleted

$ kubectl get pods -l tier=backend --watch
NAME                                  READY   STATUS    RESTARTS   AGE
backend-deployment-55f8f9b46b-jcdnx   1/1     Running   0          34s
```

The ReplicaSet immediately created a replacement Pod (`...jcdnx`) after the original
(`...g87pj`) was deleted. After the new Pod became `Running`, the tasks created before the
deletion (ids 2, 3, 4, 5) were confirmed still retrievable:

```
$ curl http://localhost:8081/api/tasks
[{"id":2, ...}, {"id":3, ...}, {"id":4, ...}, {"id":5, ...}]
```

This demonstrates Pod lifecycle (backend, ephemeral/replaceable) and PersistentVolume lifecycle
(database, durable via `db-pvc`) are independent — the backend Pod's identity changed but the
data it serves, which lives in Postgres, did not.

### 7d — Declarative vs. imperative comparison

```
$ kubectl create deployment demo-imperative --image=nginx:alpine --namespace=dso202-assignment-01 --replicas=1
deployment.apps/demo-imperative created

$ kubectl get deployment demo-imperative
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
demo-imperative   0/1     1            0           0s

$ kubectl delete deployment demo-imperative
deployment.apps "demo-imperative" deleted
```

Compared against the declarative equivalent used throughout this assignment
(`kubectl apply -f backend/deployment.yaml`, etc.):

The imperative command (`kubectl create deployment ...`) is faster for one-off testing but
leaves no durable record — if the cluster is rebuilt, the resource is gone unless the exact
command is remembered and rerun manually. The declarative manifest, applied via `kubectl apply
-f`, is version-controlled, reviewable, and idempotent — reapplying it reconciles the cluster to
match the file rather than erroring or duplicating resources. For anything meant to persist or
be reproduced (this entire assignment), declarative is the correct approach; imperative is
useful only for quick manual exploration, which is why the throwaway Deployment above was
deleted immediately after the comparison.

---

## Known Limitation — Frontend Browser Access

Opening `http://localhost:30080` in a browser shows a "Failed to fetch" error in the UI.
Diagnosed as follows:

- `kubectl exec -it deploy/frontend-deployment -- sh -c "cat /usr/share/nginx/html/config.js"`
  confirmed the entrypoint's `envsubst` substitution works correctly, rendering
  `BACKEND_URL: "http://backend-svc:8080"` into the served config.
- `kubectl exec ... "cat /etc/nginx/conf.d/default.conf"` (and the full nginx.conf) showed the
  frontend image contains **no `/api` reverse-proxy rule** — it only serves static files
  (`try_files $uri $uri/ /index.html`).
- `backend-svc` is a cluster-internal DNS name, resolvable only inside the cluster (confirmed
  working via `kubectl exec` in Task 7b). A browser on the host machine has no way to resolve
  it, and per the assignment's non-negotiable constraints the backend must never be exposed via
  NodePort/LoadBalancer to work around this.

Since fixing this would require modifying the (out-of-scope) frontend image's nginx config,
all Task 7 verification was instead performed via `curl` through a port-forwarded backend
Service and via `kubectl exec` — both explicitly permitted alternate methods per Task 7a/7b of
the brief. (Optional workaround for a live demo: `kubectl port-forward svc/backend-svc
8080:8080` plus a temporary `127.0.0.1 backend-svc` entry in the host's `hosts` file lets the
browser resolve the name locally — not required for grading, just useful to show the UI live.)

---

## Provided Images

| Tier | Image | Tag used | Internal port |
|---|---|---|---|
| Frontend | rynorbu11/dso202-frontend | 1.0 | 8080 |
| Backend  | rynorbu11/dso202-backend  | 1.0 | 8080 |
| Database | rynorbu11/dso202-db       | 1.0 | 5432 |

## Environment Variable Contract

| Variable | Consumed by | Source | Notes |
|---|---|---|---|
| DB_HOST | Backend | ConfigMap | = `db-svc` |
| DB_PORT | Backend | ConfigMap | 5432 |
| DB_NAME | Backend | ConfigMap | must equal POSTGRES_DB |
| DB_USER | Backend | Secret | must equal POSTGRES_USER |
| DB_PASSWORD | Backend | Secret | must equal POSTGRES_PASSWORD |
| APP_PORT | Backend | ConfigMap | 8080 |
| CORS_ORIGIN | Backend | ConfigMap | `*` (see caveat below) |
| POSTGRES_DB | Database | ConfigMap | = DB_NAME |
| POSTGRES_USER | Database | Secret | = DB_USER |
| POSTGRES_PASSWORD | Database | Secret | = DB_PASSWORD |
| BACKEND_URL | Frontend | ConfigMap | `http://backend-svc:8080` |

**CORS note:** CORS is enabled permissively (`*`) for classroom simplicity. A production
deployment would restrict `CORS_ORIGIN` to the known frontend origin(s) only.

## Service Naming Convention

| Tier | Type | Name |
|---|---|---|
| Frontend | NodePort | `frontend-svc` |
| Backend | ClusterIP | `backend-svc` |
| Database | Headless (`clusterIP: None`) | `db-svc` |
