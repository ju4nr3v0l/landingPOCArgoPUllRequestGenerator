# Landing POC Argo Pull Request Generator

Repositorio de codigo para la landing de la POC.

En esta version del flujo, este repo ya no es la fuente GitOps consumida por Argo CD.

Su responsabilidad es:

- almacenar el codigo fuente de la landing
- disparar GitHub Actions al abrir, actualizar o cerrar PRs
- permitir que GitHub Actions escriba metadata y manifests generados en el repo `infra`

## Estructura

```text
.
├── .github
│   └── workflows
│       ├── sync-preview-gitops.yaml
│       └── sync-prod-gitops.yaml
├── README.md
└── site
    ├── index.html
    └── style.css
```

## Flujo objetivo

### Ambiente estable

1. se hace merge a `main`
2. GitHub Actions genera o actualiza `infra/generated/environments/prod/landing`
3. Argo CD detecta cambios en el repo `infra`
4. se sincroniza `landing-prod`

### Ambiente efimero

1. se abre o actualiza un PR
2. si el PR tiene el label `preview`, GitHub Actions genera `infra/generated/previews/pr-<numero>`
3. Argo CD detecta esa carpeta por Git generator
4. crea una `Application` y un namespace efimero
5. al cerrar o mergear el PR, GitHub Actions elimina esa carpeta y Argo hace `prune`

## Secrets y variables que necesita GitHub Actions

### Secret requerido

- `INFRA_REPO_TOKEN`
  - token con permiso de escritura sobre `InfraPOCArgoPUllRequestGenerator`

### Variable opcional

- `PREVIEW_AZURE_CLIENT_ID`
  - si se define, el workflow agrega la anotacion de Azure Workload Identity al `ServiceAccount` generado para previews

## Archivo principal para cambiar la landing

- [site/index.html](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/site/index.html)
- [site/style.css](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/site/style.css)

## Notas

- este repo queda limpio de manifests Kubernetes
- todo lo derivado por ambiente se publica en `infra/generated/...`
- el repo `infra` se convierte en la unica fuente GitOps que Argo CD sincroniza
