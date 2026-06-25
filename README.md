# 🐶 Tienda de Perritos — Orquestación en AWS EKS

Plataforma de e-commerce desplegada sobre **Amazon EKS (Kubernetes)** con orquestación de contenedores, autoescalado, tolerancia a fallos y un pipeline CI/CD completamente automatizado. Desarrollada como parte de la Evaluación Parcial N°3 del curso ISY1101 - Introducción a Herramientas DevOps.

---

## 🏗️ Arquitectura

```
                    Internet
                        │
            ┌───────────▼────────────┐
            │   LoadBalancer (CLB)    │  ← URL pública
            └───────────┬────────────┘
                        │
        ┌───────────────▼───────────────┐
        │      Clúster EKS (tienda)      │
        │                                │
        │   ┌──────────────────────┐     │
        │   │  2x Pod Frontend      │     │  Nginx (puerto 80)
        │   │  (HTML/JS + proxy)    │     │
        │   └──────────┬───────────┘     │
        │              │ /api/            │
        │   ┌──────────▼───────────┐     │
        │   │  2x Pod Backend       │     │  Node.js (puerto 3001)
        │   │  (API REST)           │     │
        │   └──────────┬───────────┘     │
        │              │                  │
        │   ┌──────────▼───────────┐     │
        │   │  1x Pod MySQL         │     │  MySQL 8 (puerto 3306)
        │   └──────────────────────┘     │
        │                                │
        │   HPA: escala pods por CPU      │
        └────────────────────────────────┘
```

La aplicación corre dentro de un clúster EKS desplegado en una VPC con 2 zonas de disponibilidad. Los nodos worker viven en subredes privadas, mientras que el LoadBalancer expone el frontend públicamente a través de las subredes públicas. Kubernetes orquesta todos los contenedores, gestionando su ciclo de vida, escalado y recuperación automática.

---

## 📁 Estructura del repositorio

```
tienda-perritos-eks/
├── frontend/                      # Aplicación web (Nginx + HTML/JS)
│   ├── Dockerfile                 # Imagen basada en nginx:alpine
│   ├── default.conf               # Config Nginx + proxy inverso al backend
│   ├── index.html
│   └── app.js
├── backend/                       # API REST (Node.js)
│   ├── Dockerfile                 # Imagen basada en node:18-alpine
│   ├── server.js                  # Servidor con endpoints CRUD
│   └── package.json
├── db/                            # Base de datos (MySQL 8)
│   ├── Dockerfile                 # Imagen basada en mysql:8
│   └── init.sql                   # Script de inicialización de la BD
├── k8s/                           # Manifiestos de Kubernetes
│   ├── namespace.yaml             # Namespace "tienda"
│   ├── mysql-secret.yaml          # Contraseña de MySQL (cifrada base64)
│   ├── mysql-deployment.yaml      # Deployment de MySQL
│   ├── mysql-service.yaml         # Service interno de MySQL
│   ├── backend-deployment.yaml    # Deployment del backend (2 réplicas)
│   ├── backend-service.yaml       # Service interno del backend
│   ├── backend-hpa.yaml           # Autoescalado del backend (CPU 70%)
│   ├── frontend-deployment.yaml   # Deployment del frontend (2 réplicas)
│   ├── frontend-service.yaml      # Service LoadBalancer (público)
│   └── frontend-hpa.yaml          # Autoescalado del frontend (CPU 60%)
└── .github/
    └── workflows/
        └── deploy-eks.yml         # Pipeline CI/CD (build → push → deploy)
```

---

## 🧩 Componentes de Kubernetes

### Namespace
Todos los recursos viven dentro del namespace `tienda`, que aísla la aplicación del resto del clúster y mantiene el orden.

### Deployments
Cada servicio se define como un Deployment que especifica cuántas réplicas (pods) deben existir. Kubernetes garantiza que ese número se mantenga siempre, recreando pods automáticamente si alguno falla.

### Services
- **MySQL y Backend** usan Services de tipo `ClusterIP`, accesibles solo dentro del clúster mediante un nombre DNS interno (`tienda-db`, `tienda-backend`).
- **Frontend** usa un Service de tipo `LoadBalancer` que provisiona un balanceador de carga de AWS para exponer la aplicación a internet.

### Secret
La contraseña de MySQL se almacena en un Secret de Kubernetes (codificada en base64) y se inyecta al contenedor como variable de entorno, evitando exponerla en texto plano en los manifiestos.

### HPA (Horizontal Pod Autoscaler)
Cada servicio tiene un HPA que monitorea el uso de CPU vía Metrics Server y crea o elimina réplicas automáticamente:
- **Backend:** mínimo 2, máximo 10 réplicas, umbral 70% CPU
- **Frontend:** mínimo 2, máximo 6 réplicas, umbral 60% CPU

