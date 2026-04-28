# Práctica: Despliegue de aplicación web en Kubernetes con Minikube y Argo CD

## Descripción del proyecto

Aplicación web estática tipo "Hello World" desplegada en un clúster local de Kubernetes (Minikube) y gestionada mediante GitOps con Argo CD. El objetivo es demostrar el ciclo completo de contenerización, despliegue declarativo y sincronización desde Git.

---

## Estructura del repositorio

```
practica/
├── app/                  # Código fuente y Dockerfile de la aplicación
│   ├── Dockerfile
│   └── index.html
├── argocd/               # Configuración de Argo CD
│   └── application.yaml
├── k8s/                  # Manifiestos de Kubernetes
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── docs/                 # Capturas de pantalla y evidencias
```

- **app/**: contiene la aplicación web (HTML estático servido por nginx) y su Dockerfile.
- **k8s/**: manifiestos de Kubernetes separados por responsabilidad. Argo CD apunta a esta carpeta para sincronizar el estado del clúster.
- **argocd/**: define el recurso `Application` de Argo CD que conecta el repositorio Git con el clúster.
- **docs/**: evidencias del funcionamiento en Minikube y Argo CD.

---

## Recursos de Kubernetes

### Deployment
Define el pod que ejecuta la aplicación. Se usa una sola réplica con la imagen `hello-world:1.0.0` basada en nginx. Incluye probes de salud:

- **livenessProbe**: comprueba cada 10 segundos que el pod sigue respondiendo en `/`. Si falla, Kubernetes lo reinicia.
- **readinessProbe**: comprueba cada 5 segundos que el pod está listo para recibir tráfico antes de añadirlo al Service.

Ambas probes usan `httpGet` sobre el puerto 80, que es el puerto por defecto de nginx.

### Service
Expone el pod internamente dentro del clúster mediante un `ClusterIP` (tipo por defecto). El Ingress se conecta a este Service para enrutar el tráfico externo. No es necesario exponerlo directamente al exterior porque el Ingress actúa como punto de entrada.

### Ingress
Enruta las peticiones al host `hello-world.local` hacia el Service. Requiere que el addon `ingress` de Minikube esté habilitado. En local, se añade una entrada en `/etc/hosts` apuntando la IP de Minikube al hostname.

---

## Construcción y versionado de la imagen

La imagen se construye directamente dentro del daemon Docker de Minikube para que Kubernetes pueda acceder a ella sin necesidad de un registry externo:

```bash
eval $(minikube docker-env)
docker build -t hello-world:1.0.0 practica/app/
```

El versionado se hace mediante tags en la imagen (`1.0.0`). En un entorno de producción, el tag correspondería a la versión del commit o release de Git, y la imagen se publicaría en un registry como Docker Hub o GitHub Container Registry.

---

## Cómo ejecutar la solución

### Requisitos previos
- Minikube instalado
- kubectl instalado
- Docker instalado

### Pasos

```bash
# 1. Arrancar Minikube y habilitar ingress
minikube start
minikube addons enable ingress

# 2. Construir la imagen dentro de Minikube
eval $(minikube docker-env)
docker build -t hello-world:1.0.0 practica/app/

# 3. Crear namespace y desplegar manifiestos
kubectl create namespace practica
kubectl apply -f practica/k8s/

# 4. Instalar Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 5. Aplicar la Application de Argo CD
kubectl apply -f practica/argocd/application.yaml

# 6. Acceder a la aplicación
kubectl port-forward svc/hello-world -n practica 8081:80
# Abrir http://localhost:8081
```

### Acceso a Argo CD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Abrir https://localhost:8080
# Usuario: admin
# Contraseña: kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

---

## GitOps con Argo CD

Argo CD monitoriza la carpeta `practica/k8s/` del repositorio. Cuando se hace un push con cambios en los manifiestos, Argo CD detecta la diferencia entre el estado deseado (Git) y el estado actual del clúster y los sincroniza automáticamente gracias a `syncPolicy.automated`.

La configuración incluye:
- **prune: true** — elimina recursos del clúster que hayan sido borrados de Git.
- **selfHeal: true** — revierte cambios manuales hechos directamente en el clúster que no estén en Git.

---

## Decisiones técnicas

- **nginx:alpine** como imagen base: es ligera (~8MB), segura y perfecta para servir contenido estático sin necesidad de un servidor de aplicaciones completo.
- **ClusterIP** en el Service: suficiente porque el acceso exterior se gestiona a través del Ingress, que es el patrón estándar en Kubernetes.
- **Separación de manifiestos por archivo**: cada recurso tiene su propio archivo YAML para facilitar el mantenimiento y la lectura, en lugar de un único archivo con todo concatenado.
- **Namespace `practica`**: aislar la aplicación en su propio namespace evita conflictos con otros recursos del clúster y sigue las buenas prácticas de organización.

---

## Mejoras futuras

- Publicar la imagen en un registry (Docker Hub / GHCR) para no depender del daemon local de Minikube.
- Añadir un `HorizontalPodAutoscaler` para escalar automáticamente según la carga.
- Configurar TLS en el Ingress con cert-manager para servir la app por HTTPS.
- Añadir un pipeline de CI/CD (GitHub Actions) que construya y publique la imagen automáticamente en cada push, actualizando el tag en el Deployment para que Argo CD lo sincronice.
- Usar `Kustomize` o `Helm` para gestionar configuraciones por entorno (dev, staging, prod).
