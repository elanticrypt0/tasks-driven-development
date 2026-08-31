# Tasks-Driven-Development

*Read this in English: [README.md](README.md)*

Tres skills de agente que dividen el trabajo de features en un ciclo explícito de **analizar → planificar → implementar → auditar**, con escala de riesgo, política de selección de modelos y guardrails de orquestación. Agnósticos de proveedor: funcionan con cualquier familia de modelos.

## Los tres skills

| Skill | Modo | Salida |
|-------|------|--------|
| [`task-plan`](skills/task-plan/SKILL.md) | Investigar una tarea, asignar riesgo, producir un plan. Nunca escribe código de la feature. | Plan en `.tasks/plans/` + tarea marcada `planned` |
| [`task-impl`](skills/task-impl/SKILL.md) | Ejecutar una tarea planificada (o una consigna directa R1–R2) con guardrails de calidad y seguridad. | Cambio funcionando + tarea marcada `review`/`done` |
| [`task-audit`](skills/task-audit/SKILL.md) | Auditar código existente — correctitud, tests, seguridad, sanitización, performance. Nunca arregla código de producción. | Reporte en `.tasks/audits/` + fix tasks planificadas |

La auditoría cierra el ciclo: sus fix tasks ya quedan `planned`, así que `task-impl` las consume sin volver a analizar.

### Contrato de handoff

- El análisis termina cuando el plan existe en `.tasks/plans/plan_<task>.md` y el encabezado de la tarea en `.tasks/tasks.md` dice `planned` con nivel de riesgo.
- La implementación **exige ese handoff para toda tarea R3+**. Solo las consignas R1–R2 pueden saltear el análisis.
- Al terminar, la implementación actualiza el estado de la tarea y archiva el plan en `.tasks/plans/archive/`.
- La auditoría termina cuando existe el reporte, cada hallazgo quedó CONFIRMED o PLAUSIBLE, y cada hallazgo aceptado tiene su fix task `planned` (o figura explícitamente como aceptado-sin-tarea).

### Flujo git de la implementación

- **Una rama por tarea** (`task/<slug>`) creada antes de tocar código; nunca se implementa sobre la rama base.
- **El trabajo en paralelo comparte esa misma rama**: sin worktrees ni ramas por worker. El aislamiento es por propiedad de archivos (un archivo, un worker), los workers solo escriben archivos y el git lo maneja el orquestador: verifica cada diff acotado y recién ahí commitea.
- **Auditoría opcional**: antes de implementar se le pregunta al usuario si quiere un agente auditor sobre el código producido. Responde con el nombre del modelo, o `No`; vacío = No. `task-impl` solo pregunta — la revisión la corre `task-audit` sobre el diff de la tarea.
- El merge a la rama base **siempre es manual**, y al cerrar se sugiere una revisión humana o por agente.

## Auditoría

`task-audit` corre por su cuenta sobre cualquier alcance (`diff`, `module`, `repo`, o un solo eje), o lo invoca `task-impl` como revisión post-implementación.

- **Evidencia o nada**: todo hallazgo cita `path:line` más un escenario de falla concreto. Primero se juntan candidatos, después se verifican; lo que no sobrevive se descarta y solo se cuenta.
- **Dos escalas**: `severity` (critical/high/medium/low) para el hallazgo, `R1–R5` para el fix. Son independientes.
- **Read-only sobre código de producción.** La única excepción son los tests: puede escribir tests de caracterización/regresión en una rama `audit/<slug>` para probar hallazgos. Un test que reproduce un bug se deja fallando a propósito.
- Los hallazgos aceptados se vuelven fix tasks `planned` — `critical`/`high` tienen tarea propia, `medium`/`low` pueden agruparse dentro de un mismo módulo.

## Registro de tareas (`.tasks/`)

```
.tasks/
  tasks.md            # índice de tareas
  plans/
    plan_<task>.md    # un plan por tarea
    archive/          # planes de tareas terminadas
  audits/
    audit_<scope>_<fecha>.md
```

Formato del encabezado de tarea en `tasks.md`:

```
** <título> — <estado> · R<n> · plan: .tasks/plans/plan_<task>.md **
```

Estados: `idea` → `planned` → `review` → `done`.

## Escala de riesgo

