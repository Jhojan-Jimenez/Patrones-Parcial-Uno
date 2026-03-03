# pedido-app — Sistema de Gestión de Pedidos

Despliegue completo con **Helm** + **ArgoCD** sobre Kubernetes (DigitalOcean).

```
Stack: PostgreSQL (Bitnami) · Spring Boot · React (nginx)
GitOps: ArgoCD con sincronización automática
```

---

## Arquitectura

```
                          ┌─────────────────────────────────┐
                          │         Kubernetes Cluster       │
                          │                                  │
Browser ──► Ingress ──────┼──► /        → frontend (nginx)  │
                          │   /api/*    → backend (Spring)   │
                          │                 │                │
                          │                 ▼                │
                          │           PostgreSQL (PVC)        │
                          └─────────────────────────────────┘
                                     ▲
                          ArgoCD observa el repo Git
                          y sincroniza automáticamente
```

---

## Estructura del repositorio

```
pedido-app/
├── charts/
│   └── pedido-app/
│       ├── Chart.yaml            # Chart padre + dep. Bitnami PostgreSQL
│       ├── values.yaml           # Valores base compartidos
│       ├── values-dev.yaml       # Overrides para dev
│       ├── values-prod.yaml      # Overrides para prod
│       ├── templates/
│       │   ├── _helpers.tpl
│       │   └── ingress.yaml      # Ingress unificado /api/* y /
│       └── charts/
│           ├── backend/          # Subchart Spring Boot
│           │   ├── Chart.yaml
│           │   ├── values.yaml
│           │   └── templates/
│           │       ├── deployment.yaml
│           │       ├── service.yaml
│           │       ├── configmap.yaml
│           │       ├── secret.yaml
│           │       └── hpa.yaml
│           └── frontend/         # Subchart React/nginx
│               ├── Chart.yaml
│               ├── values.yaml
│               └── templates/
│                   ├── deployment.yaml
│                   └── service.yaml
└── environments/
    ├── dev/
    │   └── application.yaml      # ArgoCD Application → pedido-dev
    └── prod/
        └── application.yaml      # ArgoCD Application → pedido-prod
```

---

## Prerrequisitos

- `kubectl` configurado con acceso al clúster
- `helm` >= 3.x
- `argocd` CLI (opcional, para gestión desde terminal)

---

## 1. Instalar ArgoCD en el clúster

```bash
export KUBECONFIG=./pedidos-app-kubeconfig.yaml

# Crear namespace y desplegar ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que los pods estén listos
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s

# Obtener la contraseña inicial de admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Exponer la UI de ArgoCD (en otra terminal)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Acceder en: https://localhost:8080  (user: admin)
```

---

## 2. Instalar el chart manualmente con Helm

```bash
export KUBECONFIG=./pedidos-app-kubeconfig.yaml

# Bajar la dependencia de Bitnami PostgreSQL
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm dependency update charts/pedido-app

# Desplegar en dev
helm install pedido-app charts/pedido-app \
  -f charts/pedido-app/values.yaml \
  -f charts/pedido-app/values-dev.yaml \
  --namespace pedido-dev \
  --create-namespace

# Desplegar en prod
helm install pedido-app charts/pedido-app \
  -f charts/pedido-app/values.yaml \
  -f charts/pedido-app/values-prod.yaml \
  --namespace pedido-prod \
  --create-namespace

# Actualizar después de cambios
helm upgrade pedido-app charts/pedido-app \
  -f charts/pedido-app/values.yaml \
  -f charts/pedido-app/values-dev.yaml \
  --namespace pedido-dev
```

---

## 3. Configurar ArgoCD (GitOps)

### 3.1 Actualizar la URL del repositorio

Editar ambos `environments/*/application.yaml` y reemplazar:
```yaml
repoURL: https://github.com/TU_USUARIO/pedido-app
```

### 3.2 Aplicar las Applications de ArgoCD

```bash
export KUBECONFIG=./pedidos-app-kubeconfig.yaml

kubectl apply -f environments/dev/application.yaml
kubectl apply -f environments/prod/application.yaml

# Verificar estado
kubectl get applications -n argocd
```

### 3.3 Cómo funciona la sincronización automática

```
Git push  →  ArgoCD detecta cambio (~3 min polling)
          →  Calcula diff entre Git y clúster
          →  Aplica los cambios automáticamente
          →  Estado: Synced ✓
```

Con `automated.selfHeal: true`, si alguien modifica manualmente un recurso en el clúster, ArgoCD lo revierte al estado del repo.

---

## 4. Endpoints de acceso

| Ambiente | URL Frontend | URL API |
|---|---|---|
| dev | `http://dev.pedidos.local` | `http://dev.pedidos.local/api` |
| prod | `http://pedidos.example.com` | `http://pedidos.example.com/api` |

> Para pruebas locales sin DNS, agregar al `/etc/hosts`:
> ```
> <IP_DEL_INGRESS>  dev.pedidos.local
> <IP_DEL_INGRESS>  pedidos.example.com
> ```
> Obtener la IP del Ingress: `kubectl get ingress -n pedido-dev`

---

## 5. Demostración de sincronización automática

1. Cambiar el tag del backend en `values-dev.yaml`:
   ```yaml
   backend:
     image:
       tag: "3.4.0"   # antes era 3.3.0
   ```
2. Commit y push al repo
3. ArgoCD detecta el cambio y actualiza el Deployment automáticamente
4. Verificar: `kubectl get pods -n pedido-dev -w`

---

## 6. Comandos útiles de diagnóstico

```bash
# Ver todos los pods
kubectl get pods -n pedido-dev
kubectl get pods -n pedido-prod

# Ver logs del backend
kubectl logs -l app.kubernetes.io/component=backend -n pedido-dev

# Ver estado del HPA
kubectl get hpa -n pedido-dev

# Ver el PVC de PostgreSQL
kubectl get pvc -n pedido-dev

# Ver estado en ArgoCD
kubectl get applications -n argocd
```
