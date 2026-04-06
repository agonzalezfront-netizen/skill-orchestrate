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

Con las respuestas, generá `.claude/orchestrate.yml` y mostráselo al usuario para confirmar antes de guardarlo.

### Modo 2: Config existe (`.claude/orchestrate.yml` encontrado)

Leé la config, leé `memory/handoff_next_session.md` y cualquier otra memoria relevante del proyecto, y arrancá el ciclo de orquestación:

#### Paso 1: Cargar contexto
```
1. Leer .claude/orchestrate.yml → saber qué spokes tiene el proyecto
2. Leer memory/handoff_next_session.md → saber estado actual
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

¿Qué querés hacer?
```

#### Paso 4: Generar bloques para spoke sessions

Cuando el usuario decide qué sesión spoke abrir, generá el bloque copy-paste.

**IMPORTANTE**: El bloque debe incluir instrucciones para que el spoke escriba su reporte en memory al terminar (NO pedirle al usuario que copie de vuelta).

Estructura del bloque:

```markdown
## Bloque para [TIPO] Session — copiá todo esto y pegalo en una sesión nueva:

---INICIO DEL BLOQUE---

Sos una sesión de {tipo}. Tu trabajo es {descripción en 1 oración}.

## Tu contexto
{resumen del proyecto en 3-5 líneas — qué es, qué stack usa, en qué estado está}

## Verificación (corré esto primero)
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

#### Paso 5: Absorber reportes (cuando el usuario dice "absorbé los reportes" o similar)

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

# Comandos de deploy (todos opcionales)
deploy:
  build: ""
  frontend: ""
  backend: ""
  restart: ""

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

## Notas de implementación

- Este skill vive en `~/.claude/commands/orchestrate.md` (user-level, disponible en TODOS los proyectos)
- La config `.claude/orchestrate.yml` es por proyecto (cada repo tiene la suya)
- Las memorias (`memory/spoke_report_*.md`) son por proyecto (project-scoped)
- El skill NUNCA hardcodea paths, nombres de archivo, o conocimiento específico de un proyecto
- Todo lo que sabe del proyecto viene de: la config .yml + las memorias + lo que descubre con ls/git
- Si falta info, pregunta al usuario — nunca asume
- Los spoke reports sin procesar se detectan por la ausencia del sufijo `_absorbed` en el nombre
