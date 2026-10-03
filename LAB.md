# LAB — Jenkins + Docker + Artifact Registry + Cloud Run

## Objetivo

Implementar CI/CD para `aldo2510/angular-cloud-run-v2`:

```
GitHub → Webhook → Jenkins (Compute Engine) → Docker Build → Artifact Registry → Cloud Run
```

Jenkins corre dentro de Docker Compose en la VM `instance-20261003-145115`.

## Valores

| Componente | Valor |
|---|---|
| GCP Project | `sanbox-aldo-prod` |
| Región | `us-central1` |
| Compute Engine | `instance-20261003-145115` |
| Zona | `us-central1-a` |
| Artifact Registry | `container-repository-gemini-at` |
| Imagen | `gemini-angular-app` |
| Cloud Run | `gemini-angular-app-dev` |
| Environment | `dev` |
| GitHub | `aldo2510/angular-cloud-run-v2` |

## Arquitectura

```
GitHub
   │ push
   ▼
Webhook
   │
   ▼
Jenkins en Compute Engine
   │
   ├── Checkout
   ├── Docker Build
   ├── Docker Push
   └── gcloud run deploy
          │
          ├── Artifact Registry
          └── Cloud Run
```

## 1. Prerrequisitos

En la VM:

```bash
docker --version
docker compose version
gcloud --version
```

## 2. Configurar proyecto

```bash
export PROJECT_ID="sanbox-aldo-prod"
export REGION="us-central1"
export INSTANCE="instance-20261003-145115"
export ARTIFACT_REPOSITORY="container-repository-gemini-at"
export CLOUD_RUN_SERVICE="gemini-angular-app-dev"

gcloud config set project "$PROJECT_ID"
```

## 3. Habilitar APIs

> **Importante:** habilitar APIs es una tarea administrativa y **no debe requerir permisos administrativos para la Service Account de Jenkins**.
>
> En este laboratorio existe la Service Account:
>
> `jenkins-deployer@sanbox-aldo-prod.iam.gserviceaccount.com`
>
> Si `gcloud auth list` muestra esta cuenta como activa y aparece el error `serviceusage.services.enable / AUTH_PERMISSION_DENIED`, cambia temporalmente a una cuenta de usuario con permisos administrativos para habilitar las APIs.

Verificar la identidad activa:

```bash
gcloud auth list
gcloud config get-value account
```

Habilitar las APIs con una cuenta que tenga permisos para administrar Service Usage:

```bash
gcloud services enable \
  artifactregistry.googleapis.com \
  run.googleapis.com \
  iam.googleapis.com \
  compute.googleapis.com \
  cloudresourcemanager.googleapis.com
```

La Service Account de Jenkins **no necesita** `roles/owner` ni `roles/serviceusage.serviceUsageAdmin` para ejecutar el pipeline.

## 4. Artifact Registry

Verificar:

```bash
gcloud artifacts repositories describe \
  "$ARTIFACT_REPOSITORY" \
  --location="$REGION"
```

Si no existe:

```bash
gcloud artifacts repositories create "$ARTIFACT_REPOSITORY" \
  --repository-format=docker \
  --location="$REGION" \
  --description="Docker images for Angular Cloud Run LAB"
```

## 5. Identidad de la VM

Obtener la Service Account:

```bash
curl -H "Metadata-Flavor: Google" \
"http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email"
```

También:

```bash
gcloud compute instances describe "$INSTANCE" \
  --zone="us-central1-a" \
  --format="value(serviceAccounts.email)"
```

Guardar el resultado:

```bash
export VM_SERVICE_ACCOUNT="YOUR-SERVICE-ACCOUNT"
```

## 6. Permisos IAM de Jenkins

En este laboratorio se utiliza la Service Account dedicada:

```text
jenkins-deployer@sanbox-aldo-prod.iam.gserviceaccount.com
```

Definir:

```bash
export JENKINS_SERVICE_ACCOUNT="jenkins-deployer@sanbox-aldo-prod.iam.gserviceaccount.com"
```

