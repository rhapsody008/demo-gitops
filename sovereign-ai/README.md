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

## 2. On each cluster

Point `kubectl` at the **primary** cluster:

```sh
kubectl create namespace sovereign-ai
kubectl -n sovereign-ai create secret generic secure-intake-secrets \
  --from-literal=S3_ACCESS_KEY=<objects access key> \
  --from-literal=S3_SECRET_KEY=<objects secret key>
kubectl apply -f sovereign-ai/flux/primary.yaml
```

Then do the same on the **standby** cluster, with `flux/standby.yaml`.

The app's GHCR image is private by default. Make the package public, or create a pull secret on both clusters and
uncomment `imagePullSecrets` in `base/deployment.yaml`:

```sh
kubectl -n sovereign-ai create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io --docker-username=<github user> --docker-password=<PAT with read:packages>
```

If this repo is private, follow the `secretRef` comment in `flux/*.yaml`.

**Using NKP's Continuous Deployment UI instead of `kubectl apply -f flux/…`:** create one project per cluster,
each placed on only one cluster. In each project, add this repo as a GitOps source with path
`sovereign-ai/overlays/primary` or `sovereign-ai/overlays/standby`. Create `secure-intake-secrets` in each project's
namespace. The manifests set no namespace, so they work in either setup.

## 3. Check

```sh
kubectl -n sovereign-ai get kustomization,gitrepository     # READY=True
kubectl -n sovereign-ai get pods,hpa,ingress
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
