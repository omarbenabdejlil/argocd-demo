# ArgoCD + Keycloak SSO Integration — Guide Complet

## Architecture

```
Navigateur → localhost:8443 (ArgoCD)
          → ArgoCD redirige vers localhost:30090 (Keycloak login page)
          → User se connecte sur Keycloak
          → Keycloak redirige vers localhost:8443/auth/callback
          → ArgoCD valide le token via 192.168.65.3:30090 (NodePort interne)
          → Connecté avec les bons rôles (admin / readonly)
```

> **Contexte** : Cluster kubeadm sur Docker Desktop, GitHub bloqué par le réseau.
> Keycloak est exposé via NodePort (30090) accessible par le navigateur via `localhost:30090`
> et par les pods via l'IP du node `192.168.65.3:30090`.

---

## Prérequis

- Cluster Kubernetes kubeadm fonctionnel
- ArgoCD installé (voir guide ArgoCD GitOps Lab)
- `kubectl` et `helm` configurés
- Accès admin au cluster

---

## Partie 1 — Déployer Keycloak

### 1.1 Créer le namespace et déployer

```bash
kubectl create namespace keycloak

kubectl apply -n keycloak -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:26.0
          args: ["start-dev"]
          env:
            - name: KEYCLOAK_ADMIN
              value: "admin"
            - name: KEYCLOAK_ADMIN_PASSWORD
              value: "admin123"
            - name: KC_PROXY
              value: "edge"
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /realms/master
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
spec:
  selector:
    app: keycloak
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      nodePort: 30090
  type: NodePort
EOF

kubectl -n keycloak get pods -w
```

### 1.2 Vérifier l'accès

```bash
# Depuis le navigateur
# http://localhost:30090 → page admin Keycloak
# Login : admin / admin123

# Depuis les pods (vérifier le NodePort)
kubectl run test-kc --image=busybox --rm -it --restart=Never -- \
  wget -qO- http://192.168.65.3:30090/realms/master
```

---

## Partie 2 — Configurer Keycloak (automatisé via API)

### 2.1 Créer le Realm

Ouvrir **http://localhost:30090** → Login `admin` / `admin123` :

1. Menu en haut à gauche → **Create realm**
2. Realm name : `argocd`
3. **Create**

### 2.2 Configurer tout via l'API

