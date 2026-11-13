# Argo CD

Argo CD is installed once by hand and then manages itself. Every Kubernetes
manifest under `cluster/argocd/` is reconciled from this repository by Argo CD.
Other directories under `cluster/` — currently `cluster/platform/cilium/`, the
Helm values for the already-running Cilium install — are not Argo CD-managed;
adopting them is a separate change.

| item | value |
|------|-------|
| Helm repo | `https://argoproj.github.io/argo-helm` |
| Chart | `argo/argo-cd` |
| Chart version (pinned, exact) | `10.10.2` |
| Argo CD app version | `v3.5.4` |
| Namespace | `argocd` |
| repoURL | `git@github.com:stefantrost/homelab.git` |
| targetRevision | `main` |

The chart version `10.10.2` is pinned exactly: no range, no `*`, no omitted
version. Any Argo CD `Application` that installs this chart must carry the same
literal `10.10.2`. Upgrading Argo CD means editing that literal in git, not
re-running the bootstrap command.

## Files

| file | purpose |
|------|---------|
| `namespace.yaml` | the `argocd` namespace; reconciled by the `argocd-bootstrap` `Application`, and annotated `argocd.argoproj.io/sync-options: Prune=false` so no `Application` can ever prune it |
| `values.yaml` | the pinned Helm values for the `argo/argo-cd` chart. Helm input, not a Kubernetes manifest, so no `Application` applies it as a resource; it is consumed by `apps/argo-cd.yaml` through `$values` |
| `root-app.yaml` | the root `Application` (app-of-apps) that watches `cluster/argocd/apps/`; itself reconciled by the `argocd-bootstrap` `Application` |
| `apps/` | the directory the root `Application` watches; every child `Application` lives here |
| `apps/argocd-bootstrap.yaml` | child `Application` that reconciles `namespace.yaml` and `root-app.yaml` from git |
| `apps/argo-cd.yaml` | child `Application` that installs the pinned `argo/argo-cd` chart |
| `README.md` | this document (prose, not a manifest; no `Application` applies it) |

`root-app.yaml` is the only `Application` applied by hand, and only as part of
the single bootstrap command below. It installs no chart: its source is the
directory `cluster/argocd/apps/` in this repository at `main`, recursed, and
every `*.yaml` found there is applied to the cluster. Adding a component to the
cluster therefore means committing a child `Application` into
`cluster/argocd/apps/`; removing one means deleting that file and letting the
root application's `prune` remove its resources. `cluster/argocd/apps/` also
holds a `.gitkeep`, so git tracks the directory and the root application's
source path resolves on a fresh clone even when no child application is
committed.

The root `Application` syncs automatically with `prune: true` and
`selfHeal: true`, and carries the `resources-finalizer.argocd.argoproj.io`
finalizer so that deleting it cascades to the child applications it created
rather than orphaning them.

### The bootstrap manifests are themselves Argo CD-managed

The root `Application` only watches `cluster/argocd/apps/`, so on its own it
would leave `namespace.yaml` and `root-app.yaml` — the two files the bootstrap
applies by hand — outside GitOps forever. `apps/argocd-bootstrap.yaml` closes
that loop. It is an ordinary child `Application` (so the root application
reconciles it, like any other child), and its source is `cluster/argocd` with
`recurse: false` and `include: '{namespace.yaml,root-app.yaml}'`.

The include is an allow-list, not a convenience: the path `cluster/argocd` also
contains `values.yaml` and `README.md`, neither of which is a Kubernetes
manifest, and `recurse: false` keeps `apps/` out of this application's scope so
the children stay owned by the root application alone.

The effect is that the root `Application` is reconciled from git by a child it
itself reconciles. Editing `root-app.yaml` — its `targetRevision`, sync policy,
retry backoff or path — or editing the namespace's labels, converges by commit
alone; and an operator who edits or deletes the root `Application` in-cluster
has that change reverted by `argocd-bootstrap`'s `selfHeal`, instead of
silently stopping all GitOps. `argocd-bootstrap` carries the same automated
`prune`/`selfHeal` as the root application.

