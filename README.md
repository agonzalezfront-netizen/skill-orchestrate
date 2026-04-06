# Skill Orchestrate

**Session Orchestrator for Claude Code** — coordinate multiple Claude Code sessions using a hub-and-spoke pattern. Sessions are **persistent and reusable** — they accumulate context over time and get better with each run.

> **[Leer en Espanol](#espanol)**

---

## What is this?

When you work on a real project with Claude Code, you quickly realize one session isn't enough. You need:
- A **Control Session** to make decisions, design features, and coordinate
- A **QA Session** to hunt bugs systematically
- A **Prospect Journey Session** to validate your product with realistic personas
- A **Feature Session** to implement new functionality
- A **Dev Session** for quick fixes and improvements
- A **Deploy Session** to set up hosting, automate deploys, and debug production

The problem: **sessions can't talk to each other**. Each one starts fresh with no context about what the others did.

**Skill Orchestrate** solves this with two mechanisms:
1. **Control → Spoke**: you copy-paste an instruction block (one-time per spoke)
2. **Spoke → Control**: the spoke writes its report to a shared memory file + commits its work to its git branch. Control reads both automatically. **Zero copy-paste on the return trip.**

```
You open Control Session
  -> /orchestrate
  -> Control reads project state, presents options
  -> You choose: "I need a QA session"
  -> Control asks: "New session or the one that already ran QA before?"
  -> Control generates a context-aware text block
  -> You copy it

You open QA Session (new or returning)
  -> You paste the block
  -> QA works autonomously
  -> QA finishes, writes memory/spoke_report_qa.md + commits work
  -> QA tells you: "Done. Tell Control to absorb reports."

You go back to Control Session
  -> You say: "absorb reports"
  -> Control reads memory file + git branch automatically
  -> Control absorbs results, re-prioritizes, suggests next step
  -> No copy-paste needed
```

## Key concept: sessions are persistent

**Spokes are NOT disposable.** You reuse them across the project lifecycle:

| Session | 1st run | 2nd run | 3rd run | ... |
|---|---|---|---|---|
| **QA** | Full scan, finds 8 bugs | Focused re-scan on fixed modules, finds 2 regressions | Quick smoke test pre-launch | Knows all prior findings |
| **Journey** | 3 personas score 6.5-7.5 | Re-validates after fixes, scores 8.5-9.5 | Tests with a new 4th persona | Compares across rounds |
| **Feature** | Implements auth system | Adds OAuth provider | Adds MFA | Knows the auth codebase intimately |

Each subsequent run is **faster and more accurate** because the session has context from prior runs. The Control Session tracks which spokes ran when and adapts its instruction blocks accordingly (see "Smart block generation" below).

**Important for the user:** keep your spoke sessions open! Don't close a QA session after it runs — you'll want it again after the next batch of fixes.

## How it works

The skill has two parts:

| Part | Where it lives | What it does |
|---|---|---|
| **The Skill** (generic) | `~/.claude/commands/orchestrate.md` | Orchestration logic — same for ALL projects |
| **Project Config** (specific) | `.claude/orchestrate.yml` in your repo | Tells the skill about YOUR project |

The skill is project-agnostic. It doesn't know about your tech stack, your deploy process, or your domain. All project-specific knowledge comes from the config file.

## Requirements

Before installing, make sure you have:

| Dependency | Why it's needed | How to check |
|---|---|---|
| **Claude Code** | The CLI tool this skill runs inside | `claude --version` |
| **Git** | Communication channel between sessions (branches, logs, diff) | `git --version` |
| **A Git repository** | Your project must be a git repo (the skill uses branches to track spoke work) | `git status` in your project dir |
| **Bash-compatible shell** | The skill generates bash commands for verification and deploy | `bash --version` |

**Optional but highly recommended:**

| Dependency | Why | How to check |
|---|---|---|
| **GitHub CLI (`gh`)** | Automates GitHub operations (see [why `gh`](#why-github-cli-gh-is-worth-installing)) | `gh --version` |
| **Hosting access** (SSH / cPanel API / Platform CLI) | Enables fully automated deploys (see [hosting setup](#hosting-access-ssh--cpanel-api--platform-cli)) | Depends on method |
| **Superpowers skills** | TDD, plans, debugging, code review (see [recommended skills](#recommended-complementary-skills)) | Check `~/.claude/` |
| **Frontend Design skill** | High-quality UI implementation | Check available skills |
| **Node.js / Python** | Only if your project uses them for build/deploy | `node --version` / `python --version` |

**Not required:**
- No database
- No cloud account
- No paid API keys (the skill itself is pure orchestration — it doesn't call any AI APIs)

### Recommended complementary skills

This skill handles **coordination**. For the actual work inside each session, these skills significantly improve quality:

| Skill | What it does | Which spokes benefit |
|---|---|---|
| `superpowers:writing-plans` | Creates TDD implementation plans with step-by-step tasks | Feature sessions |
| `superpowers:subagent-driven-development` | Executes plans via fresh subagents with two-stage review | Feature sessions |
| `superpowers:test-driven-development` | Test first, implement second, verify always | Feature + Dev sessions |
| `superpowers:systematic-debugging` | Root-cause analysis before proposing fixes | QA + Dev sessions |
| `superpowers:brainstorming` | Explores user intent and requirements before implementation | Control session (design phase) |
| `superpowers:finishing-a-development-branch` | Merge/PR/cleanup workflow after implementation | Feature sessions |
| `superpowers:requesting-code-review` | Structured code review before merging | Feature sessions |
| `superpowers:verification-before-completion` | Evidence-based verification before claiming "done" | All sessions |
| `frontend-design:frontend-design` | Production-grade UI with distinctive aesthetics (not generic AI slop) | Feature + Dev sessions |

**How they interact:** the Control session designs and plans (brainstorming + writing-plans). Feature spokes implement (subagent-driven-development + TDD). QA spokes debug (systematic-debugging). All spokes verify (verification-before-completion). The orchestrator coordinates; the skills execute.

### Why GitHub CLI (`gh`) is worth installing

Without `gh`, these tasks require you to open the browser, navigate GitHub, and do them manually. With `gh`, they happen in 1 command from the terminal:

| Task | Without `gh` (manual) | With `gh` (automated) |
|---|---|---|
| Create a new repo | Open GitHub.com → New repo → fill form → copy URL → git remote add | `gh repo create my-project --public --clone` |
| Create a Pull Request | Push branch → open GitHub → New PR → fill title/body → submit | `gh pr create --title "..." --body "..."` |
| View PR status/checks | Open GitHub → find PR → check status | `gh pr status` |
| Merge a PR | Open GitHub → click Merge → confirm | `gh pr merge --squash` |
| Create an issue | Open GitHub → Issues → New → fill form | `gh issue create --title "..."` |

**For this skill specifically**, `gh` enables:
- Control Session can **create PRs automatically** when merging spoke branches
- Spoke sessions can **push their branches** and create draft PRs in one step
- Control can **check CI status** of spoke branches before merging

<details>
<summary><strong>How to install and authenticate GitHub CLI</strong></summary>

**Install:**
```bash
# Windows
winget install --id GitHub.cli

# macOS
brew install gh

# Linux
sudo apt install gh
```

**Authenticate:**
```bash
gh auth login
# Choose: GitHub.com → HTTPS → Yes → Login with web browser
# Follow the one-time code flow in your browser
```

**Verify:**
```bash
gh auth status
# ✓ Logged in to github.com as YOUR_USERNAME
```

| Problem | Solution |
|---|---|
| `gh: command not found` | Restart your terminal after installing |
| `error: authentication required` | Run `gh auth login` again |
| `error: insufficient scope` | Regenerate token with `repo` + `workflow` scopes |
</details>

### Hosting access (SSH / cPanel API / Platform CLI)

If your project is deployed on a hosting provider, giving the skill access to your hosting enables **fully automated end-to-end deploys** — from code change to live in production, without opening a browser.

#### The automation ladder

| Level | What you have | What the skill automates | Manual work |
|---|---|---|---|
| **0 — Nothing** | No hosting configured | Nothing | Everything |
| **1 — SSH** | SSH key to your server | `rsync` + restart remotely | Nothing |
| **2 — cPanel API** | API token (no SSH needed) | Upload via UAPI + restart | Nothing |
| **3 — Platform CLI** | Vercel/Netlify/Railway CLI | `vercel deploy --prod` | Nothing |
| **4 — CI/CD** | GitHub Actions configured | `git push` = auto deploy | Nothing |

Pick ONE that matches your hosting. Most projects only need one.

<details>
<summary><strong>Detailed setup for each option (SSH, cPanel, Platform CLI, CI/CD)</strong></summary>

#### Option A: SSH

```bash
ssh-keygen -t ed25519 -C "deploy-key"
ssh-copy-id user@myserver.com
ssh user@myserver.com "echo 'SSH works!'"
```

Configure in `orchestrate.yml`:
```yaml
deploy:
  frontend: "rsync -avz --delete dist/ deploy@myserver.com:/var/www/myapp/"
  restart: "ssh deploy@myserver.com 'sudo systemctl restart myapp'"
```

#### Option B: cPanel API (no SSH needed)

1. cPanel → Security → Manage API Tokens → Create
2. Store token in `data/cpanel_credentials.txt` (gitignored)
3. The skill can help create a `scripts/deploy_cpanel.py`

```yaml
deploy:
  frontend: "python scripts/deploy_cpanel.py"
  backend: "python scripts/deploy_cpanel.py --backend-file {file}"
  restart: "python scripts/deploy_cpanel.py --restart-only"
```

#### Option C: Platform CLI

```bash
# Vercel: npm i -g vercel && vercel login
# Netlify: npm i -g netlify-cli && netlify login
# Railway: npm i -g @railway/cli && railway login
```

```yaml
deploy:
  frontend: "vercel deploy --prod"
```

#### Option D: CI/CD (GitHub Actions)

Create `.github/workflows/deploy.yml`, then:
```yaml
deploy:
  frontend: "git push origin main"  # push = deploy
```

#### Security reminder
- **Never commit credentials to git.** Use `.gitignore`.
- **Use minimum permissions.** Deploy key ≠ admin access.
- **Rotate tokens periodically.**
- Config stores **file paths** to credentials, not the credentials themselves.
</details>

## Installation

### 1. Install the skill (one-time, user-level)

```bash
mkdir -p ~/.claude/commands
cp orchestrate.md ~/.claude/commands/orchestrate.md
```

The skill is now available as `/orchestrate` in every Claude Code session, in every project.

### 2. Configure your project (per-project)

**Option A: Let the skill guide you (recommended)**

```
/orchestrate
```

The skill detects there's no config and asks 6 friendly questions (no technical jargon):
1. **"What's your project?"** — name + one sentence
2. **"What session types do you need?"** — explained as "work modes"
3. **"How do you publish?"** — deploy commands or "I don't know yet"
4. **"Where are your files?"** — tests, plans, docs (or "I don't have any yet")
5. **"What's your main branch?"** — main/master/develop/demo
6. **"Anything to verify on startup?"** — optional health checks

**Option B: Create config manually**

```yaml
# .claude/orchestrate.yml

project:
  name: "My Project"
  description: "What it does"

branches:
  base: "main"

spokes:
  qa:
    enabled: true
  journey:
    enabled: false  # Enable when you have personas
  feature:
    enabled: true
  dev:
    enabled: true
  deploy:
    enabled: false  # Enable when you need hosting setup

deploy:
  build: "npm run build"
  frontend: ""
  backend: ""
  restart: ""

hosting:
  credentials: ""  # e.g., "data/deploy_ssh.txt"
  type: ""         # "ssh", "cpanel", "vercel", etc.

verify:
  - "git branch --show-current"
  - "git log --oneline -3"

gotchas: []
roles: []
```

## Session Types

| Session | What it does | What it produces | Persistent across runs? |
|---|---|---|---|
| **Control** (Hub) | Decides, designs, coordinates, reviews | Instruction blocks + handoff | Yes — tracks all spoke states |
| **QA** (Spoke) | Hunts bugs module × role × case | Test matrix + bug backlog | Yes — compares findings across runs |
| **Journey** (Spoke) | Simulates prospects evaluating demo | Narrative + scores + verdict | Yes — compares scores across rounds |
| **Feature** (Spoke) | Implements new functionality E2E | Commits + tests + deploy | Yes — knows the codebase deeply |
| **Dev** (Spoke) | Quick fixes and improvements | Fix list + deploy | Yes — knows prior fixes |
| **Deploy** (Spoke) | Sets up hosting, pipelines, debug prod | Working pipeline + docs | Yes — knows the infra |

### When NOT to open a new spoke

- **Don't open a new QA session** if you have one that already ran QA for this project — reuse it, it has context
- **Don't open a new Feature session** for the same area of code — the existing one knows the codebase better
- **Do open a new session** only when starting a genuinely different type of work with no prior context

## Lifecycle: what happens when you return after a break

Every time you run `/orchestrate`, the skill diagnoses which state your project is in:

| State | Situation | What the skill does |
|---|---|---|
| **First time** | Config exists but no handoff | Builds initial state from git log |
| **Fresh start** | Recent handoff, everything aligned | Normal flow — loads and presents |
| **Immediate return** | Same day, session reopened | "Resuming where we left off" |
| **Return with gap** | Days/weeks without using the skill | Scans git for untracked changes, reconstructs state |
| **Major changes** | Stale handoff + many new commits | Full rebuild from repo evidence, asks user to confirm |
| **Obsolete config** | Paths or commands don't work anymore | Checks each config entry, offers to re-configure broken parts |

This diagnosis runs **automatically** — you don't need to tell the skill "I was away for 2 weeks." It figures it out from git timestamps.

## Re-validation cycle

After shipping a batch of features/fixes, the skill recommends re-validation before continuing with new work:

```
Implement features → Deploy → Session A (QA) → Fix bugs → Session B (Journey) → Confirm scores → Next batch
```

The Control Session shows a warning when changes have been shipped without re-validation: *"N changes shipped since last QA+Journey. Recommend re-validating before adding more features."*

## Mandatory handoff on close

When you close or pause a Control Session, the skill **always** updates `memory/handoff_next_session.md` with:
- What was done (commits, features, fixes)
- Validation state (QA+Journey ran or pending)
- Current scores (confirmed or estimated)
- Next recommended step
- Active spoke sessions

Rule: **if the next `/orchestrate` needs to ask "what happened?", the handoff failed.**

## Smart block generation

The Control Session adapts blocks based on whether a spoke is new or returning:

| Scenario | Block content |
|---|---|
| **New spoke** (never used) | Full context: project description, verification, task, files, rules |
| **Returning spoke** (same type) | Short diff: "You already did X. Since then Y changed. Re-run with focus on Z." |
| **Wrong spoke** (type mismatch) | Warning with consequences + recommendation to open correct session |

The Control **always asks** before generating: *"New session or the one that already ran [type]?"*

## Communication flow

```
CONTROL → SPOKE:  user copies instruction block (1 time per spoke)
SPOKE → CONTROL:  spoke writes memory/spoke_report_{type}.md
                   + commits work to its branch
                   user says "absorb reports" in Control
                   Control reads memory + git branch automatically
                   ZERO copy-paste on return
```

**After the first connection**, the spoke-to-control channel is automatic. You don't need to copy anything back — just say "absorb reports."

## Example: Real-world usage

Built during construction of **StudentHub** (pulsed.cl):

```
Day 1 — Control Session:
  /orchestrate → "Comunicaciones redesign is the priority"
  → Designed mockups with brainstorming skill
  → Wrote 13-task TDD plan with writing-plans skill
  → Implemented with subagent-driven-development
  → Deployed to production

Day 1 — QA Session (1st run):
  → Found 4 critical bugs, 4 important, 8 UX gaps
  → Fixed and redeployed

Day 1 — Journey Session (1st run):
  → 3 personas scored: Director 7/10, Tutor 7.5/10, Assistant 6.5/10
  → Identified 3 quick wins

Day 2 — Control Session:
  → Absorbed QA + Journey reports (zero copy-paste)
  → Implemented quick wins + 3 features inline
  → Score went from 7.0 → 8.8 average

Day 2 — Journey Session (2nd run, SAME session):
  → Re-validated with same personas
  → Scores: Director 9.0, Tutor 8.5, Assistant 8.5

Day 2 — Control Session:
  → Absorbed R2 Journey report
  → Implemented 5 more fixes (global view, templates, classifier)
  → Triggered re-validation cycle

Day 2 — Journey Session (3rd run, SAME session):
  → R3: Director 9.5, Tutor 9.0, Assistant 9.0
  → Average: 7.0 → 8.7 → 9.2 across 3 rounds
  → All 3 personas would sign a pilot contract
```

Total over 2 days: 35+ commits, 50+ tests, 3 validation rounds, automated deploy pipeline, product went from "interesting prototype" to "demo-ready" — all coordinated through `/orchestrate`.

## FAQ

**Q: Can sessions talk to each other directly?**
Not directly, but close. After the initial copy-paste (Control→Spoke), the return communication is automatic via shared memory files + git branches. You just say "absorb reports" in Control.

**Q: Do I need all spoke types?**
No. Start with `feature` + `dev`. Add `qa` when your product is mature. Add `journey` when you have prospects to validate with. Add `deploy` when you need hosting automation.

**Q: Can I run multiple spokes in parallel?**
Yes, if they don't modify the same files. QA + Journey can run in parallel (both read-only). Two Feature sessions on the same code will cause merge conflicts.

**Q: What if I paste the wrong block into a session?**
The block includes a `SESSION_TYPE` check. If it detects a type mismatch (e.g., QA session receives a Journey block), it shows detailed consequences and recommends opening the correct session instead.

**Q: Should I close spoke sessions between runs?**
No! Keep them open. A QA session that already ran once will be faster and more accurate on the second run because it has context from the first.

**Q: What if I come back after weeks without using the skill?**
The skill auto-detects this. It compares the handoff date with the latest git commits. If there's a gap, it scans the git log, reconstructs the project state, and asks you to confirm before proceeding. You don't need to explain what happened — it figures it out.

**Q: What if someone else worked on the project without using the skill?**
Same as above. The skill detects commits it didn't track and incorporates them. It might ask: "I see 15 commits I don't know about. Let me analyze them." Then it updates the handoff with the new state.

**Q: Does the Journey session track score breakdowns?**
Yes. The skill requires Journey reports to include per-persona breakdowns: what added points (+X for Y), what subtracted (-X for Z), and why the score is what it is. Without the breakdown, the Control can't prioritize what to fix next.

**Q: Does this work with any project?**
Yes. The skill is generic. The config makes it project-specific. Works for React, Python, mobile, static sites — anything with git.

---

<a name="espanol"></a>

# Skill Orchestrate (Espanol)

**Orquestador de Sesiones para Claude Code** — coordiná múltiples sesiones de Claude Code usando un patrón hub-and-spoke. Las sesiones son **persistentes y reutilizables** — acumulan contexto con el tiempo y mejoran con cada corrida.

## Qué es esto?

Cuando trabajás en un proyecto real con Claude Code, una sesión no alcanza. Necesitás:
- Una **Sesión de Control** para decidir, diseñar y coordinar
- Una **Sesión de QA** para buscar bugs sistemáticamente
- Una **Sesión de Prospect Journey** para validar con prospectos simulados
- Una **Sesión de Feature** para implementar funcionalidad nueva
- Una **Sesión de Dev** para fixes rápidos y mejoras
- Una **Sesión de Deploy** para configurar hosting y automatizar deploys

**Skill Orchestrate** resuelve la comunicación entre sesiones:
1. **Control → Spoke**: copiás un bloque de instrucciones (1 vez por spoke)
2. **Spoke → Control**: el spoke escribe su reporte en memory + commitea a su branch. El Control lo lee automáticamente. **Cero copy-paste de vuelta.**

```
Abrís Sesión de Control
  → /orchestrate
  → Control pregunta: "¿Sesión nueva o la que ya hizo QA?"
  → Genera bloque adaptado al contexto del spoke
  → Lo copiás

Abrís Sesión de QA (nueva o existente)
  → Pegás el bloque
  → QA trabaja solo
  → Al terminar escribe memory/spoke_report_qa.md
  → Te dice: "Listo. Decile al Control: absorbé los reportes."

Volvés al Control
  → Decís: "absorbé los reportes"
  → Control lee memory + branch automáticamente
  → Cero copy-paste
```

## Concepto clave: las sesiones son persistentes

**Los spokes NO son descartables.** Se reusan a lo largo del proyecto:

- **QA**: la primera corrida encuentra 8 bugs. La segunda (después de fixes) hace un re-scan focalizado. La tercera es un smoke test pre-launch. Cada vez más rápido y preciso.
- **Journey**: la primera corrida da scores 6.5-7.5. La segunda (post-fixes) re-valida y compara. La tercera prueba una persona nueva.
- **Feature**: la primera implementa auth. La segunda agrega OAuth. La tercera agrega MFA. Cada vez conoce mejor el código.

**No cierres tus sesiones spoke** — las vas a necesitar de nuevo.

## Skills complementarias recomendadas

Este skill coordina. Para el trabajo dentro de cada sesión, estas skills mejoran la calidad:

| Skill | Qué hace | Qué spokes beneficia |
|---|---|---|
| `superpowers:writing-plans` | Crea planes TDD paso a paso | Feature |
| `superpowers:subagent-driven-development` | Ejecuta planes con subagents + review | Feature |
| `superpowers:test-driven-development` | Test primero, código después | Feature + Dev |
| `superpowers:systematic-debugging` | Análisis de causa raíz antes de arreglar | QA + Dev |
| `superpowers:brainstorming` | Explora requerimientos antes de implementar | Control |
| `frontend-design:frontend-design` | UI de alta calidad, no genérica | Feature + Dev |

## Requisitos

| Dependencia | Para qué | Cómo verificar |
|---|---|---|
| **Claude Code** | Donde corre el skill | `claude --version` |
| **Git** | Comunicación entre sesiones | `git --version` |
| **Repo Git** | Tu proyecto debe ser un repo | `git status` |
| **Bash** | Comandos de verificación y deploy | `bash --version` |

**Opcionales**: GitHub CLI (`gh`), acceso al hosting (SSH/cPanel/Vercel), Node.js/Python según tu stack.

<details>
<summary><strong>Cómo instalar GitHub CLI</strong></summary>

```bash
# Windows: winget install --id GitHub.cli
# macOS: brew install gh
# Linux: sudo apt install gh

gh auth login  # seguir el flujo interactivo
gh auth status  # verificar
```
</details>

<details>
<summary><strong>Cómo configurar acceso al hosting</strong></summary>

4 opciones — elegí la que matchee tu hosting:

| Opción | Para quién | Comando de deploy |
|---|---|---|
| **SSH** | VPS, dedicados | `rsync` + `ssh restart` |
| **cPanel API** | Shared hosting sin SSH | Script Python con UAPI |
| **Platform CLI** | Vercel/Netlify/Railway | `vercel deploy --prod` |
| **CI/CD** | GitHub Actions | `git push` = auto deploy |

Credenciales siempre en archivos gitignored. El config guarda el **path** al archivo, no las credenciales.
</details>

## Instalación

```bash
mkdir -p ~/.claude/commands
cp orchestrate.md ~/.claude/commands/orchestrate.md
```

Configurar por proyecto: corré `/orchestrate` y respondé 6 preguntas.

## Tipos de sesión

| Sesión | Qué hace | Persistente? |
|---|---|---|
| **Control** (Hub) | Decide, diseña, coordina, revisa | Sí — trackea todos los spokes |
| **QA** (Spoke) | Busca bugs módulo × rol × caso | Sí — compara findings entre corridas |
| **Journey** (Spoke) | Simula prospectos evaluando la demo | Sí — compara scores entre rondas |
| **Feature** (Spoke) | Implementa algo nuevo E2E | Sí — conoce el código profundamente |
| **Dev** (Spoke) | Fixes rápidos y mejoras | Sí — conoce fixes previos |
| **Deploy** (Spoke) | Setup hosting, pipeline, debug prod | Sí — conoce la infra |

## Ciclo de vida: qué pasa cuando volvés después de un tiempo

Cada vez que corrés `/orchestrate`, el skill diagnostica en qué estado está tu proyecto:

| Estado | Situación | Qué hace el skill |
|---|---|---|
| **Primera vez** | Config sin handoff | Construye estado desde git log |
| **Arranque fresco** | Handoff reciente, todo alineado | Flujo normal |
| **Retorno inmediato** | Mismo día | "Retomamos donde quedamos" |
| **Retorno con gap** | Días/semanas sin el skill | Escanea git, reconstruye estado |
| **Cambios grandes** | Handoff viejo + muchos commits | Rebuild completo desde el repo |
| **Config obsoleta** | Paths/comandos no funcionan | Verifica cada entrada, ofrece re-configurar |

El diagnóstico corre **automáticamente** — no necesitás decirle "estuve 2 semanas afuera".

## Ciclo de re-validación

Después de shipear un batch de features/fixes, el skill recomienda re-validar:

```
Implementar → Deploy → Session A (QA) → Fix bugs → Session B (Journey) → Confirmar scores → Siguiente batch
```

El Control muestra advertencia si hay cambios sin validar.

## Handoff obligatorio al cerrar

Al cerrar o pausar una Control Session, el skill **siempre** actualiza el handoff con: qué se hizo, estado de validación, scores, próximo paso, spokes activos.

Regla: **si el próximo `/orchestrate` necesita preguntar "qué pasó?", el handoff falló.**

## Generación inteligente de bloques

| Escenario | Qué genera el Control |
|---|---|
| **Spoke nuevo** | Bloque completo con todo el contexto |
| **Spoke que ya corrió** | Bloque corto: "ya hiciste X, desde entonces cambió Y, re-corré con foco en Z" |
| **Spoke equivocado** | Advertencia con consecuencias + recomendación de abrir sesión correcta |

El Control **siempre pregunta**: "¿sesión nueva o la que ya trabajó?"

## Comunicación

Después de la primera conexión (copy-paste del bloque), el retorno es automático vía memory + git. Solo decís "absorbé los reportes".

## Preguntas frecuentes

**P: Las sesiones hablan entre sí?**
Casi. Después del copy-paste inicial, el retorno es automático vía memory files compartidos + branches de git. Solo decís "absorbé los reportes" en el Control.

**P: Necesito todos los tipos de spoke?**
No. Empezá con `feature` + `dev`. Agregá `qa` cuando tu producto madure. Agregá `journey` cuando tengas prospectos.

**P: Qué pasa si pego el bloque equivocado?**
El bloque tiene un check `SESSION_TYPE`. Si detecta mismatch, muestra las consecuencias detalladas (commits mezclados, reportes sobreescritos, handoff inconsistente) y recomienda abrir la sesión correcta.

**P: Debo cerrar los spokes entre corridas?**
No. Mantené las sesiones abiertas. Un QA que ya corrió una vez será más rápido y preciso la segunda vez porque tiene contexto.

**P: Qué pasa si vuelvo después de semanas sin usar el skill?**
El skill lo detecta solo. Compara la fecha del handoff con los commits recientes. Si hay gap, escanea el git log, reconstruye el estado, y te pide confirmar antes de seguir.

**P: Qué pasa si otra persona trabajó en el proyecto sin el skill?**
Lo mismo. El skill detecta commits que no trackeó y los incorpora. Puede preguntar: "Veo 15 commits que no conozco. Los analizo."

**P: El Journey trackea desglose de scores?**
Sí. El skill exige que los reportes de Journey incluyan desglose por persona: qué sumó (+X por Y), qué restó (-X por Z), y por qué ese score. Sin desglose, el Control no puede priorizar qué arreglar.

**P: Funciona con cualquier proyecto?**
Sí. El skill es genérico. La config lo adapta. Funciona con React, Python, mobile, sitios estáticos — cualquier cosa con git.

---

## License

MIT

## Credits

Developed during the construction of [StudentHub](https://pulsed.cl) — a student tracking platform for Chilean educational organizations (OTECs).