```bash
KC_URL="http://localhost:30090"
KC_ADMIN="admin"
KC_PASS="admin123"

# 1. Token admin
TOKEN=$(curl -s -X POST "$KC_URL/realms/master/protocol/openid-connect/token" \
  -d "client_id=admin-cli" \
  -d "username=$KC_ADMIN" \
  -d "password=$KC_PASS" \
  -d "grant_type=password" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo "Token OK"

# 2. Forcer le Frontend URL (pour que les tokens utilisent localhost:30090)
curl -s -X PUT "$KC_URL/admin/realms/argocd" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"attributes": {"frontendUrl": "http://localhost:30090"}}'
echo "Frontend URL set"

# 3. Créer le client OIDC 'argocd'
curl -s -X POST "$KC_URL/admin/realms/argocd/clients" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "clientId": "argocd",
    "enabled": true,
    "clientAuthenticatorType": "client-secret",
    "standardFlowEnabled": true,
    "directAccessGrantsEnabled": true,
    "publicClient": false,
    "protocol": "openid-connect",
    "rootUrl": "https://localhost:8443",
    "redirectUris": ["https://localhost:8443/*"],
    "webOrigins": ["*"]
  }'
echo "Client created"

# 4. Récupérer le UUID et le secret du client
CLIENT_UUID=$(curl -s "$KC_URL/admin/realms/argocd/clients?clientId=argocd" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])")
echo "Client UUID: $CLIENT_UUID"

CLIENT_SECRET=$(curl -s "$KC_URL/admin/realms/argocd/clients/$CLIENT_UUID/client-secret" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print(json.load(sys.stdin)['value'])")
echo "Client Secret: $CLIENT_SECRET"

# 5. Créer le scope 'groups' avec le mapper Group Membership
SCOPE_EXISTS=$(curl -s "$KC_URL/admin/realms/argocd/client-scopes" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print(any(s['name']=='groups' for s in json.load(sys.stdin)))")

if [ "$SCOPE_EXISTS" = "False" ]; then
  curl -s -X POST "$KC_URL/admin/realms/argocd/client-scopes" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"name":"groups","protocol":"openid-connect","attributes":{"include.in.token.scope":"true"}}'

  SCOPE_ID=$(curl -s "$KC_URL/admin/realms/argocd/client-scopes" \
    -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print([s['id'] for s in json.load(sys.stdin) if s['name']=='groups'][0])")

  curl -s -X POST "$KC_URL/admin/realms/argocd/client-scopes/$SCOPE_ID/protocol-mappers/models" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
      "name": "groups",
      "protocol": "openid-connect",
      "protocolMapper": "oidc-group-membership-mapper",
      "config": {
        "full.path": "false",
        "id.token.claim": "true",
        "access.token.claim": "true",
        "claim.name": "groups",
        "userinfo.token.claim": "true"
      }
    }'

  curl -s -X PUT "$KC_URL/admin/realms/argocd/clients/$CLIENT_UUID/default-client-scopes/$SCOPE_ID" \
    -H "Authorization: Bearer $TOKEN"
  echo "Scope 'groups' created and assigned"
else
  SCOPE_ID=$(curl -s "$KC_URL/admin/realms/argocd/client-scopes" \
    -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print([s['id'] for s in json.load(sys.stdin) if s['name']=='groups'][0])")
  curl -s -X PUT "$KC_URL/admin/realms/argocd/clients/$CLIENT_UUID/default-client-scopes/$SCOPE_ID" \
    -H "Authorization: Bearer $TOKEN"
  echo "Scope 'groups' already exists, assigned"
fi

# 6. Créer les groupes
for GROUP in ArgoCDAdmins ArgoCDReadOnly; do
  EXISTS=$(curl -s "$KC_URL/admin/realms/argocd/groups?search=$GROUP" \
    -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print(len(json.load(sys.stdin)))")
  if [ "$EXISTS" = "0" ]; then
    curl -s -X POST "$KC_URL/admin/realms/argocd/groups" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"name\":\"$GROUP\"}"
    echo "Group $GROUP created"
  else
    echo "Group $GROUP exists"
  fi
done

# 7. Créer les users et assigner les groupes
for USER_DATA in "omar:omar123:ArgoCDAdmins" "viewer:viewer123:ArgoCDReadOnly"; do
  IFS=':' read -r USERNAME PASSWORD GROUP <<< "$USER_DATA"

  USER_EXISTS=$(curl -s "$KC_URL/admin/realms/argocd/users?username=$USERNAME&exact=true" \
    -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print(len(json.load(sys.stdin)))")

  if [ "$USER_EXISTS" = "0" ]; then
    curl -s -X POST "$KC_URL/admin/realms/argocd/users" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d "{
        \"username\":\"$USERNAME\",
        \"enabled\":true,
        \"emailVerified\":true,
        \"email\":\"$USERNAME@sfr.com\",
        \"credentials\":[{\"type\":\"password\",\"value\":\"$PASSWORD\",\"temporary\":false}]
      }"
    echo "User $USERNAME created"
  else
    echo "User $USERNAME exists"
  fi

  USER_ID=$(curl -s "$KC_URL/admin/realms/argocd/users?username=$USERNAME&exact=true" \
    -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])")
  GROUP_ID=$(curl -s "$KC_URL/admin/realms/argocd/groups?search=$GROUP" \
    -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d[0]['id'] if d else '')")
  curl -s -X PUT "$KC_URL/admin/realms/argocd/users/$USER_ID/groups/$GROUP_ID" \
    -H "Authorization: Bearer $TOKEN"
  echo "$USERNAME -> $GROUP"
done

echo ""
echo "============================================"
echo "  KEYCLOAK CONFIGURATION DONE"
echo "  Client Secret: $CLIENT_SECRET"
echo "============================================"
```

### Résumé de la configuration Keycloak

| Élément | Valeur |
|---------|--------|
| Realm | `argocd` |
| Client ID | `argocd` |
| Client type | OpenID Connect, confidential |
| Frontend URL | `http://localhost:30090` |
| Redirect URIs | `https://localhost:8443/*` |
| Scope | `groups` (mapper Group Membership) |
| Groupe admin | `ArgoCDAdmins` |
| Groupe readonly | `ArgoCDReadOnly` |
| User admin | `omar` / `omar123` → ArgoCDAdmins |
| User readonly | `viewer` / `viewer123` → ArgoCDReadOnly |

---

## Partie 3 — Configurer ArgoCD

### 3.1 Le problème réseau Docker Desktop

Sur Docker Desktop, il y a un mismatch d'URLs :

| Qui | Accède à Keycloak via | Pourquoi |
|-----|-----------------------|----------|
| Navigateur | `localhost:30090` | Docker Desktop forward le NodePort |
| Pods (ArgoCD) | `192.168.65.3:30090` | IP interne du node |

Le navigateur ne peut pas accéder à `192.168.65.3`, et les pods ne peuvent pas accéder à `localhost`.

**Solution** : utiliser l'OIDC direct d'ArgoCD (pas Dex) avec `insecureSkipIssuerValidation: true`
pour ignorer le mismatch d'issuer entre le token (émis avec `localhost:30090`) et l'issuer
configuré (`192.168.65.3:30090`).

### 3.2 ConfigMap ArgoCD (argocd-cm)

