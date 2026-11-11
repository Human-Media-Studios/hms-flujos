# hms-flujos

Flujos de GitHub Actions reutilizables de Human Media Studios. Sirven para que cada repositorio
gaste pocos minutos y poco espacio, siguiendo las reglas de la skill `flujos-github-hms`.

No guarda llaves. Cada repositorio le pasa las suyas como `secrets`.

| Flujo | Para qué | Cuándo llamarlo |
|---|---|---|
| `ci.yml` | Build y pruebas rápidas (y E2E si se pide). Es la puerta del despliegue. | En cada pull request y push a `main`, con `paths-ignore`. |
| `revision.yml` | Lo pesado: capturas visuales, Lighthouse, accesibilidad. | Solo a mano o con la etiqueta `revision-completa` en el PR. |
| `desplegar-siteground.yml` | Build y subida por rsync a SiteGround, sin borrar nada del servidor. | Solo desde `main`, después de que el CI pase (`workflow_run`). |

Todos traen `concurrency`, límite de minutos, caché por lockfile, informes solo si algo falla
(guardados 3 días) y acciones fijadas por SHA.

## Cómo usarlo

Fija siempre la versión por SHA (el de la etiqueta `v1`), no por rama:

```yaml
# .github/workflows/ci.yml del repositorio
name: CI
on:
  push:
    branches: [main]
    paths-ignore: ['*.md', 'docs/**']
  pull_request:
    branches: [main]
    paths-ignore: ['*.md', 'docs/**']
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: read
jobs:
  ci:
    uses: Human-Media-Studios/hms-flujos/.github/workflows/ci.yml@<SHA> # v1
    with:
      comandos: |
        npm run build
        npm run test:unit
      playwright: true
      e2e: npm run test:e2e
      artefactos: tests/.results/playwright-report/
```

```yaml
# .github/workflows/revision.yml del repositorio
on:
  pull_request:
    types: [labeled]
  workflow_dispatch:
jobs:
  revision:
    if: github.event_name == 'workflow_dispatch' || github.event.label.name == 'revision-completa'
    uses: Human-Media-Studios/hms-flujos/.github/workflows/revision.yml@<SHA> # v1
    with:
      lighthouse: npm run test:perf
      visual: npm run test:visual
```

```yaml
# .github/workflows/deploy.yml del repositorio
on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]
jobs:
  desplegar:
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'push'
    uses: Human-Media-Studios/hms-flujos/.github/workflows/desplegar-siteground.yml@<SHA> # v1
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      ruta: www/midominio.com/public_html
    secrets:
      ssh-key: ${{ secrets.SG_SSH_KEY }}
      ssh-host: ${{ secrets.SG_SSH_HOST }}
      ssh-user: ${{ secrets.SG_SSH_USER }}
      ssh-port: ${{ secrets.SG_SSH_PORT }}
```

Las opciones de cada flujo están descritas al inicio de su archivo, en `inputs` y `secrets`.

## Ojo con `paths-ignore`

Ignora solo `*.md` de la raíz y `docs/**`. No uses `**/*.md`: los blogs en Astro viven en
`src/**/*.md` y un cambio ahí sí debe probarse y desplegarse.

## Permiso de la organización

Las organizaciones de HMS solo dejan usar acciones de GitHub, verificadas y una lista corta. Para
llamar estos flujos, la organización debe tener en su lista permitida:
`Human-Media-Studios/hms-flujos/.github/workflows/*@*`.
