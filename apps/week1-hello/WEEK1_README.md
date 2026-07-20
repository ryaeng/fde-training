# Week 1 — Foundation + GitOps

Goal: stand up a local cluster, install ArgoCD, push a change, watch it
reconcile. By the end you should be able to explain "declarative state +
drift correction" without reaching for a definition.

Prerequisite: Docker Desktop (or another local Docker runtime) running.
`kind` runs Kubernetes nodes as Docker containers, so no VM/cloud needed.

---

## Session 1 — cluster + kubectl context

```bash
# install kind + kubectl if you don't have them (macOS example)
brew install kind kubectl

# create the cluster from the config in this repo
kind create cluster --config kind-config.yaml

# confirm you have 3 nodes and can talk to the cluster
kubectl get nodes
kubectl cluster-info
```

You should see 1 control-plane and 2 worker nodes, all `Ready`.

---

## Session 2 — install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# wait for it to come up (can take a couple minutes on a laptop)
kubectl get pods -n argocd -w
```

Port-forward the UI so you can watch syncs visually (optional but worth
it the first time):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Log in at https://localhost:8080 as `admin` with that password.

---

## Session 3 — push this repo and wire up the Application

1. Push this repo to your own git remote — a private GitHub repo is fine,
   ArgoCD just needs a URL it can reach.

2. Check `apps/week1-hello/argocd-app.yaml` and make sure `spec.source.repoURL`
   points at your actual remote (it currently points at
   `https://github.com/ryaeng/fde-training.git` — change it if you forked).

3. Apply it:

```bash
kubectl apply -f apps/week1-hello/argocd-app.yaml
```

4. Watch it sync:

```bash
kubectl get application -n argocd
kubectl get pods -n week1-hello -w
```

You should see `hello-gitops` come up with 2 replicas, and the
Application show `Synced` / `Healthy`.

---

## Session 4 — actually see the reconcile loop (the point of the whole week)

Do these one at a time and watch what happens each time:

**A. Change git, watch cluster follow:**
Edit `deployment.yaml` in your pushed repo — bump `replicas: 2` to
`replicas: 3` — commit, push. Within ~3 minutes (or force it with
`argocd app sync week1-hello` if you installed the CLI) you should see
a third pod appear. Nobody ran `kubectl apply` by hand.

**B. Change cluster, watch git win (selfHeal):**
```bash
kubectl scale deployment hello-gitops -n week1-hello --replicas=5
```
Watch it get scaled back down to whatever git says. This is the
"drift correction" half of GitOps — the cluster is not the source of
truth, the repo is.

**C. Delete from git, watch cluster follow (prune):**
Remove `deployment.yaml` from the repo, commit, push. The pods should
disappear from the cluster without you touching kubectl.

---

## Checkpoint before moving to Week 2

You should be able to say, in your own words, what happens in each of
A/B/C above and *why* — not just that it works, but which component
(git, ArgoCD controller, or the cluster's own control loop) is doing
what. If B or C feels like magic rather than mechanism, re-run them
once more while reading the ArgoCD UI's event log side by side.

Next: `kubernetes/sample-controller` — just enough Go to read a
reconcile loop, ahead of Volcano's scheduler source in Week 2.
