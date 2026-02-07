# Agent Development Rules

Centralized development rules for AI coding agents (Windsurf, GitHub Copilot, Cursor, Claude Code, Codex CLI, Gemini CLI, Antigravity, OpenCode).

## Structure

```
agent-development-rules/
├── manifest.yaml              # Central configuration (v2.0)
│
├── content/                   # All source content
│   ├── rules/                 # Development rules
│   │   ├── core/              # Fundamental rules
│   │   │   ├── global.md      #   Agent behavior & persona
│   │   │   ├── clean-code.md  #   CUPID, DRY, KISS, YAGNI
│   │   │   └── solid.md       #   SOLID principles
│   │   ├── languages/         # Language-specific
│   │   │   ├── go.md          #   Go idioms & tooling
│   │   │   ├── python.md      #   Type hints, Ruff, Pydantic
│   │   │   ├── nodejs.md      #   TypeScript strict, ESLint flat
│   │   │   └── bash.md        #   Shell scripting & ShellCheck
│   │   ├── infrastructure/    # IaC & DevOps
│   │   │   ├── terraform.md   #   TFLint, Checkov, tfsec
│   │   │   ├── aws.md         #   IAM, S3, Security Groups
│   │   │   ├── docker.md      #   Multi-stage, Hadolint
│   │   │   └── kubernetes.md  #   Security, probes, HPA
│   │   ├── quality/           # Code quality
│   │   │   ├── testing.md     #   TDD, pyramid, coverage
│   │   │   ├── security.md    #   OWASP Top 10:2025
│   │   │   ├── performance.md #   Profiling, caching, Big O
│   │   │   ├── scalability.md #   Stateless, resilience
│   │   │   ├── linting.md     #   Golden chain, all linters
│   │   │   └── accessibility.md # WCAG 2.1, ARIA
│   │   ├── process/           # Development processes
│   │   │   ├── git.md         #   Conventional Commits (strict)
│   │   │   ├── documentation.md # ADRs, READMEs, docstrings
│   │   │   └── operational-excellence.md # SRE, observability
│   │   └── agents/            # Agent-specific behavior
│   │       ├── windsurf.md    #   Windsurf/Cascade rules
│   │       └── copilot.md     #   GitHub Copilot rules
│   │
│   ├── workflows/             # Workflows (slash commands)
│   │   ├── validation/        #   validate, test, lint
│   │   ├── git/               #   pre-push, pr-review, fix-pr-comments
│   │   ├── infrastructure/    #   terraform-module, docker-build, k8s-validate
│   │   └── security/          #   security-check
│   │
│   └── prompts/               # Reusable prompts
│       ├── review.prompt.md   #   Code review
│       ├── security.prompt.md #   Security audit
│       ├── test.prompt.md     #   Test generation
│       ├── refactor.prompt.md #   Refactoring
│       ├── document.prompt.md #   Documentation
│       ├── validate.prompt.md #   Project validation
│       ├── commit.prompt.md   #   Conventional Commits
│       ├── explain.prompt.md  #   Code explanation
│       └── debug.prompt.md    #   Debugging assistant
│
├── templates/                 # Reusable document templates
│   ├── pull-request.md        #   PR description template
│   ├── trd.md                 #   Technical Requirements Document
│   ├── prd.md                 #   Product Requirements Document
│   ├── readme.md              #   Project README template
│   └── architecture/          #   Architecture decision templates
│       ├── ad.md              #     Architecture Decision
│       ├── adl.md             #     Architecture Decision Log
│       ├── adr.md             #     Architecture Decision Record
│       ├── akm.md             #     Architecture Knowledge Management
│       └── asr.md             #     Architecturally-Significant Requirement
│
├── config/                    # Configuration examples
│   └── .pre-commit-config.yaml.example
│
└── scripts/                   # Automation
    ├── sync.sh                #   Sync rules to agents
    ├── validate.sh            #   Validate configuration
    └── lib/
        ├── common.sh          #   Shared functions
        └── agents/            #   Agent-specific sync logic
            ├── windsurf.sh
            ├── copilot.sh
            └── cursor.sh
```

## Usage

### Sync Rules

```bash
# Sync to all enabled agents
./scripts/sync.sh

# Sync only one agent
./scripts/sync.sh --agent windsurf
./scripts/sync.sh --agent copilot-vscode
./scripts/sync.sh --agent cursor

# Dry run (see what would happen)
./scripts/sync.sh --dry-run

# Validate configuration before syncing
./scripts/sync.sh --validate

# List available agents and their status
./scripts/sync.sh --list

# Enable debug output
./scripts/sync.sh --debug
```

### Validate Configuration

```bash
# Check manifest, files, and frontmatter
./scripts/validate.sh
```

### Requirements

```bash
# Install yq (YAML parser)
brew install yq
```

## Configuration

### manifest.yaml

The `manifest.yaml` file (v2.0) defines:
- **content_dir**: Base directory for all content (`content/`).
- **Source Files**: Lists of rules, workflows, and prompts with paths relative to `content/`.
- **Agents**: Destination configuration per agent with auto-detection.

```yaml
# Agent configuration example
agents:
  windsurf:
    enabled: true
    description: "Windsurf IDE / Codeium Cascade"
    detect:
      paths:
        - "${HOME}/.codeium"
    targets:
      rules:
        path: "${HOME}/.codeium/memories/global_rules.md"
        format: merged
        strip_frontmatter: true
```

### Adding a New Agent

1. Add a section in `manifest.yaml` under `agents:`.
2. Configure `detect` with paths/commands for auto-detection.
3. Configure `targets` with destination paths and format (`merged` or `individual`).
4. Run `./scripts/sync.sh --agent <name>`.

