# exroute-deploy

Kubernetes manifests for exroute, managed with Kustomize.

```
base/              shared manifests: postgres, valhalla, exroute (API), exroute-web
overlays/local/    k3d on your machine, namespace "exroute", http://exroute.localhost:8080
overlays/prod/     k3s on the server, namespace "exroute-prod", https://exroute.benciks.uk
```

The web app and the API share one domain. The Ingress sends `/api` to `exroute:8000` and everything else to `exroute-web:3000`.

## Prerequisites

- `kubectl` 1.27+ (it includes Kustomize)
- Local: Docker and [k3d](https://k3d.io)
- Prod: a server running k3s, ports 80 and 443 open, and a DNS `A` record from `exroute.benciks.uk` to the server's IP
- A GitHub classic personal access token with only the `read:packages` scope, because the images on ghcr.io are private
- The Valhalla data (see [Valhalla data](#valhalla-data-first-boot))

## Secrets

Secrets are not stored in this repo. Create them once in every namespace (`exroute` locally, `exroute-prod` on the server), and generate fresh values for each environment.

Make sure the namespace exists first: `kubectl create namespace <ns>`, or run the apply step once beforehand.

```sh
NS=exroute   # or exroute-prod

kubectl create secret generic postgresql -n $NS \
  --from-literal=POSTGRES_USER=exroute \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -hex 24)" \
  --from-literal=POSTGRES_DB=exroute

kubectl create secret generic exroute -n $NS \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)" \
  --from-env-file=exroute-keys.env   # LLM_API_KEY, GEOCODE_API_KEY, MAPBOX_API_KEY (gitignored)

kubectl create secret docker-registry ghcr -n $NS \
  --docker-server=ghcr.io \
  --docker-username=benciks \
  --docker-password=<read:packages token>
```

Postgres only reads `POSTGRES_PASSWORD` on its very first start, when the volume is empty. To change it later, run `ALTER USER` in Postgres and then update the Secret.

## Local (k3d)

```sh
# One time. The port mapping can only be set when the cluster is created.
k3d cluster create exroute -p "8080:80@loadbalancer"

kubectl config use-context k3d-exroute
# create the secrets (see above) with NS=exroute
kubectl apply -k overlays/local
```

Then copy the Valhalla data (see below) and open http://exroute.localhost:8080.

## Prod (k3s)

```sh
kubectl config use-context <server context>   # check this, you don't want to deploy to the wrong cluster
# create the secrets (see above) with NS=exroute-prod
kubectl apply -k overlays/prod
```

Then copy the Valhalla data (see below) and open http://exroute.benciks.uk.

HTTPS is not set up yet. Traefik serves a self-signed certificate until cert-manager is added.

## Valhalla data (first boot)

Valhalla only serves tiles. It never builds them. Build the tiles locally (in the `valhalla` repo, `custom_files/`) with the **same Valhalla version** as the image. Then copy these four files into the `valhalla-pvc` volume:

```
valhalla.json  valhalla_tiles.tar  admins.sqlite  timezones.sqlite
```

The paths inside `valhalla.json` must point to `/custom_files/...`, the mount path in the container.

On the first `apply`, Valhalla crashes because the volume is empty. That's expected. The crashing pod still causes k3s to create the volume directory, which is where the files go.

**1. Find the volume directory on the node:**

```sh
PV=$(kubectl get pvc valhalla-pvc -n $NS -o jsonpath='{.spec.volumeName}')
kubectl get pv $PV -o jsonpath='{.spec.hostPath.path}{.spec.local.path}{"\n"}'
# e.g. /var/lib/rancher/k3s/storage/pvc-<uuid>_<ns>_valhalla-pvc
```

**2a. Local: copy into the k3d node container:**

```sh
docker cp custom_files/. k3d-exroute-server-0:<volume dir>/
```

Copy only the four files listed above. `custom_files/` may also contain the `.pbf` and the unpacked `valhalla_tiles/`, which aren't needed.

**2b. Prod: rsync to the server:**

```sh
rsync -avP valhalla.json valhalla_tiles.tar admins.sqlite timezones.sqlite user@server:~/valhalla/
ssh user@server 'sudo mv ~/valhalla/* <volume dir>/'
```

Moving the files on the server means Valhalla never sees a half-copied file.

**3. Restart Valhalla:**

```sh
kubectl rollout restart deployment/valhalla -n $NS
kubectl logs -n $NS deploy/valhalla -f
```

The files stay in the volume across restarts and redeploys. Deleting the PVC or the namespace deletes them (`local-path` uses `reclaimPolicy: Delete`), so keep a copy.

To update the data, repeat steps 2 and 3 with the new files.

## Users

Public registration is disabled. Create users with the CLI that is built into the API image. It runs inside the running `exroute` pod, so it uses the pod's `DATABASE_URL` and can reach Postgres.

```sh
kubectl exec -it -n $NS deploy/exroute -- \
  python -m app.cli create-user --email rider@example.com
```

It asks for the password twice. `-it` connects your terminal to the pod so you can answer those prompts.

To run it without the prompts (from a script), pipe the password in with `--password-stdin`. Use `-i` without `-t`:

```sh
read -rs PW   # type the password; it isn't echoed or saved to shell history
printf '%s\n' "$PW" | kubectl exec -i -n $NS deploy/exroute -- \
  python -m app.cli create-user --email rider@example.com --password-stdin
unset PW
```

Don't pass the password as a command-line argument, because it would show up in process lists and shell history.

Users are stored in each environment's own Postgres, so create them separately for local and prod. They stay across restarts because they're in the Postgres volume. Deleting the `postgres` PVC deletes them.

## Updating

Push a new image tag, change it in `base/<app>/deployment.yaml`, then:

```sh
kubectl apply -k overlays/<local|prod>
kubectl rollout status deploy/<app> -n $NS
```

Always use a new tag. Re-pushing the same tag does nothing, because the manifests don't change.

## Debugging

```sh
kubectl get all -n $NS
kubectl logs -n $NS deploy/<app> --previous   # output of the last crashed container
kubectl describe pod -n $NS -l app=<app>      # events: pull errors, probe failures, OOM
```
