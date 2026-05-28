# Landing POC Argo Pull Request Generator

Repositorio de frontend para la POC de previews por Pull Request con Argo CD.

## Contenido

- `k8s/base`: manifiestos base y contenido estatico de la landing
- `k8s/prod`: overlay para el entorno estable
- `k8s/preview`: overlay para previews por PR

## Flujo

Argo CD sincroniza `k8s/prod` desde `main`, y el `ApplicationSet` del repo de infraestructura crea previews usando `k8s/preview` para cada pull request etiquetado con `preview`.
