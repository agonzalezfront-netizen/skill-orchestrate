# Session Orchestrator

Skill genérico para coordinar múltiples sesiones de Claude Code en cualquier proyecto. Funciona como un hub que genera instrucciones copy-paste para sesiones spoke (QA, Journey, Feature, Dev, etc.).

**Comunicación entre sesiones**: el usuario copia el bloque inicial del Control al spoke (1 vez). Para el retorno, el spoke escribe su reporte en un memory file + commitea su trabajo a su branch. El Control lee ambos via memoria compartida + git. **Cero copy-paste de vuelta.**

## Cómo funciona

Este skill tiene 2 modos. Detectá cuál aplicar según lo que dice el usuario:

### Modo 1: Primera vez en el proyecto (no existe `.claude/orchestrate.yml`)

Si no encontrás `.claude/orchestrate.yml` en el proyecto actual, guiá al usuario para crear la config con preguntas amigables. Usá `AskUserQuestion` para cada grupo de preguntas.

**Pregunta 1 — El proyecto**
```
¿Cómo se llama tu proyecto y en qué consiste?

Ejemplo: "StudentHub — una app web para seguimiento de estudiantes"
Ejemplo: "MiCRM — un sistema de ventas para mi empresa"
Ejemplo: "LandingPage — la página web de mi negocio"

Solo necesito el nombre y una oración corta describiendo qué hace.
```

**Pregunta 2 — Tipos de sesiones**
```
¿Qué tipos de sesiones de trabajo vas a necesitar?

Pensalo así — cada tipo es un "modo de trabajo" diferente:

• QA (Testing): "Quiero probar que todo funcione bien, buscar errores
  y asegurarme que no se rompe nada"

• Prospect Journey (Validación): "Quiero simular que un cliente real
  prueba mi producto y ver si le convence para comprar"

• Feature (Funcionalidad nueva): "Quiero agregar algo que no existe
  todavía — una pantalla nueva, un botón, un reporte"

• Dev (Arreglos y mejoras): "Quiero arreglar bugs que ya conozco,
  mejorar algo que ya existe, o hacer cambios chicos"

Elegí los que apliquen. Si no estás seguro, elegí QA + Feature + Dev
que son los más comunes para cualquier proyecto.
```

**Pregunta 3 — Deploy**
```
¿Cómo se publica tu proyecto para que la gente lo vea?

Elegí la que más se parezca a tu caso:

• "Tengo un script que lo sube" → Pasame el comando
  (ejemplo: python scripts/deploy.py)

• "Uso Vercel / Netlify / Railway" → Decime cuál y el comando
  (ejemplo: vercel deploy --prod)

• "Lo subo por FTP / cPanel a mano" → Podemos automatizarlo

• "Corro un comando de build y listo" → Pasame el comando
  (ejemplo: npm run build)

• "No sé / todavía no tengo deploy" → No pasa nada, lo
  configuramos después

Si tu proyecto tiene frontend Y backend separados, decime
los comandos de cada uno.
```

**Pregunta 4 — Estructura**
```
¿Dónde viven los archivos importantes de tu proyecto?

No te preocupes si no sabés todos — puedo detectar la mayoría solo.
Solo decime lo que sepas:

• ¿Dónde están los tests? (ejemplo: tests/, src/__tests__/)
• ¿Dónde están los planes o specs? (ejemplo: docs/plans/)
• ¿Tenés documentación de QA o testing? (ejemplo: docs/qa/)
• ¿Tenés personas/perfiles de usuarios? (ejemplo: docs/personas/)

Si no tenés nada de esto todavía, decí "no tengo" y lo vamos
creando a medida que lo necesitemos.
```

**Pregunta 5 — Rama principal**
```
¿Cuál es la rama principal de tu proyecto?

Es la rama donde vive el código "oficial" — la versión que se publica
o la que todos usan como punto de partida para trabajar.

Ejemplos comunes:
• "main" — la más común en proyectos nuevos
• "master" — proyectos más antiguos
• "develop" — si usás un flujo con rama de desarrollo separada
• "demo" — si tenés una rama especial para demos

¿Por qué importa? Cuando una sesión de trabajo (QA, Feature, etc.)
termina, su trabajo se integra a esta rama. También es desde donde
se crean las nuevas ramas de trabajo.

Si no estás seguro, probablemente es "main". Podés verificar con:
  git branch
La rama con el asterisco (*) es en la que estás ahora.
```