### Artifact Registry Writer

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$JENKINS_SERVICE_ACCOUNT" \
  --role="roles/artifactregistry.writer"
```

### Cloud Run Developer

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$JENKINS_SERVICE_ACCOUNT" \
  --role="roles/run.developer"
```

### Service Account User

Cloud Run necesita que la identidad que realiza el despliegue pueda actuar como la Service Account usada por el servicio de Cloud Run. Para el laboratorio, si se utiliza la misma Service Account como identidad de runtime, puede configurarse:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  "$JENKINS_SERVICE_ACCOUNT" \
  --member="serviceAccount:$JENKINS_SERVICE_ACCOUNT" \
  --role="roles/iam.serviceAccountUser"
```

Verificar los permisos:

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:$JENKINS_SERVICE_ACCOUNT" \
  --format="table(bindings.role)"
```

> **No otorgar `roles/owner` a Jenkins.** Para producción se recomienda una Service Account dedicada, permisos mínimos y una Service Account separada para el runtime de Cloud Run.

## 7. Probar Artifact Registry

```bash
gcloud auth configure-docker \
  "$REGION-docker.pkg.dev" \
  --quiet

docker pull nginx:alpine

docker tag nginx:alpine \
  "$REGION-docker.pkg.dev/$PROJECT_ID/$ARTIFACT_REPOSITORY/test-nginx:latest"

docker push \
  "$REGION-docker.pkg.dev/$PROJECT_ID/$ARTIFACT_REPOSITORY/test-nginx:latest"
```

Validar:

```bash
gcloud artifacts docker images list \
  "$REGION-docker.pkg.dev/$PROJECT_ID/$ARTIFACT_REPOSITORY"
```

## 8. Jenkins Docker Compose

Jenkins debe tener acceso al Docker socket del host.

Ejemplo:

```yaml
services:
  jenkins:
    build:
      context: ./jenkins
    container_name: jenkins
    restart: unless-stopped

    ports:
      - "8080:8080"
      - "50000:50000"

    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  jenkins_home:
```

## 9. Imagen personalizada de Jenkins

El Dockerfile del laboratorio instala **Docker CLI, Docker Compose y Google Cloud CLI**. No instala el Docker daemon porque Jenkins utiliza el socket del Docker Engine del host.

Crear `jenkins/Dockerfile`:

```dockerfile
FROM jenkins/jenkins:lts

USER root

RUN apt-get update && apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https \
    git \
    unzip \
    && rm -rf /var/lib/apt/lists/*

# Docker CLI
RUN mkdir -p /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | \
    gpg --dearmor -o /etc/apt/keyrings/docker.gpg

RUN echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null

RUN apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

# Docker Compose
RUN mkdir -p /usr/local/lib/docker/cli-plugins/ && \
    curl -SL \
    https://github.com/docker/compose/releases/download/v2.26.1/docker-compose-linux-x86_64 \
    -o /usr/local/lib/docker/cli-plugins/docker-compose && \
    chmod +x /usr/local/lib/docker/cli-plugins/docker-compose && \
    ln -s /usr/local/lib/docker/cli-plugins/docker-compose /usr/local/bin/docker-compose

# Google Cloud CLI
RUN curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
    gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

RUN echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] \
    https://packages.cloud.google.com/apt cloud-sdk main" | \
    tee /etc/apt/sources.list.d/google-cloud-sdk.list

RUN apt-get update && \
    apt-get install -y google-cloud-cli && \
    rm -rf /var/lib/apt/lists/*

# Permitir que Jenkins use Docker CLI
RUN groupadd -f docker && \
    usermod -aG docker jenkins

USER jenkins

# Verificación durante el build
RUN docker --version && \
    docker compose version && \
    gcloud --version
```

Reconstruir:

```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

Reconstruir:

```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

## 10. Validar Jenkins

Verificar que las herramientas estén disponibles:

```bash
docker exec -it jenkins docker --version
docker exec -it jenkins docker compose version
docker exec -it jenkins gcloud --version
```

