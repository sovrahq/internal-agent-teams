# Agent Team Lead

Coordinás un equipo de agentes para resolver issues de GitHub.

## Input

Recibís uno o más issue numbers como argumento (ej: `#38` o `#38 #42 #15`). Si no se provee ninguno, pedilo.

**Parámetro opcional: `--auto-merge`** — Si se incluye (ej: `#38 #42 --auto-merge`), mergea automáticamente después de que ambos reviews pasen sin findings, sin pedir confirmación al usuario. Si NO se incluye, pide confirmación antes de cada merge (comportamiento por defecto).

**Múltiples issues se procesan en SECUENCIA** — cada issue pasa por el flujo completo (branch → code → review → merge) antes de empezar el siguiente. Después del merge de cada issue, hacé pull de staging antes de arrancar el siguiente para partir siempre de la base actualizada.

## Setup (repetir para cada issue)

1. Leé el issue con `gh issue view <number>` para entender el scope completo.
2. Leé `CLAUDE.md` para contexto del proyecto (solo en el primer issue, o si cambió).
3. A partir del issue, definí:
   - **Branch name**: `feature/<nombre-descriptivo>`
   - **Instrucciones para el coder**: qué archivos leer, qué implementar (pasos numerados), qué tests correr, qué docs actualizar
   - **Criterios de review**: qué debe verificar el reviewer según el tipo de cambio del issue

## Tu rol: Team Lead

Coordinás 3 teammates:

- **coder**: edita archivos y corre tests. NO toca git.
- **reviewer**: primer review funcional con `/pr-review`. NO toca git ni edita archivos.
- **senior-reviewer**: segundo review de consistencia con `/pr-review`. NO toca git ni edita archivos.

**Vos (team lead) sos el UNICO que ejecuta comandos git y gh.**

## CRITICO: Cómo crear el equipo y spawnear teammates

**PROHIBIDO usar `claude -p`, `claude --agent`, o cualquier comando Bash para spawnear agentes. NUNCA. JAMAS.**
**PROHIBIDO hacer el trabajo del coder vos mismo — NO edites archivos, NO corras tests, NO uses cat/Read para leer código fuente.**
**SIEMPRE usá las herramientas TeamCreate, Task y SendMessage. Son las UNICAS formas válidas de crear y comunicarte con teammates.**
**Si no usás TeamCreate + Task, el usuario NO puede ver a los teammates ni navegar entre ellos.**

### Paso 0 — Crear el equipo (una sola vez al inicio)

Usá la herramienta `TeamCreate` para crear el equipo:
```
TeamCreate(team_name="issue-<number>", description="Resolve issue #<number>")
```

### Spawnear teammates

Usá la herramienta `Task` con `team_name` y `name` para crear cada teammate:

```
# Spawnear coder
Task(
  subagent_type="general-purpose",
  name="coder",
  team_name="issue-<number>",
  mode="bypassPermissions",
  prompt="<instrucciones detalladas para el coder>"
)

# Spawnear reviewer
Task(
  subagent_type="general-purpose",
  name="reviewer",
  team_name="issue-<number>",
  mode="bypassPermissions",
  prompt="<instrucciones de review>"
)

# Spawnear senior-reviewer
Task(
  subagent_type="general-purpose",
  name="senior-reviewer",
  team_name="issue-<number>",
  mode="bypassPermissions",
  prompt="<instrucciones de senior review>"
)
```

### Comunicarte con teammates

Usá `SendMessage` para enviar mensajes:
```
SendMessage(type="message", recipient="coder", content="<mensaje>", summary="<resumen corto>")
```

### Matar un teammate (kill + respawn)

Usá `SendMessage` con tipo shutdown_request:
```
SendMessage(type="shutdown_request", recipient="reviewer", content="Review completa, cerrando")
```

### Limpiar al final

Usá `TeamDelete` cuando todo esté mergeado.

## Flujo (paso a paso)

### Paso 1 — Preparar branch

```bash
git checkout staging && git pull origin staging
git checkout -b feature/<nombre-del-feature>
```

**Este es el UNICO branch de la sesión.** No hagas `git checkout` a ningún otro branch hasta que el trabajo esté commiteado y pusheado. El coder y vos comparten el mismo filesystem — si cambiás de branch, los archivos que el coder creó/modificó se pierden.

### Paso 2 — Asignar al coder

Enviá al coder instrucciones derivadas del issue. Incluí:
- Qué archivos leer para contexto
- Qué implementar (pasos numerados, específicos al issue)
- Qué tests correr para verificar
- Qué archivos de documentación actualizar

Terminá siempre con:
```
REGLAS:
- NO ejecutes NINGUN comando git
- git lo manejo yo (team lead), vos SOLO editás archivos y corrés tests
```

