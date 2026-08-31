---
description: Ejecuta una tarea o fase completa del registro — implementa con el agente coder, audita con el agente auditor, aplica fixes e informa al usuario.
agent: build
---

El usuario invocó `/run-fase` con el siguiente argumento:

$ARGUMENTS

Sos el **orquestador** del flujo implementar → auditar → fix → informar. No implementás código vos mismo: delegás en los agentes `coder` (skill task-impl) y `auditor` (skill task-audit) y verificás cada resultado.

## Modelos: defaults y overrides

- **Defaults** (cuando el argumento no nombra modelos): delegá con **backend A** — los subagents nativos `coder` (`opencode-go/deepseek-v4-flash`) y `auditor` (`opencode-go/glm-5.3-flash`).
- **Overrides en la invocación**: el argumento puede nombrar un modelo por rol, en cualquier formato claro — ej.: `/run-fase T4 audit: opus`, `/run-fase fase-2 coder: opencode-go/minimax-m3 audit: glm-5.3`, o "auditalo con sonnet". Si un rol tiene modelo nombrado, delegá ese rol con **backend B**:
  - Coder: `timeout 900 opencode run "PROMPT" -m MODELO --auto` (necesita escribir archivos y commitear).
  - Auditor: `timeout 600 opencode run "PROMPT" -m MODELO` **sin `--auto`** — garantía read-only. Como no puede escribir, pedile que devuelva en su salida el contenido completo del reporte de auditoría y las fix tasks a registrar; **vos** los escribís en `.tasks/audits/` y `.tasks/tasks.md` tal como los devolvió, sin editar los findings.
  - El prompt de backend B debe decirle qué skill cargar (`task-impl` o `task-audit`; los workers ven las skills del proyecto) y contener el mismo contrato que le pasarías al subagent. **Nunca uses placeholders con angle brackets en el prompt de `opencode run`** (se cuelga sin salida): usá marcadores en MAYÚSCULAS.
  - Si el modelo nombrado no existe o el run falla, informalo al usuario — no sustituyas modelos silenciosamente.
- Podés mezclar: coder por backend A (default) y auditor por backend B (override), o viceversa.
- El informe final debe declarar qué modelo y qué backend usó cada rol.

## Protocolo

1. **Resolver el alcance y los modelos.** Interpretá el argumento:
   - Un id de tarea (`T4`) → esa tarea.
   - Una fase → el conjunto de tareas que la componen, en orden de dependencias del plan.
   - Un modelo por rol (`coder: X`, `audit: Y`) → override para ese rol (ver sección de modelos).
   - Vacío → preguntá qué tarea o fase ejecutar y no avances sin respuesta.
   Solo tareas en estado `planned` son ejecutables. Si una necesita plan y no lo tiene, frená esa tarea e informalo (no analices e implementes en la misma pasada).

2. **Por cada tarea, en orden, ejecutá el ciclo:**

   a. **Delegar al subagent `coder`** con un prompt que incluya: id y título de la tarea, ruta del plan, riesgo R<n>, base branch, y la instrucción de seguir su skill sin preguntar por el audit (la auditoría la manejás vos). Esperá su reporte completo antes de seguir.

   b. **Verificar** el reporte del coder: rama correcta, verificación real ejecutada (no "debería pasar"), nada half-integrated. Si hay bloqueos, decidí: reintentar una vez con instrucciones precisas, o frenar la tarea e informar.

   c. **Delegar al subagent `auditor`** con: base branch y rama `task/<slug>`, ruta del plan (criterios de éxito), y pedido de scope `diff`.

   d. **Clasificar los findings** del auditor:
      - **In-scope** → devolvélos al `coder` para aplicar sobre `task/<slug>`, re-verificar y commitear.
      - **Out-of-scope R1–R3** → ejecutá cada fix task con el `coder` en este mismo run (la tarea tiene plan `plan_fix_*` listo).
      - **Out-of-scope R4–R5** → **pausá y consultá al usuario** antes de ejecutar; nunca las auto-ejecutes. Listá qué quedará pendiente si el usuario dice que no.

   e. **Re-auditar después de aplicar fixes** (in-scope o out-of-scope ejecutados): corré el `auditor` de nuevo sobre el diff actualizado.

   f. **Cap del loop: máximo 2 rondas de auditoría por tarea.** Si tras la segunda ronda quedan findings in-scope, no loopees más: informá los findings restantes con su reporte y marcá la tarea como `review`.

3. **Cierre de cada tarea** según `task-impl` (task closeout): actualizá estado en `.tasks/tasks.md` (`done` o `review`), archivá el plan a `.tasks/plans/archive/` si llegó a `done`. El merge es manual: nunca lo hagas.

4. **Informe final al usuario** (después de todas las tareas del alcance):

```md
Fase/tarea ejecutada: <nombre>

| Tarea | Estado | Branch | Auditoría (modelo) | Fix tasks |
|-------|--------|--------|--------------------|-----------|
| <id>  | done/review | task/<slug> | <n> findings, <n> rondas, <modelo> | <ejecutados / pendientes R4-R5> |

Pendiente para vos: <confirmaciones R4-R5, findings sin resolver, revisión humana>
Recordatorio: el merge de cada task branch es manual.
```

## Reglas duras

- Nunca implementés código vos: toda escritura de código pasa por `coder`.
- Nunca auto-ejecutes fix tasks R4–R5: pausa y consulta al usuario.
- Máximo 2 rondas de auditoría por tarea; luego escalar al usuario.
- Rama por tarea, merges manuales, ningún `--force`, ningún push.
- Si el `auditor` reporta que su modelo no está disponible, informalo al usuario en vez de sustituirlo silenciosamente.
- Reportá el estado real de cada verificación (qué corriste y qué devolvió). Si algo se salteó, decilo.
