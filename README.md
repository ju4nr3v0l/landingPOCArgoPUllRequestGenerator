# Landing POC Argo Pull Request Generator

Repositorio de frontend para una POC de ambientes efimeros por Pull Request con Argo CD.

La idea central es simple:

- `main` representa el ambiente estable
- cada PR etiquetado con `preview` genera un ambiente efimero
- cuando el PR se cierra o se mergea, el ambiente efimero desaparece

## Objetivo de la POC

Validar un flujo GitOps donde el codigo de la landing vive en un repo separado del repo de infraestructura, pero Argo CD despliega ambos escenarios:

- ambiente estable desde `main`
- ambiente efimero desde el `head_sha` de cada PR

## Arquitectura funcional

Este repo no usa un frontend compilado con `npm`, `vite` o `react`.

Para acelerar la POC, la landing se publica como contenido estatico servido por `nginx`:

- HTML y CSS viven en `k8s/base/site/`
- Kustomize genera un `ConfigMap`
- el `Deployment` monta ese `ConfigMap` en `/usr/share/nginx/html`
- Argo CD sincroniza `k8s/prod`
- `ApplicationSet` sincroniza `k8s/preview` para cada PR

## Estructura

```text
.
├── README.md
└── k8s
    ├── base
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   ├── service.yaml
    │   └── site
    │       ├── index.html
    │       └── style.css
    ├── preview
    │   └── kustomization.yaml
    └── prod
        └── kustomization.yaml
```

## Archivos clave

- [k8s/base/site/index.html](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/k8s/base/site/index.html): contenido principal de la landing
- [k8s/base/site/style.css](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/k8s/base/site/style.css): estilos visuales
- [k8s/base/deployment.yaml](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/k8s/base/deployment.yaml): `Deployment` con `nginx`
- [k8s/base/kustomization.yaml](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/k8s/base/kustomization.yaml): generacion del `ConfigMap`
- [k8s/prod/kustomization.yaml](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/k8s/prod/kustomization.yaml): overlay estable
- [k8s/preview/kustomization.yaml](/Users/juanmarulanda/Documents/POCArgoPull%20Request%20Generator/landing/k8s/preview/kustomization.yaml): overlay de previews

## Prerrequisitos

Para modificar este repo no necesitas nada especial aparte de Git.

Para validar localmente la POC completa con Argo CD, el workspace global si necesita:

- Docker Desktop
- `kubectl`
- `kind`
- `argocd` CLI opcional
- acceso al repo de infraestructura
- Argo CD instalado en el cluster local

## Flujo operativo

### 1. Cambio de producto o UI

Haz cambios en:

- `k8s/base/site/index.html`
- `k8s/base/site/style.css`

Como `prod` y `preview` reutilizan `k8s/base`, un cambio alli impacta ambos flujos.

### 2. Commit y push

Trabaja sobre una rama:

```bash
git switch -c feature/cambio-landing
git add .
git commit -m "Ajusta texto de landing"
git push -u origin feature/cambio-landing
```

### 3. Pull Request

Abre un PR hacia `main`.

Si quieres ambiente efimero, agrega el label:

- `preview`

### 4. Resultado esperado

- Argo CD detecta el PR por polling
- `ApplicationSet` crea una app tipo `landing-pr-<numero>`
- se crea un namespace `preview-pr-<numero>`
- el `targetRevision` apunta al commit exacto del PR

### 5. Merge

Cuando el PR se mergea:

- `main` se actualiza
- la app `landing-prod` se resincroniza automaticamente
- el ambiente efimero deja de cumplir el filtro del generador
- Argo elimina la `Application` efimera y sus recursos

## Como validar el ambiente estable

Si el repo de infraestructura ya esta aplicado en Argo CD:

```bash
kubectl port-forward -n landing-prod svc/landingpage 8081:80
```

Abre:

