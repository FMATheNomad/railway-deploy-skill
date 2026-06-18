# Railway Deploy Skill

**Autonomous Railway deployment diagnosis & fix for AI coding agents.**

An [OpenCode](https://opencode.ai) / [Claude Code](https://claude.ai/code) skill that automatically diagnoses and fixes Railway deployment failures for **FastAPI + Next.js + PostgreSQL** projects.

## 🚂 What It Does

When your Railway deploy crashes, the agent automatically:

1. **Collects evidence** — fetches logs, checks env vars, reads your config files
2. **Classifies the error** — matches against 8 known failure patterns
3. **Applies the fix** — edits config files, sets env vars, or guides you
4. **Verifies** — checks logs after redeploy

## 🔍 Diagnosed Patterns

| Pattern | Error Signature |
|---------|----------------|
| Build OOM | `Killed`, `exit 137`, `Out of memory` |
| No Start Command | `No start command could be found` |
| Port Binding | `port already in use`, `service unavailable` |
| Database Connection | `Connection refused`, `ECONNREFUSED` |
| Healthcheck Failure | `failed with service unavailable` |
| Python Dependencies | `ModuleNotFoundError`, `pip install fails` |
| Auto-Deploy Failure | Deploy triggers on push but fails |
| Monorepo Issues | Wrong service deploys, missing files |

## 📦 Installation

### For OpenCode

Copy the skill to your OpenCode skills directory:

```bash
cp -r skills/railway-deploy ~/.config/opencode/skills/
```

Or install via npx skills:

```bash
npx skills add FMATheNomad/railway-deploy-skill@railway-deploy -g -y
```

### Prerequisites

- [Railway CLI](https://docs.railway.app/cli) installed & authenticated (`railway login`)
- Your project linked via `railway link`

## 🚀 Usage

Start a new OpenCode session and say:

> "Railway deploy saya crash, diagnosa pakai railway-deploy skill"

Or be more specific:

> "Build gagal dengan exit code 137, cek memory issue"
> "Auto-deploy error setelah git push"
> "FastAPI app deploy sukses tapi healthcheck gagal"

The agent will autonomously:

```
railway logs --lines 100
railway logs --build --lines 50
railway variable list --kv
railway status
→ read package.json, railway.json, Dockerfile, next.config.ts
→ match error with pattern
→ suggest + apply fix
→ verify with railway up
```

## 🛠 Supported Stack

| Technology | Notes |
|------------|-------|
| **Next.js** | Requires `output: "standalone"` |
| **FastAPI** | Tested with uvicorn & hypercorn |
| **PostgreSQL** | Railway plugin & external |
| **Docker** | Multi-stage build guidance |
| **Railpack** | Default Railway builder |
| **Python** | pip, requirements.txt |
| **TypeScript/JS** | npm, yarn, pnpm |

## 📁 Project Structure

```
railway-deploy-skill/
├── skills/
│   └── railway-deploy/
│       └── SKILL.md          # The skill definition
├── README.md
├── LICENSE
└── .gitignore
```

## 📚 References

- [Railway Documentation](https://docs.railway.app)
- [Railway CLI Reference](https://docs.railway.app/cli)
- [Railway Config as Code](https://docs.railway.app/config-as-code)
- [OpenCode Skills](https://opencode.ai/docs/skills)

## 🤝 Contributing

Found a missing pattern? Open an issue or PR!

## 📄 License

MIT © [FMA Software Labs](https://fmasoftwarelabs.up.railway.app)
