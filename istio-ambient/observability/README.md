# Prometheus + Kiali for Istio ambient (Argo CD)

## Layout

```
observability/
├── root-app.yaml                  # parent app, applied once by hand
├── argocd-cm-health-patch.yaml    # one-time Argo CD setting
├── apps/
│   ├── kube-prometheus-stack.yaml # wave 0  -> namespace monitoring
│   ├── istio-monitors.yaml        # wave 1  -> syncs observability/monitors
│   └── kiali.yaml                 # wave 2  -> namespace istio-system
└── monitors/
    ├── istiod-servicemonitor.yaml
    ├── ztunnel-podmonitor.yaml
    └── istio-proxies-podmonitor.yaml
```

## Steps

### 1. Check prerequisites

```bash
kubectl get pods -n istio-system          # istiod, istio-cni-node, ztunnel Running
kubectl get sc gp3                        # StorageClass for the Prometheus volume
aws eks describe-addon --cluster-name <cluster> --addon-name aws-ebs-csi-driver
```

No gp3 StorageClass or EBS CSI driver? Either install them, or remove
`storageSpec` from `apps/kube-prometheus-stack.yaml` for a test setup.

### 2. Fill in placeholders

| Placeholder | File(s) | Value |
|---|---|---|
| `<YOUR_GIT_REPO_URL>` | root-app.yaml, apps/istio-monitors.yaml | Git repo holding this folder |
| `<KUBE_PROMETHEUS_STACK_VERSION>` | apps/kube-prometheus-stack.yaml | Exact chart version |
| `<KIALI_VERSION>` | apps/kiali.yaml | Kiali 2.x version compatible with Istio 1.31 (Kiali's version compatibility table) |

If the folder isn't at the repo root, update the `path:` fields too.

### 3. Enable Application health checks (one time)

```bash
kubectl -n argocd patch configmap argocd-cm --type merge \
  --patch-file observability/argocd-cm-health-patch.yaml
```

### 4. Commit and push

```bash
git add observability/
git commit -m "Add Prometheus and Kiali"
git push
```

### 5. Apply the parent app

```bash
kubectl apply -f observability/root-app.yaml
```

Argo CD syncs: kube-prometheus-stack → istio-monitors → kiali.

```bash
kubectl -n argocd get applications observability kube-prometheus-stack istio-monitors kiali
```

All should reach `Synced` / `Healthy` within a few minutes.

### 6. Verify Prometheus is scraping Istio

```bash
kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090
```

Open http://localhost:9090/targets and confirm these are UP:

- `serviceMonitor/istio-system/istiod/0`
- `podMonitor/istio-system/ztunnel/0` (one target per node)
- `podMonitor/istio-system/istio-proxies/0` (only once a waypoint or Istio gateway exists)

Then send some traffic to your apps and query:

```promql
istio_tcp_connections_opened_total     # from ztunnel (L4)
istio_requests_total                   # from waypoints (L7, needs a waypoint)
```

### 7. Open Kiali

```bash
kubectl -n istio-system port-forward svc/kiali 20001:20001
kubectl -n istio-system create token kiali
```

Open http://localhost:20001 and paste the token. In **Traffic Graph**, select
the `default` namespace. Edges with a lock icon mean mTLS via ztunnel is active.

Don't expose Kiali on the public ALB with token auth. For shared access,
switch `auth.strategy` to `openid` or expose it only internally.

## Troubleshooting

| Symptom | Check |
|---|---|
| istio-monitors: "no matches for kind PodMonitor" | kube-prometheus-stack not finished; it retries automatically. Confirm step 3 was applied. |
| Prometheus pod Pending | PVC can't bind: EBS CSI driver / gp3 StorageClass missing. |
| ztunnel targets DOWN | Node security groups must allow node→node on 15020. |
| Kiali graph empty | Prometheus targets (step 6), traffic actually flowing, namespace labelled `istio.io/dataplane-mode=ambient`. |
| Kiali "could not fetch metrics" | Prometheus URL in apps/kiali.yaml; `kubectl -n monitoring get svc prometheus-operated`. |
| Only TCP edges, no HTTP rates | Expected without a waypoint; deploy one for L7 metrics. |
