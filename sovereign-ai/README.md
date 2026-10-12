# sovereign-ai: Secure Intake (PII Shield)

Flux manifests for the [sovereign-ai-demo](https://github.com/rhapsody008/sovereign-ai-demo) app on two NKP clusters.
The two clusters are a primary and a standby. Both store records in the same Nutanix Objects bucket and use the same
NAI endpoint, so the standby shows the same records after the primary goes down.

```
sovereign-ai/
├── base/                 Deployment, Service, HPA, Ingress, shared ConfigMap
├── overlays/
│   ├── primary/          CLUSTER_NAME / DISPLAY_NAME / ROLE=primary  (blue banner)
│   └── standby/          CLUSTER_NAME / DISPLAY_NAME / ROLE=standby  (orange banner)
├── cluster-config.example.yaml   per-cluster identity ConfigMap (NKP project setup; not applied by Flux)
├── flux/
│   ├── primary.yaml      Namespace + GitRepository + Kustomization → overlays/primary
│   └── standby.yaml      Namespace + GitRepository + Kustomization → overlays/standby
└── secret.example.yaml   Objects credentials template (not applied by Flux)
```

## 1. Before the first sync

Edit these values and commit them:

| File | Value |
|---|---|
| `base/kustomization.yaml` | `S3_ENDPOINT`: the Objects store host. Both clusters use the same value |
| `base/kustomization.yaml` | `images.newTag`: pin it to a `sha-<short>` tag from GHCR before the event (it is `latest` now) |
| `overlays/*/kustomization.yaml` | `CLUSTER_NAME` / `CLUSTER_DISPLAY_NAME`: the names shown in the banner |

## 2. Deploy

### Option A — NKP project (recommended): one project, both clusters

1. Create an NKP project and place it on **both** clusters. Note the project namespace.
2. On **each** cluster, create the Objects Secret and that cluster's identity ConfigMap in the project namespace.
   The Secret is the same on both clusters. The ConfigMap is not:

   ```sh
   NS=<project-namespace>
   kubectl -n $NS create secret generic secure-intake-secrets \
     --from-literal=S3_ACCESS_KEY=<objects access key> \
     --from-literal=S3_SECRET_KEY=<objects secret key>

   # primary cluster
   kubectl -n $NS create configmap secure-intake-cluster \
     --from-literal=CLUSTER_NAME=workload-demo-primary \
     --from-literal=CLUSTER_DISPLAY_NAME="Workload Demo Primary Cluster" \
     --from-literal=CLUSTER_ROLE=primary

   # standby cluster
   kubectl -n $NS create configmap secure-intake-cluster \
     --from-literal=CLUSTER_NAME=workload-demo-standby \
     --from-literal=CLUSTER_DISPLAY_NAME="Workload Demo Standby Cluster" \
     --from-literal=CLUSTER_ROLE=standby
   ```

3. In the project, go to **Continuous Deployment** and add this repo as a GitOps source with path **`sovereign-ai/base`**.

NKP deploys the same manifests to both clusters. Each pod reads `secure-intake-cluster` from its own cluster
(see [cluster-config.example.yaml](cluster-config.example.yaml)), so the banner shows the right name and colour.
If you create or change the ConfigMap after the pods have started, run
`kubectl -n $NS rollout restart deployment/secure-intake`.

### Option B — per-cluster Flux files (no NKP project)

On each cluster: create the `sovereign-ai` namespace and the Secret (as above, `NS=sovereign-ai`), then run
`kubectl apply -f sovereign-ai/flux/primary.yaml` or `flux/standby.yaml`. These point at `overlays/primary` and
`overlays/standby`, which already contain the cluster names, so no identity ConfigMap is needed. If you create one
anyway, it overrides the overlay values.

### Image pull

The app's GHCR image is private by default. Make the package public, or create a pull secret on both clusters and
uncomment `imagePullSecrets` in `base/deployment.yaml`:

```sh
kubectl -n $NS create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io --docker-username=<github user> --docker-password=<PAT with read:packages>
```

If this repo is private, give NKP (or the `secretRef` in `flux/*.yaml`) a read-only token.

## 3. Check

```sh
kubectl -n $NS get pods,hpa,ingress
kubectl -n $NS get configmap secure-intake-cluster -o yaml   # different on each cluster
kubectl -n kommander-default-workspace get svc kommander-traefik   # EXTERNAL-IP = the demo IP (namespace may differ)
```

Open `https://<traefik-external-ip>/piishield/`. Accept the self-signed certificate once, before going on stage.
The camera only works over HTTPS. The root path `/` returns 404 by design.

## Failover demo

1. Open `https://<primary-ip>/piishield/` (blue banner). File a record.
2. Shut down the primary.
3. Open `https://<standby-ip>/piishield/` (orange banner). The same records are there, the NAI settings are already
   loaded from the bucket, and new records are tagged with the standby cluster.

## Notes

- **NAI endpoint:** set it in the UI under *NAI endpoint settings*. It is saved in the bucket at `config/nai.json`,
  so you only set it once, for both clusters. `NAI_BASE_URL` in the ConfigMap is just a fallback.
- **Ingress:** it uses NKP's Traefik (`kommander-traefik`) on the `websecure` entrypoint, with no host rule, and
  passes the `/piishield` prefix through unchanged. The app expects the prefix (`BASE_PATH`).
- **Kubernetes API:** the app makes no Kubernetes API calls, so it needs no RBAC. The *NKP cluster* tile shows
  `DEMO_NODES` / `DEMO_PODS` / `DEMO_HPA_MAX` from the ConfigMap.
- **Replicas:** the Deployment has no `replicas` field. The HPA (2–6 pods at 70% CPU) owns the count, so Flux
  does not reset it.
- **Sample records:** only the primary seeds the 3 sample records, and only when the bucket is empty. To turn this
  off, set `SEED_SAMPLE_RECORDS=false`.
