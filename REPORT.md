# Internal Dashboard — Fix Report

## Scope

Fixed the container and Helm configuration without modifying application
code under `app/` or frontend sources under `frontend/`.

ResourceQuota and NetworkPolicy were preserved. Functional and concurrency
tests were executed through Traefik Ingress, not through port-forwarding.

## Root causes and fixes

### API container

- Corrected the Gunicorn application entry point to `main:app`.
- Bound the API listener to `0.0.0.0:8080`.
- Configured two gthread workers with four threads each to handle the
  concurrent smoke-test workload.
- Directed application logs to stdout/stderr.
- Configured Gunicorn temporary files under `/tmp`.

The image was also tested locally with a read-only root filesystem,
a writable `/tmp`, and container resource limits.

### API deployment and service

- Aligned the Service target port, container port, probes, and
  NetworkPolicy API ingress port to `8080`.
- Replaced the incorrect Service selector with the API selector helper.
- Set `APP_ENV` to `prod` through the API ConfigMap.
- Referenced the Redis connection string from Secret key `redis-url`.
- Used `/api/ready` for readiness and `/api/health` for liveness.
- Configured API resource requests and limits:
  - Requests: 100m CPU, 64Mi memory.
  - Limits: 500m CPU, 128Mi memory.
- Preserved the read-only root filesystem and provided a writable,
  memory-backed `/tmp` volume with a 32Mi size limit.
- Used `IfNotPresent` for the locally imported API image.

### Frontend and web service

- Corrected the Caddy document root to `/srv/www/dist`.
  The frontend init container writes `/out/dist`, and the shared volume
  is mounted at `/srv/www` in the Caddy container.
- Aligned the web container and Service target port to `80`.
- Used `IfNotPresent` for locally available images.

### Ingress

- Disabled `/api` prefix stripping because the application routes already
  include `/api`.
- Made Middleware creation conditional on `stripApiPrefix`.
- Preserved host-based routing for `dashboard.local`:
  - `/api` routes to the API Service on port 8000.
  - `/` routes to the web Service on port 80.

### Redis NetworkPolicy

- Corrected the allowed Redis ingress port to `6379`.
- Retained the restriction allowing Redis ingress from API Pods.
- Did not disable or remove NetworkPolicy.

### Configuration rollout

- Added API Pod-template checksums for ConfigMap and Secret content.
- Added a web Pod-template checksum for ConfigMap content.

The configuration checksum currently hashes the shared ConfigMap template.
Consequently, changing API configuration also rolls the web Deployment.
This is intentional in the current implementation, though splitting
checksums by component would avoid that additional rollout.

### CI

Updated the existing `.github/workflows/ci.yml` rather than adding a
duplicate workflow.

The workflow now includes:

- Push and pull-request triggers.
- Read-only repository permissions.
- An explicitly selected Helm version.
- Helm lint and template checks.
- API and frontend image builds.

GitHub Actions execution has not yet been verified remotely.
The workflow does not run the Kubernetes smoke test; runtime validation
was performed locally in k3d.

## Local validation

### Static checks

Passed:

```bash
git diff --check
helm lint ./charts/internal-dashboard

helm template dashboard ./charts/internal-dashboard \
  --namespace dashboard
```

Helm lint reported zero failures and only the optional chart icon
recommendation.

Confirmed no application or frontend source changes:

```bash
git diff HEAD -- app/ frontend/
```

The command produced no output.

### Build and deployment procedure

Local image names:

- `internal-dashboard-api:local`
- `internal-dashboard-frontend:local`

Build and import commands:

```bash
docker build -t internal-dashboard-api:local .
docker build -t internal-dashboard-frontend:local ./frontend

k3d image import \
  internal-dashboard-api:local \
  internal-dashboard-frontend:local \
  -c dashboard
```

The release was successfully installed and subsequently upgraded using:

```bash
helm upgrade --install dashboard ./charts/internal-dashboard \
  -n dashboard \
  --create-namespace \
  --wait
```

The final tested release was revision 3 in namespace `dashboard`.

### Ingress checks

Requests were sent through the locally exposed Traefik Ingress using:

```bash
curl --noproxy '*' \
  --resolve dashboard.local:80:127.0.0.1 \
  --max-time 10 \
  -i http://dashboard.local/

curl --noproxy '*' \
  --resolve dashboard.local:80:127.0.0.1 \
  --max-time 10 \
  -i http://dashboard.local/api/

curl --noproxy '*' \
  --resolve dashboard.local:80:127.0.0.1 \
  --max-time 10 \
  -i http://dashboard.local/api/health
```

Observed results:

| Endpoint | Status | Result |
|---|---|---|
| `/` | HTTP 200 | Internal Dashboard HTML |
| `/api/` | HTTP 200 | `{"cache":"ok","env":"prod","message":"Hello"}` |
| `/api/health` | HTTP 200 | `{"status":"OK"}` |

### Smoke and concurrency test

Command:

```bash
helm test dashboard -n dashboard --logs
```

Final result on release revision 3:

```text
Phase: Succeeded

[1/3] Checking /, /api/, /api/health via Ingress...
[2/3] Running concurrency burst (40 requests, 20 parallel)...
Burst finished in 2s (limit: 4s)
[3/3] Verifying API health after burst...
PASS: all smoke and concurrency checks succeeded.

helm test exit code: 0
```

### Stability

After the initial successful smoke test, the same API, web, and Redis
Pods remained Ready with zero restarts during an additional 180-second
observation period.

After the final configuration restoration and smoke test, all three
service Pods were again Ready with zero restarts. At that observation,
the new API and web Pods were approximately 2 minutes 10 seconds old.

### Automatic rollout verification

1. Changed `api.config.APP_ENV` from `prod` to `rollout-check`.
2. Ran a regular Helm upgrade without manually restarting or deleting Pods.
3. Observed new API and web Pods while Redis remained unchanged.
4. Verified through Ingress:

   ```json
   {"cache":"ok","env":"rollout-check","message":"Hello"}
   ```

5. Restored `APP_ENV: prod` and ran another Helm upgrade.
6. Verified through Ingress:

   ```json
   {"cache":"ok","env":"prod","message":"Hello"}
   ```

The ConfigMap-triggered rollout was verified at runtime.
Secret-triggered rollout was configured but was not separately exercised.

### ResourceQuota

Final observed quota accounting:

| Resource | Used | Hard |
|---|---:|---:|
| CPU limits | 900m | 2 |
| Memory limits | 320Mi | 640Mi |
| Memory requests | 112Mi | 384Mi |

These values represent quota accounting for declared resources, not
measured runtime CPU or memory consumption.

## Remaining verification

- Run the updated GitHub Actions workflow after pushing the branch.
- Secret-triggered rollout has not been tested separately.