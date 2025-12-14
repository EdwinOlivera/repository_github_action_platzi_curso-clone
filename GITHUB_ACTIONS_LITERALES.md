# Literales en GitHub Actions

## Tabla de Contenidos
1. [¿Qué son los Literales?](#qué-son-los-literales)
2. [Tipos de Literales](#tipos-de-literales)
3. [Uso de Literales en Expresiones](#uso-de-literales-en-expresiones)
4. [Ejemplos Prácticos](#ejemplos-prácticos)
5. [Best Practices](#best-practices)

---

## ¿Qué son los Literales?

Los **literales** son valores fijos que se escriben directamente en el código de tu workflow de GitHub Actions. Representan datos constantes que no cambian durante la ejecución.

En GitHub Actions, los literales se utilizan principalmente dentro de expresiones (`${{ }}`), condiciones (`if`), y como valores de configuración.

---

## Tipos de Literales

### 1. Literales de Cadena (String)

Las cadenas de texto se pueden escribir de dos formas:

#### Comillas Simples (')
```yaml
# Recomendado para cadenas simples
if: github.ref == 'refs/heads/main'
if: github.actor == 'dependabot[bot]'
if: matrix.os == 'ubuntu-latest'
```

#### Comillas Dobles (")
```yaml
# Para cadenas que necesitan escapar caracteres
if: github.event.head_commit.message == "Deploy \"production\""
```

**Ejemplos:**
```yaml
jobs:
  ejemplo:
    runs-on: ubuntu-latest
    steps:
      - name: Comparar strings
        if: github.event_name == 'push'
        run: echo "Es un push"
      
      - name: Comparar rama
        if: github.ref == 'refs/heads/develop'
        run: echo "Rama develop"
      
      - name: Verificar actor
        if: github.actor == 'octocat'
        run: echo "El actor es octocat"
```

### 2. Literales Numéricos

Los números se escriben sin comillas.

#### Números Enteros
```yaml
# Configuración con números
jobs:
  build:
    timeout-minutes: 30        # Literal numérico: 30
    strategy:
      max-parallel: 5           # Literal numérico: 5
```

#### Números con Decimales
```yaml
# En expresiones
steps:
  - name: Comparar versión
    if: matrix.node-version > 16.5
    run: echo "Versión mayor a 16.5"
```

**Ejemplos:**
```yaml
jobs:
  ejemplo-numeros:
    runs-on: ubuntu-latest
    steps:
      - name: Comparar números
        if: github.run_number > 100
        run: echo "Más de 100 ejecuciones"
      
      - name: Verificar attempt
        if: github.run_attempt == 1
        run: echo "Primer intento"
      
      - name: Check parallel jobs
        if: strategy.job-total == 10
        run: echo "10 jobs en total"
```

### 3. Literales Booleanos

Los valores booleanos representan verdadero o falso.

```yaml
# Valores: true, false
jobs:
  build:
    continue-on-error: true     # Literal booleano
    
    steps:
      - name: Puede fallar
        continue-on-error: false  # Literal booleano
        run: exit 1
```

**En expresiones:**
```yaml
steps:
  - name: Comparar boolean
    if: runner.debug == true
    run: echo "Debug activado"
  
  - name: Verificar draft
    if: github.event.pull_request.draft == false
    run: echo "No es un draft"
```

### 4. Literal Null

Representa la ausencia de valor.

```yaml
steps:
  - name: Verificar null
    if: github.event.pull_request.milestone == null
    run: echo "No tiene milestone asignado"
  
  - name: Verificar assignee
    if: github.event.issue.assignee != null
    run: echo "Tiene assignee"
```

### 5. Literales NaN (Not a Number)

Representa un valor que no es un número válido.

```yaml
steps:
  - name: Verificar NaN
    if: steps.calculate.outputs.result != NaN
    run: echo "El resultado es un número válido"
```

### 6. Literales Inf y -Inf (Infinito)

Representan infinito positivo y negativo.

```yaml
steps:
  - name: Comparar con infinito
    if: steps.calculate.outputs.max != Inf
    run: echo "El valor no es infinito"
```

---

## Uso de Literales en Expresiones

### Sintaxis Básica

```yaml
# Sintaxis de expresión
${{ expresión }}

# Ejemplos con literales
${{ 'cadena de texto' }}
${{ 42 }}
${{ true }}
${{ null }}
```

### En Condicionales (if)

```yaml
jobs:
  deploy:
    # Comparando con literal de string
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy a producción"

  test:
    # Comparando con literal numérico
    if: github.run_attempt <= 3
    runs-on: ubuntu-latest
    steps:
      - run: echo "Intento de ejecución válido"

  notify:
    # Comparando con literal booleano
    if: success() == true
    runs-on: ubuntu-latest
    steps:
      - run: echo "Notificar éxito"
```

### En Asignaciones

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # Asignar literal a variable
      - name: Set variables
        run: |
          echo "VERSION=1.0.0" >> $GITHUB_ENV
          echo "ENVIRONMENT=production" >> $GITHUB_ENV
          echo "DEBUG=false" >> $GITHUB_ENV
```

### En Outputs

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      # Output con literal
      status: 'success'
      code: '200'
      enabled: 'true'
    steps:
      - id: set-output
        run: |
          echo "status=success" >> $GITHUB_OUTPUT
          echo "code=200" >> $GITHUB_OUTPUT
          echo "enabled=true" >> $GITHUB_OUTPUT
```

---

## Ejemplos Prácticos

### Ejemplo 1: Validación de Rama

```yaml
name: Deploy

on:
  push:
    branches: [main, develop]

jobs:
  deploy-production:
    # Comparar con literal 'refs/heads/main'
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Desplegando a producción"

  deploy-staging:
    # Comparar con literal 'refs/heads/develop'
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Desplegando a staging"
```

### Ejemplo 2: Verificación de Etiquetas

```yaml
name: Label Actions

on:
  pull_request:
    types: [labeled]

jobs:
  auto-merge:
    # Comparar con literal 'auto-merge'
    if: github.event.label.name == 'auto-merge'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Activar auto-merge"

  ready-for-review:
    # Comparar con literal 'ready-for-review'
    if: github.event.label.name == 'ready-for-review'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Notificar revisores"
```

### Ejemplo 3: Control de Reintentos

```yaml
name: Test con Reintentos

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: npm test

      - name: Primer reintento
        # Comparar con literal numérico 1
        if: failure() && github.run_attempt == 1
        run: echo "Primer reintento fallido"

      - name: Segundo reintento
        # Comparar con literal numérico 2
        if: failure() && github.run_attempt == 2
        run: echo "Segundo reintento fallido"

      - name: Máximo de reintentos
        # Comparar con literal numérico 3
        if: failure() && github.run_attempt >= 3
        run: |
          echo "Se alcanzó el máximo de reintentos"
          exit 1
```

### Ejemplo 4: Verificación de Actor

```yaml
name: Special Handling

on: [push, pull_request]

jobs:
  dependabot-auto-merge:
    # Comparar con literal 'dependabot[bot]'
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Procesando PR de Dependabot"

  regular-ci:
    # Comparar que NO sea el literal 'dependabot[bot]'
    if: github.actor != 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - run: echo "CI normal"
```

### Ejemplo 5: Estados y Condiciones

```yaml
name: Conditional Steps

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build
        id: build
        run: npm run build

      - name: Si build es exitoso
        # Comparar función con literal true
        if: success() == true
        run: echo "Build exitoso"

      - name: Si build falló
        # Comparar función con literal false
        if: success() == false
        run: echo "Build falló"

      - name: Siempre ejecutar limpieza
        # always() retorna literal true
        if: always() == true
        run: echo "Limpieza"
```

### Ejemplo 6: Verificación de Eventos

```yaml
name: Event Specific Actions

on:
  push:
  pull_request:
  issues:
  workflow_dispatch:

jobs:
  handle-push:
    # Comparar con literal 'push'
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Manejando push"

  handle-pr:
    # Comparar con literal 'pull_request'
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Manejando pull request"

  handle-issue:
    # Comparar con literal 'issues'
    if: github.event_name == 'issues'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Manejando issue"

  handle-manual:
    # Comparar con literal 'workflow_dispatch'
    if: github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Ejecución manual"
```

### Ejemplo 7: Verificación de Paths

```yaml
name: Path Based Actions

on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'

jobs:
  backend-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run backend tests
        # Verificar si cambió el backend
        if: contains(github.event.head_commit.modified, 'src/backend')
        run: npm run test:backend

      - name: Run frontend tests
        # Verificar si cambió el frontend (literal en contains)
        if: contains(github.event.head_commit.modified, 'src/frontend')
        run: npm run test:frontend
```

### Ejemplo 8: Verificación de Null

```yaml
name: Milestone Check

on:
  pull_request:
    types: [opened, edited]

jobs:
  check-milestone:
    runs-on: ubuntu-latest
    steps:
      - name: PR sin milestone
        # Comparar con literal null
        if: github.event.pull_request.milestone == null
        run: |
          echo "Este PR no tiene milestone asignado"
          exit 1

      - name: PR con milestone
        # Verificar que NO sea null
        if: github.event.pull_request.milestone != null
        run: echo "Milestone: ${{ github.event.pull_request.milestone.title }}"
```

---

## Best Practices

### 1. Usa Comillas Simples para Strings

```yaml
# ✅ Recomendado
if: github.ref == 'refs/heads/main'

# ❌ Evitar (a menos que necesites escapar)
if: github.ref == "refs/heads/main"
```

### 2. Sé Específico con las Comparaciones

```yaml
# ✅ Recomendado - Comparación explícita
if: github.event_name == 'push'

# ⚠️ Puede funcionar pero menos claro
if: github.event_name
```

### 3. Usa Literales Apropiados para el Tipo

```yaml
# ✅ Correcto - Boolean sin comillas
continue-on-error: true

# ❌ Incorrecto - String en lugar de boolean
continue-on-error: 'true'

# ✅ Correcto - Número sin comillas
timeout-minutes: 30

# ❌ Incorrecto - String en lugar de número
timeout-minutes: '30'
```

### 4. Verifica Valores Nulos Explícitamente

```yaml
# ✅ Recomendado
if: github.event.pull_request.assignee != null

# ⚠️ Menos claro
if: github.event.pull_request.assignee
```

### 5. Usa Literales en Lugar de Variables cuando sea Posible

```yaml
# ✅ Más rápido - literal directo
if: github.ref == 'refs/heads/main'

# ⚠️ Más lento - requiere evaluación de variable
env:
  MAIN_BRANCH: 'refs/heads/main'
# ...
if: github.ref == env.MAIN_BRANCH
```

### 6. Documenta Literales "Mágicos"

```yaml
jobs:
  deploy:
    # Comparar con 'refs/heads/main' porque solo queremos
    # desplegar desde la rama principal
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### 7. Cuidado con Case Sensitivity

```yaml
# ✅ Correcto
if: github.event_name == 'push'

# ❌ Incorrecto - case sensitive
if: github.event_name == 'Push'
if: github.event_name == 'PUSH'
```

### 8. Usa Constantes para Literales Repetidos

```yaml
env:
  # Definir constantes globales
  MAIN_BRANCH: 'main'
  PRODUCTION_ENV: 'production'
  MIN_NODE_VERSION: '18'

jobs:
  deploy:
    if: github.ref == format('refs/heads/{0}', env.MAIN_BRANCH)
    environment: ${{ env.PRODUCTION_ENV }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.MIN_NODE_VERSION }}
```

---

## Resumen de Tipos de Literales

| Tipo | Ejemplo | Uso |
|------|---------|-----|
| **String** | `'main'`, `"hello"` | Comparaciones de texto, nombres, rutas |
| **Number** | `42`, `3.14` | Timeouts, contadores, versiones |
| **Boolean** | `true`, `false` | Flags, estados, condiciones |
| **Null** | `null` | Verificar ausencia de valor |
| **NaN** | `NaN` | Validar cálculos numéricos |
| **Inf** | `Inf`, `-Inf` | Comparaciones de límites |

---

## Referencias

- [GitHub Actions - Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [GitHub Actions - Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts)
- [GitHub Actions - Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