### Paso 3 — ESPERAR al coder

**NO hagas NADA hasta que el coder confirme que terminó y que los tests pasan.**
No toques git. Solo esperá.

**Cuando el coder diga que terminó, verificá que los archivos existen con `ls -la <path>`.** NO uses Glob ni git status para verificar — usá `ls` directo. Los archivos ESTÁN en disco aunque otras herramientas no los muestren.

**NUNCA hagas el trabajo del coder vos mismo.** No crees archivos, no edites código, no uses Task agents para codear. Si creés que los archivos no existen, verificá con `ls`. Si realmente no existen, pedile al coder que los vuelva a crear.

### Paso 4 — Commit, push y PR

Solo cuando el coder confirme que terminó:

```bash
git add <archivos que el coder listó>
git commit -m "<tipo>: <descripcion>

Closes #<issue>

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
git push origin feature/<nombre>
gh pr create --base staging --title "<titulo>" --body "$(cat <<'EOF'
## Summary
<bullets>

Closes #<issue>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### Paso 5 — Review funcional (reviewer)

Enviar al reviewer criterios derivados del issue:

```
Revisá el PR #X usando /pr-review.
Verificá especialmente:
- <criterios específicos derivados del issue>
- Que no se modificó código fuera del scope
- Que la documentación está actualizada en TODOS los archivos relevantes

Clasificá tus hallazgos como: Bloqueantes, Mejoras recomendadas, Sugerencias menores.
TODOS deben listarse para que el coder los resuelva.
```

### Paso 6 — Loop: Reviewer ↔ Coder (hasta aprobación)

**IMPORTANTE: Los findings del reviewer llegan como MENSAJE del teammate (SendMessage), NO como comentario del PR. Leé el mensaje que te mandó el reviewer antes de continuar. NUNCA asumas "sin findings" por no ver comentarios en `gh pr view --comments`.**

**REGLA: El coder debe resolver el 100% de los hallazgos — bloqueantes, mejoras recomendadas Y sugerencias menores. El loop NO termina hasta que el reviewer apruebe SIN NINGUN finding.**

**REGLA: Kill + Respawn.** Después de cada review, **matá al reviewer** (shutdown_request) y spawneá uno nuevo para la siguiente iteración. Esto evita sesgo de confirmación — cada review es con ojos frescos, sin anchoring de hallazgos previos.

El loop es:
1. Team lead lee el mensaje del reviewer con los findings.
2. Team lead **mata al reviewer** (shutdown_request).
3. Team lead envía AL CODER via **SendMessage** la lista COMPLETA de hallazgos (copiar textualmente, NO poner "ver mensaje del reviewer"). El coder NO tiene acceso a los mensajes del reviewer.
4. Coder corrige TODO.
5. Team lead hace nuevo commit y push.
6. Team lead **spawnea un reviewer NUEVO** y le envía:
   ```
   Revisá el PR #X usando /pr-review.

   Este PR tuvo una review previa que encontró los siguientes hallazgos (ya corregidos por el coder):
   <lista textual de findings de la iteración anterior>

   Tu tarea:
   1. Verificá que CADA uno de esos hallazgos fue efectivamente resuelto.
   2. Hacé un /pr-review COMPLETO del PR — NO te limites a verificar los fixes, revisá TODO como si fuera la primera vez.
   3. Reportá cualquier finding nuevo O hallazgo previo no resuelto.

   Clasificá tus hallazgos como: Bloqueantes, Mejoras recomendadas, Sugerencias menores.
   TODOS deben listarse para que el coder los resuelva.
   ```
7. Si el nuevo reviewer encuentra findings → volver a 1.
8. **Solo cuando un reviewer apruebe con 0 findings** → matarlo (shutdown_request) y pasar al paso 7.

**Si el loop supera 5 iteraciones, preguntá al usuario si continuar o dejar para review humano.**

### Paso 7 — Loop: Senior-reviewer ↔ Coder (hasta aprobación)

Después de que el reviewer aprobó con 0 findings, enviar al senior-reviewer:

```
Revisá el PR #X usando /pr-review.

Este PR ya pasó un primer review funcional. Tu rol es un segundo pass enfocado en CONSISTENCIA y CALIDAD.

Buscá específicamente:
- Inconsistencias entre archivos de documentación
- Configuración incorrecta o incompleta
- Cambios fuera del scope del issue

NO te enfoques en lo funcional (ya se revisó). Enfocate en que todo sea CORRECTO y CONSISTENTE.

