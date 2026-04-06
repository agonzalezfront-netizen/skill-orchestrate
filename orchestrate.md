# Session Orchestrator

Skill genérico para coordinar múltiples sesiones de Claude Code en cualquier proyecto. Funciona como un hub que genera instrucciones copy-paste para sesiones spoke (QA, Journey, Feature, Dev, etc.).

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

Con las respuestas, generá `.claude/orchestrate.yml` y mostráselo al usuario para confirmar antes de guardarlo.

### Modo 2: Config existe (`.claude/orchestrate.yml` encontrado)

Leé la config, leé `memory/handoff_next_session.md` y cualquier otra memoria relevante del proyecto, y arrancá el ciclo de orquestación:

#### Paso 1: Cargar contexto
```
1. Leer .claude/orchestrate.yml → saber qué spokes tiene el proyecto
2. Leer memory/handoff_next_session.md → saber estado actual
3. Leer memory/project_prospect_feedback_scores.md → si existe, prioridades
4. Leer memory/MEMORY.md → índice de todo lo que hay
5. git log --oneline -10 → últimos cambios
6. git status --short → hay algo sin commitear?
```

#### Paso 2: Presentar estado
Mostrá al usuario un resumen ejecutivo:
```
## Estado del proyecto: {nombre}

**Última actividad**: {fecha y descripción del último commit/handoff}
**Branch actual**: {branch}
**Pendientes**: {lista de todos del handoff}

## Sesiones disponibles

1. QA → {estado: no corrido / último corrido fecha / N bugs pendientes}
2. Journey → {estado}
3. Feature → {features en backlog con prioridad}
4. Dev → {quick wins pendientes}

## Recomendación
{qué haría yo primero y por qué}

¿Qué querés hacer?
```

#### Paso 3: Generar bloques para spoke sessions

Cuando el usuario decide qué sesión spoke abrir, generá el bloque copy-paste con esta estructura:

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
Si alguno falla, decile al usuario que haga: {fallback command}

## Tu tarea específica
{instrucciones detalladas de qué hacer, paso por paso}

## Archivos relevantes que ya existen
{lista de paths con 1 línea de descripción de cada uno}

## Reglas
- Commiteá tu trabajo a la branch en la que estés
- NO pidas credenciales al usuario — están en {path} si las necesitás
- NO intentes cambiar de worktree — trabajá desde donde estás
- Si algo no funciona, anotalo y seguí — no te trabes

## Deploy (si necesitás deployar)
{comandos de deploy del proyecto, sacados del .yml}

## Al terminar — IMPORTANTE
Cuando termines todo tu trabajo, generá este reporte para que el usuario
lo pegue de vuelta en su Control Session:

```
REPORTE DE SESSION {TIPO}
Fecha: {hoy}
Branch: {tu branch}
Commits: {lista de SHAs con mensajes}
Estado: COMPLETADO | PARCIAL | BLOQUEADO

Resumen:
{3-5 bullets de qué hiciste}

Findings:
{bugs encontrados, UX gaps, cosas que no funcionaron}

Para la próxima sesión:
{qué quedó pendiente, qué recomendás hacer next}
```

---FIN DEL BLOQUE---
```

#### Paso 4: Absorber reportes de spoke

Cuando el usuario pega un reporte de spoke, parseá el contenido y:
1. Actualizá `memory/handoff_next_session.md` con los resultados
2. Re-evaluá prioridades
3. Presentá el estado actualizado
4. Sugerí el siguiente paso

## Formato del archivo `.claude/orchestrate.yml`

```yaml
# Configuración de orquestación para {proyecto}
# Creado por /orchestrate, editable manualmente

project:
  name: "Nombre del proyecto"
  description: "Descripción corta de qué hace"

# Tipos de sesiones spoke disponibles
# Poné enabled: false para desactivar un tipo sin borrarlo
spokes:
  qa:
    enabled: true
    description: "Testing exhaustivo buscando bugs y edge cases"
    docs: "docs/qa/"  # opcional, si tenés docs de QA

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
  build: ""        # comando para buildear (ej: npm run build)
  frontend: ""     # comando para subir frontend
  backend: ""      # comando para subir backend (con {file} como placeholder)
  restart: ""      # comando para reiniciar el servidor

# Comandos de verificación que cada spoke corre al arrancar
verify:
  - "git branch --show-current"
  - "git log --oneline -3"
```

## Notas de implementación

- Este skill vive en `~/.claude/commands/orchestrate.md` (user-level, disponible en TODOS los proyectos)
- La config `.claude/orchestrate.yml` es por proyecto (cada repo tiene la suya)
- Las memorias (`memory/handoff_*.md`) son por proyecto (project-scoped en Claude Code)
- El skill NUNCA hardcodea paths, nombres de archivo, o conocimiento específico de un proyecto
- Todo lo que sabe del proyecto viene de: la config .yml + las memorias + lo que descubre con ls/git
- Si falta info, pregunta al usuario — nunca asume
