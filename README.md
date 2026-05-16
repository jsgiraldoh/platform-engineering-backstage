# Platform Engineering: del despliegue tradicional a un IDP con Backstage

Guía completa para evolucionar desde despliegues manuales hacia un modelo moderno de Platform Engineering usando **Docker**, **Kubernetes**, **Helm**, **Argo CD**, **GitHub Actions** y **Backstage** como Internal Developer Platform (IDP).

---

## Tabla de contenidos

1. [Arquitectura](#arquitectura)
2. [Pre-requisitos](#pre-requisitos)
3. [Estructura del repositorio](#estructura-del-repositorio)
4. [Parte 1 — Aplicación containerizada](#parte-1--aplicación-containerizada)
5. [Parte 2 — Kubernetes con manifiestos planos](#parte-2--kubernetes-con-manifiestos-planos)
6. [Parte 3 — Helm Chart](#parte-3--helm-chart)
7. [Parte 4 — GitOps con Argo CD](#parte-4--gitops-con-argo-cd)
8. [Parte 5 — CI/CD con GitHub Actions](#parte-5--cicd-con-github-actions)
9. [Parte 6 — Backstage como IDP](#parte-6--backstage-como-idp)
10. [Variables y secretos requeridos](#variables-y-secretos-requeridos)

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

> **Windows + Git Bash**: todos los comandos Docker con rutas de volumen requieren el prefijo `MSYS_NO_PATHCONV=1` o usar doble slash `//` para evitar que Git Bash convierta las rutas.

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
└── Dockerfile                   # Imagen nginx:alpine
```

---

## Parte 1 — Aplicación containerizada

### Construir la imagen

```bash
docker build -t platform-engineering .
```

### Validar localmente

```bash
docker run -d -p 8080:80 --name app platform-engineering
# Abrir http://localhost:8080
```

### Publicar en Docker Hub

```bash
docker login
docker tag platform-engineering jsgiraldoh/img-example-platform-eng
docker push jsgiraldoh/img-example-platform-eng
```

---

## Parte 2 — Kubernetes con manifiestos planos

### Crear el clúster Kind

```bash
kind create cluster --name platformengineering
kubectl cluster-info --context kind-platformengineering
kubectl get nodes
```

### Desplegar con manifiestos planos

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

kubectl get pods
kubectl get svc
```

### Acceder a la aplicación

```bash
kubectl port-forward svc/platform-engineering 8070:8080
# Abrir http://localhost:8070
```

### Limpiar recursos

```bash
kubectl delete -f k8s/
```

---

## Parte 3 — Helm Chart

El chart se encuentra en `platform-engineering/`. Parametriza imagen, réplicas y puertos vía `values.yaml`.

### Valores configurables (`platform-engineering/values.yaml`)

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
helm template platform-engineering ./platform-engineering
```

### Instalar el Chart

```bash
helm install platform-engineering ./platform-engineering
helm ls
kubectl get pods
```

### Actualizar tras cambios

```bash
helm upgrade platform-engineering ./platform-engineering
```

### Acceder

```bash
kubectl port-forward svc/platform-engineering 8070:8080
```

### Desinstalar

```bash
helm uninstall platform-engineering
```

---

## Parte 4 — GitOps con Argo CD

Argo CD sincroniza automáticamente el estado del clúster con el Helm Chart en Git.

### Instalar Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd
kubectl get pods -n argocd -w
```

### Acceder a la UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
# Abrir https://localhost:8081  (usuario: admin)
```

### Obtener la contraseña inicial

```bash
# Linux/macOS
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Windows PowerShell
kubectl get secret argocd-initial-admin-secret -n argocd `
  -o jsonpath="{.data.password}" | `
  ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

### Registrar la Application

```bash
kubectl create -f argocd.yaml
```

El archivo `argocd.yaml` apunta al path `platform-engineering/` de este repositorio:

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

Cada `git push` a `main` que modifique `platform-engineering/values.yaml` disparará un reconcile automático en Argo CD.

---

## Parte 5 — CI/CD con GitHub Actions

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

Ir a: `Repositorio → Settings → Secrets and variables → Actions`

| Secret | Valor |
|--------|-------|
| `REGISTRY_USERNAME` | Tu usuario de Docker Hub |
| `REGISTRY_PASSWORD` | Token de acceso de Docker Hub |

Para crear el token: Docker Hub → Account Settings → Security → New Access Token (permisos: `Read & Write`).

> El workflow también necesita que el repositorio tenga habilitado el permiso `contents: write` (ya está configurado en el YAML).

---

## Parte 6 — Backstage como IDP

Backstage corre dentro de un contenedor Docker montando `backstage-app/` como volumen, lo que permite editar los archivos desde el host y ver los cambios en tiempo real.

### Iniciar el contenedor de desarrollo

**Git Bash (Windows):**
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
  -v "D:\PlatformEngineering\platform-engineering-backstage\backstage-app:/app" `
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
```

### Instalar dependencias de Node

```bash
cd backstage
yarn install
```

### Configurar acceso externo (`app-config.yaml`)

Agregar bajo la sección `backend`:
```yaml
backend:
  listen:
    host: 0.0.0.0
```

### Crear `app-config.local.yaml` para desarrollo local

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

### Configurar autenticación GitHub OAuth

1. Ir a GitHub → Settings → Developer Settings → OAuth Apps → New OAuth App
2. Configurar:
   - **Homepage URL**: `http://localhost:3000`
   - **Authorization callback URL**: `http://localhost:7007/api/auth/github/handler/frame`
3. Generar el Client Secret y guardar ambos valores como variables de entorno al levantar el contenedor.

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
    email: johansebastiangh@gmail.com
  memberOf: []
```

### Iniciar Backstage

```bash
yarn start --config /app/backstage/app-config.local.yaml
# Frontend: http://localhost:3000
# Backend:  http://localhost:7007
```

### Instalar plugins adicionales

```bash
yarn add --cwd packages/app \
  @backstage/plugin-scaffolder \
  @backstage/plugin-search \
  @backstage/plugin-techdocs \
  @backstage/plugin-home \
  @backstage/plugin-user-settings
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
3. GitHub Actions (CI) construye imagen y hace push a Docker Hub con tag = commit SHA (6 chars)
4. GitHub Actions (CD) actualiza platform-engineering/values.yaml con el nuevo tag y hace commit
5. Argo CD detecta el cambio en Git y sincroniza el clúster Kubernetes automáticamente
6. El nuevo pod arranca con la imagen actualizada
```

---

## Limpiar el entorno

```bash
# Eliminar todos los contenedores Docker
docker rm -f $(docker ps -aq)

# Eliminar el clúster Kind
kind delete cluster --name platformengineering

# Desinstalar releases Helm
helm uninstall platform-engineering
helm uninstall argocd -n argocd
```
