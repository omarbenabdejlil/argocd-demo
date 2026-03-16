# ArgoCD GitOps Lab — Guide Complet

## Architecture

```
git push → Gitea (in-cluster) → ArgoCD (auto-sync) → Kubernetes kubeadm
```

> **Note** : GitHub étant bloqué par le réseau de l'organisation, on utilise **Gitea** déployé dans le cluster comme serveur Git interne. Le workflow GitOps reste identique.

---

## Prérequis

- Cluster Kubernetes kubeadm fonctionnel
- `kubectl` configuré
- `helm` installé
- Accès admin au cluster

---

## Partie 1 — Installation ArgoCD

### 1.1 Déployer ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Attendre que tous les pods soient Running
kubectl get pods -n argocd -w
```

### 1.2 Accéder à l'UI

```bash
# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Récupérer le mot de passe admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Ouvrir **https://localhost:8080** — Login : `admin` / mot de passe récupéré.

### 1.3 Installer le CLI ArgoCD

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

argocd login localhost:8080 --username admin --password <MOT_DE_PASSE> --insecure
```

---

## Partie 2 — Gitea (serveur Git in-cluster)

### 2.1 Déployer Gitea

```bash
kubectl create namespace gitea

kubectl apply -n gitea -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      containers:
        - name: gitea
          image: gitea/gitea:1.22-rootless
          ports:
            - containerPort: 3000
            - containerPort: 22
          env:
            - name: GITEA__server__ROOT_URL
              value: "http://gitea.gitea.svc.cluster.local:3000"
---
apiVersion: v1
kind: Service
metadata:
  name: gitea
spec:
  selector:
    app: gitea
  ports:
    - name: http
      port: 3000
      targetPort: 3000
    - name: ssh
      port: 22
      targetPort: 22
EOF

kubectl -n gitea get pods -w
```

### 2.2 Accéder à Gitea

```bash
kubectl port-forward svc/gitea -n gitea 3000:3000 &
```

Ouvrir **http://localhost:3000**, créer un compte utilisateur.

### 2.3 Créer un repo via l'API

```bash
curl -X POST http://localhost:3000/api/v1/user/repos \
  -u <USER>:<PASSWORD> \
  -H "Content-Type: application/json" \
  -d '{"name": "nom-du-repo", "auto_init": false, "private": false}'
```

---

## Partie 3 — App basique (manifestes YAML)

### 3.1 Créer les manifestes

```bash
mkdir -p ~/argocd-demo/manifests
cd ~/argocd-demo
```

**manifests/namespace.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
```

**manifests/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: demo-app
  labels:
    app: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
```

**manifests/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  namespace: demo-app
spec:
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
```

### 3.2 Pousser sur Gitea

```bash
cd ~/argocd-demo

curl -X POST http://localhost:3000/api/v1/user/repos \
  -u <USER>:<PASSWORD> \
  -H "Content-Type: application/json" \
  -d '{"name": "argocd-demo", "auto_init": false, "private": false}'

git init
git add .
git commit -m "Initial ArgoCD demo manifests"
git branch -M main
git remote add origin http://localhost:3000/<USER>/argocd-demo.git
git push -u origin main
```

### 3.3 Créer l'Application ArgoCD

```bash
argocd app create demo-app \
  --repo http://gitea.gitea.svc.cluster.local:3000/<USER>/argocd-demo.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace demo-app \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### 3.4 Vérifier

```bash
argocd app get demo-app
kubectl get all -n demo-app
```

### 3.5 Tester le GitOps

```bash
cd ~/argocd-demo
sed -i 's/replicas: 2/replicas: 3/' manifests/deployment.yaml
git add .
git commit -m "Scale to 3 replicas"
git push

argocd app sync demo-app
kubectl get pods -n demo-app
```

### 3.6 Tester le self-heal

```bash
# Supprimer une ressource manuellement
kubectl delete configmap <nom> -n demo-app

# ArgoCD la recrée automatiquement en quelques secondes
kubectl get configmap -n demo-app
```

---

## Partie 4 — Helm Chart avec ArgoCD

### 4.1 Créer le chart

```bash
mkdir -p ~/argocd-helm-demo
cd ~/argocd-helm-demo
helm create nginx-chart
```

### 4.2 Nettoyer les fichiers inutiles

```bash
rm -rf nginx-chart/templates/tests
rm -f nginx-chart/templates/hpa.yaml
rm -f nginx-chart/templates/ingress.yaml
rm -f nginx-chart/templates/serviceaccount.yaml
rm -f nginx-chart/templates/NOTES.txt
rm -f nginx-chart/templates/httproute.yaml
```

> **Important** : bien supprimer `httproute.yaml` sinon `helm lint` échoue avec `nil pointer evaluating interface {}.enabled`.

### 4.3 values.yaml

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 80

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 128Mi

config:
  APP_ENV: "staging"
  APP_VERSION: "1.0.0"
  TEAM: "valentine-infra"
```

### 4.4 templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
    chart: {{ .Chart.Name }}-{{ .Chart.Version }}
    managed-by: argocd
    team: valentine-infra
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          envFrom:
            - configMapRef:
                name: {{ .Release.Name }}-config
