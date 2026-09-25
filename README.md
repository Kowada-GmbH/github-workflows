# Kowada GitHub Workflows

Gemeinsam genutzte GitHub-Actions-Workflows für Projekte von Kowada.

Die Workflows werden hier zentral gepflegt und von den einzelnen Projekten als Reusable Workflow per `uses:` eingebunden, anstatt die CI-Konfiguration in jedem Repository zu duplizieren.

## Verwendung

Im aufrufenden Projekt bleibt ein schlanker Trigger-Workflow, der den Reusable Workflow einbindet:

```yaml
name: CI

on:
  push:
    branches:
      - develop
      - master
  pull_request: ~
  workflow_dispatch: ~

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  ci:
    uses: Kowada-GmbH/github-workflows/.github/workflows/symfony-bundle-ci.yml@master
```

## Verfügbare Workflows

### `symfony-bundle-ci.yml`

CI für Symfony-Bundles: PHPUnit (Matrix aus PHP-Version × niedrigster/höchster Composer-Abhängigkeiten), PHPStan sowie `composer validate` und `composer audit`.

| Input          | Beschreibung                              | Default          |
|----------------|-------------------------------------------|------------------|
| `php-versions` | JSON-Array der zu testenden PHP-Versionen | `["8.4", "8.5"]` |

### `symfony-app-ci.yml`

CI und Auslieferung für Symfony-Apps auf Basis von [Symfony Docker](https://github.com/dunglas/symfony-docker) mit FrankenPHP, PostgreSQL und AssetMapper. Der Workflow hat keine Inputs, sondern setzt einen einheitlichen Projektaufbau voraus und erkennt den Rest selbst.

| Job                | Inhalt                                                                                                                                                                                                                                          |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Tests`            | Dev-Stack per Docker Compose starten, HTTP/HTTPS erreichbar, Testdatenbank samt Migrations, PHPUnit, Doctrine-Schema, PHPStan, PHP CS Fixer, `lint:container`, `lint:twig`, `lint:yaml`, `composer validate` und `composer audit`                |
| `Production image` | Prod-Image bauen und starten: verweigert den Start mit Platzhalter-Konfiguration, HTTPS erreichbar, Profiler nicht erreichbar, `lint:container` im Prod-Modus. Existiert `public/site.webmanifest`, zusätzlich Manifest, Icons, Service Worker und Offline-Seite |
| `Lint`             | super-linter nur mit actionlint, zizmor, gitleaks, hadolint, shellcheck, ESLint, Merge-Konfliktmarkern und Prettier (CSS, JS, JSON, Markdown, YAML)                                                                                              |
| `Publish image`    | Nur bei Pushes auf `master`, wenn alle anderen Jobs grün sind: Prod-Image nach `ghcr.io/<owner>/<repo>` mit den Tags `latest` und Commit-SHA pushen, danach optional den Portainer-Stack per Webhook neu deployen                              |

**Voraussetzungen im Projekt:**

- `compose.yaml`, `compose.override.yaml` und `compose.prod.yaml` wie in Symfony Docker, mit einem `php`-Service, dessen Prod-Image `app-php-prod` heißt
- ein `Dockerfile` mit dem Target `frankenphp_prod` und einem Entrypoint, der sich im Prod-Modus mit fehlendem `APP_SECRET`, `MAILER_DSN` oder Platzhalter-Datenbankpasswort mit Exit-Code 1 beendet
- Doctrine mit Migrations, PHPUnit, PHPStan und PHP CS Fixer als Composer-Abhängigkeiten
- für PWAs: `public/site.webmanifest`, `public/sw.js`, `public/favicon.ico`, `public/apple-touch-icon.png` und eine Route `/offline`

**Aufrufender Workflow:** Die Berechtigungen für GHCR (`packages: write`) und super-linter (`statuses: write`) muss der Aufrufer freigeben, sie dürfen die des Reusable Workflows nicht unterschreiten.

```yaml
name: CI

on:
  push:
    branches:
      - develop
      - master
  pull_request: ~
  workflow_dispatch: ~

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  ci:
    uses: Kowada-GmbH/github-workflows/.github/workflows/symfony-app-ci.yml@master
    permissions:
      contents: read
      packages: write
      statuses: write
```

**Deployment:** Der Job `Publish image` läuft in der GitHub-Environment `production`. Beide Werte sind optional und werden dort hinterlegt:

| Name                    | Art      | Wirkung                                                                                     |
|-------------------------|----------|---------------------------------------------------------------------------------------------|
| `PORTAINER_WEBHOOK_URL` | Secret   | Stack-Webhook (Portainer Business Edition), der nach dem Push aufgerufen wird. Fehlt er, wird nur das Image veröffentlicht, z. B. für Server, die nur per VPN erreichbar sind |
| `PRODUCTION_URL`        | Variable | URL der App, die GitHub bei jedem Deployment verlinkt                                       |

**Linting:** Hat das Projekt keine eigene `.github/linters/zizmor.yaml`, stellt der Workflow eine bereit, die Actions und Reusable Workflows per Branch oder Tag statt per Commit-Hash erlaubt, damit der Aufruf `@master` nicht beanstandet wird.
