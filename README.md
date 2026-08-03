# stackiq-infra

Infrastructure for deploying the StackIQ project on your own bare metal server.

This repository contains everything needed to run StackIQ on a Kubernetes cluster at home: the application manifests, the supporting infrastructure (ingress, TLS, load balancing, container registry), and an optional full CI/CD pipeline powered by Flux and Gitea Actions.

## Table of Contents

- [Used Technologies](#used-technologies)
- [Prerequisites](#prerequisites)
- [Choosing Your Path](#choosing-your-path)
- [Customizing for Your Own Fork](#customizing-for-your-own-fork)
- [Server Setup](#server-setup)
- [Gitea Setup](#gitea-setup)
- [Kubernetes and Flux Setup](#kubernetes-and-flux-setup)
- [Environment Secrets](#environment-secrets)
- [kubectl common commands](#kubectl-common-commands)

## Used Technologies

| Technology | Role |
|---|---|
| Kubernetes | Container orchestration. Everything runs as pods in a cluster. |
| Flux | GitOps engine. Watches this repository and keeps the cluster in sync with it. |
| Gitea | Self-hosted git server, container registry, and CI runner (Actions). |
| ingress-nginx | Routes incoming HTTP/HTTPS traffic to the right service. |
| cert-manager | Automatically issues and renews Let's Encrypt TLS certificates. |
| MetalLB | Gives the cluster a LoadBalancer IP on your home network. |
| PostgreSQL | Main application database, managed with Prisma migrations. |
| Redis | Cache and BullMQ job queue backend. |
| DuckDNS | Free dynamic DNS pointing hostnames at your home IP. |

The application itself consists of four services: a frontend, a backend API, a worker that processes analysis jobs, and the databases that support them.

## Prerequisites

Hardware and network:

- A machine that will act as the server. Any modern x64 machine with 8 GB or more of RAM is fine.
- A router you control. You will need to forward ports 80 and 443 to the server.
- A free static IP on your LAN for MetalLB to hand out (for example `192.168.0.50`). Pick an address outside your router's DHCP range.

Software on the server:

- A Linux distribution (Ubuntu Server is recommended).
- A running Kubernetes cluster. Using k3s is the simplest and recommended option:

  ```bash
  curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -
  ```

Software on your workstation:

- `kubectl`, configured to talk to your cluster. Copy the kubeconfig from the server (for k3s it is at `/etc/rancher/k3s/k3s.yaml`) and point the `server:` field at your server's LAN IP.
- The `flux` CLI (only needed for the full CI/CD path). Install instructions: https://fluxcd.io/flux/installation/
- `docker` (only needed for the static path, to build and push images manually).

DNS:

- Two hostnames pointing at your public IP. We recommend DuckDNS for a free DNS solution:
  1. Create an account at https://www.duckdns.org
  2. Create two subdomains, one for the app (for example `myserver.duckdns.org`) and one for Gitea (for example `myservergit.duckdns.org`).
  3. Set both to your home's public IP. Install the DuckDNS update client (or a cron job) on the server so the records follow your IP if it changes.

## Choosing Your Path

There are two ways to deploy this project. Read this section first and pick one.

### Path A: Full GitOps with CI/CD (recommended)

Flux is bootstrapped against your fork of this repository. Gitea Actions builds a new image on every push to the app repository, pushes it to the Gitea registry, and Flux image automation detects the new tag and commits the version bump back to this repository automatically. 

Use this path if you plan to develop the application or keep it updated.

### Path B: Static one-time deployment

You build and push the images once by hand, then apply the manifests once with `kubectl apply -k`. No Flux, no Actions runner, no image automation. The cluster runs whatever you deployed until you manually apply again.

Use this path if you just want to run the app and do not care about automatic updates.

Both paths use Gitea. The manifests pull images from the Gitea container registry, so Gitea is a required component either way. The difference is only whether images get built and rolled out automatically.

Components used per path:

| Component | Path A | Path B |
|---|---|---|
| Gitea (registry) | Yes | Yes |
| Gitea Actions runner | Yes | No |
| Flux | Yes | No |
| Flux image automation | Yes | No |

## Customizing for Your Own Fork

Fork this repository before doing anything else. Then change the following values to match your environment. Search the repository for the old value and replace every occurrence.

| What to change | Current value | Files |
|---|---|---|
| App hostname | `jethroserver.duckdns.org` | `apps/stackiq/ingress.yaml`, `apps\stackiq\backend-deployment.yaml`, `apps\stackiq\worker-deployment.yaml` |
| Gitea hostname | `jethroservergit.duckdns.org` | `infrastructure/controllers/gitea.yaml`, `infrastructure/controllers/gitea-runner.yaml`, `apps/stackiq/*-deployment.yaml` (image fields), `clusters/home/image-policies.yaml` |
| MetalLB IP | `192.168.0.50/32` | `infrastructure/configs/metallb-config.yaml` |
| email | `jethroroy@gmail.com` | `infrastructure/configs/cluster-issuer.yaml` |
| Gitea admin email | `jethroroy@gmail.com` | `infrastructure/controllers/gitea.yaml` |
| Git repository URL | `https://github.com/stackiq-team/stackiq-infra.git` | Set automatically when you bootstrap Flux against your fork (Path A). Not used in Path B. |
| Registry org/repo names | `stackiq-team/backend` etc. | `apps/stackiq/*-deployment.yaml`, `clusters/home/image-policies.yaml` |


## Server Setup

These steps assume a fresh Kubernetes cluster and a workstation with `kubectl` commands working.

### 1. Verify cluster access

```bash
kubectl get nodes
```

You should see your server node in `Ready` state.

### 2. Understand what gets installed

The infrastructure layer is defined in two folders:

- `infrastructure/controllers/`: the software itself, installed as Flux HelmReleases. This includes ingress-nginx, cert-manager, MetalLB, Gitea, and the Gitea Actions runner.
- `infrastructure/configs/`: configuration that can only apply after the controllers exist. This includes the Let's Encrypt ClusterIssuer and the MetalLB address pool.

On Path A, Flux applies both folders for you in the correct order (configs declare a dependency on controllers). On Path B you apply them manually, which is covered below.

### 3. Path B only: install the infrastructure by hand

Path B has no Flux, but the controller manifests are written as Flux HelmRelease resources. The simplest approach that avoids rewriting them is to install Flux's controllers only so they can process the HelmReleases:

```bash
flux install
kubectl apply -k infrastructure/controllers
```
> wait for the helm releases to be ready, then:

```bash
kubectl apply -k infrastructure/configs
```

### 4. Port forwarding

Port forwarding means opening up a port to the internet. Make sure you know what this means and look up ways to do this safely without compromising your server.

On your router, forward external ports 80(HTTP) and 443(SSL/TLS) to the IP you configured (for example `192.168.0.50`). The ingress-nginx service claims that IP as a LoadBalancer. Verify with:

```bash
kubectl get svc -n ingress-nginx
```

The `EXTERNAL-IP` column should show your IP address.

### 5. TLS certificates

Check your certificates with:

```bash
kubectl get certificate -A
```

## Gitea Setup

Gitea is installed by the infrastructure layer. These steps configure it after it is running.

### 1. Create the admin secret

The Gitea Helm chart reads its admin credentials from an existing Secret named `gitea-admin-secret`:

```bash
kubectl create secret generic gitea-admin-secret -n gitea \
  --from-literal=username=gitea_admin \
  --from-literal=password=CHOOSE_A_STRONG_PASSWORD
```

Create this before (or right after) the Gitea HelmRelease installs. If Gitea started without it, restart the release after creating it.

### 2. Log in and create the organization

Open `https://<your-gitea-hostname>` in a browser, log in with the admin account, and create an organization matching the registry paths in the manifests (default: `stackiq-team`). Create repositories for the app source code.

### 3. Registry credentials for the cluster

Every Deployment pulls images from the Gitea registry using an image pull secret named `gitea-registry` in the `stackiq` namespace. Create it:

```bash
kubectl create namespace stackiq
kubectl create secret docker-registry gitea-registry -n stackiq \
  --docker-server=<your-gitea-hostname> \
  --docker-username=gitea_admin \
  --docker-password=YOUR_GITEA_PASSWORD
```

Path A also needs a copy for Flux's image automation to scan the registry, named `gitea-registry-flux` in the `flux-system` namespace:

```bash
kubectl create secret docker-registry gitea-registry-flux -n flux-system \
  --docker-server=<your-gitea-hostname> \
  --docker-username=gitea_admin \
  --docker-password=YOUR_GITEA_PASSWORD
```

### 4. Build and push the images

Log in to the registry from your workstation:

```bash
docker login <your-gitea-hostname>
```

Path B (manual, one time): build and push each service from the application repository:

```bash
docker build -t <your-gitea-hostname>/stackiq-team/backend:1.0.0 ./backend
docker push <your-gitea-hostname>/stackiq-team/backend:1.0.0
docker build -t <your-gitea-hostname>/stackiq-team/frontend:1.0.0 ./frontend
docker push <your-gitea-hostname>/stackiq-team/frontend:1.0.0
docker build -t <your-gitea-hostname>/stackiq-team/worker:1.0.0 ./worker
docker push <your-gitea-hostname>/stackiq-team/worker:1.0.0
```

Then set the `image:` fields in `apps/stackiq/*-deployment.yaml` to the tags you pushed.

Path A (automatic): the application repository contains a Gitea Actions workflow (`.gitea/workflows/build.yaml`) that builds and pushes all three images with an auto-incrementing version on every push. For it to run you need the Actions runner:

1. In the Gitea web UI, go to Site Administration, then Actions, then Runners, and copy a registration token.
2. Store it as the secret the runner Deployment expects:

   ```bash
   kubectl create secret generic gitea-runner-secret -n gitea-runner \
     --from-literal=token=YOUR_REGISTRATION_TOKEN
   ```

3. The runner Deployment in `infrastructure/controllers/gitea-runner.yaml` picks it up and registers itself. Confirm the runner appears as online in the Gitea admin UI.

## Kubernetes and Flux Setup

### Path A: bootstrap Flux

Flux bootstrap connects your cluster to your fork and installs its own controllers. Using a GitHub fork:

```bash
flux bootstrap github \
  --owner=YOUR_GITHUB_USER_OR_ORG \
  --repo=stackiq-infra \
  --branch=main \
  --path=clusters/home \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```

Notes:

- `--path=clusters/home` matters. That folder contains the Kustomizations that pull in everything else.
- `--components-extra` installs the two controllers needed for image automation. Without them, automatic image updates will not work.
- Bootstrap will ask for a GitHub personal access token with repo permissions. Flux uses it to commit its own manifests and, later, image version bumps.

After bootstrap, Flux reconciles in this order automatically:

1. `infrastructure-controllers` (ingress, cert-manager, MetalLB, Gitea, runner)
2. `infrastructure-configs` (ClusterIssuer, MetalLB pool), which waits for step 1
3. `apps` (the StackIQ application), which also waits for step 1

Once everything is Ready, the app is live at your app hostname.

How the image automation loop works on this path:

1. You push code to the app repository in Gitea.
2. The Actions workflow builds images tagged `1.0.<run number>` and pushes them to the registry.
3. Flux's ImageRepository resources scan the registry every minute.
4. The ImagePolicy picks the highest server tag.
5. The ImageUpdateAutomation commits the new tag into `apps/stackiq/*-deployment.yaml` in this repository (the `# {"$imagepolicy": ...}` comments mark the lines it edits).
6. Flux applies the change to the cluster.

### Path B: apply the app manifests once

With infrastructure and images in place:

```bash
kubectl apply -k apps/stackiq
```

## Environment Secrets

The application reads its configuration from a single Kubernetes Secret named `stackiq-secrets` in the `stackiq` namespace. The Deployments map individual keys from it into environment variables. The Secret is not stored in git; you create it directly in the cluster.

### Initial creation

```bash
kubectl create secret generic stackiq-secrets \
  -n stackiq \
  --from-literal=DATABASE_URL='postgresql://postgres:YOUR_DB_PASSWORD@postgres:5432/stackiq?schema=public' \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=YOUR_DB_PASSWORD \
  --from-literal=POSTGRES_DB=stackiq \
  --from-literal=REDIS_URL=redis://redis:6379 \
  --from-literal=BULLMQ_QUEUE_NAME=stackiq-analysis \
  --from-literal=OPENAI_API_KEY=YOUR_OPENAI_KEY \
  --from-literal=GITHUB_API_TOKEN=YOUR_GITHUB_TOKEN \
  --from-literal=GITHUB_MINER_TIMEOUT_MS=20000 \
  --from-literal=ISSUES_MINING_TIMEOUT_MS=300000 \
  --from-literal=ISSUES_MINING_LOOKBACK_DAYS=180 \
  --from-literal=ISSUES_MINING_MAX_ISSUES=100 \
  --from-literal=ISSUES_MINING_TIMELINE_ITEMS=100 \
  --from-literal=ISSUES_MINING_MAX_TIMELINE_PAGES=1 \
  --from-literal=ISSUES_MINING_INCLUDE_DEV_DEPENDENCIES=true \
  --from-literal=DEPENDENCY_CACHE_VERSION=v1 \
  --from-literal=DEPENDENCY_CACHE_TTL_DAYS=14 \
  --from-literal=PARTIAL_DEPENDENCY_CACHE_TTL_DAYS=1 \
  --from-literal=WORKER_CONCURRENCY=4 \
  --from-literal=MAILER_USER=YOUR_GMAIL_ADDRESS \
  --from-literal='GMAIL_APP_PASSWORD=YOUR GMAIL APP PASSWORD' \
  --from-literal=DEPENDENCY_SYNC_ENABLED=true \
  --from-literal=DEPENDENCY_SYNC_INTERVAL_MS=604800000 \
  --from-literal=DEPENDENCY_SYNC_RUN_ON_START=false \
  --from-literal=DEPENDENCY_SYNC_TOP_LIMIT=10 \
  --from-literal=DEPENDENCY_RELATIONSHIPS_ENABLED=true \
  --from-literal=DEPENDENCY_RELATIONSHIP_MAX_PAIRS=30 \
  --from-literal=DEPENDENCY_CACHE_LOCK_WAIT_MS=300000 \
  --from-literal=DEPENDENCY_RELATIONSHIP_SEARCH_RESULTS=5 \
  --from-literal=EXPLORE_TOP_LIMIT=12 \
  --from-literal=EXPLORE_REFRESH_INTERVAL_MS=604800000 \
  --from-literal=EXPLORE_RUN_ON_START=true
```

Notes:

- OPENAI key not necessary to run the app
- Gmail app passwords contain spaces, so that entry must be quoted as a whole.
- On PowerShell, replace the trailing `\` line continuations with backticks and use double quotes around any value containing spaces.
- The GitHub token needs public repo read scope. The Gmail credentials are used to send result emails and are optional if you do not need email.

## kubectl common commands

Seeing what is running:

```bash
kubectl get pods -n stackiq                # list pods and their status
kubectl get deployments -n stackiq         # list deployments and replica counts
kubectl get svc -n stackiq                 # list services
kubectl get pods -A                        # everything across all namespaces
```

Reading logs:

```bash
kubectl logs deployment/backend -n stackiq --tail=100   # last 100 lines
kubectl logs -f deployment/worker -n stackiq            # follow live
kubectl logs <pod-name> -n stackiq --previous           # logs from before the last crash
```

Running commands inside a pod:

```bash
kubectl exec -it deployment/backend -n stackiq -- sh                            # interactive shell
kubectl exec -it deployment/backend -n stackiq -- sh -c 'echo $DATABASE_URL'
```

On PowerShell, use single quotes around the inner command as shown above. Double quotes make PowerShell substitute `$VARIABLES` on your workstation before the command ever reaches the pod, which silently produces empty values.

Restarting and scaling:

```bash
kubectl rollout restart deployment/backend -n stackiq   # replace pods with fresh ones
kubectl rollout restart deployment -n stackiq           # restart every deployment in the namespace
kubectl scale deployment/worker -n stackiq --replicas=0 # stop a deployment
kubectl scale deployment/worker -n stackiq --replicas=1 # start it again
```

Note for Path A: Flux enforces what is in git. If you scale or edit a Flux-managed resource by hand, Flux will revert it on the next reconcile. Manual changes are fine for quick debugging, but permanent changes belong in the repository.

Inspecting details:

```bash
kubectl describe pod <pod-name> -n stackiq       # events, restart reasons, image pulls
kubectl get deployment backend -n stackiq -o yaml
kubectl get secret stackiq-secrets -n stackiq -o jsonpath="{.data}"   # keys, base64 values
```

Database access:

```bash
kubectl exec -it deployment/postgres -n stackiq -- psql -U postgres -d stackiq
```

Resetting all application data (destructive, keeps the schema by replaying migrations):

```bash
kubectl exec -it deployment/backend -n stackiq -- sh -c "npx prisma migrate reset --force"
kubectl exec -it deployment/redis -n stackiq -- redis-cli FLUSHALL
kubectl rollout restart deployment/backend -n stackiq
kubectl rollout restart deployment/worker -n stackiq
```

Flux status (Path A):

```bash
flux get kustomizations                                 # sync status of each layer
flux reconcile kustomization apps -n flux-system        # force an immediate sync
kubectl get gitrepository -n flux-system                # is Flux pulling the repo correctly
kubectl get kustomization apps -n flux-system -o yaml   # full status, conditions, applied revision
```