---

## 🐳 Imágenes Docker

Las imágenes se almacenan en **Amazon ECR** (Elastic Container Registry):

| Servicio | Imagen base | Puerto | Repositorio ECR |
|---|---|---|---|
| Frontend | nginx:alpine | 80 | tienda-frontend |
| Backend | node:18-alpine | 3001 | tienda-backend |
| Base de datos | mysql:8 | 3306 | tienda-db |

---

## ⚙️ Pipeline CI/CD

El pipeline está definido en `.github/workflows/deploy-eks.yml` y se ejecuta automáticamente con cada `push` a la rama `main`, o manualmente mediante `workflow_dispatch`.

### Flujo del pipeline:
```
push a main
     ↓
1. Checkout del código
2. Configurar credenciales AWS
3. Login en Amazon ECR
4. Build & push de las 3 imágenes (tag = hash del commit)
5. Configurar kubeconfig del clúster EKS
6. Aplicar manifiestos (namespace, MySQL, services)
7. Actualizar deployments backend y frontend (kubectl set image)
8. Esperar rollout (rolling update sin downtime)
9. Aplicar HPA
10. Mostrar estado final (pods, services, hpa)
```

### Secrets requeridos en GitHub:
| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial de acceso AWS |
| `AWS_SECRET_ACCESS_KEY` | Clave secreta AWS |
| `AWS_SESSION_TOKEN` | Token de sesión (AWS Academy) |
| `AWS_REGION` | Región (us-east-1) |
| `EKS_CLUSTER_NAME` | Nombre del clúster (tienda-eks) |
| `EKS_NAMESPACE` | Namespace de la app (tienda) |

> ⚠️ **Nota AWS Academy:** las credenciales (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) se renuevan cada vez que se reinicia el laboratorio. Deben actualizarse en los secrets de GitHub para que el pipeline funcione.

---

## 🚀 Despliegue manual (paso a paso)

### Requisitos previos
- AWS CLI configurado con credenciales válidas
- kubectl instalado
- Docker Desktop
- Clúster EKS creado y activo

### 1. Conectar kubectl al clúster
```bash
aws eks update-kubeconfig --region us-east-1 --name tienda-eks
kubectl get nodes
```

### 2. Login en ECR y subir imágenes
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

# Repetir para frontend, backend y db
docker build -t tienda-frontend ./frontend
docker tag tienda-frontend:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend:eks-v1
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend:eks-v1
```

### 3. Desplegar en EKS (orden importante)
```bash
cd k8s
kubectl apply -f namespace.yaml
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
kubectl apply -f backend-hpa.yaml
kubectl apply -f frontend-hpa.yaml
```

### 4. Obtener la URL pública
```bash
kubectl get svc tienda-frontend -n tienda
```
Copiar el `EXTERNAL-IP` y abrirlo en el navegador.

---

## 📊 Validación y monitoreo

### Ver estado de los pods
```bash
kubectl get pods -n tienda
```

### Ver métricas de uso (CPU/memoria)
```bash
kubectl top nodes
kubectl top pods -n tienda
```

### Ver autoescalado
```bash
kubectl get hpa -n tienda
```

### Logs del control plane
Disponibles en CloudWatch → Logs → Log groups → `/aws/eks/tienda-eks/cluster`

---

## 🔧 Características técnicas destacadas

- **Alta disponibilidad:** múltiples réplicas de frontend y backend distribuidas por el clúster.
- **Auto-healing:** Kubernetes recrea automáticamente cualquier pod que falle, gracias a los Deployments y health checks (readiness/liveness probes).
- **Autoescalado:** los HPA ajustan la cantidad de réplicas según la carga de CPU en tiempo real.
- **Rolling updates:** las actualizaciones se aplican gradualmente sin tiempo de inactividad.
- **Proxy inverso:** Nginx sirve el frontend y redirige las llamadas `/api/` al backend mediante DNS interno del clúster.
- **Despliegue continuo:** cada commit a `main` dispara el pipeline que reconstruye y redespliega automáticamente.

---

## 🛠️ Tecnologías utilizadas

| Categoría | Tecnología |
|---|---|
| Orquestación | Amazon EKS (Kubernetes 1.35) |
| Contenedores | Docker |
| Registro de imágenes | Amazon ECR |
| Frontend | Nginx + HTML/JavaScript |
| Backend | Node.js |
| Base de datos | MySQL 8 |
| CI/CD | GitHub Actions |
| Red | AWS VPC (2 AZs, subredes públicas/privadas) |
| Balanceo | AWS Classic Load Balancer |
| Monitoreo | CloudWatch + Metrics Server |