```

### 4.5 templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
      protocol: TCP
```

### 4.6 templates/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  labels:
    app: {{ .Release.Name }}
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

### 4.7 Valider le chart

```bash
helm lint nginx-chart/
helm template my-app nginx-chart/
```

### 4.8 Pousser sur Gitea

```bash
cd ~/argocd-helm-demo

curl -X POST http://localhost:3000/api/v1/user/repos \
  -u <USER>:<PASSWORD> \
  -H "Content-Type: application/json" \
  -d '{"name": "argocd-helm-demo", "auto_init": false, "private": false}'

git init
git add .
git commit -m "Initial Helm chart for nginx"
git branch -M main
git remote add origin http://localhost:3000/<USER>/argocd-helm-demo.git
git push -u origin main
```

### 4.9 Créer les apps ArgoCD (staging + production)

On utilise `--helm-set` pour override les values par environnement.

> **Note** : ajouter `--sync-option CreateNamespace=true` pour que ArgoCD crée les namespaces automatiquement, sinon les apps restent en `OutOfSync` / `Missing`.

**Staging (2 replicas)** :

```bash
argocd app create nginx-staging \
  --repo http://gitea.gitea.svc.cluster.local:3000/<USER>/argocd-helm-demo.git \
  --path nginx-chart \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace staging \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --helm-set replicaCount=2 \
  --helm-set config.APP_ENV=staging

argocd app set nginx-staging --sync-option CreateNamespace=true
```

**Production (4 replicas)** :

```bash
argocd app create nginx-prod \
  --repo http://gitea.gitea.svc.cluster.local:3000/<USER>/argocd-helm-demo.git \
  --path nginx-chart \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --helm-set replicaCount=4 \
  --helm-set config.APP_ENV=production \
  --helm-set resources.requests.cpu=100m \
  --helm-set resources.requests.memory=128Mi

argocd app set nginx-prod --sync-option CreateNamespace=true
```

### 4.10 Vérifier

```bash
argocd app list
kubectl get pods -n staging
kubectl get pods -n production
```

Résultat attendu : 2 pods staging, 4 pods production, les deux apps **Synced** / **Healthy**.

---

## Partie 5 — Tests GitOps

### 5.1 Modifier le chart et push

```bash
cd ~/argocd-helm-demo
sed -i 's/tag: "1.27-alpine"/tag: "1.26-alpine"/' nginx-chart/values.yaml

git add .
git commit -m "Downgrade nginx to 1.26 for testing"
git push

argocd app sync nginx-staging
argocd app sync nginx-prod

# Vérifier l'image
kubectl get pods -n staging -o jsonpath='{.items[0].spec.containers[0].image}'
echo
```

### 5.2 Scaler via helm-set (sans toucher au repo)

```bash
argocd app set nginx-staging --helm-set replicaCount=5
argocd app sync nginx-staging
kubectl get pods -n staging
```

### 5.3 Self-heal (ArgoCD recrée les ressources supprimées)

```bash
kubectl delete configmap nginx-staging-config -n staging

# Attendre 15 sec
sleep 15
kubectl get configmap -n staging
```

La ConfigMap est recréée automatiquement par ArgoCD.

### 5.4 Rollback

```bash
# Voir l'historique
argocd app history nginx-staging

# Désactiver auto-sync (obligatoire pour rollback)
argocd app set nginx-staging --sync-policy none

# Rollback vers une version précédente
argocd app rollback nginx-staging <REVISION_ID>

# Vérifier
kubectl get pods -n staging -o jsonpath='{.items[0].spec.containers[0].image}'
echo

# Réactiver auto-sync
argocd app set nginx-staging --sync-policy automated --auto-prune --self-heal
```

> **Important** : le rollback est impossible tant que auto-sync est actif. Il faut d'abord le désactiver avec `--sync-policy none`, puis le réactiver après.

---

## Partie 6 — Commandes utiles

| Commande | Description |
|----------|------------|
| `argocd app list` | Lister toutes les apps |
| `argocd app get <app>` | Détails d'une app |
| `argocd app sync <app>` | Forcer un sync |
| `argocd app diff <app>` | Voir les diffs repo vs cluster |
| `argocd app history <app>` | Historique des déploiements |
| `argocd app rollback <app> <id>` | Rollback (auto-sync doit être off) |
| `argocd app set <app> --helm-set key=val` | Override une value Helm |
| `argocd app set <app> --sync-policy none` | Désactiver auto-sync |
| `argocd app set <app> --sync-policy automated` | Réactiver auto-sync |
| `argocd app set <app> --sync-option CreateNamespace=true` | Créer le namespace auto |
| `argocd app terminate-op <app>` | Annuler une opération en cours |
| `argocd app delete <app> --cascade` | Supprimer l'app + ressources |

---

## Cleanup

```bash
# Supprimer les apps ArgoCD
argocd app delete demo-app --cascade
argocd app delete nginx-staging --cascade
argocd app delete nginx-prod --cascade

# Supprimer Gitea
kubectl delete namespace gitea

# Supprimer ArgoCD
kubectl delete namespace argocd
```
