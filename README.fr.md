# stackiq-infra

Infrastructure pour déployer le projet StackIQ sur votre propre serveur bare metal.

Ce dépôt contient tout ce qu'il faut pour faire tourner StackIQ sur un cluster Kubernetes à la maison : les manifestes de l'application, l'infrastructure de support (ingress, TLS, load balancing, registre de conteneurs), et un pipeline CI/CD complet optionnel propulsé par Flux et Gitea Actions.

## Table des matières

- [Technologies utilisées](#technologies-utilisées)
- [Prérequis](#prérequis)
- [Choisir votre approche](#choisir-votre-approche)
- [Personnaliser pour votre propre fork](#personnaliser-pour-votre-propre-fork)
- [Configuration du serveur](#configuration-du-serveur)
- [Configuration de Gitea](#configuration-de-gitea)
- [Configuration de Kubernetes et Flux](#configuration-de-kubernetes-et-flux)
- [Secrets d'environnement](#secrets-denvironnement)
- [Commandes kubectl courantes](#commandes-kubectl-courantes)

## Technologies utilisées

| Technologie | Rôle |
|---|---|
| Kubernetes | Orchestration de conteneurs. Tout s'exécute sous forme de pods dans un cluster. |
| Flux | Moteur GitOps. Surveille ce dépôt et maintient le cluster synchronisé avec celui-ci. |
| Gitea | Serveur git auto-hébergé, registre de conteneurs, et runner CI (Actions). |
| ingress-nginx | Route le trafic HTTP/HTTPS entrant vers le bon service. |
| cert-manager | Émet et renouvelle automatiquement les certificats TLS Let's Encrypt. |
| MetalLB | Fournit au cluster une IP LoadBalancer sur votre réseau local. |
| PostgreSQL | Base de données principale de l'application, gérée avec les migrations Prisma. |
| Redis | Cache et backend de la queue de jobs BullMQ. |
| DuckDNS | DNS dynamique gratuit qui pointe des noms d'hôtes vers votre IP domestique. |

L'application elle-même se compose de quatre services : un frontend, une API backend, un worker qui traite les jobs d'analyse, et les bases de données qui les supportent.

## Prérequis

Matériel et réseau :

- Une machine qui servira de serveur. N'importe quelle machine x64 moderne avec 8 Go de RAM ou plus convient.
- Un routeur que vous contrôlez. Vous devrez rediriger les ports 80 et 443 vers le serveur.
- Une IP statique libre sur votre LAN pour que MetalLB puisse l'attribuer (par exemple `192.168.0.50`). Choisissez une adresse en dehors de la plage DHCP de votre routeur.

Logiciels sur le serveur :

- Une distribution Linux (Ubuntu Server est recommandé).
- Un cluster Kubernetes en cours d'exécution. Utiliser k3s est l'option la plus simple et recommandée :

  ```bash
  curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -
  ```

Logiciels sur votre poste de travail :

- `kubectl`, configuré pour communiquer avec votre cluster. Copiez le kubeconfig depuis le serveur (pour k3s il se trouve dans `/etc/rancher/k3s/k3s.yaml`) et pointez le champ `server:` vers l'IP LAN de votre serveur.
- Le CLI `flux` (nécessaire uniquement pour l'approche CI/CD complète). Instructions d'installation : https://fluxcd.io/flux/installation/
- `docker` (nécessaire uniquement pour l'approche statique, pour construire et pousser les images manuellement).

DNS :

- Deux noms d'hôtes pointant vers votre IP publique. Nous recommandons DuckDNS comme solution DNS gratuite :
  1. Créez un compte sur https://www.duckdns.org
  2. Créez deux sous-domaines, un pour l'application (par exemple `myserver.duckdns.org`) et un pour Gitea (par exemple `myservergit.duckdns.org`).
  3. Configurez les deux vers l'IP publique de votre domicile. Installez le client de mise à jour DuckDNS (ou un cron job) sur le serveur pour que les enregistrements suivent votre IP si elle change.

## Choisir votre approche

Il y a deux façons de déployer ce projet. Lisez cette section en premier et choisissez-en une.

### Approche A : GitOps complet avec CI/CD (recommandé)

Flux est bootstrappé sur votre fork de ce dépôt. Gitea Actions construit une nouvelle image à chaque push sur le dépôt de l'application, la pousse vers le registre Gitea, et l'automatisation d'images de Flux détecte le nouveau tag et committe automatiquement la mise à jour de version dans ce dépôt.

Utilisez cette approche si vous prévoyez de développer l'application ou de la garder à jour.

### Approche B : Déploiement statique ponctuel

Vous construisez et poussez les images une fois à la main, puis appliquez les manifestes une fois avec `kubectl apply -k`. Pas de Flux, pas de runner Actions, pas d'automatisation d'images. Le cluster fait tourner ce que vous avez déployé jusqu'à ce que vous appliquiez à nouveau manuellement.

Utilisez cette approche si vous voulez simplement faire tourner l'application sans vous soucier des mises à jour automatiques.

Les deux approches utilisent Gitea. Les manifestes récupèrent les images depuis le registre de conteneurs Gitea, donc Gitea est un composant requis dans les deux cas. La seule différence est de savoir si les images sont construites et déployées automatiquement.

Composants utilisés selon l'approche :

| Composant | Approche A | Approche B |
|---|---|---|
| Gitea (registre) | Oui | Oui |
| Runner Gitea Actions | Oui | Non |
| Flux | Oui | Non |
| Automatisation d'images Flux | Oui | Non |

## Personnaliser pour votre propre fork

Forkez ce dépôt avant toute autre chose. Modifiez ensuite les valeurs suivantes pour correspondre à votre environnement. Recherchez l'ancienne valeur dans le dépôt et remplacez chaque occurrence.

| Quoi changer | Valeur actuelle | Fichiers |
|---|---|---|
| Nom d'hôte de l'app | `stackiq.duckdns.org` | `apps/stackiq/ingress.yaml`, `apps\stackiq\backend-deployment.yaml`, `apps\stackiq\worker-deployment.yaml` |
| Nom d'hôte de Gitea | `jethroservergit.duckdns.org` | `infrastructure/controllers/gitea.yaml`, `infrastructure/controllers/gitea-runner.yaml`, `apps/stackiq/*-deployment.yaml` (champs image), `clusters/home/image-policies.yaml` |
| IP MetalLB | `192.168.0.50/32` | `infrastructure/configs/metallb-config.yaml` |
| Email | `jethroroy@gmail.com` | `infrastructure/configs/cluster-issuer.yaml` |
| Email admin Gitea | `jethroroy@gmail.com` | `infrastructure/controllers/gitea.yaml` |
| URL du dépôt Git | `https://github.com/stackiq-team/stackiq-infra.git` | Défini automatiquement lors du bootstrap de Flux sur votre fork (Approche A). Non utilisé pour l'Approche B. |
| Noms d'org/dépôt du registre | `stackiq-team/backend` etc. | `apps/stackiq/*-deployment.yaml`, `clusters/home/image-policies.yaml` |

## Configuration du serveur

Ces étapes supposent un cluster Kubernetes tout neuf et un poste de travail où les commandes `kubectl` fonctionnent.

### 1. Vérifier l'accès au cluster

```bash
kubectl get nodes
```

Vous devriez voir votre nœud serveur à l'état `Ready`.

### 2. Comprendre ce qui est installé

La couche infrastructure est définie dans deux dossiers :

- `infrastructure/controllers/` : les logiciels eux-mêmes, installés en tant que HelmReleases Flux. Cela inclut ingress-nginx, cert-manager, MetalLB, Gitea, et le runner Gitea Actions.
- `infrastructure/configs/` : la configuration qui ne peut s'appliquer qu'une fois les contrôleurs existants. Cela inclut le ClusterIssuer Let's Encrypt et le pool d'adresses MetalLB.

Pour l'Approche A, Flux applique les deux dossiers pour vous dans le bon ordre (configs déclare une dépendance vers controllers). Pour l'Approche B, vous les appliquez manuellement, ce qui est couvert ci-dessous.

### 3. Approche B uniquement : installer l'infrastructure à la main

L'Approche B n'a pas de Flux, mais les manifestes de contrôleurs sont écrits comme des ressources HelmRelease de Flux. L'approche la plus simple qui évite de les réécrire est d'installer uniquement les contrôleurs de Flux afin qu'ils puissent traiter les HelmReleases :

```bash
flux install
kubectl apply -k infrastructure/controllers
```
> attendez que les helm releases soient prêtes, puis :

```bash
kubectl apply -k infrastructure/configs
```

### 4. Redirection de ports

La redirection de ports signifie ouvrir un port vers internet. Assurez-vous de bien comprendre ce que cela implique et renseignez-vous sur les façons de le faire en toute sécurité sans compromettre votre serveur.

Sur votre routeur, redirigez les ports externes 80 (HTTP) et 443 (SSL/TLS) vers l'IP que vous avez configurée (par exemple `192.168.0.50`). Le service ingress-nginx réclame cette IP en tant que LoadBalancer. Vérifiez avec :

```bash
kubectl get svc -n ingress-nginx
```

La colonne `EXTERNAL-IP` devrait afficher votre adresse IP.

### 5. Certificats TLS

Vérifiez vos certificats avec :

```bash
kubectl get certificate -A
```

## Configuration de Gitea

Gitea est installé par la couche infrastructure. Ces étapes le configurent une fois qu'il tourne.

### 1. Créer le secret admin

Le chart Helm de Gitea lit ses identifiants admin depuis un Secret existant nommé `gitea-admin-secret` :

```bash
kubectl create secret generic gitea-admin-secret -n gitea \
  --from-literal=username=gitea_admin \
  --from-literal=password=CHOOSE_A_STRONG_PASSWORD
```

Créez-le avant (ou juste après) l'installation du HelmRelease Gitea. Si Gitea a démarré sans lui, redémarrez la release après l'avoir créé.

### 2. Se connecter et créer l'organisation

Ouvrez `https://<your-gitea-hostname>` dans un navigateur, connectez-vous avec le compte admin, et créez une organisation correspondant aux chemins du registre dans les manifestes (par défaut : `stackiq-team`). Créez les dépôts pour le code source de l'application.

### 3. Identifiants du registre pour le cluster

Chaque Deployment récupère les images depuis le registre Gitea via un secret d'accès (image pull secret) nommé `gitea-registry` dans le namespace `stackiq`. Créez-le :

```bash
kubectl create namespace stackiq
kubectl create secret docker-registry gitea-registry -n stackiq \
  --docker-server=<your-gitea-hostname> \
  --docker-username=gitea_admin \
  --docker-password=YOUR_GITEA_PASSWORD
```

L'Approche A a aussi besoin d'une copie pour que l'automatisation d'images de Flux puisse scanner le registre, nommée `gitea-registry-flux` dans le namespace `flux-system` :

```bash
kubectl create secret docker-registry gitea-registry-flux -n flux-system \
  --docker-server=<your-gitea-hostname> \
  --docker-username=gitea_admin \
  --docker-password=YOUR_GITEA_PASSWORD
```

### 4. Construire et pousser les images

Connectez-vous au registre depuis votre poste de travail :

```bash
docker login <your-gitea-hostname>
```

Approche B (manuel, ponctuel) : construisez et poussez chaque service depuis le dépôt de l'application :

```bash
docker build -t <your-gitea-hostname>/stackiq-team/backend:1.0.0 ./backend
docker push <your-gitea-hostname>/stackiq-team/backend:1.0.0
docker build -t <your-gitea-hostname>/stackiq-team/frontend:1.0.0 ./frontend
docker push <your-gitea-hostname>/stackiq-team/frontend:1.0.0
docker build -t <your-gitea-hostname>/stackiq-team/worker:1.0.0 ./worker
docker push <your-gitea-hostname>/stackiq-team/worker:1.0.0
```

Définissez ensuite les champs `image:` dans `apps/stackiq/*-deployment.yaml` avec les tags que vous avez poussés.

Approche A (automatique) : le dépôt de l'application contient un workflow Gitea Actions (`.gitea/workflows/build.yaml`) qui construit et pousse les trois images avec une version auto-incrémentée à chaque push. Pour qu'il fonctionne, vous avez besoin du runner Actions :

1. Dans l'interface web de Gitea, allez dans Site Administration, puis Actions, puis Runners, et copiez un token d'enregistrement.
2. Stockez-le dans le secret attendu par le Deployment du runner :

   ```bash
   kubectl create secret generic gitea-runner-secret -n gitea-runner \
     --from-literal=token=YOUR_REGISTRATION_TOKEN
   ```

3. Le Deployment du runner dans `infrastructure/controllers/gitea-runner.yaml` le récupère et s'enregistre lui-même. Vérifiez que le runner apparaît en ligne dans l'interface admin de Gitea.

## Configuration de Kubernetes et Flux

### Approche A : bootstrap de Flux

Le bootstrap de Flux connecte votre cluster à votre fork et installe ses propres contrôleurs. Avec un fork GitHub :

```bash
flux bootstrap github \
  --owner=YOUR_GITHUB_USER_OR_ORG \
  --repo=stackiq-infra \
  --branch=main \
  --path=clusters/home \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```

Notes :

- `--path=clusters/home` est important. Ce dossier contient les Kustomizations qui importent tout le reste.
- `--components-extra` installe les deux contrôleurs nécessaires à l'automatisation d'images. Sans eux, les mises à jour automatiques d'images ne fonctionneront pas.
- Le bootstrap demandera un personal access token GitHub avec les permissions repo. Flux l'utilise pour committer ses propres manifestes et, plus tard, les mises à jour de version d'images.

Après le bootstrap, Flux réconcilie automatiquement dans cet ordre :

1. `infrastructure-controllers` (ingress, cert-manager, MetalLB, Gitea, runner)
2. `infrastructure-configs` (ClusterIssuer, pool MetalLB), qui attend l'étape 1
3. `apps` (l'application StackIQ), qui attend aussi l'étape 1

Une fois que tout est à l'état Ready, l'application est en ligne à votre nom d'hôte d'app.

Comment fonctionne la boucle d'automatisation d'images pour cette approche :

1. Vous poussez du code vers le dépôt de l'application dans Gitea.
2. Le workflow Actions construit des images taguées `1.0.<numéro de run>` et les pousse vers le registre.
3. Les ressources ImageRepository de Flux scannent le registre chaque minute.
4. L'ImagePolicy sélectionne le tag serveur le plus élevé.
5. L'ImageUpdateAutomation committe le nouveau tag dans `apps/stackiq/*-deployment.yaml` de ce dépôt (les commentaires `# {"$imagepolicy": ...}` marquent les lignes qu'elle modifie).
6. Flux applique le changement au cluster.

### Approche B : appliquer les manifestes de l'app une fois

Une fois l'infrastructure et les images en place :

```bash
kubectl apply -k apps/stackiq
```

## Secrets d'environnement

L'application lit sa configuration depuis un seul Secret Kubernetes nommé `stackiq-secrets` dans le namespace `stackiq`. Les Deployments mappent des clés individuelles de ce secret vers des variables d'environnement. Le Secret n'est pas stocké dans git ; vous le créez directement dans le cluster.

### Création initiale

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

Notes :

- La clé OPENAI n'est pas nécessaire pour faire tourner l'application.
- Les mots de passe d'application Gmail contiennent des espaces, donc cette entrée doit être entourée de guillemets dans son ensemble.
- Sous PowerShell, remplacez les continuations de ligne `\` par des backticks et utilisez des guillemets doubles autour de toute valeur contenant des espaces.
- Le token GitHub a besoin du scope de lecture des repos publics. Les identifiants Gmail sont utilisés pour envoyer les emails de résultat et sont optionnels si vous n'avez pas besoin des emails.

## Commandes kubectl courantes

Voir ce qui tourne :

```bash
kubectl get pods -n stackiq                # liste les pods et leur statut
kubectl get deployments -n stackiq         # liste les deployments et le nombre de replicas
kubectl get svc -n stackiq                 # liste les services
kubectl get pods -A                        # tout, dans tous les namespaces
```

Lire les logs :

```bash
kubectl logs deployment/backend -n stackiq --tail=100   # les 100 dernières lignes
kubectl logs -f deployment/worker -n stackiq            # suivre en direct
kubectl logs <pod-name> -n stackiq --previous           # logs d'avant le dernier crash
```

Exécuter des commandes dans un pod :

```bash
kubectl exec -it deployment/backend -n stackiq -- sh                            # shell interactif
kubectl exec -it deployment/backend -n stackiq -- sh -c 'echo $DATABASE_URL'
```

Sous PowerShell, utilisez des guillemets simples autour de la commande interne comme montré ci-dessus. Les guillemets doubles font que PowerShell substitue les `$VARIABLES` sur votre poste de travail avant même que la commande n'atteigne le pod, ce qui produit silencieusement des valeurs vides.

Redémarrer et scaler :

```bash
kubectl rollout restart deployment/backend -n stackiq   # remplace les pods par des neufs
kubectl rollout restart deployment -n stackiq           # redémarre tous les deployments du namespace
kubectl scale deployment/worker -n stackiq --replicas=0 # arrête un deployment
kubectl scale deployment/worker -n stackiq --replicas=1 # le redémarre
```

Note pour l'Approche A : Flux impose ce qui est dans git. Si vous scalez ou éditez à la main une ressource gérée par Flux, Flux l'annulera à la prochaine réconciliation. Les changements manuels sont acceptables pour du debug rapide, mais les changements permanents doivent être faits dans le dépôt.

Inspecter les détails :

```bash
kubectl describe pod <pod-name> -n stackiq       # événements, raisons de redémarrage, pulls d'images
kubectl get deployment backend -n stackiq -o yaml
kubectl get secret stackiq-secrets -n stackiq -o jsonpath="{.data}"   # clés, valeurs en base64
```

Accès à la base de données :

```bash
kubectl exec -it deployment/postgres -n stackiq -- psql -U postgres -d stackiq
```

Réinitialiser toutes les données de l'application (destructif, conserve le schéma en rejouant les migrations) :

```bash
kubectl exec -it deployment/backend -n stackiq -- sh -c "npx prisma migrate reset --force"
kubectl exec -it deployment/redis -n stackiq -- redis-cli FLUSHALL
kubectl rollout restart deployment/backend -n stackiq
kubectl rollout restart deployment/worker -n stackiq
```

Statut de Flux (Approche A) :

```bash
flux get kustomizations                                 # statut de synchronisation de chaque couche
flux reconcile kustomization apps -n flux-system        # force une synchronisation immédiate
kubectl get gitrepository -n flux-system                # Flux récupère-t-il correctement le dépôt
kubectl get kustomization apps -n flux-system -o yaml   # statut complet, conditions, révision appliquée
```