**Pregunta 6 — Verificación automática** (opcional, avanzada)
```
¿Hay algo que siempre debería verificar antes de empezar a trabajar?

Esto es opcional. Son comandos que cada sesión corre al arrancar para
asegurarse de que todo está en orden. Ejemplos:

• "Que exista el archivo de credenciales" → ls data/secrets.txt
• "Que el servidor responda" → curl -s https://miapp.com | head
• "Que los tests pasen" → npm test

Si no se te ocurre nada, saltá esta pregunta — siempre se pueden
agregar después.
```

**Pregunta 7 — Revisión manual personal** (solo si habilitó Journey)
```
¿Cuándo te gustaría revisar la app personalmente?

Además de las pruebas automáticas (QA y Journey que yo corro),
hay cosas que solo vos podés notar probando la app en tu browser:
la velocidad real, si los textos suenan naturales, si funciona
bien en el celular, si el copy-paste anda, etc.

Te puedo generar un checklist interactivo para que lo hagas cuando
quieras. ¿Cuándo preferís?

• "Después de cada Journey" — siempre te lo genero y te aviso
• "Cuando los scores superen 8" — solo cuando el producto ya
  esté maduro (recomendado)
• "Yo te aviso" — nunca te lo pido, solo lo hago si vos me decís
```

Guardar la preferencia en el config como:
```yaml
manual_review:
  trigger: "after_journey" | "scores_above_8" | "on_demand"
```

Con las respuestas, generá `.claude/orchestrate.yml` y mostráselo al usuario para confirmar antes de guardarlo.

### Modo 2: Config existe (`.claude/orchestrate.yml` encontrado)

Leé la config y ejecutá la secuencia de diagnóstico para determinar en qué
ESTADO está el proyecto antes de hacer cualquier otra cosa.

#### Paso 0: Diagnóstico de estado (SIEMPRE correr primero)

Antes de presentar opciones al usuario, el skill debe diagnosticar en cuál de los
6 estados posibles se encuentra. Esto se hace con 4 checks automáticos:

```bash
# Check A: ¿Existe handoff?
cat memory/handoff_next_session.md 2>/dev/null | head -5

# Check B: ¿Cuándo fue el último commit? (frescura del proyecto)
git log --oneline -1 --format="%ar"

# Check C: ¿Hubo commits DESPUÉS del último handoff?
# Comparar la fecha del handoff con el último commit
git log --oneline --since="<fecha del handoff>" | wc -l

# Check D: ¿La config refleja la realidad?
# Verificar que los paths en orchestrate.yml existen
```

**Matriz de decisión:**

| Check A (handoff) | Check B (frescura) | Check C (commits post-handoff) | → Estado |
|---|---|---|---|
| No existe | - | - | **Estado 1**: Primera vez parcial (tiene config pero no handoff). Crear handoff desde git log. |
| Existe, reciente (<24h) | Reciente | 0 nuevos | **Estado 3**: Retorno inmediato. Arrancar donde quedó. |
| Existe, reciente (<24h) | Reciente | >0 nuevos | **Estado 2**: Arranque fresco con cambios. Presentar diff. |
| Existe, viejo (>3 días) | Reciente | >0 nuevos | **Estado 4**: Retorno con gap. Reconstruir contexto. |
| Existe, viejo (>7 días) | Cualquiera | Muchos | **Estado 5**: Proyecto cambió mucho. Reconstruir estado. |
| Existe | - | Comandos de config fallan | **Estado 6**: Config obsoleta. Ofrecer re-configurar. |

**Acciones por estado:**

**Estado 1 — Config sin handoff:**
El usuario configuró el skill pero nunca corrió una sesión completa.
→ Crear handoff inicial desde `git log` + `git status` + explorar estructura.
→ Presentar: "Encontré tu config pero no hay historial de sesiones previas. Voy a
  armar el estado inicial desde tu repo."

**Estado 2 — Arranque fresco (normal):**
Todo reciente y alineado.
→ Cargar handoff, presentar estado, sugerir siguiente paso.

**Estado 3 — Retorno inmediato:**
Mismo día, probablemente cerró y reabrió. Handoff está fresco.
→ "Retomamos donde quedamos. Último estado: [resumen del handoff]"

**Estado 4 — Retorno con gap (días/semanas sin usar el skill):**
El handoff tiene datos viejos pero hubo actividad en el repo.
→ Leer handoff PERO verificar contra `git log` qué cambió desde entonces.
→ Mostrar: "Tu último handoff es del {fecha}. Desde entonces hubo {N} commits
  que no trackeé. Los analizo para reconstruir el estado actual."
