# cn-kb-freshrss

[FreshRSS](https://www.freshrss.org/) — a self-hosted RSS aggregator — on the
[amun-kubernetes](https://github.com/GonzaloAlvarez/amun-kubernetes) cluster.

Reachable as:
- **`http://freshrss.k8s.lan`** — LAN, via pfSense Domain Override + ingress-nginx
- **`https://freshrss.lab.gn.al`** — over the tailnet, via VPS Traefik wildcard +
  the k8s-router subnet route

## Architecture

| Component | Mode | Resource |
|---|---|---|
| `freshrss` | Deployment (replicas: 1, sqlite-backed) | 512Mi RAM cap |
| `freshrss-data` | Longhorn RWO PVC (sqlite DB, feed cache, user state) | 5Gi |
| `freshrss-extensions` | Longhorn RWO PVC (themes, filters, plugins) | 1Gi |

SQLite is single-writer ⇒ exactly one replica, `strategy: Recreate`. Default
backend is sqlite; switch to PostgreSQL by re-running the in-UI installer if
you outgrow it.

The container runs Apache as `www-data` (uid/gid 33). `securityContext.fsGroup: 33`
makes the Longhorn-mounted PVCs writable by it.

A `CRON_MIN=*/20` env var runs the in-container feed refresher every 20 minutes.

## Deploy

```sh
# 1. Append to amun-kubernetes/deployment/kustomization.yaml:
#      - github.com/GonzaloAlvarez/cn-kb-freshrss//k8s?ref=main
# 2. Apply
export KUBECONFIG=~/.kube/config-rpid
kubectl apply -k ~/dev/amun-kubernetes/deployment/
```

## First-time setup

On first boot, FreshRSS launches its install wizard at `/i/install.php`. Hit
`http://freshrss.k8s.lan/` (or `https://freshrss.lab.gn.al/`) and walk through:

1. Database type: **SQLite** (no extra params).
2. Default user: pick an admin name + password.
3. Done — the wizard writes its state to `freshrss-data` (the PVC), so it never
   re-prompts.

## Health

```sh
kubectl -n kb-freshrss get deploy,pvc,svc,ingress

# Local smoke
kubectl -n kb-freshrss port-forward svc/freshrss 8080:80
# → http://localhost:8080

# LAN smoke
curl -sI http://freshrss.k8s.lan/

# Tailnet smoke (from a tailnet device or via cn-socksnode SOCKS proxy)
curl -skI https://freshrss.lab.gn.al/

# Tail logs
kubectl -n kb-freshrss logs deploy/freshrss --tail=50
```

## Common tasks

```sh
# Bump the image tag — edit deployment.yaml, commit, push, then:
kubectl apply -k ~/dev/amun-kubernetes/deployment/
kubectl -n kb-freshrss rollout status deploy/freshrss

# Force a feed refresh (the cron normally runs every 20 minutes)
kubectl -n kb-freshrss exec deploy/freshrss -- \
  su -s /bin/sh www-data -c "php /var/www/FreshRSS/app/actualize_script.php"

# Backup the data PVC out of band (full FreshRSS state in a single sqlite file)
kubectl -n kb-freshrss exec deploy/freshrss -- \
  tar c -C /var/www/FreshRSS data | gzip > freshrss-data-$(date +%Y%m%dT%H%M).tar.gz

# Resize the data PVC (Longhorn handles online expansion)
kubectl -n kb-freshrss patch pvc freshrss-data --type merge \
  -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# Reset everything (DESTRUCTIVE)
kubectl -n kb-freshrss delete pvc freshrss-data freshrss-extensions
kubectl -n kb-freshrss rollout restart deploy/freshrss
```

## Observability

A Grafana dashboard for this service lives at
`https://grafana.lab.gn.al/d/freshrss` (provisioned from
[cn-root-docker/tailnet/grafana/provisioning/dashboards/freshrss.json](https://github.com/GonzaloAlvarez/cn-root-docker/blob/main/tailnet/grafana/provisioning/dashboards/freshrss.json)).
It pulls from the cluster's in-cluster Prometheus (`cluster-prometheus`
datasource) and Loki (`cluster-loki`) — pod status, CPU/mem, PVC usage,
ingress req/s by host, HTTP status codes, recent error logs.

## License

GNU GPL v3
