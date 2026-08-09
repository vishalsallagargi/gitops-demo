# gitops-demo

A hands-on GitOps learning project: nginx deployed to `dev`, `staging`, `prod`, and `qa`
on Kubernetes, entirely driven by Git and reconciled by [Argo CD](https://argo-cd.readthedocs.io/).

Nothing in the cluster is ever changed by hand. Every change starts as a commit.

---

## Architecture

```
apps/nginx/
  base/                   # shared Deployment + Service (no environment-specific values)
  site/                   # the actual app source: Dockerfile + index.html, built by CI

environments/
  dev/                    # overlay: 1 replica, auto-updated image tag from CI
  staging/                # overlay: 2 replicas
  prod/                   # overlay: 3 replicas, resource requests/limits
  qa/                     # overlay: 1 replica - added later with zero Argo CD config

bootstrap/
  applicationset.yaml     # one ApplicationSet that generates an Argo CD Application
                           # per folder under environments/

.github/workflows/
  build-and-deploy.yml    # CI: build image -> push to GHCR -> commit new tag to environments/dev
```

Each environment overlay is a [Kustomize](https://kustomize.io/) build on top of `apps/nginx/base`,
patching only what differs (replica count, namespace, resources, image tag). Argo CD watches
this repo and applies whatever `kubectl kustomize environments/<env>` renders.

---

## Step 1 — Local cluster

Used [`kind`](https://kind.sigs.k8s.io/) to run a disposable local Kubernetes cluster, so nothing
touches a real environment while learning.

```bash
kind create cluster --name gitops-demo
```

`kubectl`'s context automatically switches to `kind-gitops-demo`. (If you also work with a real
cluster, e.g. via `kubectl config use-context <other>`, switch back explicitly when you're done here.)

---

## Step 2 — Kustomize base + overlays

Restructured the flat `apps/nginx/*.yaml` manifests into a **base** (shared Deployment/Service)
and one **overlay** per environment. Each overlay:

- declares its own `Namespace` (`demo-dev`, `demo-staging`, `demo-prod`, `demo-qa`)
- sets `namespace:` so all resources land in the right place
- patches `spec.replicas` via a small `deployment-patch.yaml`
- (prod only) patches in CPU/memory requests+limits

Verify any overlay renders correctly before trusting it to a controller:

```bash
kubectl kustomize environments/dev
```

**Lesson**: a change to `apps/nginx/base` affects every environment that doesn't override it.
When we later changed the base image, staging and prod picked up the new image too, not just dev.

---

## Step 3 — Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side --force-conflicts
```

`--server-side` is required here — the `ApplicationSet` CRD is too large for `kubectl apply`'s
default client-side `last-applied-configuration` annotation (a known Argo CD install gotcha).

Access:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
argocd login localhost:8080 --username admin --password '<password>' --insecure
```

---

## Step 4 — Connect Argo CD to Git

Pushed the repo to GitHub (`github.com/vishalsallagargi/gitops-demo`) and created one Argo CD
`Application` per environment, each pointing at its overlay path. Manually synced first
(`argocd app sync nginx-dev`) to see an un-automated sync happen once, before turning on automation.

**Lesson**: Argo CD needs a Git URL it can actually reach. A local-only repo doesn't work without
extra plumbing (a reachable git server) — pushing to a real remote is the path of least resistance.

---

## Step 5 — Auto-sync: self-heal + prune

Set `syncPolicy.automated` on every Application:

```yaml
syncPolicy:
  automated:
    prune: true      # resources removed from Git get deleted from the cluster
    selfHeal: true    # manual cluster drift gets reverted back to what Git says
```

**Proved it live**: ran `kubectl scale deployment nginx -n demo-dev --replicas=5` by hand.
Argo CD reverted it back to 1 replica (what Git said) within seconds — no human involved.

---

## Step 6 — CI: build → commit → deploy

Added a real app to build (`apps/nginx/site/`) and a GitHub Actions workflow
(`.github/workflows/build-and-deploy.yml`) that on every push to `main`:

1. builds the Docker image from `apps/nginx/site/`
2. pushes it to `ghcr.io/<owner>/gitops-demo-nginx`, tagged with the commit SHA
3. runs `kustomize edit set image` on `environments/dev/kustomization.yaml` to point at the new tag
4. commits and pushes that change back to `main` using the workflow's own `GITHUB_TOKEN`

Argo CD's existing auto-sync then picks up that commit and redeploys `dev` — no direct
cluster access from CI at all. **The Git commit is the handoff between CI and CD.**

**Verified end-to-end**: pushed a change, watched the Action build+push+commit, watched the
new pod come up, and `curl`'d the running Service to confirm the new content was actually served.

**Note**: GHCR packages built from a public repo's Actions workflow inherit public visibility
automatically — no image pull secret needed here. A private repo would need one.

---

## Step 7 — ApplicationSets

Replaced the three hand-written `Application` YAMLs with a single `ApplicationSet`
(`bootstrap/applicationset.yaml`) using a Git **directory generator**:

```yaml
generators:
  - git:
      repoURL: https://github.com/vishalsallagargi/gitops-demo.git
      revision: main
      directories:
        - path: environments/*
```

It generates one `Application` per matching folder, named `nginx-{{path.basename}}`.

**Proved it live**: added `environments/qa/` (namespace, patch, kustomization) with
**zero Argo CD YAML**. Within a few minutes the ApplicationSet controller noticed the new
folder, created `nginx-qa` automatically, and it auto-synced and deployed — no manual step.

To swap the three manual Applications for the generated ones without disrupting running
resources: deleted the old `Application` objects (they had no `resources-finalizer`, so deleting
the object doesn't touch the underlying Deployment/Service/Namespace), then applied the
ApplicationSet, which created new Applications with the *same names* that adopted the
already-running resources with no pod restarts.

---

## Step 8 — Rollbacks

Simulated a bad deploy: pointed `environments/dev` at a nonexistent image tag and pushed it.
Result: `ImagePullBackOff` on the new pod — but the **old pod kept running** the whole time
(Kubernetes' rolling-update strategy won't kill the last healthy replica for a failing rollout).

Then tried to "fix" it the instinctive way:

```bash
kubectl set image deployment/nginx nginx=<good-tag> -n demo-dev
```

**Self-heal reverted it back to the broken tag within about 30 seconds**, because Git still said
the bad tag was correct. This is the core GitOps rule: **with self-heal on, `kubectl` cannot
out-run Git.** The only fix that sticks is a Git commit:

```bash
git revert --no-edit <bad-commit-sha>
git push
```

Argo CD picked up the revert and redeployed the good image.

**Also note**: `argocd app rollback` (rolling back via Argo CD's own sync history) has the same
problem when auto-sync is on — it would just get self-healed forward again. Argo CD rollback is
really only useful with manual sync policy; with automation on, revert in Git instead.

---

## Step 9 — Sync waves

Added `argocd.argoproj.io/sync-wave` annotations to make apply order explicit:

| Resource   | Wave |
|------------|------|
| Namespace  | `-1` |
| Service    | `0`  |
| Deployment | `1`  |

Argo CD applies lower waves first and waits for each wave to be healthy before starting the
next. This duplicates Argo CD's default ordering-by-kind here, but the same annotation is how
you'd sequence something that actually matters — e.g. a ConfigMap or Secret (wave `-1`) that a
Deployment (wave `0`) mounts and would crash-loop without.

---

## Step 10 — Slack notifications *(not yet configured)*

Argo CD ships a `notifications-controller` (already installed in step 3) that can post to Slack
on sync/health events via a config like:

```yaml
# argocd-notifications-cm (ConfigMap)
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: "{{.app.metadata.name}} is now {{.app.status.sync.status}}"
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]
```

...plus a `Secret` holding the real Slack token, and a `notifications.argoproj.io/subscribe.on-deployed.slack: <channel>`
annotation on each `Application` to opt it in. This needs a real Slack Incoming Webhook / bot
token to actually fire — wire it up whenever you have one.

---

## Known issue: local cluster instability

On this machine, `kube-controller-manager` inside the `kind` node intermittently crash-loops.
Root cause: very high cumulative disk I/O on the container (multi-TB over a few hours), driven by
disk contention with the many *other* Docker containers running on this host for unrelated
projects — not a problem with this repo or Argo CD's config. **App pods have never gone down
because of it** — only the control plane's leader-election health check flaps. If you hit this
and it does start affecting deploys, the fix that worked here was simply recreating the `kind`
cluster (`kind delete cluster --name gitops-demo && kind create cluster --name gitops-demo`)
and reapplying Argo CD + the ApplicationSet.

---

## Reproducing this from scratch

```bash
kind create cluster --name gitops-demo
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side --force-conflicts
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=180s
kubectl apply -f bootstrap/applicationset.yaml
```

That's the entire bootstrap. Everything else — every environment, every deployment — comes from
what's committed under `environments/`.
