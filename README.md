# Skill Orchestrate

**Session Orchestrator for Claude Code** — coordinate multiple Claude Code sessions using a hub-and-spoke pattern.

> **[Leer en Espanol](#espanol)**

---

## What is this?

When you work on a real project with Claude Code, you quickly realize one session isn't enough. You need:
- A **Control Session** to make decisions, design features, and coordinate
- A **QA Session** to hunt bugs systematically
- A **Prospect Journey Session** to validate your product with realistic personas
- A **Feature Session** to implement new functionality
- A **Dev Session** for quick fixes and improvements

The problem: **sessions can't talk to each other**. Each one starts fresh with no context about what the others did.

**Skill Orchestrate** solves this by making the Control Session generate **copy-paste ready instruction blocks** for each spoke session. You're the bridge — you copy from Control, paste into the spoke, and when the spoke finishes, you copy its report back to Control.

```
You open Control Session
  -> /orchestrate
  -> Control reads project state, presents options
  -> You choose: "I need a QA session"
  -> Control generates a text block with EVERYTHING the QA session needs
  -> You copy it

You open QA Session
  -> You paste the block
  -> QA session works autonomously (no questions, no "where is X?")
  -> QA finishes, generates a report block
  -> You copy the report

You go back to Control Session
  -> You paste the QA report
  -> Control absorbs results, re-prioritizes
  -> Control generates the next spoke block (Feature, Journey, etc.)
  -> Cycle continues
```

## How it works

The skill has two parts:

| Part | Where it lives | What it does |
|---|---|---|
| **The Skill** (generic) | `~/.claude/commands/orchestrate.md` | Orchestration logic — same for ALL projects |
| **Project Config** (specific) | `.claude/orchestrate.yml` in your repo | Tells the skill about YOUR project |

The skill is project-agnostic. It doesn't know about your tech stack, your deploy process, or your domain. All project-specific knowledge comes from the config file.

## Installation

### 1. Install the skill (one-time, user-level)

```bash
# Create the commands directory if it doesn't exist
mkdir -p ~/.claude/commands

# Copy the skill file
cp orchestrate.md ~/.claude/commands/orchestrate.md
```

That's it. The skill is now available as `/orchestrate` in every Claude Code session, in every project.

### 2. Configure your project (per-project)

Two options:

**Option A: Let the skill guide you (recommended for beginners)**

Open a Claude Code session in your project and run:
```
/orchestrate
```

The skill will detect there's no config and ask you 4 friendly questions (no technical jargon) to create one.

**Option B: Create the config manually**

Create `.claude/orchestrate.yml` in your project root:

```yaml
project:
  name: "My Project"
  description: "What your project does in one sentence"

spokes:
  qa:
    enabled: true
    description: "Find bugs and edge cases"
    docs: "tests/"

  journey:
    enabled: false  # Enable when you have personas

  feature:
    enabled: true
    description: "Implement new features with TDD plans"
    plans: "docs/plans/"

  dev:
    enabled: true
    description: "Quick fixes and improvements"

deploy:
  build: "npm run build"
  frontend: "vercel deploy --prod"

verify:
  - "git branch --show-current"
  - "git log --oneline -3"
```

## Session Types

### Control Session (Hub)

The brain. Opens first, closes last. Does NOT implement code directly (delegates that to spokes).

**What it does:**
1. Loads project context (memory files, handoff, git state)
2. Presents current state and priorities to you
3. Generates instruction blocks for spoke sessions
4. Absorbs spoke reports and re-prioritizes
5. Updates handoff memory for the next cycle

**When to open:** Start of each work day, or whenever you need to decide what to do next.

### QA Session (Spoke)

Systematic bug hunting. Goes module by module, role by role, testing every edge case.

**What it produces:** A matrix of test cases with scores (pass/fail/UX gap/missing feature) and a prioritized bug backlog.

### Prospect Journey Session (Spoke)

Simulates real prospects evaluating your product. Uses detailed personas with realistic backgrounds, pain points, and time constraints.

**What it produces:** A narrative per persona with scores (1-10), verdict (would they buy?), top frictions, and delights.

### Feature Session (Spoke)

Implements a new feature end-to-end: backend + frontend + tests + deploy. Follows a TDD plan.

**What it produces:** Commits, test results, deploy confirmation.

### Dev Session (Spoke)

Quick fixes, improvements, and polish. No formal plan needed — just a list of things to fix.

**What it produces:** List of what was fixed + deploy confirmation.

## The Config File

### Full reference

```yaml
# .claude/orchestrate.yml

project:
  name: "Project Name"
  description: "One-sentence description"

# Spoke types — enable/disable what you need
spokes:
  qa:
    enabled: true
    description: "What this spoke does"
    docs: "path/to/qa/docs/"           # optional
    template: "path/to/template.md"     # optional

  journey:
    enabled: true
    description: "Prospect validation"
    docs: "path/to/docs/"
    personas: "path/to/personas/"       # optional

  feature:
    enabled: true
    description: "New feature implementation"
    plans: "path/to/plans/"             # optional

  dev:
    enabled: true
    description: "Quick fixes and improvements"

# Deploy commands (all optional)
deploy:
  build: "command to build"
  frontend: "command to deploy frontend"
  backend: "command to deploy backend with {file} placeholder"
  restart: "command to restart server"

# Verification commands (spoke runs these on startup)
verify:
  - "git branch --show-current"
  - "ls some/critical/file"

# Project-specific gotchas (included in spoke blocks)
gotchas:
  - "Always do X before Y"
  - "Never do Z because..."

# Roles in your app (used for QA and Journey context)
roles:
  - admin: "Description of admin role"
  - user: "Description of user role"
```

### Minimal config

```yaml
project:
  name: "My App"
  description: "A web app"

spokes:
  feature:
    enabled: true
  dev:
    enabled: true
```

That's enough to start. Add more spokes and config as your project grows.

## Example: Real-world usage

This skill was developed while building **StudentHub**, a student tracking app for Chilean educational organizations. Here's how it was used:

```
Day 1 — Control Session:
  /orchestrate
  → "Feature C (Comunicaciones redesign) is the priority"
  → Designed mockups, wrote 13-task TDD plan
  → Implemented with subagent-driven development
  → Deployed to production

Day 1 — Delegated QA Session:
  → Control generated QA instruction block
  → QA session found 4 critical bugs, 4 important, 8 UX gaps
  → Fixed and redeployed

Day 1 — Delegated Prospect Journey:
  → Control generated Journey instruction block with 3 personas
  → Journey session scored: Director 7/10, Tutor 7.5/10, Assistant 6.5/10
  → Identified 3 quick wins that would raise scores ~1 point each

Day 2 — Control Session:
  /orchestrate
  → Absorbed QA + Journey results
  → "Quick wins first (1.5h), then Notes feature (2-3h)"
  → Implemented inline, deployed, re-validated
```

Total: 26 commits, 45 backend tests, 37 frontend tests, 3 personas evaluated, automated deploy pipeline — all coordinated through `/orchestrate`.

## FAQ

**Q: Can sessions talk to each other directly?**
No. You're the bridge. You copy blocks from Control and paste them into spokes. When spokes finish, you copy their report back to Control. The skill minimizes what you need to think about — it's just copy-paste.

**Q: Do I need all 4 spoke types?**
No. Start with `feature` + `dev`. Add `qa` when your product is mature enough to test systematically. Add `journey` when you have real or simulated prospects to validate with.

**Q: Can I run multiple spokes in parallel?**
Yes, as long as they don't modify the same files. QA + Journey can run in parallel (both are read-only). Two Feature sessions modifying the same codebase will cause merge conflicts.

**Q: What if a spoke session gets confused?**
The instruction block includes verification commands. If they fail, the block tells the spoke exactly what to do (e.g., `git merge demo`). If it's still stuck, the spoke reports BLOCKED and you bring it back to Control.

**Q: Does this work with any Claude Code project?**
Yes. The skill is generic. The config file makes it project-specific. You can use it for a React app, a Python API, a mobile app, a static site — anything.

---

<a name="espanol"></a>

# Skill Orchestrate (Espanol)

**Orquestador de Sesiones para Claude Code** — coordiná múltiples sesiones de Claude Code usando un patrón hub-and-spoke.

## Qué es esto?

Cuando trabajás en un proyecto real con Claude Code, rápidamente te das cuenta de que una sesión no alcanza. Necesitás:
- Una **Sesión de Control** para tomar decisiones, diseñar features, y coordinar
- Una **Sesión de QA** para buscar bugs sistemáticamente
- Una **Sesión de Prospect Journey** para validar tu producto con personas realistas
- Una **Sesión de Feature** para implementar funcionalidad nueva
- Una **Sesión de Dev** para fixes rápidos y mejoras

El problema: **las sesiones no pueden hablar entre sí**. Cada una arranca de cero sin contexto de lo que hicieron las otras.

**Skill Orchestrate** resuelve esto haciendo que la Sesión de Control genere **bloques de texto listos para copiar y pegar** para cada sesión spoke. Vos sos el puente — copiás del Control, pegás en el spoke, y cuando el spoke termina, copiás su reporte de vuelta al Control.

```
Abrís Sesión de Control
  → /orchestrate
  → Control lee el estado del proyecto, te presenta opciones
  → Elegís: "necesito una sesión de QA"
  → Control genera un bloque con TODO lo que la sesión QA necesita
  → Lo copiás

Abrís Sesión de QA
  → Pegás el bloque
  → La sesión QA trabaja sola (sin preguntas, sin "dónde está X?")
  → Termina, genera un bloque de reporte
  → Lo copiás

Volvés a la Sesión de Control
  → Pegás el reporte del QA
  → Control absorbe los resultados, re-prioriza
  → Control genera el siguiente bloque spoke (Feature, Journey, etc.)
  → El ciclo continúa
```

## Instalación

### 1. Instalar el skill (una vez, a nivel de usuario)

```bash
# Crear el directorio de comandos si no existe
mkdir -p ~/.claude/commands

# Copiar el archivo del skill
cp orchestrate.md ~/.claude/commands/orchestrate.md
```

Listo. El skill ahora está disponible como `/orchestrate` en cada sesión de Claude Code, en todos tus proyectos.

### 2. Configurar tu proyecto (por proyecto)

**Opción A: Que el skill te guíe (recomendado si no sos muy técnico)**

Abrí una sesión de Claude Code en tu proyecto y escribí:
```
/orchestrate
```

El skill va a detectar que no hay config y te hace 4 preguntas sencillas (sin jerga técnica) para crear una:

1. **"Cómo se llama tu proyecto y qué hace?"** — Solo el nombre y una oración. Ejemplo: "MiTienda — un e-commerce de ropa"

2. **"Qué tipos de sesiones necesitás?"** — Te explica cada tipo como si fuera un "modo de trabajo":
   - QA = "quiero probar que todo funcione"
   - Journey = "quiero simular que un cliente prueba mi producto"
   - Feature = "quiero agregar algo nuevo"
   - Dev = "quiero arreglar cosas"

3. **"Cómo se publica tu proyecto?"** — Opciones claras: "tengo un script", "uso Vercel", "lo subo por FTP", "no sé todavía"

4. **"Dónde viven los archivos importantes?"** — Tests, planes, docs. Si no tenés, decís "no tengo" y listo.

**Opción B: Crear la config a mano**

Creá `.claude/orchestrate.yml` en la raíz de tu proyecto:

```yaml
project:
  name: "Mi Proyecto"
  description: "Lo que hace mi proyecto"

spokes:
  qa:
    enabled: true
  feature:
    enabled: true
  dev:
    enabled: true

deploy:
  build: "npm run build"
```

## Tipos de sesión

| Sesión | Qué hace | Cuándo usarla |
|---|---|---|
| **Control** (Hub) | Decide, diseña, coordina, revisa | Al inicio del día o cuando necesitás decidir qué hacer |
| **QA** (Spoke) | Busca bugs módulo × rol × caso | Antes de mostrar el producto a alguien |
| **Journey** (Spoke) | Simula prospectos evaluando la demo | Cuando querés validar si tu producto convence |
| **Feature** (Spoke) | Implementa algo nuevo de punta a punta | Cuando hay una feature diseñada y priorizada |
| **Dev** (Spoke) | Arregla bugs y hace mejoras chicas | Cuando hay una lista de fixes conocidos |

## El ciclo de trabajo

```
Control Session (hub)
  → Carga contexto, prioriza
  → Genera bloque para spoke
                ↓
          [vos copiás]
                ↓
Spoke Session (QA/Journey/Feature/Dev)
  → Recibe bloque, trabaja solo
  → Genera reporte al final
                ↓
          [vos copiás]
                ↓
Control Session (hub)
  → Absorbe reporte
  → Re-prioriza
  → Genera siguiente bloque
  → ... ciclo continúa
```

## Preguntas frecuentes

**P: Las sesiones pueden hablar entre sí?**
No. Vos sos el puente. Copiás bloques del Control y los pegás en los spokes. Cuando terminan, copiás sus reportes de vuelta. El skill minimiza lo que tenés que pensar — es solo copy-paste.

**P: Necesito los 4 tipos de spoke?**
No. Empezá con `feature` + `dev`. Agregá `qa` cuando tu producto esté lo suficientemente maduro. Agregá `journey` cuando tengas prospectos reales o simulados para validar.

**P: Funciona con cualquier proyecto?**
Sí. El skill es genérico. La config lo hace específico a tu proyecto. Funciona para una app React, una API Python, una app mobile, un sitio estático — lo que sea.

**P: Qué pasa si un spoke se traba?**
El bloque de instrucciones incluye comandos de verificación. Si fallan, el bloque dice exactamente qué hacer. Si sigue trabado, el spoke reporta BLOQUEADO y vos lo llevás de vuelta al Control.

---

## License

MIT

## Credits

Developed during the construction of [StudentHub](https://pulsed.cl) — a student tracking platform for Chilean educational organizations (OTECs).
