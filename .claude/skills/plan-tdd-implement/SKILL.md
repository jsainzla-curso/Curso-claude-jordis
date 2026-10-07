---
name: plan-tdd-implement
description: Planifica un cambio en Resttek, lo implementa con TDD (rojo → verde → refactor) dentro de un git worktree aislado y avisa por Slack al terminar el plan y al terminar la implementación. Úsala cuando el usuario pida "planifica e implementa", "hazlo con TDD" o invoque /plan-tdd-implement <descripción del cambio>.
argument-hint: <descripción del cambio>
---

# Plan → TDD → Implementación (en git worktree)

Cambio solicitado: **$ARGUMENTS**

Si `$ARGUMENTS` está vacío, pregunta al usuario qué cambio quiere antes de seguir.

Sigue las fases **en orden**. No te saltes ninguna ni cambies el orden.

---

## Fase 0 — Preparar el git worktree (obligatorio)

**Todos los cambios se hacen en un git worktree, nunca en el directorio de trabajo principal.**

1. Comprueba que estás en un repositorio git: `git rev-parse --is-inside-work-tree`.
   - Si no lo es, **para** e informa al usuario. No ejecutes `git init` por tu cuenta.
2. Comprueba que el árbol principal está limpio (`git status --porcelain`). Si hay cambios sin commitear, avísalo al usuario: el worktree parte de `HEAD` y no los incluirá.
3. Elige un *slug* corto en kebab-case a partir del cambio (p. ej. `filtro-pedidos-por-mesa`).
4. Crea el worktree con su rama:
   - Preferente: la herramienta `EnterWorktree` (cárgala con `ToolSearch` → `select:EnterWorktree`), con el nombre `feat/<slug>`.
   - Alternativa por shell:
     ```bash
     git worktree add ../resttek-wt-<slug> -b feat/<slug>
     ```
5. Instala dependencias dentro del worktree: `npm install` (es un monorepo con npm workspaces).
6. A partir de aquí, **todas** las lecturas, ediciones, tests y commits se hacen con rutas dentro del worktree. Verifica con `git -C <worktree> branch --show-current` antes de editar.

---

## Fase 1 — Generar el plan

1. Lee `CLAUDE.md` y la documentación relevante de `docs/` (`arquitectura/`, `dominio/`) antes de proponer nada.
2. Explora el código afectado. Identifica:
   - Paquetes implicados (`api`, `web-admin`, `web-empleados`, `web-clientes`, `web-shared`).
   - En la API, el estilo del módulo (hexagonal/DDD en `src/contexts/employee/` o por capas en el resto). **Respeta el estilo del módulo que toques.**
3. Escribe el plan en `docs/planes/<slug>.md` **dentro del worktree**, en castellano, con esta estructura:

   ```markdown
   # Plan: <título>

   ## Objetivo
   ## Contexto y ficheros afectados
   ## Diseño propuesto
   ## Casos de test (TDD)
   - [ ] Test 1: <comportamiento esperado> → <fichero *.test.ts>
   - [ ] Test 2: ...
   ## Pasos de implementación
   1. ...
   ## Riesgos y documentación a actualizar
   ```

   - Cada caso de test describe un **comportamiento** concreto, no una implementación.
   - Los frontends no tienen tests: para cambios solo de frontend, indica cómo se verificará (p. ej. `ng build` del paquete y prueba manual con la skill `run`) y, si hay lógica extraíble, propón moverla a la API o a un servicio testeable.
4. Haz commit del plan en la rama del worktree:
   `git commit -m "docs: plan para <slug>"`.

### Aviso por Slack: plan terminado

Envía un mensaje al canal **`#planes-generales`** usando la herramienta de Slack disponible (busca con `ToolSearch` la consulta `slack`; normalmente algo como `mcp__slack__*send_message` / `post_message`). Contenido:

```
📝 Plan listo: <título>
Rama: feat/<slug>
Fichero: docs/planes/<slug>.md
Resumen: <2-3 líneas>
Tests previstos: <n>
```

Si no hay ninguna herramienta de Slack disponible o el envío falla, **no te lo inventes**: díselo al usuario, muéstrale el mensaje que se habría enviado y continúa.

---

## Fase 2 — TDD (rojo → verde → refactor)

Repite este ciclo **por cada caso de test del plan**, uno a uno:

1. **Rojo**: escribe el test (Vitest, junto al código como `*.test.ts`; dobles en carpetas `mocks/`). Ejecútalo y **confirma que falla por el motivo esperado** (no por un error de compilación o de import).
   ```bash
   npm test                                  # desde la raíz del worktree
   npx -w @resttek/api vitest run <ruta>     # un solo fichero
   ```
2. **Verde**: escribe el código mínimo para que pase. Vuelve a ejecutar y confirma que pasa.
3. **Refactor**: limpia sin cambiar el comportamiento y vuelve a ejecutar toda la suite.
4. Marca el caso como `[x]` en el plan y haz commit:
   `git commit -m "test+feat: <comportamiento>"`.

Reglas:
- Nunca escribas código de producción sin un test en rojo que lo justifique (salvo en frontend, ver Fase 1).
- No modifiques ni borres tests existentes para hacerlos pasar; si un test existente contradice el cambio, para y consulta al usuario.
- Código en inglés; textos de interfaz y mensajes de error al usuario en castellano. Valores de dominio tal como están (`manager`, `pendiente`, `cocinero`…).
- Los errores de dominio heredan de `AppError`; los imports de la API llevan extensión `.js` y usan los path aliases.

---

## Fase 3 — Implementación final y verificación

1. Completa los pasos de implementación del plan que no estén cubiertos por el ciclo TDD (rutas, wiring, frontend, etc.), siguiendo las convenciones de `CLAUDE.md`.
2. Si el cambio contradice algo de `docs/`, actualiza también la documentación.
3. Verificación final, todo desde el worktree:
   - `npm test` → toda la suite en verde.
   - Si se tocó un frontend: `npm run build -w @resttek/<paquete>` sin errores.
4. Commit final. **No hagas push, merge ni elimines el worktree** salvo que el usuario lo pida.

### Aviso por Slack: implementación terminada

Envía un mensaje al canal **`#planes-generales`** (salvo que el usuario indique otro canal):

```
✅ Implementación terminada: <título>
Rama: feat/<slug>  ·  Worktree: <ruta>
Tests: <n pasados>/<n totales>
Commits: <n>
Pendiente: <revisión / merge / nada>
```

Informa con fidelidad: si algún test falla o algún paso quedó sin hacer, dilo en el mensaje. Si Slack no está disponible, díselo al usuario y muestra el mensaje.

---

## Resumen final al usuario

Termina con un resumen breve en castellano: ruta del worktree, rama, fichero del plan, resultado de los tests, estado de los dos avisos de Slack (enviado / no enviado y por qué) y los siguientes pasos sugeridos (revisar, merge, `git worktree remove`).