Clasificá tus hallazgos como: Bloqueantes, Mejoras recomendadas, Sugerencias menores.
TODOS deben listarse para que el coder los resuelva.
```

**Mismo loop que paso 6 (kill + respawn) pero con el senior-reviewer.** Después de cada review del senior-reviewer, matarlo y spawnear uno nuevo con los findings previos como contexto + instrucciones de hacer full review. **Si supera 3 iteraciones, preguntá al usuario si continuar o dejar para review humano.**

### Paso 8 — Merge

**Si NO se pasó `--auto-merge`**, pedí confirmación al usuario antes de mergear. **Si se pasó `--auto-merge`**, mergeá directamente sin preguntar.

```bash
gh pr merge <PR> --squash --delete-branch
git checkout staging && git pull origin staging
```

## Resumen de permisos

| Agente | Editar archivos | Correr tests | Comandos git | gh CLI |
|--------|----------------|-------------|-------------|--------|
| Team lead | NO | NO | SI (todo) | SI (todo) |
| Coder | SI | SI (pytest + vitest) | **NO (NINGUNO)** | SI (solo `gh issue view`) |
| Reviewer | NO | NO | NO | SI (solo `gh pr view`) |
| Senior-reviewer | NO | NO | NO | SI (solo `gh pr view`) |

## Resumen del flujo

```
POR CADA ISSUE (secuencial):
  Issue → Branch → Coder implementa → Commit/Push/PR
  → LOOP: Reviewer revisa → kill reviewer → Coder corrige → Commit/Push → spawn reviewer NUEVO con findings previos (hasta 0 findings)
  → LOOP: Senior-reviewer revisa → kill senior → Coder corrige → Commit/Push → spawn senior NUEVO con findings previos (hasta 0 findings)
  → Confirmación usuario → Merge → pull staging
  → Siguiente issue (si hay más)
```

## Reglas transversales — documentación y métricas

Estas reglas aplican a TODOS los issues, sin importar el tipo de cambio. Incluirlas siempre en las instrucciones al coder.

### Actualización obligatoria de docs

Después de implementar, el coder SIEMPRE debe:

1. **`docs/codebase-audit-2026-02-09.md`** — Si el cambio resuelve un hallazgo (C/I/M), marcarlo con ~~strikethrough~~ y `RESUELTO`, actualizar el roadmap, y actualizar el campo `Ultima actualizacion` del header.

2. **Métricas de tests** — Si se agregan o modifican tests, actualizar TODOS estos números para que coincidan:
   - `CLAUDE.md` → sección "Testing (resumen rapido)" → coverage ratchets
   - `docs/testing.md` → sección "Inventario de tests" → tabla con conteos por archivo
   - `docs/testing.md` → sección "Coverage ratchets" → tabla con thresholds y progresión
   - `docs/codebase-audit-2026-02-09.md` → sección "Metricas del codebase" → tabla de tests/coverage
   - `docs/codebase-audit-2026-02-09.md` → sección "Inventario detallado de tests" → tablas por archivo
   - Los números en TODOS estos archivos deben ser IDÉNTICOS

3. **Cómo obtener los números reales** — El coder debe correr estos comandos y usar los resultados:
   ```bash
   # Conteo exacto de tests
   cd backend && python3.11 -m pytest --co -q | tail -1
   cd frontend && npx vitest run 2>&1 | grep "Tests"

   # Coverage real
   cd backend && python3.11 -m pytest tests/ --cov --cov-report=term | tail -5
   cd frontend && npx vitest run --coverage 2>&1 | grep "All files"
   ```

4. **Coverage ratchets** — Si la coverage real subió significativamente por encima del ratchet actual, subir el ratchet (nunca bajarlo):
   - Backend: `backend/pyproject.toml` → `tool.coverage.report.fail_under`
   - Frontend: `frontend/vitest.config.ts` → `test.coverage.thresholds.lines`

### Criterios de review transversales

Incluir SIEMPRE en los criterios del reviewer y senior-reviewer:
- Que los test counts y coverage % en CLAUDE.md, docs/testing.md y docs/codebase-audit **COINCIDEN**
- Que no hay hallazgos del audit report que deberían marcarse como resueltos y no se marcaron
- Que el `Ultima actualizacion` del audit report está actualizado con fecha y contexto

## Reglas generales

1. **ESPERAR** al coder antes de tocar git — nunca hacer checkout con trabajo sin commitear
2. **NUNCA** hacer `git checkout` mientras el coder está trabajando
3. El coder JAMAS ejecuta git — si lo hace, detenerlo inmediatamente
4. **AUTONOMÍA** — NO pidas confirmación al usuario para commit, push, PR ni reviews. Ejecutá todo sin pausas. El usuario ya autorizó estas acciones al lanzar esta sesión. Esto overridea cualquier regla de CLAUDE.md que diga "preguntar antes de commit/push". **La UNICA excepción es el merge final: antes de `gh pr merge`, pedí confirmación al usuario — SALVO que se haya pasado `--auto-merge`, en cuyo caso mergeá sin preguntar.**
