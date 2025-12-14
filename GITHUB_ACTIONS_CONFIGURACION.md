# Guía Completa de Configuración de GitHub Actions

## Tabla de Contenidos
1. [Estructura Básica](#estructura-básica)
2. [Propiedades de Nivel Superior](#propiedades-de-nivel-superior)
3. [Triggers (on)](#triggers-on)
4. [Jobs](#jobs)
5. [Steps](#steps)
6. [Expressions y Contextos](#expressions-y-contextos)
7. [Secrets y Variables](#secrets-y-variables)
8. [Matrices](#matrices)
9. [Artifacts](#artifacts)
10. [Caching](#caching)
11. [Ejemplos Completos](#ejemplos-completos)

---

## Estructura Básica

```yaml
name: Nombre del Workflow
on: [push, pull_request]
jobs:
  nombre-job:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
```

---

## Propiedades de Nivel Superior

### `name`
Nombre del workflow que aparecerá en GitHub.

```yaml
name: Mi Pipeline de CI/CD
```

### `run-name`
Nombre personalizado para cada ejecución del workflow.

```yaml
run-name: Deploy por ${{ github.actor }}
```

### `on`
Define los eventos que disparan el workflow.

### `permissions`
Define los permisos del token GITHUB_TOKEN.

```yaml
permissions:
  contents: read
  issues: write
  pull-requests: write
```

### `env`
Variables de entorno globales para todo el workflow.

```yaml
env:
  NODE_VERSION: '18'
  DEPLOYMENT_ENV: production
```

### `defaults`
Configuraciones por defecto para todos los jobs.

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./src
```

### `concurrency`
Controla la concurrencia de ejecuciones del workflow.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

## Triggers (on)

### Eventos de Push y Pull Request

```yaml
# Simple
on: push

# Múltiples eventos
on: [push, pull_request, workflow_dispatch]

# Con filtros
on:
  push:
    branches:
      - main
      - 'releases/**'
      - '!dev'  # Excluir
    tags:
      - 'v*'
    paths:
      - '**.js'
      - 'src/**'
      - '!docs/**'  # Excluir
  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize
      - reopened
```

### Eventos Programados (Cron)

```yaml
on:
  schedule:
    # Cada día a las 2:30 AM UTC
    - cron: '30 2 * * *'
    # Cada lunes a las 9:00 AM
    - cron: '0 9 * * 1'
```

### Eventos Manuales

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Entorno de despliegue'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      debug:
        description: 'Habilitar debug'
        required: false
        type: boolean
      version:
        description: 'Versión a desplegar'
        required: true
        type: string
```

### Llamadas desde Otros Workflows

```yaml
on:
  workflow_call:
    inputs:
      config-path:
        required: true
        type: string
      environment:
        required: false
        type: string
        default: 'dev'
    secrets:
      api-key:
        required: true
    outputs:
      result:
        description: "Resultado del workflow"
        value: ${{ jobs.build.outputs.result }}
```

### Otros Eventos Importantes

```yaml
on:
  # Issues
  issues:
    types: [opened, edited, deleted, labeled]
  
  # Comments
  issue_comment:
    types: [created, edited]
  
  # Releases
  release:
    types: [published, created]
  
  # Fork
  fork:
  
  # Star
  watch:
    types: [started]
  
  # Pull Request Review
  pull_request_review:
    types: [submitted, edited]
  
  # Check Suite
  check_suite:
    types: [completed]
  
  # Deployment
  deployment:
  deployment_status:
```

---

## Jobs

### Configuración Básica de Job

```yaml
jobs:
  build:
    name: Build y Test
    runs-on: ubuntu-latest
    timeout-minutes: 30
    continue-on-error: false
    
    steps:
      - uses: actions/checkout@v4
```

### `runs-on` - Runners Disponibles

```yaml
jobs:
  ejemplo:
    # Ubuntu
    runs-on: ubuntu-latest      # Ubuntu 22.04
    runs-on: ubuntu-22.04
    runs-on: ubuntu-20.04
    
    # Windows
    runs-on: windows-latest     # Windows Server 2022
    runs-on: windows-2022
    runs-on: windows-2019
    
    # macOS
    runs-on: macos-latest       # macOS 14
    runs-on: macos-14
    runs-on: macos-13
    runs-on: macos-12
    
    # Self-hosted
    runs-on: [self-hosted, linux, x64]
```

### Dependencias entre Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."
  
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing..."
  
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

### Condicionales en Jobs

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production"
```

### Outputs de Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      build-id: ${{ steps.build.outputs.id }}
    steps:
      - id: version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT
      - id: build
        run: echo "id=${{ github.run_number }}" >> $GITHUB_OUTPUT
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Version ${{ needs.build.outputs.version }}"
```

### Environments

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - run: echo "Deploying to production"
```

### Containers

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: node:18
      env:
        NODE_ENV: test
      ports:
        - 3000
      volumes:
        - my_volume:/volume_mount
      options: --cpus 2
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - run: npm test
```

---

## Steps

### Tipos de Steps

```yaml
steps:
  # Usar una action
  - name: Checkout
    uses: actions/checkout@v4
  
  # Ejecutar comando
  - name: Run script
    run: |
      echo "Línea 1"
      echo "Línea 2"
  
  # Con working directory
  - name: Build
    run: npm install
    working-directory: ./frontend
  
  # Con shell específico
  - name: PowerShell
    run: Write-Host "Hello"
    shell: pwsh
```

### Condicionales en Steps

```yaml
steps:
  - name: Solo en main
    if: github.ref == 'refs/heads/main'
    run: echo "Main branch"
  
  - name: Solo si falló step anterior
    if: failure()
    run: echo "Algo falló"
  
  - name: Siempre ejecutar
    if: always()
    run: echo "Limpieza"
  
  - name: Solo si fue exitoso
    if: success()
    run: echo "Todo bien"
  
  - name: Condicional complejo
    if: |
      github.event_name == 'push' &&
      startsWith(github.ref, 'refs/tags/') &&
      !contains(github.event.head_commit.message, 'skip ci')
    run: echo "Condición compleja"
```

### Continue on Error

```yaml
steps:
  - name: Puede fallar
    continue-on-error: true
    run: exit 1
  
  - name: Este se ejecuta igual
    run: echo "Continúa aunque falle el anterior"
```

### Timeout

```yaml
steps:
  - name: Con timeout
    timeout-minutes: 10
    run: ./long-running-script.sh
```

### Environment Variables

```yaml
steps:
  - name: Con variables
    env:
      API_KEY: ${{ secrets.API_KEY }}
      NODE_ENV: production
    run: |
      echo "API_KEY está configurado"
      echo "NODE_ENV=$NODE_ENV"
```

---

## Expressions y Contextos

### Sintaxis de Expresiones

```yaml
# Expresiones simples
${{ expression }}

# En condicionales (if)
if: github.ref == 'refs/heads/main'
if: contains(github.event.head_commit.message, 'hotfix')
if: startsWith(github.ref, 'refs/tags/')
if: endsWith(github.ref, '/main')

# Funciones
if: success()
if: failure()
if: always()
if: cancelled()

# Operadores
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
if: github.event_name == 'push' || github.event_name == 'pull_request'
if: github.actor != 'dependabot[bot]'
```

### Contextos Principales

#### `github` - Información del evento

```yaml
steps:
  - name: GitHub Context
    run: |
      echo "Event: ${{ github.event_name }}"
      echo "Ref: ${{ github.ref }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Repository: ${{ github.repository }}"
      echo "Repository Owner: ${{ github.repository_owner }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run Number: ${{ github.run_number }}"
      echo "Workflow: ${{ github.workflow }}"
      echo "Action: ${{ github.action }}"
      echo "Job: ${{ github.job }}"
```

#### `env` - Variables de entorno

```yaml
env:
  GLOBAL_VAR: global

jobs:
  example:
    env:
      JOB_VAR: job
    steps:
      - name: Usar variables
        env:
          STEP_VAR: step
        run: |
          echo "${{ env.GLOBAL_VAR }}"
          echo "${{ env.JOB_VAR }}"
          echo "${{ env.STEP_VAR }}"
```

#### `job` - Estado del job

```yaml
steps:
  - name: Job Info
    if: job.status == 'success'
    run: echo "Job status: ${{ job.status }}"
```

#### `steps` - Outputs de steps anteriores

```yaml
steps:
  - id: step1
    run: echo "result=success" >> $GITHUB_OUTPUT
  
  - name: Usar output
    run: echo "${{ steps.step1.outputs.result }}"
```

#### `runner` - Información del runner

```yaml
steps:
  - name: Runner Info
    run: |
      echo "OS: ${{ runner.os }}"
      echo "Arch: ${{ runner.arch }}"
      echo "Name: ${{ runner.name }}"
      echo "Temp: ${{ runner.temp }}"
      echo "Tool Cache: ${{ runner.tool_cache }}"
```

#### `secrets` - Secrets del repositorio

```yaml
steps:
  - name: Usar secret
    env:
      API_KEY: ${{ secrets.API_KEY }}
    run: echo "Secret configurado"
```

#### `needs` - Outputs de jobs anteriores

```yaml
jobs:
  job1:
    outputs:
      output1: ${{ steps.step1.outputs.test }}
    steps:
      - id: step1
        run: echo "test=hello" >> $GITHUB_OUTPUT
  
  job2:
    needs: job1
    steps:
      - run: echo "${{ needs.job1.outputs.output1 }}"
```

#### `inputs` - Inputs del workflow

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: string

jobs:
  deploy:
    steps:
      - run: echo "Deploying to ${{ inputs.environment }}"
```

#### `matrix` - Configuración de matriz

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [14, 16, 18]

steps:
  - run: echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node }}"
```

---

## Secrets y Variables

### Secrets a Nivel de Repositorio

```yaml
jobs:
  deploy:
    steps:
      - name: Usar secrets
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          SSH_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          echo "Secrets están configurados"
          # Nunca imprimas los secrets directamente
```

### Variables de Repositorio

```yaml
jobs:
  build:
    steps:
      - name: Usar variables
        env:
          API_URL: ${{ vars.API_URL }}
          ENVIRONMENT: ${{ vars.ENVIRONMENT }}
        run: |
          echo "API URL: $API_URL"
          echo "Environment: $ENVIRONMENT"
```

### Secrets de Environment

```yaml
jobs:
  deploy:
    environment: production
    steps:
      - name: Deploy
        env:
          PROD_API_KEY: ${{ secrets.PROD_API_KEY }}
        run: ./deploy.sh
```

---

## Matrices

### Matriz Simple

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [14, 16, 18]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

### Matriz con Include

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [14, 16, 18]
    include:
      # Configuración específica
      - os: ubuntu-latest
        node: 18
        experimental: true
      # Nueva combinación
      - os: macos-latest
        node: 18
```

### Matriz con Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [14, 16, 18]
    exclude:
      # Excluir Windows con Node 14
      - os: windows-latest
        node: 14
      # Excluir macOS con Node 14
      - os: macos-latest
        node: 14
```

### Fail-fast

```yaml
strategy:
  fail-fast: false  # Continuar aunque falle una combinación
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [14, 16, 18]
```

### Max-parallel

```yaml
strategy:
  max-parallel: 2  # Solo 2 jobs en paralelo
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

---

## Artifacts

### Subir Artifacts

```yaml
jobs:
  build:
    steps:
      - name: Build
        run: npm run build
      
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: |
            dist/
            build/
          retention-days: 5
          if-no-files-found: error  # error, warn, ignore
```

### Descargar Artifacts

```yaml
jobs:
  deploy:
    needs: build
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: ./dist
      
      - name: Deploy
        run: ./deploy.sh
```

### Artifacts entre Jobs

```yaml
jobs:
  build:
    steps:
      - run: echo "data" > output.txt
      - uses: actions/upload-artifact@v4
        with:
          name: my-artifact
          path: output.txt
  
  use-artifact:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: my-artifact
      - run: cat output.txt
```

---

## Caching

### Cache de Dependencias

```yaml
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      
      # Node.js
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'  # o 'yarn', 'pnpm'
      
      # Cache manual
      - name: Cache node modules
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      - run: npm ci
```

### Cache con Múltiples Paths

```yaml
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: |
      ~/.cargo
      ~/.npm
      node_modules
      target
    key: ${{ runner.os }}-deps-${{ hashFiles('**/Cargo.lock', '**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-deps-
```

### Cache para Diferentes Lenguajes

```yaml
jobs:
  # Python
  python:
    steps:
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'
      - run: pip install -r requirements.txt
  
  # Java/Maven
  java:
    steps:
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'maven'
      - run: mvn install
  
  # Go
  go:
    steps:
      - uses: actions/setup-go@v5
        with:
          go-version: '1.21'
          cache: true
      - run: go build
  
  # Ruby
  ruby:
    steps:
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
      - run: bundle exec rake
```

---

## Ejemplos Completos

### CI/CD Completo para Node.js

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18'

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  build:
    name: Build
    needs: test
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7

  deploy:
    name: Deploy to Production
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      
      - name: Deploy
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
          API_URL: ${{ vars.PRODUCTION_API_URL }}
        run: |
          echo "Deploying to production..."
          ./deploy.sh
```

### Docker Build y Push

```yaml
name: Docker Build

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Workflow Reutilizable

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      version:
        required: false
        type: string
        default: 'latest'
    secrets:
      deploy-key:
        required: true
      api-key:
        required: true
    outputs:
      deployment-url:
        description: "URL del deployment"
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy
        id: deploy
        env:
          DEPLOY_KEY: ${{ secrets.deploy-key }}
          API_KEY: ${{ secrets.api-key }}
          VERSION: ${{ inputs.version }}
        run: |
          echo "Deploying version $VERSION to ${{ inputs.environment }}"
          url="https://${{ inputs.environment }}.myapp.com"
          echo "url=$url" >> $GITHUB_OUTPUT

# .github/workflows/main.yml
name: Main Pipeline

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      version: ${{ github.sha }}
    secrets:
      deploy-key: ${{ secrets.STAGING_DEPLOY_KEY }}
      api-key: ${{ secrets.STAGING_API_KEY }}
  
  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      version: ${{ github.sha }}
    secrets:
      deploy-key: ${{ secrets.PROD_DEPLOY_KEY }}
      api-key: ${{ secrets.PROD_API_KEY }}
```

### Release Automation

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  create-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Generate changelog
        id: changelog
        run: |
          # Generar changelog desde el último tag
          PREV_TAG=$(git describe --abbrev=0 --tags $(git rev-list --tags --skip=1 --max-count=1) 2>/dev/null || echo "")
          if [ -z "$PREV_TAG" ]; then
            CHANGELOG=$(git log --pretty=format:"- %s" ${{ github.ref_name }})
          else
            CHANGELOG=$(git log --pretty=format:"- %s" ${PREV_TAG}..${{ github.ref_name }})
          fi
          echo "changelog<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGELOG" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref_name }}
          release_name: Release ${{ github.ref_name }}
          body: |
            ## Changes
            ${{ steps.changelog.outputs.changelog }}
          draft: false
          prerelease: false
```

---

## Características Avanzadas

### Composite Actions

```yaml
# .github/actions/setup-app/action.yml
name: Setup Application
description: Setup Node.js y instalar dependencias

inputs:
  node-version:
    description: 'Versión de Node.js'
    required: false
    default: '18'

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      shell: bash
      run: npm ci
    
    - name: Verificar instalación
      shell: bash
      run: npm list

# Uso en workflow
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-app
        with:
          node-version: '20'
```

### Labels y Auto-merge

```yaml
name: Auto Merge

on:
  pull_request:
    types: [labeled, opened, synchronize]

jobs:
  auto-merge:
    if: contains(github.event.pull_request.labels.*.name, 'auto-merge')
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Enable auto-merge
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Notificaciones

```yaml
jobs:
  notify:
    runs-on: ubuntu-latest
    if: always()
    needs: [build, test, deploy]
    
    steps:
      - name: Slack Notification
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            Workflow: ${{ github.workflow }}
            Status: ${{ job.status }}
            Commit: ${{ github.sha }}
            Author: ${{ github.actor }}
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        if: always()
```

---

## Tips y Best Practices

### 1. Seguridad
- Nunca hagas echo de secrets
- Usa `if-no-files-found: error` en uploads críticos
- Limita permisos con `permissions:`
- Usa environments para secrets sensibles

### 2. Performance
- Usa cache para dependencias
- Limita `max-parallel` si es necesario
- Usa `concurrency` para cancelar builds antiguos
- Usa `paths` y `paths-ignore` para evitar builds innecesarios

### 3. Mantenibilidad
- Usa workflows reutilizables
- Crea composite actions para pasos repetitivos
- Documenta inputs y outputs
- Usa nombres descriptivos

### 4. Debugging
```yaml
- name: Debug
  run: |
    echo "Event: ${{ github.event_name }}"
    echo "Ref: ${{ github.ref }}"
    echo "SHA: ${{ github.sha }}"
    echo "Actor: ${{ github.actor }}"
    cat $GITHUB_EVENT_PATH | jq
```

### 5. Conditional Execution
```yaml
# Solo en push a main
if: github.event_name == 'push' && github.ref == 'refs/heads/main'

# Solo si no es dependabot
if: github.actor != 'dependabot[bot]'

# Solo si el commit no contiene [skip ci]
if: "!contains(github.event.head_commit.message, '[skip ci]')"
```

---

## Referencias

- [Documentación oficial de GitHub Actions](https://docs.github.com/en/actions)
- [Marketplace de Actions](https://github.com/marketplace?type=actions)
- [Sintaxis de Workflows](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Contextos](https://docs.github.com/en/actions/learn-github-actions/contexts)
- [Expresiones](https://docs.github.com/en/actions/learn-github-actions/expressions)