→ Recorrer los commits nuevos, leer los mensajes, y ACTUALIZAR el handoff.
→ Preguntar al usuario: "¿Pasó algo relevante que no esté en los commits?"

**Estado 5 — Proyecto cambió mucho:**
Handoff viejo + muchos commits → el handoff es poco confiable.
→ "Pasaron {N} días y {M} commits desde mi último handoff. Voy a reconstruir
  el estado del proyecto desde cero usando git + archivos."
→ Escanear: `git log`, tests, docs, package.json changes, nuevos archivos.
→ Generar handoff fresco basado en evidencia del repo.
→ Pedir confirmación: "Esto es lo que entiendo del estado actual: [resumen].
  ¿Me falta algo?"

**Estado 6 — Config obsoleta:**
Los paths o comandos del config no funcionan (ej: `docs/qa/` fue movido,
deploy command cambió, un spoke fue eliminado).
→ Verificar cada path/comando del config.
→ Para cada fallo: "El config dice que {X} está en {path}, pero no existe.
  ¿Se movió o se eliminó?"
→ Ofrecer: "¿Querés que actualice la config?" + re-hacer las preguntas
  solo para las partes que cambiaron.

#### Paso 1: Cargar contexto (después del diagnóstico)

Una vez determinado el estado, cargar contexto relevante:

```
1. Leer .claude/orchestrate.yml → saber qué spokes tiene el proyecto
2. Leer memory/handoff_next_session.md → saber estado actual (puede estar
   recién creado o recién actualizado por el diagnóstico)
3. Buscar memory/spoke_report_*.md → reportes de spokes pendientes de absorber
4. Leer memory/project_prospect_feedback_scores.md → si existe, prioridades
5. Leer memory/MEMORY.md → índice de todo lo que hay
6. git log --oneline -10 → últimos cambios
7. git branch --all | grep claude/ → ramas de spokes activas
8. git status --short → hay algo sin commitear?
```

#### Paso 2: Absorber reportes pendientes de spokes

Antes de presentar estado al usuario, verificá si hay reportes de spoke sin procesar:

```bash
ls memory/spoke_report_*.md 2>/dev/null
```

Si hay archivos, para cada uno:
1. Leé el reporte (tiene branch name, commits, resumen, findings)
2. Leé la branch del spoke via git para contexto completo:
   ```bash
   git log {spoke_branch} --oneline -20
   git diff demo..{spoke_branch} --stat
   ```
3. Si el spoke commiteo un session file (ej: `docs/qa/sessions/...`), leélo:
   ```bash
   git show {spoke_branch}:docs/qa/sessions/{file} 2>/dev/null
   ```
4. Incorporá los findings al estado general
5. Renombrá el reporte a `spoke_report_{tipo}_absorbed.md` para no re-procesarlo

#### Paso 3: Presentar estado
Mostrá al usuario un resumen ejecutivo:
```
## Estado del proyecto: {nombre}

**Última actividad**: {fecha y descripción del último commit/handoff}
**Branch actual**: {branch}
**Spokes recientes**: {lista de ramas spoke detectadas con estado}
**Pendientes**: {lista de todos del handoff}

## Reportes absorbidos
{si hubo spoke reports, mostrar resumen de cada uno}

## Sesiones disponibles

1. QA → {estado: no corrido / último corrido fecha / N bugs pendientes}
2. Journey → {estado}
3. Feature → {features en backlog con prioridad}
4. Dev → {quick wins pendientes}

## Recomendación
{qué haría yo primero y por qué}

## ⚠️ Validación pendiente
{si hubo features/fixes shipeados desde la última Session A+B, mostrar:}
"Se shipearon N cambios desde la última validación. Recomiendo correr
Session A (QA focalizado en lo nuevo) + Session B (Journey re-validación)
antes de seguir con features nuevas."

¿Qué querés hacer?
```

#### Regla de re-validación obligatoria

Después de cada batch de features/fixes deployados, el Control Session DEBE
recomendar una ronda de re-validación antes de seguir con features nuevas.

El ciclo correcto es:
```
Implementar features → Deploy → Session A (QA) → Fix bugs → Session B (Journey) → Confirmar scores → Siguiente batch
```

NO es correcto:
```
Implementar features → Deploy → Implementar más features → Deploy → ... → QA al final
```

**Por qué**: cada feature puede introducir regresiones en las anteriores. Si
acumulás 5 features sin validar, encontrar cuál rompió qué es mucho más difícil.
Además, los scores estimados del handoff son PROYECCIONES — solo Session B
los confirma con datos reales.

