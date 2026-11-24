# Week 13 Runbook & Screenshot Checklist

> **PowerShell users:** the `\` line continuation used in the assignment does not work in PowerShell.
> Put the command on one line (shown below) or use a backtick `` ` `` at the end of each line.

## 01 — Tests pass → `evidence/01-tests-pass.png`
```
python -m venv .venv
.venv\Scripts\Activate.ps1        # macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
python -m pytest -v
```

## 02 — Image built → `evidence/02-image.png`
```
docker build -t week13-helpdesk:v1.0.0 .
docker images week13-helpdesk
```

## 03 — Cluster ready → `evidence/03-cluster-ready.png`
```
minikube start --driver=docker --cpus=2 --memory=4096
minikube image load week13-helpdesk:v1.0.0
minikube status
kubectl get nodes
```
Screenshot `minikube status` + `kubectl get nodes` (STATUS = Ready).
Optional, for `kubectl top` in step 07: `minikube addons enable metrics-server`

## 04 — Completed Deployment → `evidence/04-deployment-yaml.png`
Open `k8s/app-deployment.yaml` in your editor. Make sure `replicas: 2`, both probes and `resources` are visible in the screenshot.

## 05 — Stack running → `evidence/05-stack-running.png`
```
kubectl apply -f k8s/namespace.yaml
kubectl create secret generic helpdesk-secret --namespace week13 --from-literal=DB_PASSWORD=week13-demo-password
kubectl apply -k k8s/
kubectl rollout status deployment/postgres -n week13
kubectl rollout status deployment/helpdesk -n week13
kubectl get pods,svc,pvc -n week13
```
Wait until both helpdesk pods show `1/1 Running` and the PVC shows `Bound`, then screenshot.
(Don't screenshot the `create secret` line, because it contains the password.)

## 06 — Working app + data → `evidence/06-app-ticket.png`
In a **second terminal** (leave it running):
```
kubectl port-forward svc/helpdesk 8080:80 -n week13
```
Browse to http://localhost:8080 and create a ticket: `Week 13 Persistence Test — YOURNAME`. Screenshot the browser.

## 07 — Operational inspection → `evidence/07-operations.png`
```
kubectl describe deployment helpdesk -n week13
kubectl logs deployment/helpdesk -n week13 --tail=30
kubectl top pods -n week13          # only if metrics-server is enabled
```
In `describe`, the useful part shows `Replicas: 2 desired | 2 available`, the Liveness/Readiness lines and the Limits/Requests.

## 08 — Scaled to 3 → `evidence/08-scaled.png`
```
kubectl scale deployment/helpdesk --replicas=3 -n week13
kubectl get pods -n week13 -w
```
Ctrl+C once three helpdesk pods are `1/1 Running`, then `kubectl get pods -n week13` and screenshot.

## 09 — Self-healing → `evidence/09-self-healing.png`
```
kubectl get pods -n week13 -l app=helpdesk
kubectl delete pod <one-helpdesk-pod-name> -n week13
kubectl get pods -n week13 -l app=helpdesk
```
Keep all three commands in one screenshot: the old pod name, the delete, and then three pods including a new one with a low AGE.

## 10 — Persistent data → `evidence/10-persistence.png`
```
kubectl get pods -n week13 -l app=postgres
kubectl delete pod <postgres-pod-name> -n week13
kubectl get pods -n week13 -w
```
When the new postgres pod is `1/1 Running` and the helpdesk pods are Ready again, refresh the browser. The ticket is still there.
If the port-forward dropped, restart it. Tip: put the terminal (showing the new postgres pod) and the browser side by side in one screenshot.

## 11 — GitHub evidence → `evidence/11-github.png`
Commit everything (including `evidence/*.png`), push, and screenshot the repo's **Actions** tab showing the green CI run (or the PR with checks passing).

## Cleanup
```
kubectl delete namespace week13
minikube stop
```

## Troubleshooting
| Symptom | Fix |
|---|---|
| `ImagePullBackOff` / `ErrImageNeverPull` | Re-run `minikube image load week13-helpdesk:v1.0.0`; the tag must match exactly. |
| `CreateContainerConfigError` | The Secret is missing or misnamed: `kubectl get secret helpdesk-secret -n week13`. |
| Helpdesk pods `0/1` Running | `/ready` can't reach Postgres. Check `kubectl logs deployment/helpdesk -n week13` and that the postgres pod is Ready. |
| Helpdesk pod restarts once at first deploy | Normal if Postgres is slow to start. The app waits for the DB before serving `/healthz`. |
| Postgres `password authentication failed` after re-creating the Secret with a new password | The PVC still holds the old database. Run `kubectl delete namespace week13` (wipes the PVC) and redeploy. |
| Postgres pod `CrashLoopBackOff`; logs say *"in 18+, these Docker images are configured to store database data in a format..."* | The volume must mount at `/var/lib/postgresql`, not `/var/lib/postgresql/data` (already fixed in `k8s/postgres-deployment.yaml`). |
| PVC `Pending` | `kubectl get storageclass`. Minikube's default is `standard`. |
