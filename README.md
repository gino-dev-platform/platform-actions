# platform-actions

Workflows reutilizables de CI/CD para todos los repositorios de la plataforma. Centraliza la lógica de build y deploy para eliminar duplicación entre repos.

## Workflows disponibles

### `build-and-push.yml`

Buildea una imagen Docker, la publica en GHCR y actualiza el tag en `platform-gitops`.

**Uso desde cualquier repo de aplicación:**

```yaml
jobs:
  deploy:
    uses: gino-dev-platform/platform-actions/.github/workflows/build-and-push.yml@main
    with:
      image_name: mi-repo-backend
      kustomization_path: applications/mi-proyecto/overlays/development/kustomization.yaml
    secrets:
      GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}
```

**Inputs:**

| Input | Requerido | Default | Descripción |
|---|---|---|---|
| `image_name` | ✅ | — | Nombre de la imagen sin prefijo de registry |
| `kustomization_path` | ✅ | — | Path al `kustomization.yaml` en platform-gitops |
| `registry` | ❌ | `ghcr.io` | Container registry |
| `org` | ❌ | `gino-dev-platform` | Org del registry |
| `gitops_repo` | ❌ | `gino-dev-platform/platform-gitops` | Repo GitOps |
| `gitops_username` | ❌ | `ginojc` | Usuario para push al repo GitOps |

**Secrets:**

| Secret | Descripción |
|---|---|
| `GHCR_TOKEN` | PAT con `packages:write` y acceso de escritura a platform-gitops |

## Requisitos

Este repo debe ser **público** en GitHub para que los workflows puedan ser referenciados desde repos privados de la misma organización (limitación del plan gratuito de GitHub).

## Repositorios relacionados

- [template-backend](https://github.com/gino-dev-platform/template-backend) — API REST NestJS
- [template-frontend](https://github.com/gino-dev-platform/template-frontend) — frontend Next.js
- [platform-gitops](https://github.com/gino-dev-platform/platform-gitops) — manifiestos Kubernetes
- [platform-manager](https://github.com/gino-dev-platform/platform-manager) — provisioner de proyectos
