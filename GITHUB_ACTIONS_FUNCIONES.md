# Funciones en GitHub Actions

## Tabla de Contenidos
1. [¿Qué son las Funciones?](#qué-son-las-funciones)
2. [Funciones de Estado](#funciones-de-estado)
3. [Funciones de Cadenas](#funciones-de-cadenas)
4. [Funciones de Conversión](#funciones-de-conversión)
5. [Funciones de Arrays](#funciones-de-arrays)
6. [Funciones de Verificación de Tipo](#funciones-de-verificación-de-tipo)
7. [Funciones de Objeto](#funciones-de-objeto)
8. [Funciones de Hash](#funciones-de-hash)
9. [Ejemplos Prácticos por Categoría](#ejemplos-prácticos-por-categoría)
10. [Combinación de Funciones](#combinación-de-funciones)
11. [Best Practices](#best-practices)

---

## ¿Qué son las Funciones?

Las **funciones** en GitHub Actions son operaciones predefinidas que se pueden usar dentro de expresiones `${{ }}`. Permiten manipular datos, verificar condiciones, y realizar operaciones complejas en los workflows.

### Sintaxis General

```yaml
${{ nombreFuncion(argumentos) }}
```

---

## Funciones de Estado

Estas funciones verifican el estado de ejecución del workflow y sus componentes.

### `success()`

Retorna `true` cuando ninguno de los steps anteriores ha fallado o sido cancelado.

**Sintaxis:**
```yaml
success()
```

**Ejemplos:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: npm run build
      
      - name: Deploy (solo si build exitoso)
        if: success()
        run: ./deploy.sh
      
      - name: Notificar éxito
        if: success()
        run: echo "✅ Todo salió bien"

  notify:
    needs: build
    runs-on: ubuntu-latest
    # Ejecutar solo si build fue exitoso
    if: success()
    steps:
      - run: echo "Notificando éxito..."
```

### `failure()`

Retorna `true` cuando algún step anterior ha fallado.

**Sintaxis:**
```yaml
failure()
```

**Ejemplos:**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: npm test
      
      - name: Notificar fallo
        if: failure()
        run: |
          echo "❌ Los tests fallaron"
          curl -X POST $WEBHOOK_URL -d "Tests failed"
      
      - name: Crear issue en fallo
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: 'Test failure',
              body: 'Tests failed in run ${{ github.run_id }}'
            })

  cleanup:
    needs: test
    runs-on: ubuntu-latest
    # Ejecutar solo si test falló
    if: failure()
    steps:
      - run: echo "Limpiando recursos..."
```

### `cancelled()`

Retorna `true` cuando el workflow ha sido cancelado.

**Sintaxis:**
```yaml
cancelled()
```

**Ejemplos:**

```yaml
jobs:
  long-running:
    runs-on: ubuntu-latest
    steps:
      - name: Tarea larga
        run: ./long-task.sh
      
      - name: Si fue cancelado
        if: cancelled()
        run: |
          echo "⚠️ Workflow cancelado"
          ./cleanup-on-cancel.sh
      
      - name: Liberar recursos
        if: cancelled()
        run: |
          echo "Liberando recursos..."
          docker compose down

  notify-cancellation:
    needs: long-running
    runs-on: ubuntu-latest
    if: cancelled()
    steps:
      - run: echo "Notificando cancelación..."
```

### `always()`

Siempre retorna `true`, incluso si el workflow fue cancelado o falló.

**Sintaxis:**
```yaml
always()
```

**Ejemplos:**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: npm test
      
      - name: Upload logs (siempre)
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: logs
          path: logs/
      
      - name: Cleanup (siempre)
        if: always()
        run: |
          echo "Limpiando archivos temporales..."
          rm -rf /tmp/test-*
      
      - name: Notificar resultado
        if: always()
        run: |
          if [ "${{ job.status }}" == "success" ]; then
            echo "✅ Éxito"
          else
            echo "❌ Falló o fue cancelado"
          fi

  cleanup:
    needs: test
    runs-on: ubuntu-latest
    # Siempre ejecutar limpieza
    if: always()
    steps:
      - run: echo "Limpieza final..."
```

---

## Funciones de Cadenas

Funciones para manipular y verificar cadenas de texto.

### `contains()`

Verifica si una cadena contiene otra cadena, o si un array contiene un valor.

**Sintaxis:**
```yaml
contains(search, item)
```

**Ejemplos:**

```yaml
jobs:
  check-commit:
    runs-on: ubuntu-latest
    steps:
      # Verificar mensaje de commit
      - name: Hotfix check
        if: contains(github.event.head_commit.message, 'hotfix')
        run: echo "🔥 Es un hotfix"
      
      - name: Skip CI check
        if: contains(github.event.head_commit.message, '[skip ci]')
        run: |
          echo "⏭️ CI omitido por solicitud"
          exit 0
      
      - name: Breaking change check
        if: contains(github.event.head_commit.message, 'BREAKING CHANGE')
        run: echo "⚠️ Contiene cambios que rompen compatibilidad"

  check-labels:
    runs-on: ubuntu-latest
    steps:
      # Verificar etiquetas de PR
      - name: Bug label
        if: contains(github.event.pull_request.labels.*.name, 'bug')
        run: echo "🐛 Marcado como bug"
      
      - name: Priority label
        if: contains(github.event.pull_request.labels.*.name, 'priority:high')
        run: echo "⚡ Alta prioridad"
      
      - name: Auto-merge label
        if: contains(github.event.pull_request.labels.*.name, 'auto-merge')
        run: gh pr merge --auto --squash "$PR_URL"

  check-files:
    runs-on: ubuntu-latest
    steps:
      # Verificar archivos modificados
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      
      - name: Check modified files
        run: |
          FILES=$(git diff --name-only HEAD^ HEAD)
          if echo "$FILES" | grep -q "package.json"; then
            echo "📦 package.json modificado"
          fi
```

### `startsWith()`

Verifica si una cadena comienza con otra cadena específica.

**Sintaxis:**
```yaml
startsWith(searchString, searchValue)
```

**Ejemplos:**

```yaml
jobs:
  version-tags:
    # Solo para tags que empiezan con 'v'
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - name: Extract version
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          echo "Version: $VERSION"
          echo "VERSION=$VERSION" >> $GITHUB_ENV
      
      - name: Create release
        run: gh release create "v$VERSION"

  feature-branches:
    # Solo para ramas de feature
    if: startsWith(github.ref, 'refs/heads/feature/')
    runs-on: ubuntu-latest
    steps:
      - name: Feature branch CI
        run: |
          FEATURE_NAME=${GITHUB_REF#refs/heads/feature/}
          echo "Testing feature: $FEATURE_NAME"
          npm test

  conventional-commits:
    runs-on: ubuntu-latest
    steps:
      - name: Verify feat commit
        if: startsWith(github.event.head_commit.message, 'feat:')
        run: echo "✨ Nueva característica"
      
      - name: Verify fix commit
        if: startsWith(github.event.head_commit.message, 'fix:')
        run: echo "🐛 Corrección de bug"
      
      - name: Verify docs commit
        if: startsWith(github.event.head_commit.message, 'docs:')
        run: echo "📝 Actualización de documentación"
      
      - name: Verify refactor commit
        if: startsWith(github.event.head_commit.message, 'refactor:')
        run: echo "♻️ Refactorización"
      
      - name: Invalid commit format
        if: |
          !startsWith(github.event.head_commit.message, 'feat:') &&
          !startsWith(github.event.head_commit.message, 'fix:') &&
          !startsWith(github.event.head_commit.message, 'docs:') &&
          !startsWith(github.event.head_commit.message, 'chore:')
        run: |
          echo "❌ Formato de commit inválido"
          exit 1
```

### `endsWith()`

Verifica si una cadena termina con otra cadena específica.

**Sintaxis:**
```yaml
endsWith(searchString, searchValue)
```

**Ejemplos:**

```yaml
jobs:
  file-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Python files changed
        if: endsWith(github.event.pull_request.head.ref, '.py')
        run: echo "🐍 Archivos Python modificados"
      
      - name: TypeScript files
        run: |
          FILES=$(git diff --name-only HEAD^ HEAD)
          for file in $FILES; do
            if [[ $file == *.ts ]]; then
              echo "📘 TypeScript: $file"
            fi
          done

  branch-naming:
    runs-on: ubuntu-latest
    steps:
      - name: Docs branch
        if: endsWith(github.ref_name, '-docs')
        run: echo "📚 Rama de documentación"
      
      - name: Test branch
        if: endsWith(github.ref_name, '-test')
        run: echo "🧪 Rama de pruebas"
      
      - name: Hotfix branch
        if: endsWith(github.ref_name, '-hotfix')
        run: |
          echo "🔥 Rama de hotfix - prioridad alta"
          # Notificar al equipo

  environment-detection:
    runs-on: ubuntu-latest
    steps:
      - name: Development
        if: endsWith(github.ref_name, '-dev')
        run: echo "ENVIRONMENT=development" >> $GITHUB_ENV
      
      - name: Staging
        if: endsWith(github.ref_name, '-staging')
        run: echo "ENVIRONMENT=staging" >> $GITHUB_ENV
      
      - name: Production
        if: endsWith(github.ref_name, '-prod')
        run: echo "ENVIRONMENT=production" >> $GITHUB_ENV
```

### `format()`

Formatea una cadena reemplazando placeholders `{N}` con los valores proporcionados.

**Sintaxis:**
```yaml
format(string, value0, value1, ...)
```

**Ejemplos:**

```yaml
jobs:
  formatting:
    runs-on: ubuntu-latest
    steps:
      - name: Mensaje personalizado
        run: |
          MESSAGE="${{ format('Hola {0}, bienvenido a {1}!', github.actor, github.repository) }}"
          echo "$MESSAGE"
      
      - name: URL formateada
        run: |
          URL="${{ format('https://github.com/{0}/actions/runs/{1}', github.repository, github.run_id) }}"
          echo "Workflow URL: $URL"
      
      - name: Versión formateada
        run: |
          VERSION="${{ format('v{0}.{1}.{2}', '1', '2', '3') }}"
          echo "Version: $VERSION"
      
      - name: Path formateado
        run: |
          PATH="${{ format('{0}/releases/{1}/artifacts', github.repository, github.ref_name) }}"
          echo "Release path: $PATH"
      
      - name: Notificación formateada
        run: |
          NOTIFICATION="${{ format('🚀 Deploy de {0} por {1} en {2}', github.repository, github.actor, github.ref_name) }}"
          echo "$NOTIFICATION"
          # curl -X POST $WEBHOOK -d "$NOTIFICATION"

  dynamic-config:
    runs-on: ubuntu-latest
    steps:
      - name: Configuración dinámica
        env:
          DATABASE_URL: ${{ format('postgresql://{0}:{1}@{2}:{3}/{4}', 'user', secrets.DB_PASSWORD, 'localhost', '5432', 'mydb') }}
          API_ENDPOINT: ${{ format('https://api.{0}.example.com/v{1}', github.ref_name, '2') }}
        run: |
          echo "Database URL configurado"
          echo "API Endpoint: $API_ENDPOINT"
```

### `join()`

Une elementos de un array en una cadena usando un separador.

**Sintaxis:**
```yaml
join(array, separator)
```

**Ejemplos:**

```yaml
jobs:
  join-examples:
    runs-on: ubuntu-latest
    steps:
      - name: Join labels
        run: |
          LABELS="${{ join(github.event.pull_request.labels.*.name, ', ') }}"
          echo "Labels: $LABELS"
      
      - name: Join assignees
        run: |
          ASSIGNEES="${{ join(github.event.pull_request.assignees.*.login, ', ') }}"
          echo "Asignados a: $ASSIGNEES"
      
      - name: Join paths con separador personalizado
        run: |
          PATHS="${{ join(fromJSON('["src", "lib", "tests"]'), ':') }}"
          echo "PATH=$PATHS" >> $GITHUB_ENV
      
      - name: Join para comando
        run: |
          FILES="${{ join(fromJSON('["file1.js", "file2.js", "file3.js"]'), ' ') }}"
          echo "Procesando archivos: $FILES"
          # npx eslint $FILES

  matrix-join:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    runs-on: ${{ matrix.os }}
    steps:
      - name: Matrix info
        run: |
          INFO="${{ format('Testing on {0} with Node {1}', matrix.os, matrix.node) }}"
          echo "$INFO"
```

---

## Funciones de Conversión

Funciones para convertir entre diferentes tipos de datos.

### `toJSON()`

Convierte un valor a su representación JSON.

**Sintaxis:**
```yaml
toJSON(value)
```

**Ejemplos:**

```yaml
jobs:
  json-conversion:
    runs-on: ubuntu-latest
    steps:
      - name: Debug - Ver todo el contexto de GitHub
        run: |
          echo '${{ toJSON(github) }}'
      
      - name: Ver evento completo
        run: |
          echo '${{ toJSON(github.event) }}'
      
      - name: Ver matriz
        run: |
          echo '${{ toJSON(matrix) }}'
      
      - name: Ver secrets (nombres, no valores)
        run: |
          echo '${{ toJSON(secrets) }}'
      
      - name: Guardar contexto en archivo
        run: |
          echo '${{ toJSON(github) }}' > github-context.json
          cat github-context.json | jq '.'
      
      - name: Enviar a API
        run: |
          PAYLOAD='${{ toJSON(github.event) }}'
          curl -X POST https://api.example.com/webhook \
            -H "Content-Type: application/json" \
            -d "$PAYLOAD"

  debug-workflow:
    runs-on: ubuntu-latest
    steps:
      - name: Debug completo
        run: |
          echo "=== GitHub Context ==="
          echo '${{ toJSON(github) }}' | jq '.'
          
          echo "=== Job Context ==="
          echo '${{ toJSON(job) }}' | jq '.'
          
          echo "=== Runner Context ==="
          echo '${{ toJSON(runner) }}' | jq '.'
          
          echo "=== Env Context ==="
          echo '${{ toJSON(env) }}' | jq '.'
```

### `fromJSON()`

Convierte una cadena JSON a un objeto.

**Sintaxis:**
```yaml
fromJSON(string)
```

**Ejemplos:**

```yaml
jobs:
  json-parsing:
    runs-on: ubuntu-latest
    steps:
      - name: Parse JSON simple
        run: |
          VALUE='${{ fromJSON('{"name":"John","age":30}').name }}'
          echo "Name: $VALUE"
      
      - name: Parse array
        if: contains(fromJSON('["main", "develop", "staging"]'), github.ref_name)
        run: echo "Rama válida"
      
      - name: Configuración desde JSON
        env:
          CONFIG: '{"env":"production","debug":false,"port":3000}'
        run: |
          ENV='${{ fromJSON(env.CONFIG).env }}'
          DEBUG='${{ fromJSON(env.CONFIG).debug }}'
          PORT='${{ fromJSON(env.CONFIG).port }}'
          echo "Environment: $ENV"
          echo "Debug: $DEBUG"
          echo "Port: $PORT"

  dynamic-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          # Generar matriz dinámica
          MATRIX='{"include":[{"os":"ubuntu-latest","node":"18"},{"os":"windows-latest","node":"20"}]}'
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT
  
  use-matrix:
    needs: dynamic-matrix
    strategy:
      matrix: ${{ fromJSON(needs.dynamic-matrix.outputs.matrix) }}
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test

  conditional-from-json:
    runs-on: ubuntu-latest
    steps:
      - name: Verificar en lista
        if: contains(fromJSON('["prod", "staging", "dev"]'), github.ref_name)
        run: echo "Ambiente válido"
      
      - name: Verificar permiso
        if: contains(fromJSON('["admin", "maintainer", "owner"]'), github.actor)
        run: echo "Usuario autorizado"
```

---

## Funciones de Arrays

### `contains()` para Arrays

Ya visto anteriormente, pero específico para arrays:

```yaml
jobs:
  array-contains:
    runs-on: ubuntu-latest
    steps:
      - name: Verificar en array de ramas
        if: contains(fromJSON('["main", "develop", "staging"]'), github.ref_name)
        run: echo "Rama en la lista"
      
      - name: Verificar múltiples valores
        run: |
          VALID_ENVS='["development", "staging", "production"]'
          if [[ $(echo '${{ toJSON(fromJSON(env.VALID_ENVS)) }}' | jq 'contains(["${{ github.ref_name }}"])') == "true" ]]; then
            echo "Ambiente válido"
          fi
```

---

## Funciones de Verificación de Tipo

GitHub Actions no tiene funciones explícitas de verificación de tipo como `isString()` o `isNumber()`, pero puedes usar expresiones para verificar tipos.

### Verificar Nulos

```yaml
jobs:
  null-checks:
    runs-on: ubuntu-latest
    steps:
      - name: Verificar si es null
        if: github.event.pull_request.milestone == null
        run: echo "No tiene milestone"
      
      - name: Verificar si NO es null
        if: github.event.pull_request.milestone != null
        run: echo "Tiene milestone: ${{ github.event.pull_request.milestone.title }}"
      
      - name: Verificar assignee
        if: github.event.issue.assignee != null
        run: echo "Asignado a: ${{ github.event.issue.assignee.login }}"
```

### Verificar Booleanos

```yaml
jobs:
  boolean-checks:
    runs-on: ubuntu-latest
    steps:
      - name: Verificar draft
        if: github.event.pull_request.draft == true
        run: echo "Es un borrador"
      
      - name: Verificar merged
        if: github.event.pull_request.merged == true
        run: echo "PR mergeado"
      
      - name: Verificar fork
        if: github.event.pull_request.head.repo.fork == true
        run: echo "Es un fork"
```

---

## Funciones de Objeto

### Acceso a Propiedades

```yaml
jobs:
  object-access:
    runs-on: ubuntu-latest
    steps:
      - name: Acceso directo
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Actor: ${{ github.actor }}"
          echo "Ref: ${{ github.ref }}"
          echo "SHA: ${{ github.sha }}"
      
      - name: Acceso anidado
        run: |
          echo "PR Title: ${{ github.event.pull_request.title }}"
          echo "PR Number: ${{ github.event.pull_request.number }}"
          echo "Base Branch: ${{ github.event.pull_request.base.ref }}"
          echo "Head Branch: ${{ github.event.pull_request.head.ref }}"
      
      - name: Acceso con corchetes
        run: |
          echo "Ref: ${{ github['ref'] }}"
          echo "Event: ${{ github['event_name'] }}"
```

---

## Funciones de Hash

### `hashFiles()`

Genera un hash SHA-256 de uno o más archivos. Útil para cache keys.

**Sintaxis:**
```yaml
hashFiles(pattern1, pattern2, ...)
```

**Ejemplos:**

```yaml
jobs:
  caching:
    runs-on: ubuntu-latest
    steps:
      # Cache básico con hash
      - name: Cache dependencies
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      # Cache con múltiples archivos
      - name: Cache multi-file
        uses: actions/cache@v4
        with:
          path: |
            ~/.npm
            ~/.cache
          key: ${{ runner.os }}-deps-${{ hashFiles('**/package-lock.json', '**/yarn.lock', '**/pnpm-lock.yaml') }}
      
      # Cache para diferentes lenguajes
      - name: Cache Python
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt', '**/Pipfile.lock') }}
      
      - name: Cache Rust
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

  conditional-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate cache key
        id: cache-key
        run: |
          HASH="${{ hashFiles('**/package-lock.json') }}"
          echo "hash=$HASH" >> $GITHUB_OUTPUT
          echo "Cache key hash: $HASH"
      
      - name: Use cache
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-modules-${{ steps.cache-key.outputs.hash }}
```

---

## Ejemplos Prácticos por Categoría

### Ejemplo 1: Pipeline de CI/CD Completo con Funciones

```yaml
name: Complete CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate commit message
        if: |
          !startsWith(github.event.head_commit.message, 'feat:') &&
          !startsWith(github.event.head_commit.message, 'fix:') &&
          !startsWith(github.event.head_commit.message, 'docs:') &&
          !startsWith(github.event.head_commit.message, 'chore:')
        run: |
          echo "❌ Commit message must start with: feat:, fix:, docs:, or chore:"
          exit 1
      
      - name: Check for breaking changes
        if: contains(github.event.head_commit.message, 'BREAKING CHANGE')
        run: |
          echo "⚠️ Breaking change detected!"
          echo "BREAKING_CHANGE=true" >> $GITHUB_ENV

  test:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Cache dependencies
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      
      - run: npm ci
      - run: npm test
      
      - name: Upload coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  build:
    needs: test
    if: success()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy:
    needs: build
    if: |
      success() &&
      github.event_name == 'push' &&
      (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop')
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
      
      - name: Deploy info
        run: |
          ENV=${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
          MESSAGE="${{ format('Deploying {0} to {1} by {2}', github.repository, env.ENV, github.actor) }}"
          echo "$MESSAGE"
      
      - run: ./deploy.sh

  notify:
    needs: [test, build, deploy]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Success notification
        if: success()
        run: |
          MESSAGE="${{ format('✅ {0} - All jobs succeeded!', github.workflow) }}"
          echo "$MESSAGE"
      
      - name: Failure notification
        if: failure()
        run: |
          MESSAGE="${{ format('❌ {0} - Some jobs failed!', github.workflow) }}"
          echo "$MESSAGE"
```

### Ejemplo 2: Gestión Avanzada de Labels

```yaml
name: Label Management

on:
  pull_request:
    types: [opened, labeled, unlabeled]

jobs:
  check-labels:
    runs-on: ubuntu-latest
    steps:
      - name: List all labels
        run: |
          LABELS="${{ join(github.event.pull_request.labels.*.name, ', ') }}"
          echo "Current labels: $LABELS"
      
      - name: Priority check
        if: |
          contains(github.event.pull_request.labels.*.name, 'priority:high') ||
          contains(github.event.pull_request.labels.*.name, 'priority:critical')
        run: |
          echo "⚡ High priority PR!"
          # Notify team
      
      - name: Auto-merge ready
        if: |
          contains(github.event.pull_request.labels.*.name, 'auto-merge') &&
          !contains(github.event.pull_request.labels.*.name, 'blocked') &&
          github.event.pull_request.draft == false
        run: |
          echo "🚀 Ready for auto-merge"
          gh pr merge --auto --squash "${{ github.event.pull_request.html_url }}"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Needs review
        if: |
          !contains(github.event.pull_request.labels.*.name, 'reviewed') &&
          !github.event.pull_request.draft
        run: |
          REVIEWERS="${{ join(github.event.pull_request.requested_reviewers.*.login, ', ') }}"
          echo "Waiting for review from: $REVIEWERS"
```

### Ejemplo 3: Matriz Dinámica con JSON

```yaml
name: Dynamic Matrix

on: [push]

jobs:
  setup-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Determine matrix
        id: set-matrix
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            # Full matrix for main
            MATRIX='{"os":["ubuntu-latest","windows-latest","macos-latest"],"node":["16","18","20"]}'
          else
            # Reduced matrix for other branches
            MATRIX='{"os":["ubuntu-latest"],"node":["18"]}'
          fi
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT
      
      - name: Show matrix
        run: |
          echo "Matrix: ${{ steps.set-matrix.outputs.matrix }}"
          echo '${{ toJSON(fromJSON(steps.set-matrix.outputs.matrix)) }}'

  test:
    needs: setup-matrix
    strategy:
      matrix: ${{ fromJSON(needs.setup-matrix.outputs.matrix) }}
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      
      - name: Test info
        run: |
          INFO="${{ format('Testing on {0} with Node {1}', matrix.os, matrix.node) }}"
          echo "$INFO"
      
      - run: npm ci
      - run: npm test
```

### Ejemplo 4: Verificación de Archivos Modificados

```yaml
name: File Change Detection

on:
  pull_request:
    paths:
      - 'src/**'
      - 'tests/**'

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.changes.outputs.backend }}
      frontend: ${{ steps.changes.outputs.frontend }}
      docs: ${{ steps.changes.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      
      - name: Detect changes
        id: changes
        run: |
          FILES=$(git diff --name-only HEAD^ HEAD)
          
          if echo "$FILES" | grep -q "^src/backend/"; then
            echo "backend=true" >> $GITHUB_OUTPUT
          else
            echo "backend=false" >> $GITHUB_OUTPUT
          fi
          
          if echo "$FILES" | grep -q "^src/frontend/"; then
            echo "frontend=true" >> $GITHUB_OUTPUT
          else
            echo "frontend=false" >> $GITHUB_OUTPUT
          fi
          
          if echo "$FILES" | grep -q "^docs/"; then
            echo "docs=true" >> $GITHUB_OUTPUT
          else
            echo "docs=false" >> $GITHUB_OUTPUT
          fi
      
      - name: Show detected changes
        run: |
          echo "Backend changed: ${{ steps.changes.outputs.backend }}"
          echo "Frontend changed: ${{ steps.changes.outputs.frontend }}"
          echo "Docs changed: ${{ steps.changes.outputs.docs }}"

  test-backend:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running backend tests..."
      - run: npm run test:backend

  test-frontend:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running frontend tests..."
      - run: npm run test:frontend

  build-docs:
    needs: detect-changes
    if: needs.detect-changes.outputs.docs == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Building documentation..."
      - run: npm run docs:build
```

---

## Combinación de Funciones

### Ejemplo 1: Funciones Anidadas

```yaml
jobs:
  nested-functions:
    runs-on: ubuntu-latest
    steps:
      # Combinar format + join
      - name: Format with join
        run: |
          LABELS="${{ join(github.event.pull_request.labels.*.name, ', ') }}"
          MESSAGE="${{ format('PR #{0} tiene las labels: {1}', github.event.pull_request.number, join(github.event.pull_request.labels.*.name, ', ')) }}"
          echo "$MESSAGE"
      
      # Combinar contains + fromJSON
      - name: Check in JSON array
        if: contains(fromJSON('["main", "develop", "staging"]'), github.ref_name)
        run: echo "Rama válida"
      
      # Combinar toJSON + format
      - name: Debug formatted
        run: |
          echo "${{ format('Event data: {0}', toJSON(github.event)) }}"
      
      # Combinar startsWith + format
      - name: Version tag check
        if: startsWith(github.ref, 'refs/tags/v')
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          MESSAGE="${{ format('Releasing version {0}', '') }}${VERSION}"
          echo "$MESSAGE"
```

### Ejemplo 2: Condiciones Complejas

```yaml
jobs:
  complex-conditions:
    runs-on: ubuntu-latest
    steps:
      - name: Multi-condition check
        if: |
          success() &&
          github.event_name == 'push' &&
          startsWith(github.ref, 'refs/heads/') &&
          !contains(github.event.head_commit.message, '[skip ci]') &&
          (
            endsWith(github.ref_name, '-prod') ||
            contains(fromJSON('["main", "master"]'), github.ref_name)
          )
        run: echo "All conditions met!"
      
      - name: Label and state check
        if: |
          contains(github.event.pull_request.labels.*.name, 'ready') &&
          !contains(github.event.pull_request.labels.*.name, 'blocked') &&
          github.event.pull_request.draft == false &&
          github.event.pull_request.mergeable == true
        run: echo "Ready to merge!"
```

---

## Best Practices

### 1. Usa Funciones de Estado Apropiadamente

```yaml
# ✅ Recomendado
jobs:
  cleanup:
    if: always()  # Siempre ejecutar limpieza
    steps:
      - run: ./cleanup.sh
  
  notify-failure:
    if: failure()  # Solo si algo falló
    steps:
      - run: ./notify-team.sh

# ❌ Evitar - No usar funciones de estado innecesariamente
jobs:
  build:
    if: success()  # Redundante, se ejecuta por defecto si todo está bien
    steps:
      - run: npm build
```

### 2. Combina Funciones de String Eficientemente

```yaml
# ✅ Recomendado - Usar contains para múltiples opciones
if: |
  contains(github.event.head_commit.message, 'hotfix') ||
  contains(github.event.head_commit.message, 'urgent')

# ⚠️ Menos eficiente - Múltiples startsWith
if: |
  startsWith(github.event.head_commit.message, 'hotfix:') ||
  startsWith(github.event.head_commit.message, 'urgent:')
```

### 3. Usa format() para Mensajes Dinámicos

```yaml
# ✅ Recomendado
steps:
  - name: Deploy notification
    run: |
      MESSAGE="${{ format('Deploying {0} v{1} to {2}', github.repository, github.ref_name, 'production') }}"
      echo "$MESSAGE"

# ❌ Evitar - Concatenación manual compleja
steps:
  - name: Deploy notification
    run: |
      echo "Deploying ${{ github.repository }} v${{ github.ref_name }} to production"
```

### 4. Usa hashFiles() para Cache Inteligente

```yaml
# ✅ Recomendado - Cache específico
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# ❌ Evitar - Cache genérico sin hash
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node
```

### 5. Debugging con toJSON()

```yaml
# ✅ Útil para debugging
steps:
  - name: Debug context
    run: |
      echo "GitHub context:"
      echo '${{ toJSON(github) }}' | jq '.'
      
      echo "Matrix:"
      echo '${{ toJSON(matrix) }}' | jq '.'
```

### 6. Verifica Valores Null Explícitamente

```yaml
# ✅ Recomendado
if: github.event.pull_request.milestone != null

# ⚠️ Menos claro
if: github.event.pull_request.milestone
```

### 7. Usa fromJSON() para Configuración Dinámica

```yaml
# ✅ Recomendado - Configuración flexible
jobs:
  setup:
    outputs:
      config: ${{ steps.config.outputs.config }}
    steps:
      - id: config
        run: |
          CONFIG='{"env":"prod","replicas":3,"regions":["us-east","eu-west"]}'
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
  
  deploy:
    needs: setup
    steps:
      - name: Deploy
        run: |
          ENV='${{ fromJSON(needs.setup.outputs.config).env }}'
          REPLICAS='${{ fromJSON(needs.setup.outputs.config).replicas }}'
          echo "Deploying to $ENV with $REPLICAS replicas"
```

### 8. Funciones en Expresiones Cortas

```yaml
# ✅ Recomendado - Expresión concisa
timeout-minutes: ${{ github.ref == 'refs/heads/main' && 60 || 30 }}

# ✅ También válido - Más explícito
steps:
  - name: Set timeout
    run: |
      if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
        echo "TIMEOUT=60" >> $GITHUB_ENV
      else
        echo "TIMEOUT=30" >> $GITHUB_ENV
      fi
```

---

## Tabla Resumen de Funciones

| Función | Categoría | Descripción | Ejemplo |
|---------|-----------|-------------|---------|
| `success()` | Estado | Retorna true si todo fue exitoso | `if: success()` |
| `failure()` | Estado | Retorna true si algo falló | `if: failure()` |
| `cancelled()` | Estado | Retorna true si fue cancelado | `if: cancelled()` |
| `always()` | Estado | Siempre retorna true | `if: always()` |
| `contains()` | String/Array | Verifica si contiene valor | `contains(string, search)` |
| `startsWith()` | String | Verifica inicio de string | `startsWith(string, search)` |
| `endsWith()` | String | Verifica final de string | `endsWith(string, search)` |
| `format()` | String | Formatea string con placeholders | `format('Hello {0}', name)` |
| `join()` | Array | Une array con separador | `join(array, ',')` |
| `toJSON()` | Conversión | Convierte a JSON | `toJSON(object)` |
| `fromJSON()` | Conversión | Parsea JSON | `fromJSON('{"key":"value"}')` |
| `hashFiles()` | Hash | Genera hash SHA-256 | `hashFiles('**/*.lock')` |

---

## Funciones No Soportadas

GitHub Actions **NO soporta**:

- ❌ Expresiones regulares (regex) directamente
- ❌ Funciones matemáticas complejas (`Math.floor`, `Math.random`, etc.)
- ❌ Funciones de fecha/tiempo personalizadas
- ❌ Funciones de tipo (`typeof`, `isString`, `isNumber`)
- ❌ Operaciones de array avanzadas (`map`, `filter`, `reduce`)

**Alternativa:** Usa scripts de shell para funcionalidad avanzada:

```yaml
steps:
  - name: Complex operations
    run: |
      # Usar bash/python/node para operaciones complejas
      result=$(echo "scale=2; 10/3" | bc)
      echo "Result: $result"
```

---

## Referencias

- [GitHub Actions - Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [GitHub Actions - Functions](https://docs.github.com/en/actions/learn-github-actions/expressions#functions)
- [GitHub Actions - Status Check Functions](https://docs.github.com/en/actions/learn-github-actions/expressions#status-check-functions)
- [GitHub Actions - Object Filters](https://docs.github.com/en/actions/learn-github-actions/expressions#object-filters)
