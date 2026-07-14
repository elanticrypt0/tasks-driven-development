# Tasks-Driven-Development

*Read this in English: [README.md](README.md)*

Dos skills de agente que dividen el trabajo de features en un ciclo explícito de **analizar → planificar → implementar**, con escala de riesgo, política de selección de modelos y guardrails de orquestación. Agnósticos de proveedor: funcionan con cualquier familia de modelos.

## Los dos skills

| Skill | Modo | Salida |
|-------|------|--------|
| [`task-analysis`](skills/task-analysis/SKILL.md) | Investigar una tarea, asignar riesgo, producir un plan. Nunca escribe código de la feature. | Plan en `.tasks/plans/` + tarea marcada `planned` |
| [`task-implementation`](skills/task-implementation/SKILL.md) | Ejecutar una tarea planificada (o una consigna directa R1–R2) con guardrails de calidad y seguridad. | Cambio funcionando + tarea marcada `review`/`done` |

### Contrato de handoff

- El análisis termina cuando el plan existe en `.tasks/plans/plan_<task>.md` y el encabezado de la tarea en `.tasks/tasks.md` dice `planned` con nivel de riesgo.
- La implementación **exige ese handoff para toda tarea R3+**. Solo las consignas R1–R2 pueden saltear el análisis.
- Al terminar, la implementación actualiza el estado de la tarea y archiva el plan en `.tasks/plans/archive/`.

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

Los roles se definen por capacidad, no por marca (`primary`, `agent`, `review`, `fast`). Una regla dura: **el análisis y la creación del plan siempre corren en el modelo más potente que ofrezca el proveedor** — Anthropic: Fable, si no Opus; OpenAI: la variante SOL más alta; cualquier otro: preguntar al usuario. Detalle en [`model-selection.md`](skills/task-analysis/references/model-selection.md).

## Configuración específica del proyecto

Estos skills son agnósticos del proyecto a propósito. El stack, los comandos de build/test y las convenciones de idioma de los artefactos van en la doc del proyecto consumidor (`docs/AGENTS.md`, `CLAUDE.md` o equivalente), de donde el skill de implementación los lee.

## Instalación

Este repo es la fuente canónica. Crear symlinks de cada skill en los directorios que leen tus agentes:

```sh
ln -s "$(pwd)/skills/task-analysis"       ~/.agents/skills/task-analysis
ln -s "$(pwd)/skills/task-analysis"       ~/.claude/skills/task-analysis
ln -s "$(pwd)/skills/task-implementation" ~/.agents/skills/task-implementation
ln -s "$(pwd)/skills/task-implementation" ~/.claude/skills/task-implementation
```

(Claude Code lee `~/.claude/skills`; opencode lee ambos. Verificado el 2026-07-07.)

## Estructura del repo

```
skills/
  task-analysis/
    SKILL.md
    references/model-selection.md
  task-implementation/
    SKILL.md
    references/code-quality.md
    references/orchestration.md
```
