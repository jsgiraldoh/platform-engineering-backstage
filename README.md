# Platform Engineering: del despliegue tradicional a un IDP con Backstage

Guía completa para evolucionar desde despliegues manuales hacia un modelo moderno de Platform Engineering usando **Docker**, **Kubernetes**, **Helm**, **Argo CD**, **GitHub Actions** y **Backstage** como Internal Developer Platform (IDP).

---

## Tabla de contenidos

1. [Arquitectura](#arquitectura)
2. [Pre-requisitos](#pre-requisitos)
3. [Estructura del repositorio](#estructura-del-repositorio)
4. [Parte 1 — Clonar y configurar el repositorio](#parte-1--clonar-y-configurar-el-repositorio)
5. [Parte 2 — Aplicación containerizada](#parte-2--aplicación-containerizada)
6. [Parte 3 — Kubernetes con manifiestos planos](#parte-3--kubernetes-con-manifiestos-planos)
7. [Parte 4 — Helm Chart](#parte-4--helm-chart)
8. [Parte 5 — GitOps con Argo CD](#parte-5--gitops-con-argo-cd)
9. [Parte 6 — CI/CD con GitHub Actions](#parte-6--cicd-con-github-actions)
10. [Parte 7 — Backstage como IDP](#parte-7--backstage-como-idp)
11. [Variables y secretos requeridos](#variables-y-secretos-requeridos)
12. [Troubleshooting](#troubleshooting)

---

## Arquitectura

```
┌─────────────┐    push     ┌──────────────────┐    build/push    ┌─────────────┐
│  Developer  │ ──────────► │  GitHub Actions  │ ───────────────► │  Docker Hub │
└─────────────┘             └──────────────────┘                  └─────────────┘
                                     │                                     │
                              update values.yaml                      pull image
                                     │                                     │
                                     ▼                                     ▼
                            ┌─────────────────┐    sync       ┌───────────────────┐
                            │   Git (Helm)    │ ◄──────────── │     Argo CD       │
                            └─────────────────┘               └───────────────────┘
                                                                        │
                                                                  deploy/reconcile
                                                                        │
                                                                        ▼
                                                              ┌──────────────────┐
                                                              │   Kubernetes     │
                                                              │  (Kind cluster)  │
                                                              └──────────────────┘
```

---

## Pre-requisitos

| Herramienta | Versión mínima | Instalación (Windows) |
|-------------|---------------|----------------------|
| Docker Desktop | 24+ | [docker.com](https://www.docker.com/products/docker-desktop/) |
| Kind | 0.20+ | `winget install Kubernetes.kind` |
| kubectl | 1.28+ | `winget install Kubernetes.kubectl` |
| Helm | 3.12+ | `winget install Helm.Helm` |
| Git | 2.40+ | `winget install Git.Git` |

> **Chocolatey en Windows**: los comandos `choco install` / `choco upgrade` requieren una terminal con **"Ejecutar como administrador"**. Sin permisos de admin fallará con "Acceso denegado".

> **Git Bash en Windows**: el comando `watch` no existe. Para monitorear pods usa `kubectl get pods` repetido o `kubectl get pods -w` (solo funciona con un tipo de recurso a la vez).

---

## Estructura del repositorio

```
platform-engineering-backstage/
├── .github/
│   └── workflows/
│       └── ci-cd.yaml          # Pipeline CI/CD completo
├── backstage-app/               # Aplicación Backstage (IDP)
├── k8s/
│   ├── deployment.yaml          # Manifiesto Kubernetes plano
│   └── service.yaml
├── platform-engineering/        # Helm Chart
│   ├── Chart.yaml
│   ├── charts/
│   ├── templates/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── values.yaml
├── src/
│   ├── index.html               # Aplicación web estática
│   └── fondo.png
├── argocd.yaml                  # Application CRD para Argo CD
├── Dockerfile                   # Imagen nginx:alpine
└── .gitignore
```

---

## Parte 1 — Clonar y configurar el repositorio

Clonar el repositorio base del taller y apuntarlo a tu propio repositorio:

```bash
git clone https://github.com/roko1987-k8s/platform-engineering-backstage.git
cd platform-engineering-backstage

git remote remove origin
git remote add origin https://github.com/<TU-USUARIO>/platform-engineering-backstage.git
git branch -M main
git push -u origin main
```

Configurar tu identidad de Git (si aún no lo tienes):

```bash
git config --global user.name "tu-usuario"
git config --global user.email "tu@email.com"
```

---

## Parte 2 — Aplicación containerizada

### Construir la imagen

```bash
docker build -t img-example-platform-eng .
```

### Validar localmente

```bash
docker run -d -p 8080:80 --name app img-example-platform-eng
# Abrir http://localhost:8080
```

### Login en Docker Hub

> **Importante**: el login con `-u user` usando un Personal Access Token (PAT) puede fallar con `authentication required - access token has insufficient scopes`. Usa el login web en su lugar:

```bash
# Recomendado: login vía navegador (evita problemas de scopes con PAT)
docker login
# Sigue las instrucciones del navegador

# Alternativa con contraseña real (no PAT):
docker login -u <tu-usuario>
```

### Etiquetar y publicar

```bash
docker tag img-example-platform-eng:latest jsgiraldoh/img-example-platform-eng:latest
docker push jsgiraldoh/img-example-platform-eng:latest
```

> **Asegúrate de que el push fue exitoso antes de continuar.** Si aplicas los manifiestos de Kubernetes antes de que la imagen esté en Docker Hub, obtendrás `ImagePullBackOff`.

---

## Parte 3 — Kubernetes con manifiestos planos

### Crear el clúster Kind

```bash
kind create cluster --name platform-engineering
kubectl cluster-info --context kind-platform-engineering
kubectl get nodes
```

### Desplegar con manifiestos planos

Los manifiestos están en `k8s/`. Ejecutar desde la raíz del repositorio:

```bash
kubectl apply -f k8s/
kubectl get pods
kubectl get svc
```

### Verificar el estado del pod

```bash
# Monitorear (Git Bash no tiene 'watch', usar -w con recurso único)
kubectl get pods -w

# Ver detalle si hay errores
kubectl describe pod <nombre-del-pod>
kubectl logs <nombre-del-pod>
```

### Acceder a la aplicación

El Service de `k8s/service.yaml` expone el puerto **80**:

```bash
kubectl port-forward svc/platform-engineering 8070:80
# Abrir http://localhost:8070
```

### Limpiar recursos

```bash
kubectl delete -f k8s/
```

---

## Parte 4 — Helm Chart

El chart se encuentra en `platform-engineering/`. Todos los comandos de Helm deben ejecutarse **desde dentro de ese directorio**.

```bash
cd platform-engineering/
```

### Valores configurables (`values.yaml`)

```yaml
replicaCount: 1

app:
  name: platform-engineering

image:
  repository: jsgiraldoh/img-example-platform-eng
  tag: latest

containerPort: 80

service:
  port: 8080
  targetPort: 80
```

### Validar templates sin desplegar

```bash
# Desde dentro de platform-engineering/
helm template platform-engineering .
```

### Instalar el Chart

```bash
# Desde dentro de platform-engineering/
helm install platform-engineering .
helm ls
kubectl get all
```

### Acceder a la aplicación

El Service del Helm chart expone el puerto **8080** (definido en `values.yaml`):

```bash
kubectl port-forward svc/platform-engineering 8080:8080
# Abrir http://localhost:8080
```

> Nota: aunque el port-forward dice `8080:8080`, internamente Kubernetes redirige al `targetPort: 80` del contenedor.

### Actualizar tras cambios

```bash
helm upgrade platform-engineering .
```

### Desinstalar

```bash
helm uninstall platform-engineering
```

---

## Parte 5 — GitOps con Argo CD

Argo CD sincroniza automáticamente el estado del clúster con el Helm Chart en Git.

### Instalar Argo CD

> **Importante**: `helm repo update` es **obligatorio** antes de instalar. Sin él obtendrás el error `no cached repo found`.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd
```

Verificar que todos los pods estén corriendo:

```bash
kubectl get pods -n argocd -w
```

### Acceder a la UI de Argo CD

Argo CD usa HTTPS en el puerto 443. Mapearlo a un puerto local diferente para no chocar con la app:

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
# Abrir https://localhost:8081 (aceptar el certificado autofirmado)
# Usuario: admin
```

### Obtener la contraseña inicial

```bash
# Git Bash / Linux / macOS
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# PowerShell
kubectl -n argocd get secret argocd-initial-admin-secret `
  -o jsonpath="{.data.password}" | `
  ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

### Registrar la Application

```bash
kubectl create -f argocd.yaml
```

El archivo `argocd.yaml` en la raíz del repositorio:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-engineering
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/jsgiraldoh/platform-engineering-backstage.git
    targetRevision: HEAD
    path: platform-engineering
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Cada `git push` a `main` que modifique `platform-engineering/values.yaml` (por ejemplo, el tag de imagen) disparará un reconcile automático en Argo CD.

---

## Parte 6 — CI/CD con GitHub Actions

El workflow `.github/workflows/ci-cd.yaml` automatiza el ciclo completo:

```
push a main
    │
    ├── Job CI
    │   ├── Checkout código
    │   ├── Calcular commit_id (6 chars del SHA)
    │   ├── Login Docker Hub
    │   └── Build & Push imagen con tag = commit_id
    │
    └── Job CD (necesita CI)
        ├── Checkout código
        ├── Actualizar image.tag en platform-engineering/values.yaml
        ├── Commit automático del cambio
        └── Push → Argo CD detecta y sincroniza
```

### Configurar secretos en GitHub

Ir a: `Repositorio → Settings → Secrets and variables → Actions → New repository secret`

| Secret | Valor |
|--------|-------|
| `REGISTRY_USERNAME` | Tu usuario de Docker Hub |
| `REGISTRY_PASSWORD` | Token de acceso de Docker Hub con permisos **Read & Write** |

Para crear el token: Docker Hub → Account Settings → Security → New Access Token.

---

## Parte 7 — Backstage como IDP

Backstage corre dentro de un contenedor Docker montando `backstage-app/` como volumen, lo que permite editar los archivos desde el host y ver los cambios en tiempo real.

### Iniciar el contenedor de desarrollo

**Git Bash (Windows) — opción recomendada:**

```bash
MSYS_NO_PATHCONV=1 docker run --rm \
  --platform=linux/amd64 \
  -p 3000:3000 -p 7007:7007 \
  -e AUTH_GITHUB_CLIENT_ID=<tu-client-id> \
  -e AUTH_GITHUB_CLIENT_SECRET=<tu-client-secret> \
  -ti \
  -v "D:/PlatformEngineering/platform-engineering-backstage/backstage-app:/app" \
  -w /app \
  node:20-bookworm-slim bash
```

**PowerShell:**

```powershell
docker run --rm `
  --platform=linux/amd64 `
  -p 3000:3000 -p 7007:7007 `
  -e AUTH_GITHUB_CLIENT_ID=<tu-client-id> `
  -e AUTH_GITHUB_CLIENT_SECRET=<tu-client-secret> `
  -ti `
  -v "D:\PlatformEngineering\platform-engineering-backstage\backstage-app://app" `
  -w //app `
  node:20-bookworm-slim bash
```

### Instalar dependencias del sistema (dentro del contenedor)

```bash
apt update && apt install -y python3 make g++ build-essential
```

### Crear la aplicación Backstage

```bash
corepack enable
npx @backstage/create-app@latest --path backstage --skip-install
# Cuando pregunte el nombre de la app: backstage
```

### Instalar dependencias de Node

```bash
cd backstage
yarn install
```

### Iniciar Backstage (modo básico)

```bash
yarn start
# Frontend: http://localhost:3000
```

### Configurar acceso externo

Modificar `backstage/app-config.yaml`, agregar bajo `backend`:

```yaml
backend:
  listen:
    host: 0.0.0.0
```

### Configurar autenticación GitHub OAuth

1. Ir a GitHub → Settings → Developer Settings → OAuth Apps → **New OAuth App**
2. Completar:
   - **Homepage URL**: `http://localhost:3000`
   - **Authorization callback URL**: `http://localhost:7007/api/auth/github/handler/frame`
3. Generar un Client Secret y guardar ambos valores.

### Crear `app-config.local.yaml` (contiene secretos, no se sube al repo)

```yaml
app:
  title: Taller Platform Engineering con Backstage
  baseUrl: http://localhost:3000
  listen:
    host: 0.0.0.0

backend:
  baseUrl: http://localhost:7007
  listen:
    port: 7007
    host: 0.0.0.0

auth:
  environment: development
  providers:
    github:
      development:
        clientId: ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}
        signIn:
          resolvers:
            - resolver: usernameMatchingUserEntityName

catalog:
  rules:
    - allow: [User, Component, System, API, Resource, Location]
  locations:
    - type: file
      target: /app/backstage/users/user.yaml
```

### Instalar plugin de autenticación GitHub en el backend

```bash
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider
```

Agregar en `packages/backend/src/index.ts`:

```ts
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```

### Crear usuario en el catálogo

```bash
mkdir backstage/users
```

Crear `backstage/users/user.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: jsgiraldoh
spec:
  profile:
    displayName: Johan Giraldo
    email: tu@email.com
  memberOf: []
```

### Iniciar Backstage con config local

```bash
yarn start --config /app/backstage/app-config.local.yaml
# Frontend: http://localhost:3000
# Backend:  http://localhost:7007
```

---

## Variables y secretos requeridos

| Contexto | Variable / Secret | Descripción |
|----------|------------------|-------------|
| GitHub Actions | `REGISTRY_USERNAME` | Usuario Docker Hub |
| GitHub Actions | `REGISTRY_PASSWORD` | Token Docker Hub con permisos Read/Write |
| Backstage (env) | `AUTH_GITHUB_CLIENT_ID` | Client ID de la OAuth App en GitHub |
| Backstage (env) | `AUTH_GITHUB_CLIENT_SECRET` | Client Secret de la OAuth App en GitHub |

---

## Flujo completo de un cambio

```
1. Modificar src/index.html (o cualquier archivo de la app)
2. git add . && git commit -m "feat: ..." && git push origin main
3. GitHub Actions (CI) construye imagen y hace push a Docker Hub con tag = SHA (6 chars)
4. GitHub Actions (CD) actualiza platform-engineering/values.yaml con el nuevo tag y hace commit
5. Argo CD detecta el cambio en Git y sincroniza el clúster automáticamente
6. El nuevo pod arranca con la imagen actualizada
```

---

## Troubleshooting

| Error | Causa | Solución |
|-------|-------|----------|
| `ImagePullBackOff` | La imagen no existe en Docker Hub cuando se aplicaron los manifiestos | Hacer `docker push` primero, luego `kubectl apply` |
| `authentication required - access token has insufficient scopes` | PAT de Docker Hub sin permisos suficientes | Usar `docker login` (web-based) en lugar de `docker login -u user` |
| `no cached repo found` en Helm | Falta ejecutar `helm repo update` | Ejecutar `helm repo update` antes de `helm install` |
| `unable to detect chart` en `helm template` | Ejecutar el comando desde la raíz en lugar del directorio del chart | Entrar a `cd platform-engineering/` antes de ejecutar comandos de Helm |
| `Service does not have a service port 80` en port-forward | El Service del Helm chart usa puerto 8080, no 80 | Usar `kubectl port-forward svc/platform-engineering 8080:8080` |
| `watch: command not found` | `watch` no existe en Git Bash | Usar `kubectl get pods` repetido o `kubectl get pods -w` |
| `error: you may only specify a single resource type` | `-w` no funciona con `kubectl get all` | Usar `kubectl get pods -w` (un solo tipo de recurso) |
| Chocolatey `Acceso denegado` | Sin permisos de administrador | Abrir Git Bash / PowerShell como **Administrador** |

---

## Limpiar el entorno

```bash
# Eliminar todos los contenedores Docker
docker rm -f $(docker ps -aq)

# Eliminar el clúster Kind
kind delete cluster --name platform-engineering

# Desinstalar releases Helm
helm uninstall platform-engineering
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```
