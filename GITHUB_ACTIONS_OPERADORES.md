# Operadores en GitHub Actions

## Tabla de Contenidos
1. [¿Qué son los Operadores?](#qué-son-los-operadores)
2. [Operadores de Comparación](#operadores-de-comparación)
3. [Operadores Lógicos](#operadores-lógicos)
4. [Operadores de Agrupación](#operadores-de-agrupación)
5. [Operadores de Índice](#operadores-de-índice)
6. [Funciones como Operadores](#funciones-como-operadores)
7. [Ejemplos Prácticos](#ejemplos-prácticos)
8. [Precedencia de Operadores](#precedencia-de-operadores)
9. [Best Practices](#best-practices)

---

## ¿Qué son los Operadores?

Los **operadores** en GitHub Actions son símbolos o palabras clave que realizan operaciones sobre valores (literales, variables, o expresiones). Se utilizan principalmente en:

- Expresiones (`${{ }}`)
- Condicionales (`if:`)
- Cálculos y comparaciones

---

## Operadores de Comparación

Los operadores de comparación evalúan la relación entre dos valores y retornan un valor booleano (`true` o `false`).

### Igualdad (`==`)

Comprueba si dos valores son iguales.

```yaml
jobs:
  deploy:
    # Verificar si la rama es main
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Rama principal"

  production:
    # Verificar evento
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Es un push"

  check-actor:
    # Verificar actor
    if: github.actor == 'octocat'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Actor es octocat"
```

### Desigualdad (`!=`)

Comprueba si dos valores son diferentes.

```yaml
jobs:
  skip-dependabot:
    # Ejecutar solo si NO es dependabot
    if: github.actor != 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - run: echo "No es dependabot"

  non-draft:
    # Solo si NO es draft
    if: github.event.pull_request.draft != true
    runs-on: ubuntu-latest
    steps:
      - run: echo "No es borrador"

  has-milestone:
    # Solo si tiene milestone
    if: github.event.pull_request.milestone != null
    runs-on: ubuntu-latest
    steps:
      - run: echo "Tiene milestone"
```

### Mayor que (`>`)

Comprueba si el valor de la izquierda es mayor que el de la derecha.

```yaml
jobs:
  check-runs:
    # Solo si hay más de 100 ejecuciones
    if: github.run_number > 100
    runs-on: ubuntu-latest
    steps:
      - run: echo "Más de 100 ejecuciones"

  check-version:
    # Solo para versiones mayores a 16
    if: matrix.node-version > 16
    runs-on: ubuntu-latest
    steps:
      - run: echo "Node > 16"
```

### Mayor o igual que (`>=`)

Comprueba si el valor de la izquierda es mayor o igual que el de la derecha.

```yaml
jobs:
  minimum-version:
    # Node 18 o superior
    if: matrix.node-version >= 18
    runs-on: ubuntu-latest
    steps:
      - run: echo "Node >= 18"

  retry-limit:
    # Hasta 3 intentos
    if: github.run_attempt >= 3
    runs-on: ubuntu-latest
    steps:
      - run: echo "Máximo de reintentos alcanzado"
```

### Menor que (`<`)

Comprueba si el valor de la izquierda es menor que el de la derecha.

```yaml
jobs:
  early-runs:
    # Primeras 10 ejecuciones
    if: github.run_number < 10
    runs-on: ubuntu-latest
    steps:
      - run: echo "Ejecución temprana"

  old-version:
    # Versiones anteriores a 18
    if: matrix.node-version < 18
    runs-on: ubuntu-latest
    steps:
      - run: echo "Versión antigua de Node"
```

### Menor o igual que (`<=`)

Comprueba si el valor de la izquierda es menor o igual que el de la derecha.

```yaml
jobs:
  retry-check:
    # Hasta el tercer intento
    if: github.run_attempt <= 3
    runs-on: ubuntu-latest
    steps:
      - run: echo "Intento permitido"

  version-check:
    # Node 16 o inferior
    if: matrix.node-version <= 16
    runs-on: ubuntu-latest
    steps:
      - run: echo "Node <= 16"
```

---

## Operadores Lógicos

Los operadores lógicos combinan múltiples expresiones booleanas.

### AND Lógico (`&&`)

Retorna `true` solo si ambas expresiones son verdaderas.

```yaml
jobs:
  deploy-production:
    # Ambas condiciones deben ser verdaderas
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy a producción desde main"

  tag-release:
    # Múltiples condiciones con AND
    if: |
      github.event_name == 'push' &&
      startsWith(github.ref, 'refs/tags/v') &&
      github.actor != 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Release tag válido"

  advanced-check:
    # Condiciones complejas
    if: |
      success() &&
      github.ref == 'refs/heads/main' &&
      !contains(github.event.head_commit.message, '[skip ci]') &&
      github.run_attempt == 1
    runs-on: ubuntu-latest
    steps:
      - run: echo "Todas las condiciones se cumplen"
```

### OR Lógico (`||`)

Retorna `true` si al menos una expresión es verdadera.

```yaml
jobs:
  ci-trigger:
    # Ejecutar en push O pull_request
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - run: echo "CI ejecutándose"

  multiple-branches:
    # Ejecutar en main O develop
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Rama importante"

  special-users:
    # Usuarios especiales
    if: |
      github.actor == 'admin' ||
      github.actor == 'maintainer' ||
      github.actor == 'owner'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Usuario con privilegios especiales"

  version-check:
    # Node 18 O Node 20
    if: matrix.node-version == 18 || matrix.node-version == 20
    runs-on: ubuntu-latest
    steps:
      - run: echo "Versión LTS de Node"
```

### NOT Lógico (`!`)

Invierte el valor booleano de una expresión.

```yaml
jobs:
  not-draft:
    # NO es draft
    if: '!github.event.pull_request.draft'
    runs-on: ubuntu-latest
    steps:
      - run: echo "No es borrador"

  skip-ci-check:
    # NO contiene [skip ci]
    if: "!contains(github.event.head_commit.message, '[skip ci]')"
    runs-on: ubuntu-latest
    steps:
      - run: echo "No se solicitó skip CI"

  not-dependabot:
    # NO es dependabot
    if: "!(github.actor == 'dependabot[bot]')"
    runs-on: ubuntu-latest
    steps:
      - run: echo "No es dependabot"

  complex-not:
    # NOT con múltiples condiciones
    if: |
      !failure() &&
      !cancelled() &&
      !startsWith(github.ref, 'refs/heads/feature/')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Condiciones negativas cumplidas"
```

---

## Operadores de Agrupación

Los paréntesis `()` se usan para agrupar expresiones y controlar la precedencia de evaluación.

### Agrupación Básica

```yaml
jobs:
  complex-condition:
    # Agrupar con paréntesis
    if: (github.event_name == 'push' || github.event_name == 'pull_request') && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Condición compleja"
```

### Agrupación Múltiple

```yaml
jobs:
  advanced-grouping:
    # Múltiples niveles de agrupación
    if: |
      (github.event_name == 'push' && github.ref == 'refs/heads/main') ||
      (github.event_name == 'pull_request' && github.base_ref == 'main')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Condición con agrupación múltiple"

  complex-logic:
    # Lógica compleja con agrupación
    if: |
      (
        (github.actor == 'admin' || github.actor == 'maintainer') &&
        (github.event_name == 'push' || github.event_name == 'workflow_dispatch')
      ) ||
      (
        github.event_name == 'schedule' &&
        github.ref == 'refs/heads/main'
      )
    runs-on: ubuntu-latest
    steps:
      - run: echo "Lógica muy compleja"
```

### Sin Agrupación vs Con Agrupación

```yaml
jobs:
  # ❌ Puede ser ambiguo
  without-grouping:
    if: github.event_name == 'push' || github.event_name == 'pull_request' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Puede no ser lo que esperas"

  # ✅ Claro y explícito
  with-grouping:
    if: (github.event_name == 'push' || github.event_name == 'pull_request') && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Condición clara"
```

---

## Operadores de Índice

Los operadores de índice permiten acceder a elementos en arrays u objetos.

### Notación de Corchetes (`[]`)

```yaml
jobs:
  access-array:
    runs-on: ubuntu-latest
    steps:
      # Acceder a elementos por índice
      - name: Primer elemento
        run: echo "${{ github.event.commits[0].message }}"
      
      # Acceder a propiedades de objeto
      - name: Acceder a propiedad
        run: echo "${{ github.event.pull_request['head']['ref'] }}"
```

### Notación de Punto (`.`)

```yaml
jobs:
  dot-notation:
    runs-on: ubuntu-latest
    steps:
      # Acceder a propiedades anidadas
      - name: Usar notación de punto
        run: |
          echo "Ref: ${{ github.event.pull_request.head.ref }}"
          echo "Repo: ${{ github.repository }}"
          echo "Actor: ${{ github.actor }}"
          echo "SHA: ${{ github.event.pull_request.head.sha }}"
```

### Combinación de Notaciones

```yaml
jobs:
  mixed-notation:
    runs-on: ubuntu-latest
    steps:
      - name: Notación mixta
        run: |
          echo "Autor del primer commit: ${{ github.event.commits[0].author.name }}"
          echo "Email: ${{ github.event.commits[0].author.email }}"
          echo "Label: ${{ github.event.pull_request.labels[0].name }}"
```

---

## Funciones como Operadores

GitHub Actions proporciona funciones integradas que actúan como operadores.

### `contains()`

Verifica si una cadena contiene otra cadena o si un array contiene un valor.

```yaml
jobs:
  check-message:
    # Verificar si el mensaje contiene una palabra
    if: contains(github.event.head_commit.message, 'hotfix')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Es un hotfix"

  check-label:
    # Verificar si tiene una etiqueta específica
    if: contains(github.event.pull_request.labels.*.name, 'bug')
    runs-on: ubuntu-latest
    steps:
      - run: echo "PR marcado como bug"

  check-branch:
    # Verificar si está en el array de ramas
    if: contains(fromJSON('["main", "develop", "staging"]'), github.ref_name)
    runs-on: ubuntu-latest
    steps:
      - run: echo "Rama importante"
```

### `startsWith()`

Verifica si una cadena comienza con otra cadena.

```yaml
jobs:
  tag-check:
    # Verificar si es un tag de versión
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Tag de versión"

  feature-branch:
    # Verificar si es rama de feature
    if: startsWith(github.ref, 'refs/heads/feature/')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Rama de feature"

  dependency-update:
    # Verificar si el commit es de actualización
    if: startsWith(github.event.head_commit.message, 'chore(deps):')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Actualización de dependencias"
```

### `endsWith()`

Verifica si una cadena termina con otra cadena.

```yaml
jobs:
  python-files:
    # Verificar si el archivo es Python
    if: endsWith(github.event.pull_request.head.ref, '.py')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Archivo Python"

  docs-branch:
    # Verificar si termina en -docs
    if: endsWith(github.ref_name, '-docs')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Rama de documentación"

  test-files:
    # Verificar archivos de test
    if: endsWith(github.event.head_commit.modified[0], '.test.js')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Archivo de test modificado"
```

### `format()`

Formatea una cadena con placeholders.

```yaml
jobs:
  format-message:
    runs-on: ubuntu-latest
    steps:
      - name: Mensaje formateado
        run: |
          echo "${{ format('Hola {0}, bienvenido a {1}', github.actor, github.repository) }}"
      
      - name: URL formateada
        run: |
          echo "${{ format('https://github.com/{0}/actions/runs/{1}', github.repository, github.run_id) }}"
      
      - name: Versión formateada
        run: |
          echo "${{ format('v{0}.{1}.{2}', '1', '2', '3') }}"
```

### `join()`

Une elementos de un array con un separador.

```yaml
jobs:
  join-example:
    runs-on: ubuntu-latest
    steps:
      - name: Unir array
        run: |
          echo "${{ join(github.event.pull_request.labels.*.name, ', ') }}"
      
      - name: Unir paths
        run: |
          echo "${{ join(fromJSON('["src", "lib", "tests"]'), ':') }}"
```

### `toJSON()` y `fromJSON()`

Convierte entre JSON y objetos.

```yaml
jobs:
  json-operations:
    runs-on: ubuntu-latest
    steps:
      - name: Convertir a JSON
        run: |
          echo '${{ toJSON(github.event) }}'
      
      - name: Parsear JSON
        run: |
          echo "${{ fromJSON('{"name":"value"}').name }}"
      
      - name: Array desde JSON
        if: contains(fromJSON('["main","develop"]'), github.ref_name)
        run: echo "Rama válida"
```

### `hashFiles()`

Genera un hash de archivos (útil para cache).

```yaml
jobs:
  cache-dependencies:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      
      - name: Multi-file hash
        run: |
          echo "${{ hashFiles('**/package-lock.json', '**/yarn.lock') }}"
```

### Funciones de Estado

```yaml
jobs:
  always-run:
    runs-on: ubuntu-latest
    if: always()  # Siempre se ejecuta
    steps:
      - run: echo "Limpieza"

  on-success:
    runs-on: ubuntu-latest
    if: success()  # Solo si todo fue exitoso
    steps:
      - run: echo "Éxito"

  on-failure:
    runs-on: ubuntu-latest
    if: failure()  # Solo si algo falló
    steps:
      - run: echo "Hubo un error"

  on-cancelled:
    runs-on: ubuntu-latest
    if: cancelled()  # Solo si fue cancelado
    steps:
      - run: echo "Workflow cancelado"
```

---

## Ejemplos Prácticos

### Ejemplo 1: Deploy Condicional Complejo

```yaml
name: Deploy

on: [push, pull_request]

jobs:
  deploy-production:
    # Múltiples operadores combinados
    if: |
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main' &&
      !contains(github.event.head_commit.message, '[skip ci]') &&
      github.actor != 'dependabot[bot]'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: echo "Desplegando a producción"

  deploy-staging:
    if: |
      (github.event_name == 'push' && github.ref == 'refs/heads/develop') ||
      (github.event_name == 'pull_request' && github.base_ref == 'develop')
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - run: echo "Desplegando a staging"
```

### Ejemplo 2: Validación de PR

```yaml
name: PR Validation

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  validate-title:
    runs-on: ubuntu-latest
    steps:
      - name: Verificar formato del título
        if: |
          !startsWith(github.event.pull_request.title, 'feat:') &&
          !startsWith(github.event.pull_request.title, 'fix:') &&
          !startsWith(github.event.pull_request.title, 'docs:') &&
          !startsWith(github.event.pull_request.title, 'chore:')
        run: |
          echo "El título del PR debe comenzar con: feat:, fix:, docs:, o chore:"
          exit 1

  require-labels:
    runs-on: ubuntu-latest
    steps:
      - name: Verificar etiquetas
        if: |
          !contains(github.event.pull_request.labels.*.name, 'bug') &&
          !contains(github.event.pull_request.labels.*.name, 'enhancement') &&
          !contains(github.event.pull_request.labels.*.name, 'documentation')
        run: |
          echo "El PR debe tener al menos una etiqueta: bug, enhancement, o documentation"
          exit 1

  check-size:
    runs-on: ubuntu-latest
    steps:
      - name: PR muy grande
        if: github.event.pull_request.changed_files > 50
        run: |
          echo "⚠️ Este PR modifica más de 50 archivos. Considera dividirlo."
      
      - name: PR enorme
        if: github.event.pull_request.changed_files > 100
        run: |
          echo "❌ Este PR es demasiado grande. Debe dividirse."
          exit 1
```

### Ejemplo 3: Matriz Condicional

```yaml
name: Matrix CI

on: [push, pull_request]

jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
        include:
          - os: ubuntu-latest
            node: 20
            experimental: true
        exclude:
          - os: windows-latest
            node: 16
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      
      - name: Instalar dependencias
        run: npm ci
      
      - name: Run tests
        # Continue on error solo para builds experimentales
        continue-on-error: ${{ matrix.experimental == true }}
        run: npm test
      
      - name: Notificar si es experimental
        if: matrix.experimental == true && failure()
        run: echo "⚠️ Build experimental falló"
      
      - name: Test adicionales para Node 20
        if: matrix.node >= 20
        run: npm run test:experimental
      
      - name: Verificar solo en Ubuntu
        if: matrix.os == 'ubuntu-latest'
        run: npm run lint
```

### Ejemplo 4: Manejo de Reintentos

```yaml
name: Tests con Reintentos

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests
        id: test
        continue-on-error: true
        run: npm test
      
      - name: Primer reintento
        if: steps.test.outcome == 'failure' && github.run_attempt == 1
        run: |
          echo "Primer intento falló, reintentando..."
          npm test
      
      - name: Segundo reintento
        if: steps.test.outcome == 'failure' && github.run_attempt == 2
        run: |
          echo "Segundo intento falló, último reintento..."
          npm test
      
      - name: Notificar fallo final
        if: |
          failure() &&
          github.run_attempt >= 3
        run: |
          echo "❌ Tests fallaron después de 3 intentos"
          exit 1
```

### Ejemplo 5: Filtrado por Paths

```yaml
name: Path-based CI

on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package*.json'

jobs:
  backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      
      - name: Verificar cambios en backend
        id: backend-changed
        run: |
          if git diff --name-only HEAD^ HEAD | grep -E '^src/backend/'; then
            echo "changed=true" >> $GITHUB_OUTPUT
          else
            echo "changed=false" >> $GITHUB_OUTPUT
          fi
      
      - name: Tests de backend
        if: steps.backend-changed.outputs.changed == 'true'
        run: npm run test:backend
  
  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      
      - name: Verificar cambios en frontend
        id: frontend-changed
        run: |
          if git diff --name-only HEAD^ HEAD | grep -E '^src/frontend/'; then
            echo "changed=true" >> $GITHUB_OUTPUT
          else
            echo "changed=false" >> $GITHUB_OUTPUT
          fi
      
      - name: Tests de frontend
        if: steps.frontend-changed.outputs.changed == 'true'
        run: npm run test:frontend
```

### Ejemplo 6: Autenticación Condicional

```yaml
name: Conditional Auth

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Login solo en push a main
      - name: Login a Docker Hub
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build imagen
        run: docker build -t myapp:latest .
      
      # Push solo si estamos en main y es push
      - name: Push imagen
        if: |
          github.event_name == 'push' &&
          github.ref == 'refs/heads/main' &&
          success()
        run: docker push myapp:latest
```

---

## Precedencia de Operadores

La precedencia determina el orden en que se evalúan los operadores. De mayor a menor precedencia:

1. **`()`** - Agrupación
2. **`[]`** - Índice
3. **`.`** - Acceso a propiedad
4. **`!`** - NOT lógico
5. **`<`, `<=`, `>`, `>=`** - Comparación
6. **`==`, `!=`** - Igualdad
7. **`&&`** - AND lógico
8. **`||`** - OR lógico

### Ejemplos de Precedencia

```yaml
jobs:
  # Sin paréntesis (puede ser confuso)
  precedence-1:
    # Se evalúa como: A || (B && C)
    if: github.event_name == 'push' || github.ref == 'refs/heads/main' && success()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Puede no ser lo esperado"

  # Con paréntesis (claro)
  precedence-2:
    # Se evalúa explícitamente: (A || B) && C
    if: (github.event_name == 'push' || github.ref == 'refs/heads/main') && success()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Intención clara"

  # NOT tiene alta precedencia
  precedence-3:
    # Se evalúa como: (!A) && B
    if: '!failure() && github.ref == ''refs/heads/main'''
    runs-on: ubuntu-latest
    steps:
      - run: echo "NOT se evalúa primero"

  # Comparación antes que lógicos
  precedence-4:
    # Se evalúa como: (A == B) && (C == D)
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Las comparaciones se evalúan primero"
```

---

## Best Practices

### 1. Usa Paréntesis para Claridad

```yaml
# ✅ Recomendado - Claro y explícito
if: (github.event_name == 'push' || github.event_name == 'pull_request') && github.ref == 'refs/heads/main'

# ❌ Evitar - Confuso
if: github.event_name == 'push' || github.event_name == 'pull_request' && github.ref == 'refs/heads/main'
```

### 2. Separa Condiciones Complejas en Múltiples Líneas

```yaml
# ✅ Recomendado - Fácil de leer
if: |
  github.event_name == 'push' &&
  github.ref == 'refs/heads/main' &&
  !contains(github.event.head_commit.message, '[skip ci]') &&
  github.actor != 'dependabot[bot]'

# ❌ Evitar - Difícil de leer
if: github.event_name == 'push' && github.ref == 'refs/heads/main' && !contains(github.event.head_commit.message, '[skip ci]') && github.actor != 'dependabot[bot]'
```

### 3. Usa Funciones en Lugar de Operadores Complejos

```yaml
# ✅ Recomendado - Usa contains()
if: contains(github.event.head_commit.message, 'hotfix')

# ❌ Evitar - Regex complejo (no soportado directamente)
# GitHub Actions no soporta regex en expresiones directamente
```

### 4. Combina Operadores de Forma Lógica

```yaml
# ✅ Recomendado - Lógica clara
if: |
  (github.event_name == 'push' && github.ref == 'refs/heads/main') ||
  (github.event_name == 'workflow_dispatch' && inputs.environment == 'production')

# ⚠️ Menos ideal - Condiciones separadas
# Considera usar un solo if bien estructurado
```

### 5. Evita Doble Negación

```yaml
# ✅ Recomendado
if: github.event.pull_request.draft == false

# ❌ Evitar - Confuso
if: '!(!github.event.pull_request.draft)'
```

### 6. Documenta Condiciones Complejas

```yaml
jobs:
  deploy:
    # Deploy solo cuando:
    # - Es push a main
    # - No es dependabot
    # - No contiene [skip ci]
    # - Es el primer intento
    if: |
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main' &&
      github.actor != 'dependabot[bot]' &&
      !contains(github.event.head_commit.message, '[skip ci]') &&
      github.run_attempt == 1
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### 7. Usa Variables para Condiciones Reutilizables

```yaml
env:
  IS_MAIN: ${{ github.ref == 'refs/heads/main' }}
  IS_PUSH: ${{ github.event_name == 'push' }}

jobs:
  deploy:
    # Usar variables de entorno
    if: env.IS_MAIN == 'true' && env.IS_PUSH == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy"
```

### 8. Testa Condiciones Complejas

```yaml
jobs:
  debug-conditions:
    runs-on: ubuntu-latest
    steps:
      - name: Debug información
        run: |
          echo "Event: ${{ github.event_name }}"
          echo "Ref: ${{ github.ref }}"
          echo "Actor: ${{ github.actor }}"
          echo "Run attempt: ${{ github.run_attempt }}"
      
      - name: Test condición
        if: |
          github.event_name == 'push' &&
          github.ref == 'refs/heads/main'
        run: echo "✅ Condición cumplida"
```

---

## Tabla Resumen de Operadores

| Operador | Tipo | Descripción | Ejemplo |
|----------|------|-------------|---------|
| `==` | Comparación | Igualdad | `a == b` |
| `!=` | Comparación | Desigualdad | `a != b` |
| `>` | Comparación | Mayor que | `a > b` |
| `>=` | Comparación | Mayor o igual | `a >= b` |
| `<` | Comparación | Menor que | `a < b` |
| `<=` | Comparación | Menor o igual | `a <= b` |
| `&&` | Lógico | AND | `a && b` |
| `\|\|` | Lógico | OR | `a \|\| b` |
| `!` | Lógico | NOT | `!a` |
| `()` | Agrupación | Paréntesis | `(a \|\| b) && c` |
| `[]` | Índice | Acceso a array | `array[0]` |
| `.` | Acceso | Propiedad | `object.property` |

---

## Referencias

- [GitHub Actions - Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [GitHub Actions - Operators](https://docs.github.com/en/actions/learn-github-actions/expressions#operators)
- [GitHub Actions - Functions](https://docs.github.com/en/actions/learn-github-actions/expressions#functions)
- [GitHub Actions - Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts)
