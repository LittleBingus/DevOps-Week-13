# SDI 4213 Week 13 — Kubernetes Individual Exercise

## Help Desk on Kubernetes

This starter repository contains a small browser-based Help Desk application and partially completed Kubernetes manifests. Your job is to deploy the application and PostgreSQL to **Minikube**, verify persistence and health, scale the application, and demonstrate Kubernetes self-healing.

### Architecture

Browser → Help Desk Service → Help Desk Pods (2+) → PostgreSQL Service → PostgreSQL Pod → PVC

### Important

- Do **not** commit a real password or a generated Kubernetes Secret manifest containing the actual password.
- The provided `k8s/secret-template.yaml` is a template only and is intentionally excluded from `kustomization.yaml`.
- Create the real Secret locally with `kubectl create secret ...` as described in the assignment.

### Quick local test

```bash
python -m venv .venv
# Windows PowerShell: .venv\Scripts\Activate.ps1
# macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
python -m pytest -v
```

See `docs/Week13_Individual_Exercise.docx` for the full assignment.

---

## What I completed

| File | What I changed |
|---|---|
| `k8s/app-deployment.yaml` | Completed the four TODOs: 2 replicas, liveness probe on `/healthz`, readiness probe on `/ready`, requests 100m/64Mi, limits 500m/256Mi |
| `k8s/postgres-deployment.yaml` | Changed the volume `mountPath` from `/var/lib/postgresql/data` to `/var/lib/postgresql`. The `postgres:18` image refuses to start with the old path (it now keeps data in `/var/lib/postgresql/18/docker`), so the pod crash-looped. |
| `tests/test_app.py` | Registered the dynamically loaded module in `sys.modules` so Flask finds `app/templates/` when pytest runs from the repo root |
| `.github/workflows/ci.yml` | New: runs tests, builds the image, renders the manifests, and fails if a Secret is committed |
| `docs/RUNBOOK.md` | New: commands and screenshot checklist |

All other starter files are unchanged.

The database password is the only secret. It lives in the `helpdesk-secret` Secret, which is created locally with `kubectl create secret` and is never committed.

Step-by-step commands and the screenshot checklist are in [`docs/RUNBOOK.md`](docs/RUNBOOK.md).

## Reflection

**1. What did Kubernetes reconcile automatically?**
A Deployment stores a *desired state* (for example, 3 Help Desk replicas). The Deployment's ReplicaSet controller keeps comparing that to the actual state. When I scaled to 3, it created one extra pod. When I deleted a Help Desk pod, only 2 were left, so it created a replacement right away without any command from me. The same happened when I deleted the PostgreSQL pod: the Deployment wanted 1 pod, saw 0, and started a new one. I never restarted anything by hand.

**2. Why is readiness different from liveness?**
Liveness (`/healthz`) only asks whether the Flask process is alive. If it fails, the kubelet restarts the container. Readiness (`/ready`) asks whether the pod can do useful work right now, which here means reaching PostgreSQL. If it fails, the pod is removed from the `helpdesk` Service endpoints, so it gets no traffic, but it is not restarted. If the database goes down, restarting a healthy web process would fix nothing and could cause a restart loop. Marking the pod not-ready instead keeps it running and sends traffic back automatically once the database returns. That is why `/healthz` must not check the database.

**3. Why did the database record survive pod deletion?**
PostgreSQL writes its data under `/var/lib/postgresql`, which is mounted from the PersistentVolumeClaim `postgres-data`. The PVC and its volume are separate objects from the pod and outlive it. When I deleted the pod, the Deployment created a new one that mounted the same PVC and found the existing data files, including my ticket. A container's own filesystem is temporary, so without the PVC the ticket would have been lost. Deleting the PVC or the whole namespace would also delete the data.