### Adding a New Rule

1. Create a file in `content/rules/<category>/<name>.md` with frontmatter.
2. Add it to `manifest.yaml` under `rules.files` (path relative to `content/rules/`).
3. Run `./scripts/sync.sh`.

## Rule Content

### Core
- **global.md** - Agent behavior, persona, validation chain, zero trust
- **clean-code.md** - CUPID principles, DRY, KISS, YAGNI, code smells
- **solid.md** - SOLID with Python/Go examples, refactoring triggers

### Languages
- **go.md** - Error handling, golangci-lint, testing, security
- **python.md** - Type hints, Ruff, Pydantic, Pytest, Bandit
- **nodejs.md** - TypeScript strict, ESLint flat config, Vitest, Zod
- **bash.md** - `set -euo pipefail`, shfmt, ShellCheck

### Infrastructure
- **terraform.md** - TFLint, Checkov, tfsec, pre-commit, CI/CD
- **aws.md** - IAM least privilege, Security Groups, S3, secrets
- **docker.md** - Multi-stage builds, non-root, Hadolint, secrets
- **kubernetes.md** - SecurityContext, probes, NetworkPolicy, HPA

### Quality
- **testing.md** - TDD, test pyramid, coverage, AAA, test doubles
- **security.md** - OWASP Top 10:2025, input validation, cryptography
- **performance.md** - Profiling, Big O, N+1, caching, memory
- **scalability.md** - Stateless, connection pooling, resilience patterns
- **linting.md** - Golden chain, linters by language, pre-commit, CI/CD
- **accessibility.md** - WCAG 2.1 POUR, semantic HTML, ARIA

### Process
- **git.md** - Conventional Commits (strict), branching, PRs
- **documentation.md** - ADRs, README standard, docstrings, Mermaid
- **operational-excellence.md** - Observability, alerting, SLI/SLO, incidents

### Agents
- **windsurf.md** - Windsurf/Cascade specific behavior
- **copilot.md** - GitHub Copilot specific behavior

## Quality Gates

### Linting & Formatting

All code **MUST** pass linting before commit/PR:

| Language | Tools | Configuration |
|----------|-------|---------------|
| **Go** | `golangci-lint v2`, `go fmt`, `go vet` | `.golangci.yml` |
| **Python** | `ruff` (format + lint), `mypy`, `bandit` | `pyproject.toml` |
| **TypeScript** | `prettier`, `eslint` (flat config), `tsc` | `eslint.config.js`, `tsconfig.json` |
| **Terraform** | `terraform fmt`, `tflint`, `checkov`, `tfsec` | `.tflint.hcl` |
| **Bash** | `shfmt`, `shellcheck` | `.shellcheckrc` |
| **Docker** | `hadolint`, `docker scout` | `.hadolint.yaml` |
| **Kubernetes** | `kubeconform`, `helm lint`, `kube-linter` | - |

See `content/rules/quality/linting.md` for detailed configuration.

### Testing & Coverage

Minimum coverage requirements:

| Code Type | Coverage Target |
|-----------|----------------|
| **Critical Logic** | 90% |
| **Public API** | 80% |
| **Overall** | 70% |

**Commands**:
- Go: `go test -race -cover ./...`
- Python: `pytest --cov=src --cov-fail-under=70`
- Node/TS: `vitest run --coverage`

See `content/workflows/validation/test.md` for detailed testing guidelines.

### Pre-commit Hooks

Use `config/.pre-commit-config.yaml.example` to catch issues before commit:

```bash
pip install pre-commit
cp config/.pre-commit-config.yaml.example .pre-commit-config.yaml
pre-commit install
```

## Supported Agents

| Agent | Status | Description |
|-------|--------|-------------|
| **windsurf** | ✅ Enabled | Windsurf IDE / Codeium Cascade |
| **copilot-cli** | ✅ Enabled | GitHub Copilot CLI (`gh copilot`) |
| **copilot-vscode** | ✅ Enabled | GitHub Copilot (VS Code) |
| **copilot-intellij** | ✅ Enabled | GitHub Copilot (IntelliJ/JetBrains) |
| **cursor** | ✅ Enabled | Cursor IDE |
| **claude-code** | ✅ Enabled | Claude Code (Anthropic) |
| **codex-cli** | ✅ Enabled | OpenAI Codex CLI |
| **gemini-cli** | ✅ Enabled | Google Gemini CLI |
| **antigravity** | ✅ Enabled | Google Antigravity IDE |
| **opencode** | ✅ Enabled | OpenCode |
| **local-project** | ⬜ Disabled | Copy rules to local project |

## Development

### Rule Structure

Rules use YAML frontmatter for trigger configuration:

```markdown
---
trigger: glob
globs: ["*.py", "*.go"]
---

# Rule Title

## Section 1

Content...

## Section 2

Content...
```

### Workflow Structure

```markdown
---
description: Workflow description
---

# /workflow-name

## Steps

1. Step 1
2. Step 2
```

### Prompt Structure

```markdown
---
name: Prompt Name
description: What this prompt does
trigger: manual
tags:
  - category
---

# Prompt Name

Instructions...
```

## Scripts

### `sync.sh`

Reads `manifest.yaml` and syncs rules, workflows, and prompts to each agent's configured destination. Supports auto-detection of installed agents, dry-run mode, backups, and per-agent sync.

### `validate.sh`

Validates the entire configuration: manifest syntax, file references, content directory structure, and frontmatter consistency.

### `lib/`

- **common.sh** - Shared logging, file operations, frontmatter extraction.
- **agents/** - Agent-specific sync logic (`windsurf.sh`, `copilot.sh`, `cursor.sh`).

## License

MIT
