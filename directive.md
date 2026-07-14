# Directivas operativas

Reglas para cualquier modelo o agente que trabaje en este repo. Agnósticas de proveedor: aplican igual sin importar la familia de modelos.

## Principios
- Objetivo antes que proceso: resolver la consigna con el mínimo de iteraciones.
- Respuestas cortas y directas. Sin saludos ni relleno. Fragmentos OK.
- Inspeccionar el código real antes de afirmar o proponer. No asumir nombres ni rutas; no inventar datos. Si falta un dato, verificarlo en el código o preguntarlo.
- Cambios reversibles y de bajo alcance. Un cambio = un objetivo.
- Sin valores hardcodeados: configuración por variables de entorno o constantes nombradas.
- Documentar el flujo y las decisiones tomadas por el LLM para que un developer humano pueda comprenderlas y auditarlas.

## Eficiencia (menos iteraciones / tokens)
- Batchear: agrupar todas las preguntas en un solo mensaje; no preguntar de a una.
- No re-leer archivos ya leídos en la sesión. Leer solo el fragmento necesario, no el archivo entero.
- No repetir en la respuesta el contenido de archivos ni volcar outputs largos: citar `ruta:línea` y resumir.
- Llamadas a herramientas independientes → en paralelo, no en serie.
- Entregar el artefacto concreto cuando es reversible y barato, en vez de pedir confirmación para algo trivial.

## Idioma de los artefactos
- Prosa, docs, reportes y Markdown en español: ortografía natural (acentos, ñ, ¿, ¡), Unicode/UTF-8 normal.
- Código, identificadores y comandos: en inglés, siguiendo el estilo del archivo vecino.

