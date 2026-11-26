# Trazabilidad

## Requisito → gate → evidencia

| Requisito | Gate o test | Bloquea / avisa | Etapa | Run fallido | Run corregido |
|---|---|---|---|---|---|
| **2.1** `aprobados` no es arreglo → 400 (hallazgo 3) | Test `2.1: aprobados = "1INF25", 123, null, {} → 400` en `tests/habilitados.test.js` | Bloquea (job `test`) | PR y push a `main` | [run 45903653155](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903653155) | [run 45903954511](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903954511) |
| **3.1** Más de 100 elementos → 400 (hallazgo 1) | Tests `3.1: 101 elementos → 400` y `5 000 elementos → 400 o 413` | Bloquea (job `test`) | PR y push a `main` | [run 45903653155](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903653155) | [run 45903954511](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903954511) |
| **3.3** 100 elementos responde en < 500 ms (hallazgo 1) | Test `3.3: 100 elementos responde en < 500 ms` | Bloquea (job `test`) | PR y push a `main` | [run 45903653155](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903653155) | [run 45903954511](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903954511) |
| **4.3** Código fuera de `^[0-9][A-Z]{3}[0-9]{2}$` → 400 sin eco (hallazgo 2) | Test `4.3: ["1INF30<script>"] → 400 y body sin <script>` | Bloquea (job `test`) | PR y push a `main` | [run 45903653155](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903653155) | [run 45903954511](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903954511) |
| **5.1 / 5.2** Errores sin stack trace ni rutas; excepción → 500 genérico (hallazgo 2) | Tests `5.1: body solo contiene error…` y `5.2: mock repositorio.listar lanza → 500` | Bloquea (job `test`) | PR y push a `main` | [run 45903653155](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903653155) | [run 45903954511](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903954511) |
| **6.1** Cobertura de líneas < 80 % → pipeline falla | `jest --coverage` con `coverageThreshold.global.lines = 80` | Bloquea (job `test`) | PR y push a `main` | [run 45903653155](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903653155): cobertura 42.18 % | [run 45903954511](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45903954511): cobertura 100 % |
| **7.1** Dependencia high/critical → pipeline falla | `npm audit --audit-level=high` | Bloquea (job `sca`) | PR y push a `main` | [run 45904626522](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45904626522): `lodash 4.17.20` con vulnerabilidad alta | [run 45905396991](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45905396991): `lodash ^4.18.1` |
| **8.1** Secreto detectado → pipeline falla | gitleaks (`gitleaks/gitleaks-action@v2`), umbral 0 hallazgos | Bloquea (job `secrets`) | PR y push a `main` | No observable; ver diagnóstico | [run 45905396991](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45905396991) |
| SAST (extra) | CodeQL `javascript` | Avisa (`continue-on-error: true`) | PR y push a `main` | — | [run 45905396991](https://github.com/ezeAst/INF384-lab6-base/actions/runs/45905396991) |

### Diagnóstico del run 45903653155 (job `test` en rojo)
Falló con `Cannot find module './routes/habilitados'`: `src/app.js` montaba la ruta, pero `src/routes/habilitados.js` y `src/middleware/validarAprobados.js` no estaban en el commit. Por eso fallaron los tests de los requisitos 1 a 5 y la cobertura bajó a 42.18 %. Se corrigió agregando ambos archivos (commit `7fbec37`), y el run 45903954511 quedó en verde.

### Diagnóstico del run 45904626522 (job `sca` en rojo)
`npm audit --audit-level=high` reportó una vulnerabilidad alta en `lodash 4.17.20`. Se corrigió actualizando a `lodash ^4.18.1` (commit `71bb3a4`), que no tiene advisories high/critical. El run 45905396991 quedó en verde.

### Diagnóstico del gate de secretos (modo de escaneo)
Actions estaba desactivado en el fork, así que el primer run fue el del commit `c750857`, cuando la clave ya se había eliminado. En un evento `push`, `gitleaks-action` escanea solo los commits incluidos en ese push, no el historial completo. Por eso no detectó la clave `CATALOGO_API_KEY` agregada en el commit `7506fec`, y no hubo un run en rojo.
Corrección aplicada: la clave se eliminó de `config/ejemplo.env` (commit `774bd42`), se considera **rotada** y su fingerprint quedó registrado en `.gitleaksignore` con la razón. Así, un escaneo del historial completo tampoco falla por una clave que ya no es válida.

## Cambios a la propuesta de Kiro

| Documento | Qué propuso Kiro | Qué cambié | Por qué |
|---|---|---|---|
| `requirements.md` | Límite de 500 elementos en `aprobados`, sin tiempo de respuesta | Límite de 100 y criterio 3.3: respuesta < 500 ms con 100 elementos | El catálogo tiene 87 cursos; 500 no corta un abuso realista y "sin procesar" no era medible. 100 deja margen, y el tiempo con número hace verificable el hallazgo 1. |
| `requirements.md` | Elementos solo con caracteres `[A-Za-z0-9]` | Formato exacto `^[0-9][A-Z]{3}[0-9]{2}$` y el `error` no repite el valor | El charset aceptaba `"abc"`; el formato exacto rechaza `1INF30<script>` por definición, y no repetir el valor evita un XSS reflejado (hallazgo 2). |
| `tasks.md` | Tests marcados como opcionales (`*`), gates dependientes de `test` y sin tareas para corregir lodash ni la clave | Tests obligatorios escritos primero, jobs independientes y tareas 8 y 9 de corrección | La rúbrica exige un test por criterio y evidencia de fallo y de corrección; con `needs: test`, un fallo de tests ocultaba los gates de seguridad. |
| `design.md` | Dependencia nueva `fast-check` para property-based tests | Tests de ejemplo con supertest, sin dependencias nuevas | `tech.md` exige justificar cada dependencia, y cada una amplía la superficie que revisa el gate de SCA. |