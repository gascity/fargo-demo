# rewind-demo

A tiny GitOps repo for demonstrating [rewind](https://github.com/luthermonson/rewind):
a transactional-deployment controller on a k3s-on-beads cluster. Both Argo CD
and Flux can point at it.

## What's here

```
apps/podinfo/       podinfo Deployment (2 replicas), Service, ConfigMap, Namespace,
                    kustomization.yaml   — the app both tools deploy
argo/               Application (manual sync, destination = the k3s target) +
                    a kustomize overlay that adds the rewind PreSync hook
flux/               GitRepository + Kustomization (wait: true, timeout 2m,
                    healthChecks on the Deployment)
```

### Branches

- **`main`** — the good image, `ghcr.io/stefanprodan/podinfo:6.17.0`.
- **`broken`** — identical, but the image tag is `6.17.0-does-not-exist`, which
  never pulls (`ImagePullBackOff`). This is the failing rollout rewind rolls
  back.

Everything else (replica count, ports, probes) is the same on both branches, so
the only difference a rollout sees is the image.

## The demo flow

rewind must be running in the matching mode and pointed at the cluster's kine
(`--kine-endpoint`). The `/txn` control API has to be live in kine for the
rollback to actually happen; until then, run rewind with `--dry-run` to watch it
decide begin/commit/rollback without sending.

### 1. Good deploy (commit)

**Argo:** apply `argo/application.yaml`, then sync `main`.

```sh
kubectl apply -f argo/application.yaml
argocd app sync podinfo-demo        # or: kubectl -n argocd patch app ... (manual sync)
```

rewind sees `operationState.phase=Running`, opens a transaction
(`argo/argocd/podinfo-demo/<revision>`), the PreSync hook waits until the
savepoint exists, podinfo rolls out, goes `Healthy`, and rewind **commits**.
Nothing visibly changed; the Application is annotated
`rewind.gascity.dev/outcome: committed`.

**Flux:** apply `flux/` and let it reconcile `main`.

```sh
kubectl apply -f flux/
```

rewind sees `Reconciling=True` with a new `lastAttemptedRevision`, begins, and
**commits** when `Ready=True` and `lastAppliedRevision == lastAttemptedRevision`.

### 2. Failed deploy (rollback)

Point the source at `broken` and sync again:

- **Argo:** set `spec.source.targetRevision: broken` and sync.
- **Flux:** set the GitRepository `spec.ref.branch: broken`.

Now:

1. rewind begins a transaction for the new revision.
2. podinfo's new pod can't pull `6.17.0-does-not-exist` → `ImagePullBackOff`.
3. Argo reports the app `Degraded`; Flux reports `Ready=False`
   (HealthCheckFailed) after `timeout: 2m`.
4. rewind **rolls back**: it emits the inverse of the in-scope changes as fresh
   revisions, restoring the previous Deployment/ReplicaSet/Pods.

Verify the rollback restored real state, not a re-reconcile:

```sh
kubectl -n demo get all
# The previous ReplicaSet and Pods are back with their ORIGINAL uids:
kubectl -n demo get rs -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.metadata.uid}{"\n"}{end}'

# The failed attempt is still in the datastore timeline (nothing was erased):
bd history <the podinfo Deployment bead>   # via kourt / bd against the beads store
```

The deployment system's own objects (the Application / Kustomization /
GitRepository) are never in scope, so they are untouched by the rollback — only
the app's objects are reverted.

## Notes

- The kine endpoint in `argo/presync-hook.yaml` (`http://192.168.5.2:2381`) and
  in the Flux/rewind config must match your cluster's kine. Change it to suit.
- `argo/` is synced by Argo (it includes the PreSync hook); Flux syncs
  `apps/podinfo/` directly (Flux has no pre-apply hook — see the rewind README
  for how that window is narrowed).

MIT licensed.

## mirage-fork: the node-pool example (Cluster API + mirage)

`apps/mirage-fork/` is a Cluster API cluster for the
[mirage](https://github.com/luthermonson/cluster-api-provider-mirage) fake
infrastructure provider: 3 control-plane machines and 5 MachineDeployments
of 5 workers, plus the rewind PreSync hook (it reads the kine token from a
Secret `rewind-kine-token` in the destination namespace; create that by hand).
`argo/mirage-fork-application.yaml` is the manual-sync Application.

Branches (only `apps/mirage-fork/` differs):

- **`main`** — 5 pools x 5 workers (28 Nodes).
- **`scale-10x10`** — 10 pools x 10 workers (103 Nodes): the committed scale-up.
- **`scale-5x10`** — 5 pools x 10 workers (53 Nodes): the smaller scale-up for a
  rig near its churn ceiling.
- **`broken-10x10`** / **`broken-5x10`** — the same shape, but pool
  `mirage-fork-md-7` (`md-4` on 5x10) references a MirageMachineTemplate that
  does not exist, so that pool never becomes healthy and rewind rolls the
  whole sync back after its health deadline.
- **`invalid-5x5`** — `main` with a MachineDeployment whose `replicas` is a
  string, so the sync fails at apply time (`operationState.phase=Failed`).