**Cuándo omitir la re-validación**: solo si los cambios son puramente de
infraestructura (deploy scripts, config, docs) que no afectan la UI ni el
comportamiento del producto.

#### Paso 4: Generar bloques para spoke sessions

Cuando el usuario decide qué sesión spoke abrir, generá el bloque copy-paste.

**IMPORTANTE**: El bloque debe incluir instrucciones para que el spoke escriba su reporte en memory al terminar (NO pedirle al usuario que copie de vuelta).

**Contexto del producto en cada bloque**: Si `product` está definido en el config,
incluir en la sección "Tu contexto" del bloque:
- Lista de módulos con descripción + URL + roles que lo ven
- Cuentas de test con login URL + nombre
- Escala de datos de demo

Esto permite que el spoke:
- **QA**: sepa qué módulos testear y cómo loguearse como cada rol
- **Journey**: sepa qué features existen para guiar a cada persona
- **Feature**: sepa qué módulos existen para no duplicar
- **Dev**: sepa dónde está cada cosa para aplicar fixes
- **Deploy**: sepa qué archivos y comandos de build/deploy hay

Si `product` NO está definido, el bloque solo incluye lo que haya en
`project.description`. Pero recomendar al usuario que llene `product`
si va a correr QA o Journey ("Para que el QA sepa qué testear, te
recomiendo llenar la sección product de tu orchestrate.yml").

Estructura del bloque:

```markdown
## Bloque para [TIPO] Session — copiá todo esto y pegalo en una sesión nueva:

---INICIO DEL BLOQUE---

## ⚠️ VALIDACIÓN DE SESIÓN (corré esto ANTES de hacer cualquier otra cosa)

SESSION_TYPE: {tipo}

Antes de ejecutar CUALQUIER instrucción de este bloque, verificá:

1. **¿Ya hiciste trabajo de OTRO tipo en esta sesión?**
   ```bash
   git log --oneline -10
   ```
   Si ves commits de otro tipo de sesión (ej: commits de "qa(session)" y este bloque
   dice SESSION_TYPE: journey), PARÁ y decile al usuario:

   > "⚠️ BLOQUE EQUIVOCADO DETECTADO
   >
   > Esta sesión ya tiene trabajo de tipo [QA]. Me llegó un bloque de tipo [JOURNEY].
   > Probablemente pegaste el bloque equivocado.
   >
   > **Consecuencias de ejecutar de todas formas:**
   > - Los commits de Journey quedarían MEZCLADOS con los de QA en la misma branch
   > - El reporte spoke_report se sobreescribiría, perdiendo los findings del QA
   > - El Control Session no podrá distinguir qué commits son de QA y cuáles de Journey
   > - Si hay fixes del QA sin mergear, el merge a demo podría incluir trabajo
   >   incompleto del Journey mezclado con el QA
   > - El handoff queda inconsistente: dice 'QA completado' pero la branch tiene
   >   Journey encima — la próxima sesión se confunde
   >
   > **Lo correcto**: abrí una sesión NUEVA para Journey y pegá el bloque ahí.
   > Esta sesión debería quedarse como QA puro.
   >
   > ¿Querés que ignore esto y ejecute de todas formas? (no recomendado)"

   NO ejecutes silenciosamente — SIEMPRE mostrá las consecuencias y esperá respuesta.

2. **¿El bloque matchea lo que el usuario te pidió?**
   Si el usuario dijo "hacé QA" pero el bloque dice SESSION_TYPE: journey,
   avisale del mismatch antes de proceder. Consecuencia: el usuario puede
   estar confundido sobre qué sesión es esta y terminar con trabajo mezclado.

3. **¿Ya existe un spoke_report de otro tipo en memory?**
   ```bash
   ls memory/spoke_report_*.md 2>/dev/null
   ```
   Si existe un spoke_report de otro tipo sin absorber, advertir:
   "Hay un reporte de [otro tipo] pendiente. Si ejecuto [este tipo] ahora,
   podría sobreescribirlo. ¿El Control ya absorbió el anterior?"

Si todo OK, seguí con la verificación de archivos.

---

Sos una sesión de {tipo}. Tu trabajo es {descripción en 1 oración}.

## Tu contexto
{resumen del proyecto en 3-5 líneas — qué es, qué stack usa, en qué estado está}

## Verificación de archivos (corré esto segundo)
```bash
{comandos ls/git para verificar que los archivos necesarios existen}
```
Si alguno falla, intentá: {fallback command}

## Tu tarea específica
{instrucciones detalladas de qué hacer, paso por paso}

## Archivos relevantes que ya existen
{lista de paths con 1 línea de descripción de cada uno}

## Reglas
- Commiteá tu trabajo a la branch en la que estés (NO cambies de branch)
- NO pidas credenciales al usuario — están en {path} si las necesitás
- NO intentes cambiar de worktree — trabajá desde donde estás
- Si algo no funciona, anotalo y seguí — no te trabes
- Commiteá frecuentemente — cada unidad de trabajo terminada = 1 commit

## Deploy (si necesitás deployar)
{comandos de deploy del proyecto, sacados del .yml}

## AL TERMINAR — CRÍTICO (leé esto ANTES de empezar)

Cuando termines todo tu trabajo, hacé estas 3 cosas en orden:

### 1. Commiteá todo lo pendiente
```bash
git add -A && git status
# Si hay cambios, commitealos
git commit -m "{tipo}(session): {resumen de lo que hiciste}"
```

### 2. Escribí tu reporte en memory (para que Control lo lea)

**IMPORTANTE para Journey sessions**: el reporte DEBE incluir el DESGLOSE
de cada score, no solo el número. Para cada persona:
- Qué sumó (cada item con puntos: "+1 por X, +0.5 por Y")
- Qué restó (cada item con puntos: "-0.5 por Z")
- Por qué el score es ese y no más alto
- Si es una re-corrida: qué cambió vs la ronda anterior

Sin desglose, el Control no puede priorizar qué arreglar primero.

**IMPORTANTE para Journey sessions — sugerencia al final del resumen:**
Al final del resumen que el Journey muestra en el chat (no en el spoke_report,
sino en la conversación con el usuario), incluir como última línea una
sugerencia sutil, NO como pregunta:

> "💡 Si quieres, puedes probar la app tú mismo — dile al Control
> 'quiero revisar la app' y te genera un checklist interactivo."

Esto va SIEMPRE, como parte natural del cierre del resumen. No es un modal,
no es una pregunta que espere respuesta — es información que el usuario
puede actuar o ignorar.

Creá el archivo memory/spoke_report_{tipo}.md con este contenido:

```markdown
---
name: Spoke Report — {tipo}
description: Reporte de sesión {tipo} completada
type: project
---

## Reporte de Session {TIPO}

**Fecha**: {fecha de hoy}
**Branch**: {tu branch actual — corré git branch --show-current}
**HEAD**: {corré git rev-parse --short HEAD}
**Estado**: COMPLETADO | PARCIAL | BLOQUEADO

### Commits realizados
{corré git log demo..HEAD --oneline y pegá el output}

### Resumen
{3-5 bullets de qué hiciste}

### Findings
{bugs encontrados, UX gaps, features faltantes}

### Para la próxima sesión
{qué quedó pendiente, qué recomendás hacer next}
```

### 3. Decile al usuario
Mostrá este mensaje al usuario:

> Terminé. Mi reporte está en `memory/spoke_report_{tipo}.md` y mi trabajo
> está commiteado en la branch `{tu branch}`. Cuando vuelvas a tu Control
> Session, decile: **"absorbé los reportes"** y va a leer todo
> automáticamente. No necesitás copiar nada.

---FIN DEL BLOQUE---
```

#### Paso 5b: Checklist HTML para QA manual del usuario (condicional)

La generación del checklist depende de la preferencia `manual_review.trigger`
del config:

- `"after_journey"` → generar después de CADA Journey
- `"scores_above_8"` → generar SOLO cuando TODOS los roles tienen score > 8
  (leer scores del último spoke_report_journey o del handoff)
- `"on_demand"` → NO generar automáticamente. Solo si el usuario dice
  "quiero revisar la app" o similar

Si la condición no se cumple, NO mencionar el checklist. El Journey ya
incluye la sugerencia sutil al final del resumen ("si quieres, puedes
probar la app tú mismo").

Cuando la condición SÍ se cumple, generar el checklist y decir:
> "Los scores de los 3 roles superaron 8. Es buen momento para que pruebes
> la app personalmente. Generé un checklist en `docs/qa/demo-checklist.html`."

Después de cada ciclo de validación (Session A + B) y si la condición aplica,
el Control genera un **checklist HTML interactivo** para que el usuario pruebe
la app manualmente. Las sesiones automáticas (QA + Journey) cubren el 80% de los
casos, pero siempre hay cosas que solo un humano nota:

- Sensación de velocidad (¿se siente lento?)
- Flujo natural (¿dónde haría click instintivamente?)
- Textos y tono (¿suena natural o robótico?)
- Mobile real en un celular real (no un resize de browser)
- Copy-paste funciona (clipboard, WhatsApp, Excel)
- Acentos, formato de fechas, moneda, RUT

**Cómo generar el checklist:**

1. Leer `product.modules` y `product.test_accounts` del config
2. Para cada rol de test_account, crear un flujo con pasos que cubran:
   - Login → primera vista → navegación principal
   - La feature/acción más importante de ese rol
   - Un caso cross-role (bidireccionalidad entre roles)
3. Generar un archivo HTML autocontenido en `docs/qa/demo-checklist.html`
4. El HTML debe tener:
   - **Dos botones por paso**: ✓ Pass (verde) y ! Issue (rojo)
   - **Textarea** que aparece al marcar Issue (para describir el problema)
   - **Barra de progreso** global con ratio pass/issue
   - **Badges por flujo** que cambian de color según estado
   - **Persistencia en localStorage** (el usuario puede cerrar y volver)
   - **Botón "Copiar issues"** que exporta todos los issues al clipboard formateados para pegar en el Control
   - **Links directos de login** para cada rol (clickeables)
   - **Tags por paso**: verificar (azul), acción (morado), crítico (rojo)
5. Decirle al usuario:
   > "Generé un checklist en `docs/qa/demo-checklist.html`. Abrilo en tu
   > browser, probá cada paso, marcá pass o issue. Cuando termines, click
   > 'Copiar issues' y pegame el resultado acá."

**Cuándo re-generar el checklist:**
- Después de cada batch de features/fixes (los pasos pueden haber cambiado)
- Cuando se agrega un módulo nuevo al config
- Cuando el usuario pide "actualizá el checklist"

**El checklist NO reemplaza Session A ni B** — las complementa. Es la capa
de validación humana que cubre lo que la IA no puede ver.

#### Paso 6: Cierre de Control Session (obligatorio antes de cerrar o pausar)

Cuando el usuario indica que quiere cerrar, pausar, o "retomar mañana", SIEMPRE
actualizar `memory/handoff_next_session.md` con:

1. **Qué se hizo** en esta sesión (lista de commits/features/fixes)
2. **Estado de validación** (¿se corrió Session A+B después de los últimos cambios?)
3. **Scores actuales** (confirmados por Journey o estimados — marcar cuál)
4. **Spoke reports pendientes** (¿hay alguno sin absorber?)
5. **Próximo paso recomendado** (qué haría la próxima Control Session primero)
6. **Spokes activos** (qué sesiones spoke están abiertas y en qué estado)

El handoff debe ser suficiente para que la próxima invocación de `/orchestrate`
arranque sin preguntarle nada al usuario. Si `/orchestrate` necesita preguntar
"¿qué pasó desde la última vez?", el handoff falló.

**Template mínimo del handoff:**
```markdown
## Estado al {fecha}
- Branch: {branch}, HEAD: {sha}
- Scores: Director {X}, Tutor {Y}, Ayudante {Z} ({confirmados/estimados})
- Validación: {Session A+B corridas/pendientes}
- Último cambio: {descripción del último commit relevante}

## Qué se hizo
{lista numerada de features/fixes/infra}

## Próximo paso
{recomendación concreta — no "decidir qué hacer" sino "hacer X porque Y"}

## Spokes activos
- QA: {última corrida, estado}
- Journey: {última corrida, ronda N, estado}
- Feature: {abierta/cerrada}
```

NUNCA cerrar una Control Session sin actualizar el handoff. Si el usuario dice
"cerremos" y el handoff no está actualizado, actualizalo ANTES de cerrar.

#### Paso 6: Absorber reportes (cuando el usuario dice "absorbé los reportes" o similar)

1. Buscar `memory/spoke_report_*.md` (excluyendo los `*_absorbed.md`)
2. Para cada reporte:
   a. Leer el contenido (branch, commits, findings)
   b. Leer la branch del spoke via git:
      ```bash
      git log {branch} --oneline | head -20
      git diff demo..{branch} --stat
      ```
   c. Si hay session files commiteados, leerlos via `git show`
   d. Presentar un resumen al usuario
   e. Preguntar: "¿mergeo esta branch a demo?" → si sí, hacer el merge
   f. Renombrar el reporte a `spoke_report_{tipo}_absorbed.md`
3. Actualizar `memory/handoff_next_session.md` con los resultados absorbidos
4. Presentar estado actualizado + recomendación de siguiente paso

## Formato del archivo `.claude/orchestrate.yml`

```yaml
# Configuración de orquestación para {proyecto}
# Creado por /orchestrate, editable manualmente

project:
  name: "Nombre del proyecto"
  description: "Descripción corta de qué hace"

# Rama principal — a donde se mergean los spokes al terminar
# También es la base desde donde se crean nuevas ramas de trabajo
branches:
  base: "main"  # o "master", "develop", "demo", etc.

# Tipos de sesiones spoke disponibles
# Poné enabled: false para desactivar un tipo sin borrarlo
spokes:
  qa:
    enabled: true
    description: "Testing exhaustivo buscando bugs y edge cases"
    docs: "docs/qa/"  # opcional

  journey:
    enabled: true
    description: "Simular prospectos evaluando el producto"
    docs: "docs/qa/"
    personas: "docs/qa/personas/"

  feature:
    enabled: true
    description: "Implementar funcionalidad nueva con plan TDD"
    plans: "docs/superpowers/plans/"

  dev:
    enabled: true
    description: "Quick wins, fixes, mejoras chicas"

  deploy:
    enabled: true
    description: "Setup de hosting, configurar pipeline de deploy, debugging de producción"

# ─── Contexto del producto ───
# Esta sección le da a TODOS los spokes el contexto necesario para entender
# qué hace la app, qué módulos tiene, qué roles existen, y cómo loguearse
# para probar. SIN esta sección, cada spoke necesita leer N archivos dispersos
# para entender el producto. CON esta sección, el Control incluye todo en
# el bloque generado y el spoke arranca inmediato.

product:
  # Módulos/pantallas de la app — qué existe y qué hace
  modules:
    - name: ""           # nombre del módulo (ej: "Dashboard")
      description: ""    # 1 oración de qué hace
      url: ""           # ruta en la app (ej: "/dashboard")
      roles: []         # qué roles lo ven (ej: ["tutor", "ayudante"])

  # Cuentas de test / demo para probar la app
  # El QA y Journey las usan para loguearse como cada rol
  test_accounts:
    - role: ""          # nombre del rol (ej: "director")
      login: ""         # comando o URL para loguearse (ej: "/auth/demo-login?role=director")
      name: ""          # nombre que aparece en la app después del login
      description: ""   # contexto del rol en 1 oración

  # Datos de demo — qué data tiene la app para testing
  demo_data:
    description: ""     # cómo se generó la data (ej: "scripts/generate_demo_data.py")
    scale: ""          # volumen (ej: "10 cursos, 300 alumnos, 5 tutores")

# ─── Comandos de deploy ───
deploy:
  build: ""
  frontend: ""
  backend: ""
  restart: ""

# Archivo de credenciales del hosting (gitignored — NO las credenciales, sino el PATH al archivo)
hosting:
  credentials: ""
  type: ""

# Comandos de verificación que cada spoke corre al arrancar
verify:
  - "git branch --show-current"
  - "git log --oneline -3"

# Gotchas (incluidos en los bloques spoke)
gotchas: []

# Roles de la app (para QA y Journey)
roles: []
```

### Cómo el skill usa `branches.base`

1. **Al generar bloques spoke**: el bloque incluye `git diff {base}..HEAD --stat` en las instrucciones de reporte, para que el spoke sepa qué cambió respecto a la rama principal.

2. **Al absorber reportes**: Control corre `git diff {base}..{spoke_branch} --stat` para ver el diff completo del spoke.

3. **Al mergear**: Control pregunta "¿mergeo {spoke_branch} a {base}?" y ejecuta `git merge {spoke_branch}` desde la rama base.

4. **Si no está configurado**: el skill intenta auto-detectar con `git symbolic-ref refs/remotes/origin/HEAD` o busca `main` / `master` / `develop` en ese orden. Si no encuentra ninguna, pregunta al usuario.

## Comunicación entre sesiones — resumen

```
CONTROL → SPOKE:  usuario copia bloque inicial (1 vez, inevitable)
SPOKE → CONTROL:  spoke escribe memory/spoke_report_{tipo}.md
                   + commitea trabajo a su branch
                   usuario solo dice "absorbé los reportes"
                   Control lee memory + git automáticamente
                   CERO copy-paste de vuelta
```

**Canales de comunicación**:
1. `memory/spoke_report_*.md` — project-scoped, compartido entre todas las sesiones del proyecto
2. Git branches — las ramas de cada spoke son visibles desde cualquier worktree via `git log {branch}`
3. `git show {branch}:{path}` — Control lee archivos de un spoke sin estar en su worktree

## Generación inteligente de bloques spoke

El Control Session DEBE adaptar los bloques según el estado real de cada sesión spoke. No todos los bloques son iguales — hay 3 escenarios:

### Escenario 1: Sesión spoke NUEVA (nunca usada)
Bloque completo con todo el contexto: qué es el proyecto, qué hacer, verificación, etc. El spoke no sabe nada — hay que darle todo.

### Escenario 2: Sesión spoke que YA CORRIÓ este tipo antes
El spoke ya tiene contexto del trabajo anterior. El bloque debe ser MÁS CORTO y decir:
- "Ya corriste un [tipo] antes en esta sesión con estos resultados: [resumen]"
- "Desde entonces se hicieron estos cambios: [lista de fixes/features]"
- "Re-corré el [tipo] con foco en lo que cambió, no desde cero"
- Ejemplo: un Journey que se re-valida después de fixes → "compará ronda 1 vs ronda 2"

### Escenario 3: Sesión spoke que corrió OTRO tipo (error del usuario)
Detectado por SESSION_TYPE validation. Mostrar consecuencias y recomendar sesión nueva.

### Reglas para el Control al generar bloques

1. **Preguntale al usuario**: "¿Ya tenés una sesión de [tipo] abierta o abrís una nueva?"
   - Si ya tiene una abierta → generar bloque Escenario 2 (corto, con diff del anterior)
   - Si abre nueva → generar bloque Escenario 1 (completo)

2. **Incluir siempre qué cambió** desde la última vez que ese spoke corrió. El Control sabe esto porque tiene el handoff + git log. Ejemplo:
   ```
   "Desde tu última corrida se shipearon:
   - fix: CSRF localhost para demo sessions
   - feat: notas por alumno
   - feat: export SENCE CSV
   Re-validá con estos cambios en mente."
   ```

3. **Nombrar el session file con versión** si es una re-corrida:
   - Primera vez: `2026-04-06-prospect-journey.md`
   - Re-corrida: `2026-04-06-prospect-journey-v2.md`
   Esto evita sobreescribir el resultado anterior.

4. **NUNCA generar un bloque de sesión nueva para una sesión que ya tiene trabajo**. Si el usuario dice "volvé a correr el Journey en la misma sesión", el bloque debe reconocer el trabajo previo y pedir comparación, no empezar de cero como si fuera virgin.

## Errores comunes del usuario y cómo prevenirlos

### Error: pegar bloque de tipo B en sesión de tipo A
**Prevención**: SESSION_TYPE check al inicio de cada bloque (ya implementado).
**Consecuencias**: commits mezclados, reportes sobreescritos, handoff inconsistente.

### Error: pedir al Control "pasame el bloque para Session B" cuando Session B ya corrió algo
**Prevención**: el Control debe preguntar "¿Session B ya corrió antes o es nueva?" y adaptar el bloque.
**Consecuencia si no se previene**: el spoke recibe instrucciones de "empezá de cero" cuando ya tiene trabajo hecho → confusión, trabajo duplicado, o peor: sobreescribe su propio session file.

### Error: olvidar que una sesión spoke ya tiene contexto
**Prevención**: el Control debe trackear en el handoff qué spokes corrieron y cuándo. Al generar un bloque, siempre verificar si ese tipo de spoke ya corrió en el proyecto recientemente.

### Error: no distinguir entre "sesión nueva" y "misma sesión que ya trabajó"
**Prevención**: el Control debe preguntar SIEMPRE antes de generar el bloque:
> "¿Vas a pegar esto en una sesión nueva o en [la sesión que ya hizo X]?"
Y generar el bloque apropiado (Escenario 1 o 2).

## Notas de implementación

- Este skill vive en `~/.claude/commands/orchestrate.md` (user-level, disponible en TODOS los proyectos)
- La config `.claude/orchestrate.yml` es por proyecto (cada repo tiene la suya)
- Las memorias (`memory/spoke_report_*.md`) son por proyecto (project-scoped)
- El skill NUNCA hardcodea paths, nombres de archivo, o conocimiento específico de un proyecto
- Todo lo que sabe del proyecto viene de: la config .yml + las memorias + lo que descubre con ls/git
- Si falta info, pregunta al usuario — nunca asume
- Los spoke reports sin procesar se detectan por la ausencia del sufijo `_absorbed` en el nombre
