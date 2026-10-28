# shorty

a link shortener I use as a playground for running an app the way it's run in prod.

the app itself (flask + postgres, ~150 lines in `app.py`) came ready-made with the task, I didn't write it and it's not the point.
the point is everything around it: the image, the kubernetes setup, and next ci and a real deploy.

## what it does

| path | what |
|---|---|
| `GET /` | form + last 20 links + which pod answered |
| `POST /api/links` `{"url": "https://..."}` | create a link, returns the code |
| `GET /<code>` | 302 to the original url, counts hits |
| `GET /healthz` | process is alive. doesn't touch the db |
| `GET /readyz` | 200 if the db is reachable, 503 if not. creates the table on first success |
| `GET /metrics` | prometheus metrics |

config is env only: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `APP_TITLE`.

## what's here

```
Dockerfile
k8s/
  00-namespace.yaml   namespace, pod security "restricted" in warn mode
  10-config.yaml      configmap: db host/name/user, gunicorn workers
  20-postgres.yaml    postgres statefulset + volume
  40-web.yaml         the app, 2 replicas
  50-network.yaml     service for the app + headless service for postgres
```

## how it's set up and why

**image**

- `python:3.14-slim`, requirements copied and installed before the code -> changing the code doesn't reinstall deps
- runs as uid 10001, not root
- `CMD` in exec form -> gunicorn is pid 1 and gets SIGTERM from kubelet directly. delete a pod and you see `got SIGTERM` in the logs instead of a 30s wait and SIGKILL
- gunicorn heartbeat files go to `/dev/shm` (memory), so a slow disk doesn't make workers look dead
- 2 workers per pod (`WEB_CONCURRENCY`). scaling is done with replicas, not with more workers

**postgres**

- statefulset with `volumeClaimTemplates`, so data survives `kubectl delete pod`. the pod comes back as `postgres-0` and gets the same volume
- headless service (`clusterIP: None`), `postgres` resolves straight to the pod ip
- the volume is mounted at `/var/lib/postgresql`, not `.../data` like older guides say. that's how the postgres 18 image wants it
- password comes from a secret that isn't in git (command below)
- startup probe gives it ~25s for the first start before liveness kicks in
- readiness checks over tcp (`-h 127.0.0.1`), liveness over the socket. during the first init postgres only listens on the socket, so a socket check says "ready" too early

**app**

- liveness -> `/healthz`, readiness -> `/readyz`. not the other way round: if liveness checked the db, a dead postgres would make kubelet restart every web pod in a loop, and restarting doesn't fix the db. this way the pods just drop out of the service until the db is back
- config from the configmap via `envFrom`, password from the secret
- requests/limits on both

## run it on kind

```sh
kind create cluster --name shorty

# the package on ghcr is private for now, so build and load it yourself
docker build -t ghcr.io/iswaff/shorty:0.1 .
kind load docker-image ghcr.io/iswaff/shorty:0.1 --name shorty

kubectl apply -f k8s/00-namespace.yaml
kubectl -n shorty create secret generic postgres-auth --from-literal=password="$(openssl rand -hex 16)"
kubectl apply -f k8s/

kubectl -n shorty get pods -w      # wait for postgres-0 and both web pods to be READY
kubectl -n shorty port-forward svc/web 8080:80
```

open http://localhost:8080 or:

```sh
curl -X POST localhost:8080/api/links -H 'content-type: application/json' -d '{"url": "https://example.com"}'
```

port-forward sticks to one pod, so "served by" won't change on refresh. to see the service balancing you need to go through ingress (todo).

## things that bit me

- the secret isn't in git, so if you recreate the namespace the secret is gone too and the pods sit in `CreateContainerConfigError`. that's why the command is written down above
- deleting the namespace deletes the pvc, and kind's default storage class has `reclaimPolicy: Delete`. so the database goes with it. a pvc doesn't mean the data is safe
- `POSTGRES_PASSWORD` is only read on the very first start, when the data dir is empty. change the secret later and postgres still has the old password, the web pods can't log in. to rotate it you also run `ALTER USER` inside postgres

## todo

- [ ] ci: build on every push, push the image to ghcr, tag = commit sha
- [ ] ingress instead of port-forward
- [ ] helm chart with dev/prod values
- [ ] `securityContext` so the restricted pod security policy stops warning
- [ ] run it for real at shorty.iswaff.com through a cloudflare tunnel
- [ ] daily `pg_dump` backups (cronjob) and actually test a restore
- [ ] scrape `/metrics` with prometheus