`argocd-bootstrap` deliberately carries **no**
`resources-finalizer.argocd.argoproj.io`. The root application watches
`cluster/argocd/apps/` with `prune: true`, so deleting
`apps/argocd-bootstrap.yaml` from git makes the root application delete the
`argocd-bootstrap` `Application` object. With a cascade finalizer that deletion
would propagate to everything `argocd-bootstrap` manages — including the
`argocd` namespace, and therefore the Argo CD installation that is doing the
deleting; `selfHeal` could not recover, because the controller that heals would
be gone. Without the finalizer the same deletion is non-cascading: the managed
resources are **orphaned**, left running and untouched in the cluster, merely no
longer reconciled. The root application keeps its finalizer (deleting it is an
explicit, deliberate teardown of the app-of-apps tree, not a side effect of
removing one file).

As a second, independent guard, `namespace.yaml` carries the annotation
`argocd.argoproj.io/sync-options: Prune=false`, so no `Application` may ever
prune the `argocd` namespace, however it goes out of scope.

The mutual reference is deliberate and is not a deadlock: both objects already
exist in the cluster after the single bootstrap command below, and from then on
each one merely re-asserts the other's git state.

`values.yaml` deliberately disables the components that are out of scope for
this cluster: Dex (SSO), notifications, the commit server and `redis-ha`. Chart
`10.10.2` exposes no switch for the ApplicationSet controller — it has no
`applicationSet.enabled` key — so that controller is installed by the chart and
is simply left unused: this repository defines no `ApplicationSet`. The API
server is exposed as a `ClusterIP` with no ingress; reach it with
`kubectl port-forward` until an ingress is introduced as its own change.

## Operator prerequisites

These are operator responsibilities, not steps this repository performs.

1. **Tooling on the workstation** (no cluster mutation):

   ```sh
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update argo
   ```

2. **A read-only deploy key for this repository.** The `repoURL`
   `git@github.com:stefantrost/homelab.git` is an SSH URL, so Argo CD cannot
   clone it until an SSH private key is available to the repo server. Argo CD
   reads that key from a repository-credentials `Secret` in the `argocd`
   namespace, labelled `argocd.argoproj.io/secret-type: repository`, with at
   least the keys `type: git`, `url: git@github.com:stefantrost/homelab.git`
   and `sshPrivateKey`.

   **This repository does not contain that key, that Secret, or any generator
   for either, and must not.** The operator generates the key pair, registers
   the public half as a deploy key on the GitHub repository, and supplies the
   private half to the cluster through whatever secret-management mechanism the
   cluster adopts. Secret management is out of scope for this change, so the
   mechanism is not prescribed here. Until that Secret exists, the root
   `Application` in `root-app.yaml` will report the repository as inaccessible
   and will not sync. This is a known limitation of this bootstrap.

## Bootstrap

Exactly one imperative command touches the cluster. It is run once, from the
repository root, and never again:

```sh
helm template argocd argo/argo-cd \
  --version 10.10.2 \
  --namespace argocd \
  --include-crds \
  -f cluster/argocd/values.yaml \
| kubectl apply --server-side -n argocd \
    -f cluster/argocd/namespace.yaml \
    -f - \
    -f cluster/argocd/root-app.yaml
```

This renders the pinned chart with the git-tracked values and applies it
together with the git-tracked namespace. It installs no Helm release, so there
is no release state for Argo CD to fight with when it subsequently takes
ownership of its own manifests.

The `-f` order is load-bearing: `cluster/argocd/namespace.yaml` is listed before
stdin so the `argocd` namespace exists before the rendered chart objects are
created in it, and `cluster/argocd/root-app.yaml` is listed after stdin so the
`applications.argoproj.io` CRD exists before the root `Application` is created.
If the API server has not finished establishing that CRD when the root
`Application` is submitted, kubectl reports `no matches for kind "Application"`;
re-running the same command is safe and idempotent, and no other command is
needed.

`--server-side` is required, not cosmetic: the `applicationsets.argoproj.io`
CRD shipped by this chart exceeds the 262144-byte limit on the
`last-applied-configuration` annotation that a client-side `kubectl apply`
writes, and a client-side apply of it is rejected by the API server.

After this command the cluster runs Argo CD and the root `Application` exists,
pointing at `git@github.com:stefantrost/homelab.git` at `main`. Once the
deploy-key `Secret` from the prerequisites also exists, the root application
reaches the repository and takes over every child `Application` committed to
`cluster/argocd/apps/` — including `argocd-bootstrap`, which then takes over
`namespace.yaml` and `root-app.yaml`, the very files this command applied.

