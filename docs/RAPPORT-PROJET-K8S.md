# Projet Kubernetes — 5SDWANA
## POC Kubernetes pour un client e-commerce

**Date** : 25/03/2026  
**Équipe** : Clément GRECO & Vincent  
**Convention de nommage** : `cv-25-03-2026`  
**Repository** : [https://github.com/ClementGRC/projet-k8s-CV-25-03-2026](https://github.com/ClementGRC/projet-k8s-CV-25-03-2026)

---

## Table des matières

1. [Contexte du projet](#1-contexte-du-projet)
2. [Architecture globale](#2-architecture-globale)
3. [Choix techniques et justifications](#3-choix-techniques-et-justifications)
4. [Étape 1 — Initialisation du projet et Git](#4-étape-1--initialisation-du-projet-et-git)
5. [Étape 2 — Installation et configuration de Minikube](#5-étape-2--installation-et-configuration-de-minikube)
6. [Étape 3 — Création des namespaces](#6-étape-3--création-des-namespaces)
7. [Étape 4 — Analyse et optimisation des applications](#7-étape-4--analyse-et-optimisation-des-applications)
8. [Étape 5 — Build des images Docker optimisées](#8-étape-5--build-des-images-docker-optimisées)
9. [Étape 6 — Déploiement en environnement dev](#9-étape-6--déploiement-en-environnement-dev)
10. [Étape 7 — Déploiement en environnement preprod](#10-étape-7--déploiement-en-environnement-preprod)
11. [Étape 8 — Déploiement en environnement prod](#11-étape-8--déploiement-en-environnement-prod)
12. [Gestion des secrets et sécurité](#12-gestion-des-secrets-et-sécurité)
13. [Avantages et inconvénients des solutions choisies](#13-avantages-et-inconvénients-des-solutions-choisies)
14. [Conclusion](#14-conclusion)

---

## 1. Contexte du projet

Dans le cadre de la formation 5SDWANA, nous avons intégré le rôle d'une équipe SRE dans un grand groupe français. Le scrum master nous demande de déployer 3 applications conteneurisées pour un nouveau client e-commerce qui doit gérer plusieurs milliers de clients connectés en simultané.

Le client souhaite un POC basé sur Kubernetes. L'une des 3 applications doit obligatoirement intégrer la passerelle de paiement Stripe pour réaliser des tests de paiement.

**Contraintes du sujet :**

- 3 applications conteneurisées, dont une obligatoirement avec Stripe
- Applications les plus légères possible
- 3 réplicas par application
- Applications accessibles depuis l'extérieur
- Images disponibles depuis un registry (ou en local via Minikube)
- 3 environnements : dev, preprod, prod
- Autoscaling (HPA) sur preprod et prod
- Système de mise à jour (rollout) et de retour en arrière (rollback)
- Tout déployé de façon "as code" via des manifests YAML
- Repo Git avec commits réguliers et messages explicites

---

## 2. Architecture globale

```
                        ┌─────────────────────────────────────────────┐
                        │            Cluster Minikube                 │
                        │                                             │
                        │  ┌───────────┐ ┌───────────┐ ┌───────────┐ │
                        │  │ Namespace │ │ Namespace │ │ Namespace │ │
                        │  │    dev    │ │  preprod  │ │   prod    │ │
                        │  │           │ │           │ │           │ │
                        │  │ ┌───────┐ │ │ ┌───────┐ │ │ ┌───────┐ │ │
                        │  │ │Ecomm. │ │ │ │Ecomm. │ │ │ │Ecomm. │ │ │
                        │  │ │x3 rep.│ │ │ │x3 rep.│ │ │ │x3 rep.│ │ │
                        │  │ │+Stripe│ │ │ │+Stripe│ │ │ │+Stripe│ │ │
                        │  │ └───────┘ │ │ │+ HPA  │ │ │ │+ HPA  │ │ │
                        │  │ ┌───────┐ │ │ └───────┘ │ │ └───────┘ │ │
                        │  │ │Django │ │ │ ┌───────┐ │ │ ┌───────┐ │ │
                        │  │ │Pro    │ │ │ │Django │ │ │ │Django │ │ │
                        │  │ │x3 rep.│ │ │ │Pro    │ │ │ │Pro    │ │ │
                        │  │ └───────┘ │ │ │x3+HPA│ │ │ │x3+HPA│ │ │
                        │  │ ┌───────┐ │ │ └───────┘ │ │ └───────┘ │ │
                        │  │ │Dashb. │ │ │ ┌───────┐ │ │ ┌───────┐ │ │
                        │  │ │x3 rep.│ │ │ │Dashb. │ │ │ │Dashb. │ │ │
                        │  │ └───────┘ │ │ │x3+HPA│ │ │ │x3+HPA│ │ │
                        │  │           │ │ └───────┘ │ │ └───────┘ │ │
                        │  │ NodePort  │ │ LoadBal.  │ │ LoadBal.  │ │
                        │  └───────────┘ └───────────┘ └───────────┘ │
                        │                                             │
                        │  27 pods total (3 apps × 3 replicas × 3 envs)│
                        └─────────────────────────────────────────────┘
```

**Par application, chaque environnement contient :**

- 1 ConfigMap (variables d'environnement non sensibles)
- 1 Secret (clés Stripe, secrets applicatifs)
- 1 Deployment (avec 3 réplicas)
- 1 Service (NodePort en dev, LoadBalancer en preprod/prod)
- 1 HPA (preprod et prod uniquement, min 3 → max 10 pods, seuil CPU 50%)

---

## 3. Choix techniques et justifications

### 3.1 Orchestrateur : Minikube

Minikube a été choisi comme solution locale pour ce POC car il permet de simuler un cluster Kubernetes complet sur un seul nœud. Il est léger, facile à installer, et supporte tous les composants k8s nécessaires (Deployments, Services, HPA, Namespaces, Secrets, ConfigMaps).

**Driver utilisé** : Docker. C'est le driver recommandé pour les environnements Linux virtualisés.

### 3.2 Applications

Les 3 applications proviennent des templates AppSeed fournis par le formateur :

| Application | Framework | Port | Rôle | Image Docker |
|---|---|---|---|---|
| Rocket Ecommerce | Django + Stripe | 5005 | E-commerce avec paiement | `ecommerce-stripe-cv:1.0` |
| Rocket Django Pro | Django | 5005 | Application secondaire | `django-pro-cv:1.0` |
| Soft UI Dashboard | Django | 5005 | Dashboard d'administration | `dashboard-cv:1.0` |

### 3.3 Optimisation des images Docker

Le sujet exigeant des applications "les plus légères possible", nous avons appliqué plusieurs optimisations :

| Optimisation | Description | Impact |
|---|---|---|
| Multi-stage build | Stage 1 (Node.js) compile le frontend, Stage 2 (Python slim) ne contient que le runtime | Suppression de Node.js/npm de l'image finale |
| `python:3.11-slim` | Image de base minimaliste (~150 Mo vs ~900 Mo pour l'image full) | Réduction de ~80% de la taille de base |
| `.dockerignore` | Exclusion de `.git`, `node_modules`, `__pycache__`, `.env`, etc. | Réduction du build context |
| `--no-cache-dir` pip | Pas de cache pip stocké dans l'image | Réduction de ~50 Mo par image |
| Layers combinées | `RUN` multiples regroupés avec `&&` | Réduction du nombre de layers |

**Résultats des optimisations :**

| Image | Taille optimisée | Taille estimée sans optimisation |
|---|---|---|
| `ecommerce-stripe-cv:1.0` | **256 Mo** | ~1.5 Go |
| `django-pro-cv:1.0` | **428 Mo** | ~1.5 Go |
| `dashboard-cv:1.0` | **284 Mo** | ~1.2 Go |

![Images Docker buildées avec tailles](screenshots/08-images-built-sizes.png)

### 3.4 Stratégie de Services par environnement

| Environnement | Type de Service | Justification |
|---|---|---|
| Dev | NodePort | Simple, suffisant pour les tests locaux. Port aléatoire entre 30000-32767 |
| Preprod | LoadBalancer | Simule la production. Équilibrage de charge entre les nodes |
| Prod | LoadBalancer | Standard en production pour l'accès externe via IP dédiée |

### 3.5 Gestion des secrets

Les secrets (clés Stripe) sont gérés via des objets Kubernetes `Secret` avec le type `Opaque`. Les fichiers de secrets sont exclus du dépôt Git via `.gitignore` pour ne jamais exposer de données sensibles dans le code source.

---

## 4. Étape 1 — Initialisation du projet et Git

### 4.1 Création de la structure du projet

Le repo Git a été initialisé dès le début du projet pour assurer la traçabilité de chaque action.

```bash
mkdir projet-k8s-CV-25-03-2026
cd projet-k8s-CV-25-03-2026
git init
```

**Structure de dossiers créée :**

```
projet-k8s-CV-25-03-2026/
├── README.md
├── .gitignore
├── apps/
│   ├── priv-rocket-ecommerce-main/     # App Stripe (obligatoire)
│   ├── priv-rocket-django-pro-main/     # App Django
│   └── priv-django-soft-ui-dashboard-pro-master/  # Dashboard
├── manifests/
│   ├── namespaces.yaml
│   ├── dev/        # Manifests environnement dev
│   ├── preprod/    # Manifests environnement preprod
│   └── prod/       # Manifests environnement prod
├── infra/          # Infrastructure as code
├── docs/           # Rapport et schémas
└── logs/           # Historiques (shell, docker, k8s events)
```

### 4.2 Configuration Git et premier commit

```bash
git config --global user.name "ClementGRC"
git config --global user.email "greco.clement57@gmail.com"
git add .
git commit -m "init: creation structure projet k8s - equipe CV"
```

![Premier commit réussi](screenshots/01-premier-commit.png)

### 4.3 Push vers GitHub

Le repo a été connecté à GitHub pour le versionning distant :

```bash
git remote add origin https://github.com/ClementGRC/projet-k8s-CV-25-03-2026.git
git branch -M main
git push -u origin main
```

![Push GitHub réussi](screenshots/03-github-push.png)

---

## 5. Étape 2 — Installation et configuration de Minikube

### 5.1 Vérification des prérequis

```bash
minikube version   # v1.38.1
docker --version   # Docker 29.3.0
```

### 5.2 Démarrage du cluster

Le cluster a été démarré avec le driver Docker et une allocation mémoire adaptée à la VM :

```bash
minikube start --driver=docker --memory=6144 --cpus=2
```

**Note** : La mémoire a été augmentée à 6 Go (et la VM à 8 Go de RAM) pour supporter les 27 pods (3 apps × 3 réplicas × 3 environnements) + le système Kubernetes.

![Minikube démarré avec node Ready](screenshots/02-minikube-start.png)

### 5.3 Activation du metrics-server

Le metrics-server est nécessaire pour le fonctionnement du HPA (autoscaling). Il collecte les métriques CPU et mémoire des pods :

```bash
minikube addons enable metrics-server
```

---

## 6. Étape 3 — Création des namespaces

Les namespaces permettent d'isoler les 3 environnements (dev, preprod, prod) au sein du même cluster. Chaque environnement a ses propres ressources, secrets et configurations.

**Fichier `manifests/namespaces.yaml` :**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    env: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: preprod
  labels:
    env: preprod
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    env: prod
```

```bash
kubectl apply -f manifests/namespaces.yaml
```

![Namespaces créés](screenshots/04-namespaces-created.png)

Les labels `env` permettent de filtrer facilement les ressources par environnement : `kubectl get all -l env=dev`.

---

## 7. Étape 4 — Analyse et optimisation des applications

### 7.1 Extraction des applications

Les 3 applications fournies par le formateur ont été extraites dans le dossier `apps/` :

- `priv-rocket-ecommerce-main/` — Application e-commerce avec Stripe
- `priv-rocket-django-pro-main/` — Application Django Pro
- `priv-django-soft-ui-dashboard-pro-master/` — Dashboard Django

Chaque application contenait déjà un Dockerfile et un `requirements.txt`.

![Dockerfiles et requirements trouvés](screenshots/05-dockerfiles-found.png)

### 7.2 Analyse des ports

Les 3 applications utilisent toutes le port **5005** via Gunicorn :

![Ports des applications](screenshots/06-ports-apps.png)

### 7.3 Analyse des dépendances (exemple : ecommerce)

L'application e-commerce utilise Django, Stripe, Celery, Redis, et d'autres dépendances Python :

![Requirements ecommerce](screenshots/07-requirements-ecommerce.png)

### 7.4 Création des `.dockerignore`

Un `.dockerignore` a été ajouté à chaque application pour exclure les fichiers inutiles du build context :

```
.git
.gitignore
__pycache__
*.pyc
*.pyo
node_modules
.env
.env.*
*.md
LICENSE
.vscode
.idea
*.log
docker-compose*
.dockerignore
```

### 7.5 Réécriture des Dockerfiles optimisés

**Ecommerce et Django Pro (multi-stage build) :**

```dockerfile
# ---- Stage 1 : Build des assets frontend ----
FROM node:18-slim AS frontend

WORKDIR /app
COPY package*.json ./
RUN npm i
COPY . .
RUN npm run build

# ---- Stage 2 : Image de production légère ----
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY . .
COPY --from=frontend /app/static /app/static

RUN python manage.py collectstatic --no-input && \
    python manage.py makemigrations && \
    python manage.py migrate

EXPOSE 5005
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

**Dashboard (pas de Node.js, image simple) :**

```dockerfile
FROM python:3.10-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY . .

RUN python manage.py collectstatic --no-input && \
    python manage.py makemigrations && \
    python manage.py migrate

EXPOSE 5005
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

**Justification du multi-stage build** : Les applications ecommerce et Django Pro nécessitent Node.js pour compiler les assets frontend (webpack). Avec un multi-stage build, le Stage 1 utilise `node:18-slim` pour le build, et le Stage 2 ne copie que les fichiers statiques compilés. Résultat : Node.js et npm ne sont pas présents dans l'image finale, ce qui réduit considérablement la taille.

---

## 8. Étape 5 — Build des images Docker optimisées

### 8.1 Connexion au Docker daemon de Minikube

Pour éviter l'utilisation d'un registry externe (Docker Hub), nous avons buildé les images directement dans le Docker daemon interne de Minikube :

```bash
eval $(minikube docker-env)
```

Cette commande redirige le client Docker local vers le daemon Docker qui tourne à l'intérieur de Minikube. Les images buildées sont immédiatement disponibles pour Kubernetes sans push/pull.

### 8.2 Build des 3 images

```bash
cd apps/
docker build -t ecommerce-stripe-cv:1.0 priv-rocket-ecommerce-main/
docker build -t django-pro-cv:1.0 priv-rocket-django-pro-main/
docker build -t dashboard-cv:1.0 priv-django-soft-ui-dashboard-pro-master/
```

![Build de l'app ecommerce réussi](screenshots/09-build-ecommerce-success.png)

### 8.3 Vérification des tailles d'images

```bash
docker images | grep cv
```

![Tailles des 3 images optimisées](screenshots/08-images-built-sizes.png)

**Résultat** : les 3 images totalisent ~968 Mo au lieu de ~4.2 Go estimés sans optimisation, soit une **réduction de ~77%**.

---

## 9. Étape 6 — Déploiement en environnement dev

### 9.1 Structure des manifests par application

Chaque application est déployée avec un triplet de manifests :

- **ConfigMap** : variables d'environnement non sensibles (`DEBUG`, `FLASK_APP`, etc.)
- **Secret** : données sensibles (clés Stripe)
- **Deployment + Service** : configuration du déploiement et de l'exposition réseau

### 9.2 Exemple : manifest de l'application e-commerce

**ConfigMap** (`manifests/dev/ecommerce-configmap-cv.yaml`) :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecommerce-configmap-cv
  namespace: dev
  labels:
    app: ecommerce
    env: dev
data:
  DEBUG: "True"
  FLASK_APP: "run.py"
  FLASK_DEBUG: "True"
  SERVER_ADDRESS: "http://localhost:5005/"
```

**Deployment + Service** (`manifests/dev/ecommerce-app-cv.yaml`) :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-deployment-cv
  namespace: dev
  labels:
    app: ecommerce
    env: dev
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecommerce
  template:
    metadata:
      labels:
        app: ecommerce
        env: dev
        tier: frontend
    spec:
      containers:
        - name: ecommerce-stripe
          image: ecommerce-stripe-cv:1.0
          imagePullPolicy: Never
          ports:
            - containerPort: 5005
          envFrom:
            - configMapRef:
                name: ecommerce-configmap-cv
            - secretRef:
                name: ecommerce-secret-cv
          resources:
            requests:
              memory: "128Mi"
              cpu: "50m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-service-cv
  namespace: dev
  labels:
    app: ecommerce
    env: dev
spec:
  type: NodePort
  selector:
    app: ecommerce
  ports:
    - port: 5005
      targetPort: 5005
      protocol: TCP
      name: ecommerce-np
```

**Points importants :**

- `imagePullPolicy: Never` : les images sont dans le Docker local de Minikube, pas besoin de registry
- `envFrom` : injecte toutes les variables du ConfigMap et du Secret dans les conteneurs automatiquement
- `resources.requests` : ressources minimales demandées au scheduler pour placer le pod
- `resources.limits` : plafond que le pod ne peut pas dépasser
- `replicas: 3` : conformément à l'exigence du sujet

### 9.3 Déploiement et vérification

```bash
kubectl apply -f manifests/dev/
kubectl get pods -n dev
```

![9 pods Running en dev](screenshots/10-9pods-dev-running.png)

**Résultat** : 9 pods (3 apps × 3 réplicas), tous en état Running, 0 restarts.

### 9.4 Test d'accès à l'application e-commerce

L'application est accessible depuis le navigateur via port-forwarding :

```bash
kubectl port-forward -n dev service/ecommerce-service-cv 5005:5005 --address 0.0.0.0
```

![Application e-commerce accessible dans le navigateur](screenshots/11-app-ecommerce-navigateur.png)

L'application Rocket Ecommerce est fonctionnelle et accessible depuis le PC hôte via l'IP de la VM.

---

## 10. Étape 7 — Déploiement en environnement preprod

### 10.1 Différences avec l'environnement dev

| Paramètre | Dev | Preprod |
|---|---|---|
| `DEBUG` | `True` | `False` |
| Type de Service | NodePort | LoadBalancer |
| Autoscaling (HPA) | Non | Oui (min 3, max 10) |
| Seuil CPU HPA | — | 50% |

### 10.2 HorizontalPodAutoscaler (HPA)

Le HPA est ajouté sur chaque application en preprod. Il surveille la consommation CPU et scale automatiquement le nombre de pods :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-hpa-cv
  namespace: preprod
  labels:
    app: ecommerce
    env: preprod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-deployment-cv
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
```

**Explication** :
- `minReplicas: 3` / `maxReplicas: 10` : le HPA maintient au minimum 3 pods et peut monter jusqu'à 10
- `averageUtilization: 50` : quand la moyenne CPU des pods dépasse 50%, le HPA ajoute des pods
- `stabilizationWindowSeconds: 60` : attend 60 secondes avant de scale down, pour éviter le "yo-yo" lors des pics de trafic

### 10.3 Déploiement et vérification

```bash
kubectl apply -f manifests/preprod/
kubectl get pods -n preprod
kubectl get hpa -n preprod
```

![Preprod — 9 pods Running + 3 HPA actifs](screenshots/12-preprod-pods-hpa.png)

**Résultat** : 9 pods Running en preprod, 3 HPA configurés avec target 50% CPU.

---

## 11. Étape 8 — Déploiement en environnement prod

### 11.1 Stratégie

L'environnement prod est identique à la preprod (LoadBalancer + HPA). Les manifests preprod ont été dupliqués et adaptés :

```bash
cp manifests/preprod/*.yaml manifests/prod/
sed -i 's/preprod/prod/g' manifests/prod/*.yaml
kubectl apply -f manifests/prod/
```

### 11.2 Gestion des ressources

Avec 27 pods déployés sur un seul nœud Minikube, le dimensionnement des ressources a été un enjeu important. Le CPU requests a été optimisé à 50m par pod pour tenir dans les 4 CPU alloués au cluster :

![Allocation des ressources du node](screenshots/16-resource-allocation.png)

### 11.3 Vérification finale — 27 pods Running

```bash
kubectl get pods -A | grep -E "dev|preprod|prod"
```

![Tous les pods Running sur les 3 environnements](screenshots/13-all-27-pods-running.png)

**Résultat final** : 27 pods (3 apps × 3 réplicas × 3 environnements) tous en état Running.

---

## 12. Gestion des secrets et sécurité

### 12.1 Secrets Kubernetes

Les clés Stripe (secret key et publishable key) sont injectées via des objets `Secret` Kubernetes avec le type `Opaque`. Le champ `stringData` permet de les écrire en clair dans le manifest, et Kubernetes les encode automatiquement en base64.

### 12.2 Protection du code source

Lors du push initial sur GitHub, le système **GitHub Push Protection** a détecté une clé Stripe API dans un fichier `.env` et a bloqué le push :

![GitHub Push Protection](screenshots/14-github-push-protection.png)

Pour résoudre ce problème :

1. Ajout d'un `.gitignore` excluant les fichiers `.env` et les manifests secrets (`*-secret-*.yaml`)
2. Nettoyage de l'historique Git avec `git filter-branch` pour supprimer les secrets des anciens commits
3. Push forcé pour réécrire l'historique distant

![Push réussi après nettoyage](screenshots/15-push-force-success.png)

**Leçon apprise** : ne jamais commiter de fichiers contenant des secrets dans Git, même dans un repo privé. Les secrets doivent être gérés via des mécanismes dédiés (Kubernetes Secrets, gestionnaires de secrets comme Infisical/Vault).

---

## 13. Avantages et inconvénients des solutions choisies

### Avantages

| Solution | Avantage |
|---|---|
| Minikube | Simple à installer, simule un vrai cluster k8s, gratuit |
| Docker local (eval minikube docker-env) | Pas besoin de registry externe, build rapide, images immédiatement disponibles |
| Multi-stage build | Images 3 à 5 fois plus légères, pas de dépendances de build en production |
| python:slim | Image de base ~6x plus petite que l'image full |
| Namespaces | Isolation propre des environnements, filtrage par labels |
| HPA | Scaling automatique, absorbe les pics de trafic sans intervention manuelle |
| ConfigMap + Secret | Séparation claire entre config, secrets et code applicatif |
| Infrastructure as code (YAML) | Tout est reproductible, versionné, déclaratif |

### Inconvénients

| Solution | Inconvénient |
|---|---|
| Minikube | Single node, pas de haute disponibilité, limité en ressources |
| Docker local sans registry | Images non partagées entre machines, perdues si `minikube delete` |
| python:slim | Certains packages Python nécessitent des dépendances système manquantes |
| NodePort en dev | Port aléatoire, pas d'équilibrage de charge |
| LoadBalancer sur Minikube | IP en `<pending>`, nécessite `minikube tunnel` ou port-forward |
| Secrets en YAML | Encodage base64 ≠ chiffrement, stockés en clair dans etcd par défaut |

### Alternatives envisageables

| Aspect | Solution actuelle | Alternative possible |
|---|---|---|
| Cluster | Minikube (single node) | Kind, K3S, ou cloud managé (GKE, EKS, AKS) |
| Registry | Docker local | Docker Hub, GitHub Container Registry, Harbor |
| Templating | YAML dupliqués | Helm Charts ou Kustomize |
| Secrets | Kubernetes Secrets | Infisical, HashiCorp Vault, Sealed Secrets |
| CI/CD | Manuel | GitOps (ArgoCD, FluxCD) |

---

## 14. Conclusion

Ce projet nous a permis de mettre en pratique l'ensemble de la chaîne de déploiement Kubernetes :

- **Conteneurisation** : création et optimisation d'images Docker avec multi-stage builds
- **Orchestration** : déploiement de 27 pods répartis sur 3 environnements avec gestion des réplicas
- **Scaling** : mise en place du HPA pour l'autoscaling automatique en preprod et prod
- **Sécurité** : gestion des secrets séparée du code source, protection GitHub Push Protection
- **Infrastructure as code** : tous les déploiements sont déclaratifs et reproductibles via des manifests YAML versionnés dans Git

Le POC démontre qu'un cluster Kubernetes peut gérer efficacement plusieurs applications e-commerce conteneurisées avec mise à l'échelle automatique, tout en maintenant une séparation propre entre les environnements de développement, pré-production et production.

**Reste à réaliser** :
- Tests de paiement Stripe de bout en bout
- Rollout (mise à jour) et rollback (retour en arrière)
- Infrastructure as code avec Helm ou Kustomize (Partie 2)
- Gestionnaire de secrets Infisical (bonus)
- Collecte des logs (historiques shell, docker logs, k8s events)
