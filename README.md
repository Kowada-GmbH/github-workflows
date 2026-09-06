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
| `php-versions` | JSON-Array der zu testenden PHP-Versionen | `["8.3", "8.4"]` |