| Nivel | Significado |
|-------|-------------|
| R1 | Cosmético / UI local. Reversible. |
| R2 | Lógica local. Reversible. Sin migraciones ni contratos externos. |
| R3 | Varios módulos, contratos de API o estado compartido. |
| R4 | Migraciones de DB, auth, concurrencia, datos difíciles de revertir. |
| R5 | Seguridad, escritura/borrado masivo, cambios irreversibles o de infraestructura. |

R4–R5 siempre requieren confirmación del usuario antes de ejecutar y nunca se delegan a workers.

## Política de modelos

Los roles se definen por capacidad, no por marca (`primary`, `agent`, `review`, `fast`). Una regla dura: **el análisis y la creación del plan siempre corren en el modelo más potente que ofrezca el proveedor** — Anthropic: Fable, si no Opus; OpenAI: la variante SOL más alta; cualquier otro: preguntar al usuario. Auditar es análisis, así que sigue la misma regla — salvo que el usuario haya nombrado un modelo para la auditoría, en cuyo caso se usa ese exactamente. Detalle en [`model-selection.md`](skills/task-plan/references/model-selection.md).

## Agentes de opencode y `/run-fase` (automatización)

Este repo incluye dos subagents de opencode y un comando orquestador que encadenan las skills en un flujo automático:

```
.opencode/
  agent/
    coder.md      # ejecuta task-impl  · modelo default: opencode-go/deepseek-v4-flash
    auditor.md    # ejecuta task-audit · modelo default: opencode-go/glm-5.3-flash
  command/
    run-fase.md   # orquestador /run-fase
```

- `/run-fase T4` o `/run-fase fase-2` — resuelve la tarea o fase desde `.tasks/tasks.md`, implementa cada tarea `planned` con el **coder**, audita el diff con el **auditor**, aplica los hallazgos in-scope y las fix tasks out-of-scope R1–R3, re-audita (máximo 2 rondas por tarea, luego escala al usuario) y cierra con una tabla de resumen.
- Los fixes out-of-scope **R4–R5 siempre pausan y consultan** antes de ejecutar. El merge sigue siendo manual.
- El coder maneja git en su rama `task/<slug>`; el auditor es estrictamente read-only sobre código de producción (en el flujo automático no escribe tests ni ramas `audit/` — los tests de prueba son trabajo del coder en los fixes).
- **Overrides de modelo en la invocación**: `/run-fase T4 audit: opus`, `/run-fase fase-2 coder: opencode-go/minimax-m3 audit: glm-5.3`. Un rol con modelo nombrado corre por backend B (`opencode run -m MODELO`): el coder con `--auto` (necesita escribir), el auditor sin `--auto` (read-only — devuelve el reporte y el orquestador lo persiste). Los roles sin override usan los subagents nativos con sus modelos default. Un modelo no disponible se informa, nunca se sustituye en silencio.

## Configuración específica del proyecto

Estos skills son agnósticos del proyecto a propósito. El stack, los comandos de build/test y las convenciones de idioma de los artefactos van en la doc del proyecto consumidor (`docs/AGENTS.md`, `CLAUDE.md` o equivalente), de donde el skill de implementación los lee.

## Instalación

Este repo es la fuente canónica. Crear symlinks de cada skill en los directorios que leen tus agentes:

```sh
ln -s "$(pwd)/skills/task-plan"  ~/.agents/skills/task-plan
ln -s "$(pwd)/skills/task-plan"  ~/.claude/skills/task-plan
ln -s "$(pwd)/skills/task-impl"  ~/.agents/skills/task-impl
ln -s "$(pwd)/skills/task-impl"  ~/.claude/skills/task-impl
ln -s "$(pwd)/skills/task-audit" ~/.agents/skills/task-audit
ln -s "$(pwd)/skills/task-audit" ~/.claude/skills/task-audit
```

(Claude Code lee `~/.claude/skills`; opencode lee ambos. Verificado el 2026-07-07.)

## Estructura del repo

```
skills/
  task-plan/
    SKILL.md
    references/model-selection.md
  task-impl/
    SKILL.md
    references/code-quality.md
    references/orchestration.md
  task-audit/
    SKILL.md
    references/audit-checklist.md
.opencode/
  agent/
    coder.md
    auditor.md
  command/
    run-fase.md
  commands/
    worktrees.md
```
