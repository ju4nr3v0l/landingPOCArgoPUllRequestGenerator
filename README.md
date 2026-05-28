# Landing POC Argo Pull Request Generator

Repositorio de codigo para la landing de la POC.

En esta version del flujo, este repo ya no es la fuente GitOps consumida por Argo CD.

Su responsabilidad es:

- almacenar el codigo fuente de la landing
- construir una imagen OCI versionada en Docker Hub
- disparar GitHub Actions al abrir, actualizar o cerrar PRs
- permitir que GitHub Actions escriba metadata y manifests generados en el repo `infra`

## Estructura

```text
.
├── .github
│   └── workflows
│       ├── sync-preview-gitops.yaml
│       └── sync-prod-gitops.yaml
├── .dockerignore
├── Dockerfile
├── README.md
└── site
    ├── index.html
    └── style.css
```

## Flujo objetivo

### Ambiente estable

1. se hace merge a `main`
2. GitHub Actions construye y publica una imagen en Docker Hub
3. GitHub Actions genera o actualiza `infra/generated/environments/prod/landing` con la referencia de imagen por digest
4. Argo CD detecta cambios en el repo `infra`
5. se sincroniza `landing-prod`

### Ambiente efimero

1. se abre o actualiza un PR
2. si el PR tiene el label `preview`, GitHub Actions construye y publica una imagen del PR
3. GitHub Actions genera `infra/generated/previews/pr-<numero>` con la referencia de imagen por digest
4. Argo CD detecta esa carpeta por Git generator
5. crea una `Application` y un namespace efimero
6. al cerrar o mergear el PR, GitHub Actions elimina esa carpeta y Argo hace `prune`

## Secrets y variables que necesita GitHub Actions

### Secret requerido

- `INFRA_REPO_TOKEN`
  - token con permiso de escritura sobre `InfraPOCArgoPUllRequestGenerator`
  - ubicacion exacta: `landing repo > Settings > Secrets and variables > Actions > Repository secrets`
  - uso: el workflow lo usa solo para hacer `push` al repo `infra`

- `DOCKERHUB_USERNAME`
  - usuario de Docker Hub con permiso para publicar en `juanmarulanda/landingpocargoprpreview`
  - ubicacion exacta: `landing repo > Settings > Secrets and variables > Actions > Repository secrets`

- `DOCKERHUB_TOKEN`
  - access token o password de Docker Hub con permiso para publicar en `juanmarulanda/landingpocargoprpreview`
  - ubicacion exacta: `landing repo > Settings > Secrets and variables > Actions > Repository secrets`

### Variable opcional

- `PREVIEW_AZURE_CLIENT_ID`
  - si se define, el workflow agrega la anotacion de Azure Workload Identity al `ServiceAccount` generado para previews
  - ubicacion exacta: `landing repo > Settings > Secrets and variables > Actions > Variables`

## Workflows

### `sync-prod-gitops.yaml`

- se ejecuta en cada `push` a `main`
- construye la imagen de la landing
- publica imagen multi-arquitectura (`linux/amd64` y `linux/arm64`)
- renderiza `infra/generated/environments/prod/landing`
- hace commit al repo `infra`
- Argo CD sincroniza `landing-prod`

### `sync-preview-gitops.yaml`

- se ejecuta en eventos de `pull_request`
- si el PR tiene label `preview`, construye y publica una imagen del PR
- la imagen del PR tambien se publica como multi-arquitectura (`linux/amd64` y `linux/arm64`)
- si el PR tiene label `preview`, genera `infra/generated/previews/pr-<numero>`
- si el PR se cierra o pierde el label `preview`, elimina esa carpeta
- Argo CD crea o destruye el ambiente efimero en funcion del estado de Git

## Como visualizar la landing

### Prod

Ejecuta:

```bash
kubectl port-forward -n landing-prod svc/landingpage 8081:80
```

Luego abre:

- [http://localhost:8081](http://localhost:8081)

### Preview por PR

Cuando exista un preview para un PR, el namespace seguira este patron:

- `preview-pr-<numero>`

Y el servicio seguira este patron:

- `landingpage-pr-<numero>`

Ejemplo para el PR 7:

```bash
kubectl port-forward -n preview-pr-7 svc/landingpage-pr-7 8082:80
```

Luego abre:

- [http://localhost:8082](http://localhost:8082)

Si no recuerdas el numero del PR o quieres confirmar que el preview ya existe:

```bash
kubectl get applications -n argocd
kubectl get svc -n preview-pr-<numero>
```

## Troubleshooting rapido

### Error: `Input required and not supplied: token`

Ese error corresponde a una version vieja del workflow.

Valida que `main` del repo `landing` ya tenga el commit con el fix del workflow y vuelve a ejecutar el job.

### Error: `Missing INFRA_REPO_TOKEN`

Debes crear el secret:

- `INFRA_REPO_TOKEN`

en:

- `landing repo > Settings > Secrets and variables > Actions > Repository secrets`

### Error: `Missing Docker Hub secrets`

Debes crear estos secrets en el repo `landing`:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`

Ubicacion:

- `landing repo > Settings > Secrets and variables > Actions > Repository secrets`

### Error en Kubernetes: `no match for platform in manifest`

Ese error indica que la imagen publicada no incluia la arquitectura del nodo del cluster.

La configuracion actual ya construye imagenes multi-arquitectura:

- `linux/amd64`
- `linux/arm64`

## Archivo principal para cambiar la landing

- [site/index.html](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/site/index.html)
- [site/style.css](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/site/style.css)

## Notas

- este repo queda limpio de manifests Kubernetes
- la entrega hacia Argo CD ocurre por imagen OCI, no copiando `html` o `css` a `infra`
- todo lo derivado por ambiente se publica en `infra/generated/...`
- el repo `infra` se convierte en la unica fuente GitOps que Argo CD sincroniza