```bash
kubectl -n argocd delete configmap argocd-cm

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  url: "https://localhost:8443"
  oidc.config: |
    name: Keycloak
    issuer: http://192.168.65.3:30090/realms/argocd
    clientID: argocd
    clientSecret: <CLIENT_SECRET>
    requestedScopes: ["openid", "profile", "email", "groups"]
    insecureSkipIssuerValidation: true
EOF
```

> **Important** : on utilise `delete` puis `apply` pour s'assurer qu'il n'y a pas
> d'ancienne config (comme `dex.config` ou un ancien `oidc.config`) qui traîne.
> Un simple `kubectl patch` fait un merge et peut laisser des clés obsolètes.

### 3.3 ConfigMap RBAC (argocd-rbac-cm)

```bash
kubectl -n argocd patch configmap argocd-rbac-cm --type merge -p '{
  "data": {
    "policy.default": "role:readonly",
    "policy.csv": "g, ArgoCDAdmins, role:admin\ng, ArgoCDReadOnly, role:readonly\n",
    "scopes": "[groups]"
  }
}'
```

| Groupe Keycloak | Rôle ArgoCD | Droits |
|-----------------|-------------|--------|
| ArgoCDAdmins | `role:admin` | Créer, sync, supprimer des apps |
| ArgoCDReadOnly | `role:readonly` | Voir les apps uniquement |
| (défaut) | `role:readonly` | Lecture seule |

### 3.4 Redémarrer et accéder

```bash
kubectl -n argocd rollout restart deployment argocd-server

sleep 15

pkill -f "port-forward.*argocd-server"
sleep 2
kubectl port-forward svc/argocd-server -n argocd 8443:443 &
```

Ouvrir **https://localhost:8443** → cliquer **LOG IN VIA KEYCLOAK**.

---

## Partie 4 — Tester

### 4.1 Test admin

1. Ouvrir **https://localhost:8443**
2. Cliquer **LOG IN VIA KEYCLOAK**
3. Login : `omar` / `omar123`
4. Vérifier : accès complet, possibilité de sync/créer/supprimer des apps

### 4.2 Test readonly

1. Ouvrir en **navigation privée**
2. **LOG IN VIA KEYCLOAK**
3. Login : `viewer` / `viewer123`
4. Vérifier : voir les apps mais impossible de sync ou supprimer

### 4.3 Test via CLI

```bash
# Login OIDC via CLI
argocd login localhost:8443 --sso --insecure

# Voir le user connecté
argocd account get-user-info

# Test admin (omar) — doit marcher
argocd app sync nginx-staging

# Test readonly (viewer) — doit échouer
argocd app sync nginx-staging
# → "permission denied"
```

---

## Partie 5 — En production (avec Ingress)

En environnement réel (Valentine/SFR), le problème de mismatch d'URLs n'existe pas.
Keycloak serait exposé via un **Ingress** avec un vrai nom DNS :

```yaml
# Exemple de config production (pas de insecureSkipIssuerValidation)
data:
  url: "https://argocd.sfr.internal"
  oidc.config: |
    name: Keycloak
    issuer: https://keycloak.sfr.internal/realms/argocd
    clientID: argocd
    clientSecret: <SECRET>
    requestedScopes: ["openid", "profile", "email", "groups"]
```

Avec un Ingress, l'URL est la même partout (navigateur et pods), donc
`insecureSkipIssuerValidation` n'est pas nécessaire.

---

## Partie 6 — Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|----------|-------|----------|
| `ERR_NAME_NOT_RESOLVED` sur `*.svc.cluster.local` | Le navigateur ne résout pas les DNS Kubernetes | Utiliser NodePort + localhost ou IP du node |
| `connection refused` sur `localhost:9090` depuis un pod | Pour un pod, `localhost` = lui-même | Utiliser l'IP du node (`192.168.65.3`) ou le service interne |
| `issuer did not match` (expected X got Y) | Keycloak émet le token avec l'URL d'accès du navigateur, pas celle des pods | `insecureSkipIssuerValidation: true` ou aligner les URLs avec un Ingress |
| `oidc.config` persiste après `kubectl apply` | `kubectl apply` fait un merge, ne supprime pas les clés | `kubectl delete configmap` puis `kubectl apply` |
| `Invalid parameter: redirect_uri` | L'URI de callback n'est pas dans la liste autorisée du client Keycloak | Ajouter `https://localhost:8443/*` dans les redirect URIs |
| `failed to get provider` dans les logs Dex | Dex ne peut pas contacter l'issuer (localhost = le pod) | Utiliser OIDC direct au lieu de Dex, ou utiliser l'IP du node |
| `CrashLoopBackOff` sur applicationset-controller | CRD `ApplicationSet` manquant | `kubectl apply -n argocd -f https://...argo-cd/stable/manifests/install.yaml` |

---

## Cleanup

```bash
# Supprimer Keycloak
kubectl delete namespace keycloak

# Restaurer ArgoCD sans OIDC
kubectl -n argocd delete configmap argocd-cm
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data: {}
EOF

kubectl -n argocd rollout restart deployment argocd-server
```
