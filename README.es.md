# Tasks-Driven-Development

*Read this in English: [README.md](README.md)*

Dos skills de agente que dividen el trabajo de features en un ciclo explícito de **analizar → planificar → implementar**, con escala de riesgo, política de selección de modelos y guardrails de orquestación. Agnósticos de proveedor: funcionan con cualquier familia de modelos.

## Los dos skills

| Skill | Modo | Salida |
|-------|------|--------|
| [`task-plan`](skills/task-plan/SKILL.md) | Investigar una tarea, asignar riesgo, producir un plan. Nunca escribe código de la feature. | Plan en `.tasks/plans/` + tarea marcada `planned` |
| [`task-impl`](skills/task-impl/SKILL.md) | Ejecutar una tarea planificada (o una consigna directa R1–R2) con guardrails de calidad y seguridad. | Cambio funcionando + tarea marcada `review`/`done` |

### Contrato de handoff

- El análisis termina cuando el plan existe en `.tasks/plans/plan_<task>.md` y el encabezado de la tarea en `.tasks/tasks.md` dice `planned` con nivel de riesgo.
- La implementación **exige ese handoff para toda tarea R3+**. Solo las consignas R1–R2 pueden saltear el análisis.
- Al terminar, la implementación actualiza el estado de la tarea y archiva el plan en `.tasks/plans/archive/`.

### Flujo git de la implementación

- **Una rama por tarea** (`task/<slug>`) creada antes de tocar código; nunca se implementa sobre la rama base.
- **Trabajo en paralelo = un worktree por worker** (`task/<slug>/w1`, `w2`, …). El orquestador verifica cada diff y recién ahí mergea a la rama de tarea y elimina el worktree.
- **Auditoría opcional**: antes de implementar se le pregunta al usuario si quiere un agente auditor sobre el código producido. Responde con el nombre del modelo, o `No`; vacío = No.
- El merge a la rama base **siempre es manual**, y al cerrar se sugiere una revisión humana o por agente.

## Registro de tareas (`.tasks/`)

```
.tasks/
  tasks.md            # índice de tareas
  plans/
    plan_<task>.md    # un plan por tarea
    archive/          # planes de tareas terminadas
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

Los roles se definen por capacidad, no por marca (`primary`, `agent`, `review`, `fast`). Una regla dura: **el análisis y la creación del plan siempre corren en el modelo más potente que ofrezca el proveedor** — Anthropic: Fable, si no Opus; OpenAI: la variante SOL más alta; cualquier otro: preguntar al usuario. Detalle en [`model-selection.md`](skills/task-plan/references/model-selection.md).

## Configuración específica del proyecto

Estos skills son agnósticos del proyecto a propósito. El stack, los comandos de build/test y las convenciones de idioma de los artefactos van en la doc del proyecto consumidor (`docs/AGENTS.md`, `CLAUDE.md` o equivalente), de donde el skill de implementación los lee.

## Instalación

Este repo es la fuente canónica. Crear symlinks de cada skill en los directorios que leen tus agentes:

```sh
ln -s "$(pwd)/skills/task-plan"       ~/.agents/skills/task-plan
ln -s "$(pwd)/skills/task-plan"       ~/.claude/skills/task-plan
ln -s "$(pwd)/skills/task-impl" ~/.agents/skills/task-impl
ln -s "$(pwd)/skills/task-impl" ~/.claude/skills/task-impl
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
```