- [http://localhost:8081](http://localhost:8081)

## Como validar el ambiente efimero de un PR

Supongamos que el PR es el `2`:

```bash
kubectl port-forward -n preview-pr-2 svc/landingpage-pr-2 8082:80
```

Abre:

- [http://localhost:8082](http://localhost:8082)

## Convenciones de esta POC

- `main` representa el ambiente estable
- el label `preview` habilita ambiente efimero
- los previews usan `nameSuffix` por numero de PR
- el namespace de preview es desechable
- el contenido se publica desde `ConfigMap`, no desde una imagen propia

## Beneficios esperados en Sistecredito

En un contexto como Sistecredito, donde producto, QA, arquitectura, seguridad y desarrollo necesitan validar cambios con rapidez, este enfoque puede aportar:

- menos dependencia de ambientes compartidos para revisar cambios pequeños de frontend
- menos friccion entre desarrollo y QA al tener una URL por PR
- trazabilidad clara entre commit, PR y ambiente desplegado
- menor riesgo de validar una rama equivocada
- menor tiempo de espera para demo interna de cambios

## Ahorro potencial de costo y tiempo

Esta POC no pretende dar una cifra financiera exacta, pero si un marco de estimacion.

### Ahorro operativo directo

Si hoy una validacion manual requiere:

- pedir despliegue a otra persona
- esperar una ventana de ambiente compartido
- coordinar QA sobre una rama temporal

entonces cada PR con preview puede ahorrar entre 10 y 30 minutos de coordinacion.

### Escenarios de referencia

- 40 PRs al mes x 10 min = 6.7 horas/mes evitadas
- 80 PRs al mes x 20 min = 26.7 horas/mes evitadas
- 120 PRs al mes x 30 min = 60 horas/mes evitadas

La forma recomendada de convertir eso a dinero en Sistecredito es:

`PRs/mes x minutos ahorrados por PR x costo blended por hora del equipo`

### Otros costos evitados

- menos necesidad de ambientes largos por rama
- menos reprocesos por diferencias entre QA y desarrollo
- menos costo de oportunidad por esperas entre equipos

## Riesgos y limites de esta POC

- el contenido de la landing esta en `ConfigMap`; esto es util para demo, pero no es el patron final ideal para apps web mas grandes
- no hay pipeline de build ni tests automatizados en este repo
- un cambio de HTML o CSS queda acoplado a la estructura Kubernetes
- no hay `Ingress` por hostname; el acceso se hace por `port-forward`
- la deteccion del PR hoy depende de polling, no de webhook

## Recomendaciones de mejores practicas

- evolucionar de `ConfigMap` a imagen versionada cuando la app deje de ser una landing minima
- agregar pruebas de smoke o snapshots antes de merge
- usar convenciones de ramas y labels simples y consistentes
- mantener los overlays de `prod` y `preview` tan delgados como sea posible
- no meter secretos en este repo
- tratar este repo solo como fuente de la app, no como fuente de credenciales o configuracion sensible

## Roadmap sugerido para este repo

### Fase 1. Consolidar la POC

- mantener la landing estatica
- probar varios PRs concurrentes
- validar cleanup automatico despues de merge y cierre

### Fase 2. Hacer la app mas real

- mover la landing a Vite o framework equivalente
- generar artefacto compilado
- publicar imagen en registry
- desplegar imagen por tag o digest

### Fase 3. Calidad y seguridad

- agregar tests de UI
- agregar escaneo de dependencias
- fijar imagenes por digest
- agregar politica de revision minima antes de merge

### Fase 4. Uso organizacional

- convertir este patron en plantilla reutilizable para otros frontends
- publicar lineamientos de nombres, labels y ownership
- integrar el preview en el flujo de QA y demo

## Dependencias con el repo de infraestructura

Este repo depende de que el repo de `infra` publique:

- una `Application` estable apuntando a `k8s/prod`
- un `ApplicationSet` apuntando a `k8s/preview`
- un `github-token` en el namespace `argocd`

Repositorio relacionado:

- [InfraPOCArgoPUllRequestGenerator](https://github.com/ju4nr3v0l/InfraPOCArgoPUllRequestGenerator)

## Referencias oficiales

- [Argo CD Automated Sync](https://argo-cd.readthedocs.io/en/release-3.2/user-guide/auto_sync/)
- [Argo CD Sync Options](https://argo-cd.readthedocs.io/en/release-3.4/user-guide/sync-options/)
- [Argo CD ApplicationSet](https://argo-cd.readthedocs.io/en/release-3.4/operator-manual/applicationset/)
