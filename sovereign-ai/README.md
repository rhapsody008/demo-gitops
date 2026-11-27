# sovereign-ai: Secure Intake (PII Shield)

These are the Kubernetes manifests for the [sovereign-ai-demo](https://github.com/rhapsody008/sovereign-ai-demo) app.
Two NKP projects deploy them: one to the primary cluster and one to the standby cluster. Both clusters use the same Nutanix Objects
bucket and the same NAI endpoint, so the standby shows the same records after the primary goes down.

```
sovereign-ai/
├── base/
│   ├── kustomization.yaml        resources, image tag, shared settings (ConfigMap secure-intake-config)
│   ├── rbac.yaml                 ServiceAccount + Role: list pods in its own namespace (live pod count)
│   ├── deployment.yaml           app pods: probes, Downward API, hardened security context
│   ├── service.yaml              ClusterIP :80 → :8080
│   ├── hpa.yaml                  1–6 pods at 70% CPU (not applied, see Notes)
│   ├── ingress.yaml              NKP Traefik, HTTPS, path /piishield
│   └── secret.example.yaml       Objects credentials: create by hand, same on both clusters
└── overlays/
    ├── primary/kustomization.yaml   2 replicas, cluster identity "primary"
    └── standby/kustomization.yaml   0 replicas, cluster identity "standby"
```

Flux applies only the files listed in the `kustomization.yaml` files. `secret.example.yaml` is a template and is
never applied from git.

## How the two clusters differ

Each overlay generates a ConfigMap named `secure-intake-cluster`, which sets `CLUSTER_NAME`, `CLUSTER_DISPLAY_NAME`
and `CLUSTER_ROLE`, and sets the replica count:

| | Primary (`overlays/primary`) | Standby (`overlays/standby`) |
|---|---|---|
| Replicas | 2 | 0: no pods until failover |
| `CLUSTER_NAME` | `workload-primary` | `workload-standby` |
| `CLUSTER_ROLE` | `primary`: blue banner. Writes 3 sample records if the bucket is empty | `standby`: orange banner |

Everything else is shared through `base/` or through the bucket (NAI settings, records, photos). The ConfigMap names
are hashed, so editing the identity in git rolls the pods automatically.

## Setup

### 1. Edit the shared settings in `base/kustomization.yaml`

| Setting | Value |
|---|---|
| `S3_ENDPOINT` | Objects store host (currently `10.8.79.10`) |
| `S3_BUCKET` | `secure-intake`: the bucket already created in Objects |
| `images.newTag` | Currently `latest`. Pin it to a `sha-<short>` tag from GHCR before the event |

### 2. On each cluster, create the Secret

Create it in the NKP project namespace, **before** the first sync. Pods only read it at start-up.

```sh
NS="your-project-namespace"   # differs per NKP project

# Same on both clusters. The key needs read/write access to the secure-intake bucket.
kubectl -n $NS create secret generic secure-intake-secrets \
  --from-literal=S3_ACCESS_KEY=<objects access key> \
  --from-literal=S3_SECRET_KEY=<objects secret key>
```

`NAI_API_KEY` and `ADMIN_TOKEN` are optional keys you can add to the Secret (see
[base/secret.example.yaml](base/secret.example.yaml)). You normally don't need them, because the NAI endpoint and key are set
in the UI.

### 3. Add the GitOps source to each NKP project

| NKP project | Placed on | Continuous Deployment path |
|---|---|---|
| primary project | primary cluster | `sovereign-ai/overlays/primary` |
| standby project | standby cluster | `sovereign-ai/overlays/standby` |

If an existing NKP project still points at the old path `sovereign-ai`, change it to one of the overlay paths. The
old path no longer has a `kustomization.yaml`, so Flux would pick up both overlays and fail on duplicate resources.

### 4. Image pull (if the GHCR package is private)

Make the package public, or do both of these:

- create a pull secret on both clusters:

  ```sh
  kubectl -n $NS create secret docker-registry ghcr-pull \
    --docker-server=ghcr.io --docker-username=<github user> --docker-password=<PAT with read:packages>
  ```

- uncomment `imagePullSecrets` in `base/deployment.yaml`

## Check

On each cluster (the standby has no pods until it is scaled up):

```sh
kubectl -n $NS get deploy,pods,ingress
kubectl -n $NS get configmap | grep secure-intake-cluster           # secure-intake-cluster-<hash>, differs per cluster
kubectl get svc -A | grep traefik                            # EXTERNAL-IP is the demo IP
```

Open `https://<traefik-external-ip>/piishield/`. Accept the self-signed certificate once, before going on stage.
The camera only works over HTTPS. The root path `/` returns 404 by design.

All three tiles should be green:
- **NAI endpoint**: Connected.
- **Storage on Nutanix Cloud Infrastructure**: Reachable, with a record count.
- **NKP cluster**: Healthy.

NAI is set once in the UI under *NAI endpoint settings*. The settings are saved in the bucket at `config/nai.json`,
so the other cluster picks them up automatically.

## Failover demo

1. Open `https://<primary-ip>/piishield/` (blue banner). File a record, using text and/or a photo.
2. Shut down the primary.
3. Scale up the standby: set `count: 2` in `overlays/standby/kustomization.yaml` and push (Flux applies it), or for a
   quick demo run `kubectl -n $NS scale deployment/secure-intake --replicas=2` (Flux resets it to 0 on its next sync).
4. Open `https://<standby-ip>/piishield/` (orange banner). The same records are there, and NAI is already configured.
   New records are tagged with the standby cluster's name.

## Self-healing demo

On the primary (2 pods), deleting a pod shows Kubernetes replacing it while the other pod keeps serving:

```sh
kubectl -n $NS get pods -w &                     # watch in a second terminal
kubectl -n $NS get pods -l app=secure-intake     # pick one pod name
kubectl -n $NS delete pod <pod-name>
```

1. Kubernetes starts a replacement pod within seconds. Delete both pods to see the banner turn red
   ("Connection lost") until a pod is Ready again.
2. Refresh the page to see the banner show a **new pod name**.
3. The records are all still there, because they live in Objects, not in the pod.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Standby shows no pods | Expected: the standby overlay sets 0 replicas. Scale it up (see Failover demo) |
| Both clusters show a blue banner | The standby project points at `overlays/primary`. Set the path to `sovereign-ai/overlays/standby` |
| Cluster tile says "Status unavailable" with a 403 | The `secure-intake` ServiceAccount, Role or RoleBinding is missing. Check `kubectl -n $NS get sa,role,rolebinding` and `kubectl -n $NS auth can-i list pods --as=system:serviceaccount:$NS:secure-intake` |
| Pods in `CreateContainerConfigError` | `secure-intake-secrets` doesn't exist in the project namespace |
| Pods not Ready, Storage tile says "Unreachable" | `/readyz` fails when the bucket can't be reached. Check `S3_ENDPOINT`, the keys, and the key's access to `secure-intake` |
| `ImagePullBackOff` | The GHCR package is private. See step 4 |
| "Camera needs HTTPS" message | The page was opened over `http://`. Use `https://…/piishield/` |
| 404 at `https://<ip>/` | Expected. The app is only served under `/piishield/` |

## Notes

- **Ingress:** it uses NKP's Traefik (`kommander-traefik`) on the `websecure` entrypoint, with no host rule, and
  passes the `/piishield` prefix through to the app unchanged (`BASE_PATH`).
- **Replicas:** the base Deployment has no `replicas` field; each overlay sets it (primary 2, standby 0) and Flux
  enforces it. `base/hpa.yaml` is not applied, because an HPA and a git-pinned replica count would fight over the primary's count.
  If you enable it, remove the `replicas` field from the primary overlay. An HPA does nothing while its target is at
  0 replicas, so it won't wake the standby.
- **NKP cluster tile:** the app pod count is live. The app lists the Ready `app=secure-intake` pods in its own
  namespace, using the `secure-intake` ServiceAccount and a namespaced Role that only allows `list pods`
  (`base/rbac.yaml`). The node count and HPA max are still fixed values (`DEMO_NODES` / `DEMO_HPA_MAX` in
  `base/kustomization.yaml`). No ClusterRole is needed.
- **Sample records:** to stop the primary seeding them, set `SEED_SAMPLE_RECORDS=false` in `base/kustomization.yaml`.
- **COSI:** not used. The app connects to the existing bucket with the static key in `secure-intake-secrets`.
