# Gedeelde workflows

Herbruikbare GitHub Actions-workflows voor de repositories van developer.overheid.nl. Een wijziging
hier geldt voor alle repo's die de workflow aanroepen. Je hoeft hem dus niet in elke repo apart aan
te passen.

Roep een workflow aan via de tag `v1`:

```yaml
uses: developer-overheid-nl/.github/.github/workflows/<workflow>.yml@v1
```

Na een merge naar `main` verplaats je de tag om de wijziging uit te rollen:

```bash
git tag -f v1 && git push -f origin v1
```

Bij een wijziging die niet compatibel is met bestaande aanroepen maak je een nieuwe major-tag (`v2`).

## deploy.yml

Bouwt de Docker-image(s) en zet de image-tag in de infra-repository.

| `environment` | Image-tags | Infra-repository |
| --- | --- | --- |
| `test` | `test`, `test-<sha>` | commit direct op `main` (test-overlay) |
| `prod` | `latest`, `<sha>` | release-PR `release/<repo>`; oudere release-PR's van de repo worden gesloten |

Een test-deploy start bij een handmatige run (`workflow_dispatch`) of als de commit-message
`[deploy-test]` bevat. Vanaf `main`, `changeset-release/**`, `changesets-ghcommit-temp/**` en `dependabot/**`
start nooit een test-deploy. Test-images hebben een eigen tag (`test-<sha>`), zodat een test-build nooit de
image overschrijft die productie deployt.

```yaml
# .github/workflows/deploy-test.yml
name: Deploy to Test

on:
  workflow_dispatch:
  push:
    branches-ignore:
      - main
      - "changeset-release/**"
      - "changesets-ghcommit-temp/**"

jobs:
  deploy:
    uses: developer-overheid-nl/.github/.github/workflows/deploy.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      environment: test
      infra-repo: ${{ vars.INFRA_REPO }}
      kustomize-path: ${{ vars.KUSTOMIZE_PATH }}
    secrets: inherit
```

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy to Production

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  deploy:
    uses: developer-overheid-nl/.github/.github/workflows/deploy.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      environment: prod
      infra-repo: ${{ vars.INFRA_REPO }}
      kustomize-path: ${{ vars.KUSTOMIZE_PATH }}
    secrets: inherit
```

Inputs:

| Input | Standaard | Omschrijving |
| --- | --- | --- |
| `environment` | — | `test` of `prod` |
| `infra-repo` | — | `owner/repo` van de infra-repository |
| `kustomize-path` | `""` | pad naar de overlays; de workflow gebruikt `<pad>/<environment>/kustomization.yaml` |
| `images` | `ghcr.io/<repository>` | JSON-lijst `[{"name", "target", "kustomize-path"}]` voor meerdere images of Docker-targets |
| `build` | `true` | `false` als de image al elders is gebouwd en gepusht |
| `sha` | `github.sha` | commit voor de image-tag, bijvoorbeeld `github.event.workflow_run.head_sha` |
| `context` | `.` | Docker build context |
| `build-args` | `""` | Docker build-args, één per regel |
| `deploy-keyword` | `[deploy-test]` | keyword voor een test-deploy |

Secrets: `RELEASE_PROCES_APP_ID` en `RELEASE_PROCES_APP_PRIVATE_KEY` zijn nodig voor een deploy. Ze staan niet op
`required`, omdat Dependabot-runs geen secrets krijgen; de infra-job faalt met een duidelijke melding als ze ontbreken. `BUILD_SECRETS`
is optioneel en bevat Docker build-secrets, één per regel in de vorm `"ID=waarde"`. Gebruik
`secrets: inherit`, of geef de secrets expliciet door als je `BUILD_SECRETS` nodig hebt.

## go-ci.yml

golangci-lint, `go test` met coverage-drempel en een Docker-build als controle (zonder push).

| Input | Standaard |
| --- | --- |
| `golangci-lint-version` | `v2.12.2` |
| `coverage-threshold` | `80` (`0` schakelt de drempel uit) |
| `coverpkg` | `""` (bijv. `./...`) |
| `docker-build` | `true` |

## json-ci.yml

JSON-lint en OpenAPI-validatie met Super-Linter. Input `filter-regex-exclude` bepaalt welke bestanden
worden overgeslagen. De aanroepende job heeft `statuses`, `issues` en `pull-requests: write` nodig.

## oas-adr.yml

Valideert een OpenAPI-document tegen de DON ADR-ruleset met `don-checker`. Inputs:
`openapi-file` (standaard `api/openapi.json`), `adr-version` en `don-checker-version`.

## changie.yml

Plaatst bij een pull request een reminder als er geen Changie-fragment in zit. Na een push naar
`main` maakt hij een PR met `changie batch auto` en `changie merge`. De aanroepende job heeft
`contents`, `pull-requests` en `issues: write` nodig.