> El acceso al Docker Engine se obtiene mediante `/var/run/docker.sock`. La autenticación de Google Cloud debe provenir de la identidad configurada para la VM/Jenkins; no se debe guardar una clave JSON dentro de la imagen.



```bash
docker compose ps
docker exec -it jenkins docker --version
docker exec -it jenkins gcloud --version
```

Validar identidad de Compute Engine desde Jenkins:

```bash
docker exec -it jenkins curl -H "Metadata-Flavor: Google" \
"http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email"
```

Debe devolver la Service Account de la VM.

## 11. Jenkinsfile

El repositorio contiene el `Jenkinsfile` que ejecuta el pipeline completo.

### Autenticación de Artifact Registry

**Importante:** `gcloud auth configure-docker` debe ejecutarse en el **mismo agente Jenkins** que realiza `docker build` y `docker push`.

El flujo correcto es:

```
Checkout
   ↓
GCP Authentication
   ↓
gcloud auth configure-docker
   ↓
Docker Build
   ↓
Docker Push → Artifact Registry
   ↓
Cloud Run Deploy
```

Ejemplo:

```groovy
stage('GCP & Docker Auth') {
    steps {
        sh """
            gcloud config set project ${GCP_PROJECT_ID}
            gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev --quiet
        """
    }
}

stage('Build and Push Image') {
    steps {
        sh """
            docker build -t ${IMAGE_TAG} .
            docker push ${IMAGE_TAG}
        """
    }
}
```

No se debe ejecutar la autenticación dentro de un agente Docker temporal diferente del agente que realizará el `docker push`. De hacerlo, el `~/.docker/config.json` puede quedar dentro del contenedor temporal y no estar disponible para el agente del build/push.

El archivo de Docker de Jenkins debe quedar configurado con el helper de Artifact Registry:

```json
{
  "credHelpers": {
    "us-central1-docker.pkg.dev": "gcloud"
  }
}
```

### Error `Unauthenticated request`

Si el pipeline construye la imagen correctamente pero `docker push` termina con:

```text
error from registry: Unauthenticated request
Unauthenticated requests do not have permission "artifactregistry.repositories.uploadArtifacts"
```

verificar desde la VM:

```bash
docker exec -it jenkins gcloud auth list
docker exec -it jenkins gcloud config get-value account
docker exec -it jenkins gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
docker exec -it jenkins cat /var/jenkins_home/.docker/config.json
```

La cuenta esperada es:

```text
jenkins-deployer@sanbox-aldo-prod.iam.gserviceaccount.com
```

> Si el `docker push` funciona directamente desde la VM pero falla desde Jenkins, revisar primero la configuración de autenticación Docker dentro del agente Jenkins. No es necesario generar una clave JSON para este laboratorio.

El pipeline debe realizar:

```
Checkout
   ↓
GCP Authentication
   ↓
gcloud auth configure-docker
   ↓
Docker Build
   ↓
Docker Push
   ↓
Cloud Run Deploy
```

La imagen debe utilizar el formato:

```
us-central1-docker.pkg.dev/sanbox-aldo-prod/container-repository-gemini-at/gemini-angular-app:<TAG>
```

Se recomienda utilizar el SHA corto del commit como tag.

## 12. Configurar Jenkins

En Jenkins:

```
New Item
→ Pipeline
→ Pipeline script from SCM
→ Git
```

Repository:

```
https://github.com/aldo2510/angular-cloud-run-v2.git
```

Branch:

```
*/main
```

Script Path:

```
Jenkinsfile
```

## 13. Ejecutar manualmente

Seleccionar:

```
Build Now
```

Esperado:

```
Checkout
GCP Authentication
Build Image
Push Image
Deploy to Cloud Run
```

## 14. Validar Artifact Registry

```bash
gcloud artifacts docker images list \
  "$REGION-docker.pkg.dev/$PROJECT_ID/$ARTIFACT_REPOSITORY"
```

Debe aparecer `gemini-angular-app`.

## 15. Validar Cloud Run

```bash
gcloud run services describe \
  "$CLOUD_RUN_SERVICE" \
  --region="$REGION"
```

