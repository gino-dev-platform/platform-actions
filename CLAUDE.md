# CLAUDE.md — platform-actions

## Propósito

Repositorio de CI/CD centralizado. Contiene reusable workflows y custom actions
reutilizables por todos los repos de aplicación de la plataforma.

## Problema que resuelve

Antes de este repo, `template-backend` y `template-frontend` tenían workflows
casi idénticos de ~60 líneas cada uno. Cualquier cambio (nuevo registry, nuevo
cluster, lógica de tagging) requería editar ambos por separado.

Con `platform-actions`, los repos de app tienen workflows de ~10 líneas que
simplemente invocan el reusable workflow con sus parámetros.

## Estructura

```
platform-actions/
└── .github/
    └── workflows/
        └── build-and-push.yml    # reusable workflow (workflow_call)
```

## Reusable Workflows

### `build-and-push.yml`

Invocado con `uses: gino-dev-platform/platform-actions/.github/workflows/build-and-push.yml@main`

**Inputs:**

| Input | Requerido | Default | Descripción |
|---|---|---|---|
| `image_name` | ✅ | — | Nombre de la imagen sin prefijo de registry (ej. `template-backend`) |
| `kustomization_path` | ✅ | — | Path al `kustomization.yaml` dentro de `platform-gitops` |
| `registry` | ❌ | `ghcr.io` | Container registry |
| `org` | ❌ | `gino-dev-platform` | Org / namespace del registry |
| `gitops_repo` | ❌ | `gino-dev-platform/platform-gitops` | Slug del repo GitOps |
| `gitops_username` | ❌ | `ginojc` | Usuario git para push al repo GitOps |

**Secrets:**

| Secret | Requerido | Descripción |
|---|---|---|
| `GHCR_TOKEN` | ✅ | PAT con permisos `packages:write` y `contents:write` en platform-gitops |

**Qué hace:**

1. Checkout del repo de la aplicación
2. Genera el tag `sha_short` del commit (7 caracteres)
3. Login a GHCR
4. Build de la imagen con dos tags: `latest` y `{sha_short}`
5. Push de ambas imágenes
6. Clona `platform-gitops`, actualiza `newTag` en el kustomization.yaml indicado y hace push

**Cómo se actualiza el tag en kustomization.yaml:**

```bash
# Usa | como delimitador en sed para evitar escapar / en el nombre de imagen
sed -i "\|name: ghcr.io/gino-dev-platform/{image_name}|{n;s/newTag:.*/newTag: \"{sha}\"/;}" kustomization.yaml
```

El valor siempre queda entre comillas dobles para evitar que YAML interprete SHAs
numéricos como enteros (bug conocido que generó el error de Kustomize en el pasado).

## Cómo lo usan los repos de aplicación

```yaml
# template-backend/.github/workflows/deploy.yml
jobs:
  deploy:
    uses: gino-dev-platform/platform-actions/.github/workflows/build-and-push.yml@main
    with:
      image_name: template-backend
      kustomization_path: applications/template/overlays/development/kustomization.yaml
    secrets:
      GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}
```

El secret `GHCR_TOKEN` debe estar configurado en cada repo de aplicación
(Settings → Secrets and variables → Actions).

## Prerequisitos para que funcione

1. El repo `platform-actions` debe ser **público** en GitHub (o la org debe tener
   GitHub Enterprise para compartir workflows entre repos privados de la misma org).
2. El secret `GHCR_TOKEN` en los repos de app debe tener permisos:
   - `read:packages` + `write:packages` → para push a GHCR
   - `contents:write` en `platform-gitops` → para el push del tag actualizado

## Cómo agregar nuevos workflows

Crear un nuevo archivo en `.github/workflows/` con `on: workflow_call`.
Cada nuevo workflow sigue el mismo patrón: inputs + secrets + jobs.

## Cómo testear workflows localmente

Usar [`act`](https://github.com/nektos/act) para simular GitHub Actions sin push real:

```bash
# Instalar act (Windows)
winget install nektos.act

# Simular el workflow de build (requiere Docker)
act push --workflows .github/workflows/build-and-push.yml
```

`act` no soporta todos los features de GitHub Actions (ej. `workflow_call`),
pero sirve para validar la sintaxis y pasos básicos antes de hacer push.
