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

```bash
gcloud services enable \
  artifactregistry.googleapis.com \
  run.googleapis.com \
  iam.googleapis.com \
  compute.googleapis.com \
  cloudresourcemanager.googleapis.com
```

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

## 6. Permisos IAM

Para el laboratorio:

### Artifact Registry Writer

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$VM_SERVICE_ACCOUNT" \
  --role="roles/artifactregistry.writer"
```

### Cloud Run Developer

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$VM_SERVICE_ACCOUNT" \
  --role="roles/run.developer"
```

### Service Account User

```bash
gcloud iam service-accounts add-iam-policy-binding \
  "$VM_SERVICE_ACCOUNT" \
  --member="serviceAccount:$VM_SERVICE_ACCOUNT" \
  --role="roles/iam.serviceAccountUser"
```

> En producción se recomienda utilizar una Service Account dedicada y mínimo privilegio.

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

Crear `jenkins/Dockerfile`:

```dockerfile
FROM jenkins/jenkins:lts

USER root

RUN apt-get update && \
    apt-get install -y docker.io curl ca-certificates gnupg && \
    rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
    | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

RUN echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] \
    https://packages.cloud.google.com/apt cloud-sdk main" \
    > /etc/apt/sources.list.d/google-cloud-sdk.list

RUN apt-get update && \
    apt-get install -y google-cloud-cli && \
    rm -rf /var/lib/apt/lists/*

USER jenkins
```

Reconstruir:

```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

## 10. Validar Jenkins

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

El repositorio ya contiene el `Jenkinsfile`.

El pipeline debe realizar:

```
Checkout
   ↓
GCP Authentication
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

Se recomienda utilizar `BUILD_NUMBER` o el SHA corto del commit como tag.

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

La autenticación contra Google Cloud utiliza la identidad de la instancia de Compute Engine, evitando almacenar una clave JSON de Service Account dentro de Jenkins.