Obtener URL:

```bash
gcloud run services describe \
  "$CLOUD_RUN_SERVICE" \
  --region="$REGION" \
  --format="value(status.url)"
```

Revisar revisiones:

```bash
gcloud run revisions list \
  --service="$CLOUD_RUN_SERVICE" \
  --region="$REGION"
```

## 16. GitHub Webhook

En GitHub:

```
Settings
→ Webhooks
→ Add webhook
```

URL para laboratorio:

```
http://PUBLIC_IP:8080/github-webhook/
```

Content type:

```
application/json
```

Evento:

```
Just the push event
```

Activar `Active`.

> Si Jenkins solo está en HTTP, GitHub debe poder alcanzar públicamente la URL del webhook. Para producción se recomienda HTTPS mediante reverse proxy o Load Balancer.

## 17. Firewall de laboratorio

Si es necesario:

```bash
gcloud compute firewall-rules create allow-jenkins-8080 \
  --allow=tcp:8080 \
  --target-tags=jenkins \
  --description="Allow Jenkins HTTP for LAB"

gcloud compute instances add-tags "$INSTANCE" \
  --zone="us-central1-a" \
  --tags=jenkins
```

## 18. Prueba end-to-end

Realizar un cambio:

```bash
git add .
git commit -m "test: trigger Cloud Run deployment"
git push origin main
```

Flujo esperado:

```
GitHub
  ↓
Webhook
  ↓
Jenkins
  ↓
Docker Build
  ↓
Docker Push
  ↓
Artifact Registry
  ↓
Cloud Run
```

## 19. Troubleshooting

### Docker no encontrado

```
docker: command not found
```

Probar:

```bash
docker exec -it jenkins docker --version
```

### Docker daemon inaccesible

```
Cannot connect to the Docker daemon
```

Probar:

```bash
docker exec -it jenkins ls -l /var/run/docker.sock
```

Debe existir `/var/run/docker.sock`.

### Artifact Registry Permission Denied

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:$VM_SERVICE_ACCOUNT" \
  --format="table(bindings.role)"
```

Debe incluir:

```
roles/artifactregistry.writer
```

### Cloud Run Permission Denied

Debe incluir:

```
roles/run.developer
```

### iam.serviceAccounts.actAs

Verificar:

```bash
gcloud iam service-accounts get-iam-policy \
  "$VM_SERVICE_ACCOUNT"
```

Debe existir `roles/iam.serviceAccountUser`.

### Cloud Run no inicia

```bash
gcloud run services logs read \
  "$CLOUD_RUN_SERVICE" \
  --region="$REGION" \
  --limit=100
```

## 20. Checklist

- [ ] VM funcionando
- [ ] Docker instalado
- [ ] Docker Compose funcionando
- [ ] Jenkins funcionando
- [ ] Jenkins tiene acceso a Docker socket
- [ ] Docker CLI disponible en Jenkins
- [ ] Google Cloud CLI disponible en Jenkins
- [ ] VM tiene Service Account
- [ ] Artifact Registry Writer configurado
- [ ] Cloud Run Developer configurado
- [ ] Service Account User configurado
- [ ] APIs habilitadas
- [ ] Artifact Registry creado
- [ ] Jenkinsfile configurado
- [ ] Jenkins Pipeline creado
- [ ] GitHub Webhook configurado
- [ ] Jenkins accesible desde GitHub
- [ ] Cloud Run desplegado

## Resultado final

Un:

```bash
git push origin main
```

debe producir automáticamente:

```
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Docker Build
   ↓
Docker Push
   ↓
Artifact Registry
   ↓
Cloud Run
```

La autenticación contra Google Cloud debe utilizar una identidad administrada por GCP (preferentemente la Service Account asociada a la VM mediante metadata/ADC), evitando almacenar una clave JSON dentro de Jenkins. En este laboratorio la identidad de despliegue prevista es `jenkins-deployer@sanbox-aldo-prod.iam.gserviceaccount.com`. La cuenta utilizada para habilitar APIs puede ser diferente y debe tener permisos administrativos.
