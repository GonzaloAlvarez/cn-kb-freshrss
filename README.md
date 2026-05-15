# cn-kb-freshrss

[FreshRSS](https://www.freshrss.org/) — a self-hosted RSS aggregator — on the
[amun-kubernetes](https://github.com/GonzaloAlvarez/amun-kubernetes) cluster,
with full-article enrichment via [Mercury Parser](https://github.com/postlight/parser)
through the [xExtension-Readable](https://github.com/printfuck/xExtension-Readable)
plugin.

Reachable as:
- **`http://freshrss.k8s.lan`** — LAN, via pfSense Domain Override + ingress-nginx
- **`https://freshrss.lab.gn.al`** — over the tailnet, via VPS Traefik wildcard +
  the k8s-router subnet route

## Architecture

| Component | Mode | Resource |
|---|---|---|
| `freshrss` Deployment | replicas: 1, sqlite-backed | 512Mi RAM cap |
| `mercury` Deployment | replicas: 1, stateless | 256Mi RAM cap |
| `freshrss-data` PVC | Longhorn RWO — sqlite DB, feed cache, user state | 5Gi |
| `freshrss-extensions` PVC | Longhorn RWO — themes, filters, plugins (incl. xExtension-Readable) | 1Gi |

SQLite is single-writer ⇒ exactly one freshrss replica, `strategy: Recreate`.
Default backend is sqlite; switch to PostgreSQL by re-running the in-UI installer
if you outgrow it.

The freshrss container runs Apache as `www-data` (uid/gid 33).
`securityContext.fsGroup: 33` makes the Longhorn-mounted PVCs writable by it.
`CRON_MIN=*/20` runs the in-container feed refresher every 20 minutes.

### Readable extension — auto-installed via initContainer

A short `alpine/git` initContainer clones
[`printfuck/xExtension-Readable`](https://github.com/printfuck/xExtension-Readable)
into `/var/www/FreshRSS/extensions/xExtension-Readable` on every pod start. The
commit is pinned in `k8s/deployment.yaml` (`REF=…`) for reproducibility; bump it
deliberately.

FreshRSS auto-discovers extensions on the next page load. After the first deploy,
go to **Settings → Extensions** in the FreshRSS UI and enable **Readable** —
then per feed/category, tick the "Mercury" column.

The Mercury service is reached at `http://mercury:3000` (within the cluster,
same namespace as freshrss) — pre-fill that in the extension config UI.

### Mercury sidecar

`mercury.yaml` deploys `wangqiru/mercury-parser-api:latest` on port 3000.
Multi-arch (arm64 supported, so it runs on the rpid Pis). The API is
`GET /parser?url=<encoded-url>` — returns the parsed article in JSON with
`title`, `author`, `lead_image_url`, `content` (HTML), `excerpt`, etc.

## Deploy

```sh
# 1. Append to amun-kubernetes/deployment/kustomization.yaml:
#      - github.com/GonzaloAlvarez/cn-kb-freshrss//k8s?ref=main
# 2. Apply
export KUBECONFIG=~/.kube/config-rpid
kubectl apply -k ~/dev/amun-kubernetes/deployment/
```

## First-time setup

1. **Install wizard.** Hit `http://freshrss.k8s.lan/` (or `https://freshrss.lab.gn.al/`)
   and walk through the wizard:
   - Database: **SQLite** (no extra params).
   - Default user: pick an admin name + password.
   The wizard writes its state to `freshrss-data` (the PVC), so it never re-prompts.
2. **Enable the Readable extension.** Log in → **Settings → Extensions** → tick
   **Readable**. Configure:
   - **Mercury Host**: `http://mercury:3000`
   - Readability Host / Five Filters Host: leave blank (not deployed here).
3. **Pick feeds.** Still in the Readable config page, tick the **Mercury** column
   for each feed/category you want enriched. Hit **Submit** at the bottom — the
   plugin docs are emphatic about this.

After this, every new entry inserted in those feeds calls
`http://mercury:3000/parser?url=<entry-link>` and the parsed `content` replaces
the truncated feed body. The hook is `entry_before_insert`, so existing entries
are unchanged — only new ones get enriched.

## Health

```sh
kubectl -n kb-freshrss get deploy,pvc,svc,ingress

# Mercury API direct probe — should return JSON with .content
kubectl -n kb-freshrss exec deploy/freshrss -- \
  curl -s 'http://mercury:3000/parser?url=https%3A%2F%2Fexample.com' | head -c 200

# Extension files present
kubectl -n kb-freshrss exec deploy/freshrss -- \
  ls /var/www/FreshRSS/extensions/xExtension-Readable

# LAN smoke
curl -sI http://freshrss.k8s.lan/

# Tailnet smoke
curl -skI --socks5-hostname 127.0.0.1:1055 https://freshrss.lab.gn.al/

# Tail logs
kubectl -n kb-freshrss logs deploy/freshrss --tail=50
kubectl -n kb-freshrss logs deploy/mercury  --tail=50
```

## Common tasks

```sh
# Bump the Readable extension — edit REF in deployment.yaml, commit, push, then:
kubectl apply -k ~/dev/amun-kubernetes/deployment/
kubectl -n kb-freshrss rollout restart deploy/freshrss

# Force a feed refresh (the cron normally runs every 20 minutes)
kubectl -n kb-freshrss exec deploy/freshrss -- \
  su -s /bin/sh www-data -c "php /var/www/FreshRSS/app/actualize_script.php"

# Manually exercise Mercury (good for diagnosing "extension is on but nothing changes")
kubectl -n kb-freshrss exec deploy/freshrss -- \
  curl -s 'http://mercury:3000/parser?url=https%3A%2F%2Fwww.bbc.com%2Fnews' \
  | python3 -c 'import sys,json; d=json.load(sys.stdin); print("title:", d.get("title")); print("len(content):", len(d.get("content","")))'

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
Two rows of panels:

- **FreshRSS** — pod status, restart count, request rate, CPU/mem with limit
  overlay, PVC usage (bytes + %), ingress req/s by host, HTTP status codes,
  recent Loki error log lines.
- **Mercury Parser** — pod status, restart count, CPU/mem, recent Loki log
  lines, request rate derived from Loki (`count_over_time` of `GET /parser`
  lines), error rate (Loki count of lines with `error|fail|warn`).

Datasources are `cluster-prometheus` and `cluster-loki` — the in-cluster
Prometheus + Loki, reached from the VPS Grafana via the k8s ingress at
`10.0.0.201` with `Host:` headers (already wired in
`tailnet/grafana/provisioning/datasources/k8s-cluster.yml`).

## License

GNU GPL v3