## Roles de modelo (por capacidad, no por marca)
```yaml
models:
  primary: best_available               # decisiones, plan y código principal
  agent:   best_available_for_coding    # subtareas / ejecución en paralelo
  review:  best_available_for_reasoning # revisión crítica y seguridad
  fast:    fastest_available            # tareas triviales o de alto volumen
```
Usar el rol según la herramienta disponible. Cómo se resuelve cada rol depende del backend de orquestación (ver [Orquestación](#orquestación-de-la-implementación)). Si no hay subagentes ni CLI externa, ejecutar directo con el rol primario. Mapeo de marcas: Apéndice (orientativo, puede envejecer).

## Dos modos de trabajo (no mezclar)
El usuario elige el modo; ante una consigna concreta de implementación, por defecto → **Implementar**.
- **Analizar**: investigar y producir un plan. NO escribir código de la feature.
- **Implementar**: ejecutar una tarea ya analizada o una consigna directa puntual. No exige haber analizado toda la lista: se puede implementar una sola tarea.

### Alta de una tarea nueva
1. Pensar la consigna; detectar ambigüedades y sugerir mejoras que el usuario quizá no consideró.
2. Si hacen falta preguntas, batchearlas en un solo mensaje. Si son más de 5, crear un markdown con cada pregunta seguida de `Rta:` para que el usuario complete.
3. Registrar la tarea en `tasks.md` como `idea`.

## Escala de riesgo (obligatoria al analizar)
- **R1** — Cosmético / UI local. Reversible. Sin datos ni seguridad.
- **R2** — Lógica local. Reversible. Sin migraciones ni contratos externos.
- **R3** — Toca varios módulos, contratos de API o estado compartido.
- **R4** — Migraciones de DB, auth/permisos, concurrencia o datos difíciles de revertir.
- **R5** — Seguridad, escritura/borrado masivo, cambios irreversibles o de infraestructura.

Comportamiento según riesgo:
- R1–R2: avanzar sin bloquear si la consigna es clara.
- R3: avanzar; confirmar solo si hay ambigüedad real.
- R4–R5: confirmar el plan con el usuario ANTES de ejecutar. Nunca delegar a un worker (ver Guardrails de orquestación).

## Tareas
Ubicadas en `tasks.md`. Encabezado de tarea:
```
** <título> — <estado> · R<n> · plan: _plans/plan_<tarea>.md **
```
- `<estado>`: `idea` (sin analizar) | `planificado` (analizado) | `revisión` (requiere revisar) | `ok` (hecho).
- Una tarea `planificado`, `revisión` u `ok` DEBE tener `R<n>` y `plan:`. Si falta cualquiera, tratarla como `idea`.

## Pre-análisis (modo Analizar)
1. Leer la consigna e inspeccionar el código real involucrado.
2. Detectar el estado actual de la tarea.
3. Asignar riesgo R1–R5.
4. Señalar mejoras o problemas concretos. Si se pide crítica, ser firme y claro sin faltar el respeto.
5. Sugerir alternativas o ideas que el usuario quizá no consideró.
6. Preguntar solo si falta información crítica e irreemplazable (todo en un mensaje).
7. Evaluar si la tarea es orquestable: identificar pasos independientes ejecutables en paralelo por workers (ver Orquestación) y marcarlos en el plan.

## Plan (`_plans/`)
Crear `_plans/plan_<tarea>.md` (crear el directorio si no existe). Cada paso incluye:
- archivos afectados (rutas reales);
- acción concreta;
- criterio de éxito verificable;
- riesgo del paso y cómo mitigarlo o revertir;
- `paralelizable: sí|no` — sí solo si el paso no comparte archivos ni depende del resultado de otro paso pendiente.

Registrar la ruta del plan en el encabezado de la tarea.

## Learn (aprendizaje)
Convertir conocimiento adquirido en artefactos reutilizables para no re-derivar trabajo. Un aprendizaje no es un parche de un solo uso: debe ahorrar trabajo en tareas futuras.

### Disparadores → destino
| Disparador | Artefacto |
|------------|-----------|
| **Integración**: el usuario pide usar una API/tecnología/servicio externo o comparte una URL de docs oficial | Skill nuevo (salvo que diga lo contrario) |
| **Procedimiento repetido**: una secuencia de pasos ya se ejecutó ≥2 veces (deploy, migración, seeding, reporte) | Skill nuevo que la capture |
| **Lección/corrección**: el usuario corrige un comportamiento, o se descubre algo no obvio del entorno (flags, workarounds, límites) | Actualizar el skill afectado; si es del repo y no de una integración → `docs/AGENTS.md` |

### Ubicación y formato de skills
- Canónico: `~/.agents/skills/<nombre-kebab>/SKILL.md`. Es la única copia real.
- Crear symlink en `~/.claude/skills/<nombre>` apuntando al canónico (Claude Code hoy solo lee `~/.claude/skills`; opencode lee ambos). Verificado el 2026-07-07.
- Formato `SKILL.md`: frontmatter con `name` (kebab-case) y `description` (qué hace + frases que lo disparan); cuerpo con lo mínimo para operar.
- **Antes de crear, buscar si ya existe un skill que cubra el caso**: actualizarlo en lugar de duplicar. Nunca dejar dos copias reales del mismo skill.

### Flujo (integraciones)
1. Leer la documentación oficial provista (la URL es la fuente de verdad; no inventar endpoints, parámetros ni nombres de campos).
2. Crear el skill con lo mínimo necesario: invocación, auth, llamadas clave y manejo de errores.
3. Generar **documentación breve y concreta** para el developer junto al skill:
   - qué hace la integración y cuándo usarla;
   - **setup manual** requerido (obtener API-key, crear app/proyecto, scopes/permisos, panel donde se gestiona) con pasos numerados;
   - variables de entorno o secretos que faltan y dónde van (nunca commitear el valor real);
   - link a la documentación oficial usada, con fecha: `verificado el <fecha>`.
4. Si el setup exige acción humana (p. ej. registrarse para una API-key), dejarlo explícito y marcado como **bloqueante** antes de poder usar el skill.
5. **Probar el skill** (smoke test real contra la API/procedimiento) antes de declararlo hecho; si el setup está bloqueado, decirlo y marcar el skill como no verificado.

### Mantenimiento
- Si un skill falla porque la API cambió: actualizarlo contra la doc oficial y renovar la fecha de verificación.
- Lección nueva sobre una integración → va al skill de esa integración, no a un archivo suelto.

### Relación con la orquestación
Los workers de opencode ven los skills del proyecto y los globales (verificado el 2026-07-07): cada skill aprendido amplía lo que se puede delegar. Si un paso delegado depende de una integración, indicar en el contrato de subtarea qué skill usar.

Aplica la escala de riesgo: una integración que maneja credenciales o escribe en un servicio externo suele ser R3+ y debe listar su impacto de seguridad.

## Seguridad (revisar siempre, no negociable)
- Nunca commitear secretos; usar variables de entorno. No loguear datos sensibles ni PII.
- Validar y sanear toda entrada externa; consultas parametrizadas (evitar SQL/command injection).
- Verificar autorización/permisos en cada endpoint nuevo; respetar la protección de `root`.
- Mínimo privilegio: no ampliar superficie de ataque ni permisos más de lo necesario.
- En R4–R5, listar el impacto de seguridad explícitamente antes de ejecutar.

## Ejecución (modo Implementar)
1. Cargar skills/herramientas relevantes antes de empezar.
2. Con herramienta de tareas: una tarea por paso; `WIP` al iniciar, `DONE` al terminar.
3. Si el plan tiene ≥2 pasos `paralelizable: sí` → aplicar Orquestación. Si no, ejecutar directo con el rol primario.
4. Hacer el cambio mínimo que cumple el objetivo; seguir el estilo del código vecino.
5. Si algo falla, reportar el error exacto y la solución propuesta. No ocultar fallos.

## Orquestación de la implementación

El orquestador (rol `primary`) reparte pasos del plan entre **workers** y verifica cada resultado. Un worker puede ser un **subagente nativo** (backend A) o una **corrida de `opencode`** con otro modelo (backend B).

### Cuándo orquestar
Orquestar solo si se cumplen TODAS:
- El plan tiene ≥2 pasos marcados `paralelizable: sí`.
- Cada paso delegable es R1–R3 (R4–R5 jamás se delegan).
- Los pasos no comparten archivos: un archivo pertenece a un solo worker a la vez.

Si no se cumplen, ejecutar en serie con el rol primario. Orquestar una tarea trivial cuesta más de lo que ahorra.

### Contrato de subtarea (aplica a ambos backends)
Cada delegación incluye, en el prompt del worker:
- objetivo puntual y archivos exactos a tocar (rutas reales);
- contexto mínimo necesario (no volcar el repo entero);
- criterio de éxito verificable del paso;
- restricciones: no borrar contenido, no tocar archivos fuera de la lista, no introducir secretos ni valores hardcodeados;
- formato de salida esperado (diff aplicado + resumen corto).

### Backend A — subagentes nativos
Usar cuando el runtime ofrece subagentes (p. ej. Task/Agent en Claude Code).
- Lanzar los workers independientes en paralelo con rol `agent`; búsqueda/exploración con rol `fast`.
- El orquestador no implementa mientras hay workers activos: coordina, verifica e integra.
- Revisión final del conjunto con rol `review` si la tarea es R3+.

### Backend B — `opencode` con otros modelos
Usar cuando no hay subagentes, o cuando conviene derivar volumen a modelos externos más baratos.

Invocación base (probada el 2026-07-06 con workers en paralelo sobre ambos proveedores):
```sh
timeout 240 opencode run "<prompt del contrato de subtarea>" -m <modelo> --auto
```
- `--auto` es necesario en corridas no interactivas para que el worker pueda escribir archivos; es seguro solo porque el contrato restringe los archivos a tocar.
- **Nunca usar placeholders con ángulos (`<value>`, `<path>`) en el prompt**: `opencode run` se cuelga sin emitir output (verificado el 2026-07-07 en v1.17.14, A/B con el mismo prompt). Usar marcadores en MAYÚSCULAS ("reply: TEMP Rosario NUMBER").
- Envolver siempre en `timeout`: una corrida colgada no emite nada y bloquearía la orquestación. Si expira, revisar primero el prompt (¿ángulos?) y recién después aplicar el fallback de modelos.
- `--format json` solo si se va a parsear la salida programáticamente; para verificar, mirar el diff real en disco, no el output.

Flags útiles:

| Flag | Atajo | Descripción |
|------|-------|-------------|
| `--model` | `-m` | Modelo a usar (formato `proveedor/modelo`) |
| `--agent` | — | Agente a disparar (p. ej. `plan`, `build`) |
| `--format` | — | Salida `default` (texto) o `json` |
| `--continue` | `-c` | Continúa la última sesión activa |
| `--file` | `-f` | Adjunta archivos locales al contexto |
| `--thinking` | — | Muestra los bloques de razonamiento |
| `--auto` | — | Auto-aprueba operaciones no denegadas en config |

Proveedores autenticados (verificado el 2026-07-06 con `opencode models` y `opencode auth list`):
- `opencode-go/` — modelos de cuenta OpenCode Go.
- `opencode/` — gateway **OpenCode Zen**, modelos free (sin costo); funcionan con la misma credencial.

Modelos disponibles, en orden de prioridad (1 = usar primero):

| Prioridad | Modelo | ID para `-m` | Uso sugerido |
|-----------|--------|--------------|--------------|
| 1 | MiniMax M3 | `opencode-go/minimax-m3` | subtareas de código (rol `agent`) |
| 2 | DeepSeek V4 Flash | `opencode-go/deepseek-v4-flash` | tareas triviales / alto volumen (rol `fast`) |
| 3 | Kimi K2.7-code | `opencode-go/kimi-k2.7-code` | código pesado alternativo |
| 4 | DeepSeek V4 Flash free | `opencode/deepseek-v4-flash-free` | Zen: rol `fast` sin costo |
| 5 | Nemotron 3 Ultra free | `opencode/nemotron-3-ultra-free` | Zen: respaldo |
| 6 | GLM 5.2 | `opencode-go/glm-5.2` | último recurso |

Otros modelos Zen disponibles como reemplazo: `opencode/big-pickle`, `opencode/mimo-v2.5-free`, `opencode/north-mini-code-free`.

Fallback: si una corrida falla o devuelve basura, reintentar UNA vez con el siguiente modelo en prioridad; si vuelve a fallar, el orquestador hace el paso directamente y lo reporta.

Nota: no usar el nombre de la company como proveedor: `minimax/minimax-m3` da "Unexpected server error".

### Verificación e integración (obligatoria por worker)
1. Al terminar cada worker, el orquestador revisa el diff real contra el criterio de éxito del paso (no confiar en el auto-reporte del worker).
2. Corregir o rehacer lo que no cumpla; mejoras menores las aplica el orquestador directo.
3. Integrados todos los pasos, correr la Definición de hecho sobre el conjunto.

### Guardrails de orquestación
- **PROHIBIDO ejecutar comandos que eliminen contenido a través de workers** (`rm`, `DROP`, `DELETE` masivo, `--force`). Si un paso lo requiere, lo ejecuta el orquestador con confirmación del usuario.
- R4–R5 nunca se delegan: los ejecuta el orquestador previa confirmación.
- No pasar secretos ni credenciales en prompts de workers.
- Máximo 3 workers en paralelo; un archivo pertenece a un solo worker a la vez.
- Todo resultado de worker se verifica antes de integrarse (ver arriba).

## Definición de hecho (para bajar iteraciones)
Antes de declarar terminado:
- Compila / linta / pasa tests según el stack (comandos en `docs/AGENTS.md`).
- Cumple el criterio de éxito del plan.
- Sin secretos, sin código muerto ni TODO crítico introducidos por el cambio.

Reportar el resultado real de cada verificación (qué se corrió y qué dio). Si se saltó un paso, decirlo.

## Cierre (máx. 2 líneas)
```md
Cambió: <qué cambió y dónde>.
Sigue: <próximo paso o "nada">.
```

---

## Documentación importante
- `docs/AGENTS.md`: documentación resumida para agentes.

## Ubiación de skills
- `.agents/skills`: path de skills

## Stack
| Área | Herramienta |
|------|-------------|
| Backend | Python 3, uv (package manager), FastAPI, SQLite, LangChain |
| LLM | Ollama, modelo Gemma 4b |
| Frontend | HTML, CSS, Tailwind, HTMX |

---

## Apéndice — mapeo de marcas (orientativo, puede envejecer)
El YAML por capacidad es la fuente de verdad. Si el runtime no resuelve roles solo, usar como referencia:
- Anthropic: primary=Opus/Fable · agent=Sonnet · fast=Haiku
- OpenAI: primary=GPT alto razonamiento · agent=GPT medio
- opencode: ver tabla de prioridades en [Backend B](#backend-b--opencode-con-otros-modelos)
