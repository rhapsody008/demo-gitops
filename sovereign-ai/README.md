# sovereign-ai: Secure Intake (PII Shield)

These are the Kubernetes manifests for the [sovereign-ai-demo](https://github.com/rhapsody008/sovereign-ai-demo) app.
A single NKP project deploys them to two clusters, a primary and a standby. Both clusters use the same Nutanix Objects
bucket and the same NAI endpoint, so the standby shows the same records after the primary goes down.

```
sovereign-ai/
├── kustomization.yaml            entry point: resources, image tag, shared settings (ConfigMap)
├── deployment.yaml               app pods: probes, Downward API, hardened security context
├── service.yaml                  ClusterIP :80 → :8080
├── hpa.yaml                      1–6 pods at 70% CPU (one pod, for the self-healing demo)
├── ingress.yaml                  NKP Traefik, HTTPS, path /piishield
├── secret.example.yaml           Objects credentials: create by hand, same on both clusters
└── cluster-config.example.yaml   cluster identity: create by hand, different on each cluster
```

Flux applies only the files listed in `kustomization.yaml`. The two `*.example.yaml` files are templates. They are
never applied from git.

## How the two clusters differ

NKP deploys identical manifests to both clusters. On each cluster, the pod also loads a ConfigMap named
`secure-intake-cluster`, which you create by hand on that cluster. It sets `CLUSTER_NAME`, `CLUSTER_DISPLAY_NAME`
and `CLUSTER_ROLE`:

| | Primary | Standby |
|---|---|---|
| `CLUSTER_ROLE` | `primary`: blue banner. Writes 3 sample records if the bucket is empty | `standby`: orange banner |

Everything else is shared through git (`kustomization.yaml`) or through the bucket (NAI settings, records, photos).

## Setup

### 1. Edit the shared settings in `kustomization.yaml`

| Setting | Value |
|---|---|
| `S3_ENDPOINT` | Objects store host (currently `10.8.18.10`) |
| `S3_BUCKET` | `secure-intake`: the bucket already created in Objects |
| `images.newTag` | Currently `latest`. Pin it to a `sha-<short>` tag from GHCR before the event |

### 2. On each cluster, create the Secret and the identity ConfigMap

Create them in the NKP project namespace, **before** the first sync. Pods only read them at start-up.

```sh
NS=sovereign-ai   # the NKP project namespace

# Same on both clusters. The key needs read/write access to the secure-intake bucket.
kubectl -n $NS create secret generic secure-intake-secrets \
  --from-literal=S3_ACCESS_KEY=<objects access key> \
  --from-literal=S3_SECRET_KEY=<objects secret key>

# Primary cluster only
kubectl -n $NS create configmap secure-intake-cluster \
  --from-literal=CLUSTER_NAME=workload-demo-primary \
  --from-literal=CLUSTER_DISPLAY_NAME="Workload Demo Primary Cluster" \
  --from-literal=CLUSTER_ROLE=primary

# Standby cluster only
kubectl -n $NS create configmap secure-intake-cluster \
  --from-literal=CLUSTER_NAME=workload-demo-standby \
  --from-literal=CLUSTER_DISPLAY_NAME="Workload Demo Standby Cluster" \
  --from-literal=CLUSTER_ROLE=standby
```

`NAI_API_KEY` and `ADMIN_TOKEN` are optional keys you can add to the Secret (see
[secret.example.yaml](secret.example.yaml)). You normally don't need them, because the NAI endpoint and key are set
in the UI.

### 3. Add the GitOps source to the NKP project

Place the project on **both** clusters. Under **Continuous Deployment**, add this repo with path **`sovereign-ai`**.

### 4. Image pull (if the GHCR package is private)

Make the package public, or do both of these:

- create a pull secret on both clusters:

  ```sh
  kubectl -n $NS create secret docker-registry ghcr-pull \
    --docker-server=ghcr.io --docker-username=<github user> --docker-password=<PAT with read:packages>
  ```

- uncomment `imagePullSecrets` in `deployment.yaml`

## Check

On each cluster:

```sh
kubectl -n $NS get pods,hpa,ingress
kubectl -n $NS get configmap secure-intake-cluster -o yaml   # different on each cluster
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
3. Open `https://<standby-ip>/piishield/` (orange banner). The same records are there, and NAI is already configured.
   New records are tagged with the standby cluster's name.

## Self-healing demo

The app runs as a single pod, so deleting it shows Kubernetes replacing it:

```sh
kubectl -n $NS get pods -w &                     # watch in a second terminal
kubectl -n $NS delete pod -l app=secure-intake
```

1. The banner turns red ("Connection lost") for a few seconds, while no pod is Ready.
2. Kubernetes starts a new pod, and the banner comes back with a **new pod name**.
3. The records are all still there, because they live in Objects, not in the pod.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Both clusters show "local" or a blue banner | `secure-intake-cluster` is missing or was created after the pods started. Create it, then run `kubectl -n $NS rollout restart deployment/secure-intake` |
| Pods in `CreateContainerConfigError` | `secure-intake-secrets` doesn't exist in the project namespace |
| Pods not Ready, Storage tile says "Unreachable" | `/readyz` fails when the bucket can't be reached. Check `S3_ENDPOINT`, the keys, and the key's access to `secure-intake` |
| `ImagePullBackOff` | The GHCR package is private. See step 4 |
| "Camera needs HTTPS" message | The page was opened over `http://`. Use `https://…/piishield/` |
| 404 at `https://<ip>/` | Expected. The app is only served under `/piishield/` |

## Notes

- **Ingress:** it uses NKP's Traefik (`kommander-traefik`) on the `websecure` entrypoint, with no host rule, and
  passes the `/piishield` prefix through to the app unchanged (`BASE_PATH`).
- **Replicas:** the Deployment has no `replicas` field, so it starts at 1. The HPA (min 1, max 6) owns the replica count, so Flux does not reset it.
- **NKP cluster tile:** the numbers come from `DEMO_NODES` / `DEMO_PODS` / `DEMO_HPA_MAX` in `kustomization.yaml`.
  The app makes no Kubernetes API calls and needs no RBAC.
- **Sample records:** to stop the primary seeding them, set `SEED_SAMPLE_RECORDS=false`.
- **COSI:** not used. The app connects to the existing bucket with the static key in `secure-intake-secrets`.