From that point on every Kubernetes manifest under `cluster/argocd/` is
reconciled from git: `namespace.yaml` and `root-app.yaml` by
`apps/argocd-bootstrap.yaml`, `apps/argocd-bootstrap.yaml` and
`apps/argo-cd.yaml` by `root-app.yaml`, and the Argo CD workloads themselves by
`apps/argo-cd.yaml`. (`values.yaml` is Helm input rather than a manifest, and is
read from git by `apps/argo-cd.yaml` through `$values`; `README.md` is prose.)
The workflow is: commit to `main`, let Argo CD sync. No further `kubectl
apply`, `helm install` or `helm upgrade` is run by hand — not for any file under
`cluster/argocd/`.

## Changing Argo CD after bootstrap

Edit `cluster/argocd/values.yaml` (or the pinned chart version in the
`Application` that installs it) and commit to `main`. Argo CD re-renders the
chart and applies the diff to itself. The bootstrap command above is not
re-run.

## Rollback

Rolling back an *edit* is a git operation, not a cluster operation. Rolling
back a *deletion* is a separate case — see "What reverting a deletion does, and
does not, do" below before deleting any `Application` file.

```sh
git revert <sha>        # or: git revert <oldest-bad-sha>..<newest-bad-sha>
git push origin main
```

Argo CD detects the new `main`, re-renders, and converges the cluster back to
the reverted state — automatically where the application has an automated sync
policy, otherwise with `argocd app sync argocd` (or the Sync button in the UI).
Every application in this directory has an automated sync policy, so reverting
a commit that edited the *contents* of a manifest converges without any cluster
command: `namespace.yaml` and `root-app.yaml` through
`apps/argocd-bootstrap.yaml`, the child applications through `root-app.yaml`,
and the Argo CD workloads through `apps/argo-cd.yaml`.

### What reverting a *deletion* does, and does not, do

Converging on a revert is not universal, and this README does not claim it is.
A commit that **deletes an `Application` file** is not symmetric with one that
edits a manifest, and the asymmetry matters most for
`apps/argocd-bootstrap.yaml`:

- Deleting `apps/argocd-bootstrap.yaml` from git makes the root application
  prune the `argocd-bootstrap` `Application` object. Because that object carries
  no `resources-finalizer.argocd.argoproj.io`, the prune does **not** delete what
  it managed. `namespace.yaml` and `root-app.yaml` are simply **un-managed**:
  the `argocd` namespace and the root `Application` stay in the cluster, running
  and unchanged, and merely stop being reconciled from git. Nothing is torn
  down, and the `Prune=false` annotation on `namespace.yaml` blocks a namespace
  prune independently. Reverting that deletion restores the
  `argocd-bootstrap` `Application`, which re-adopts the two files it had been
  managing.
- Deleting any other child `Application` under `cluster/argocd/apps/` — for
  example `apps/argo-cd.yaml` — *does* cascade, because those objects carry the
  finalizer: pruning them deletes the resources they created. Reverting such a
  deletion re-creates the `Application`, which then re-installs those resources
  from git; it is a real delete-and-recreate, not a no-op, and anything those
  resources held that is not in git (for example cluster-local state) does not
  come back.
- Deleting `root-app.yaml` from git is the one genuinely destructive deletion
  in this directory, and it is destructive on purpose. `argocd-bootstrap`
  manages that file with `prune: true`, so removing it prunes the root
  `Application`, which *does* carry
  `resources-finalizer.argocd.argoproj.io` — the prune cascades to every child
  `Application`, and through `apps/argo-cd.yaml` to the Argo CD workloads
  themselves. The `argocd` namespace survives (`Prune=false`), but the
  installation inside it does not, and `git revert` then re-creates the tree
  from scratch rather than restoring it. Do not delete `root-app.yaml` as a
  rollback step; use it only when a full, intentional teardown is what you
  want.

So: a revert converges for manifest edits, and restores management for a
deleted `argocd-bootstrap`; for a deleted child `Application` it re-creates
rather than restores; and for a deleted `root-app.yaml` it rebuilds the whole
tree after a cascade. Rollback by `git revert` of an *edit* is always a git
operation; rollback of a *deletion* is not uniformly so, and the three cases
above are the whole of it.

Do **not** roll back with `kubectl apply`, `kubectl patch`, `kubectl rollout
undo` or `helm rollback`. Any such change is drift against `main`: Argo CD will
either revert it on the next sync or leave the cluster permanently out of sync
with the repository. The revert commit is also the audit record of what was
rolled back and why.

If Argo CD itself is broken badly enough that it cannot sync the revert, repeat
the single bootstrap command above against the reverted checkout. That restores
Argo CD from git state and remains a git-driven rollback.
