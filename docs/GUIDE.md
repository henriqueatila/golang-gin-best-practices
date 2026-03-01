# Guia: Como usar com Claude Code

## Setup (faça uma vez)

### 1. Crie o repositório local

```bash
mkdir gin-skills
cd gin-skills
git init
```

### 2. Copie os arquivos de setup

Coloque estes 2 arquivos na raiz do repositório:

```
gin-skills/
├── CLAUDE.md          ← instruções do agente (Bloco 1)
└── SPECIFICATION.md   ← especificação detalhada do projeto
```

### 3. Crie o GIN-API-REFERENCE.md primeiro

Antes de iniciar as fases, você precisa do arquivo de referência da API do Gin.
Use este comando como Fase 0:

```
Read the official Gin documentation at https://gin-gonic.com/en/docs/ and the
gin-gonic/gin GitHub repo docs. Create GIN-API-REFERENCE.md with the complete
verified API surface: Engine methods, Context methods (request binding, response,
flow control, metadata), RouterGroup methods, binding tags, middleware signatures,
and the official graceful shutdown pattern. This file is the single source of
truth — every Gin API call in the repository will be verified against it.
```

---

## Execução por fases

Rode cada fase como um comando separado no Claude Code.
Espere completar uma fase antes de iniciar a próxima.

### Fase 1 — Core API (5 arquivos)

```
Read SPECIFICATION.md and GIN-API-REFERENCE.md. Create these files for Phase 1:
1. LICENSE (MIT)
2. gin-api/SKILL.md
3. gin-api/references/error-handling.md
4. gin-api/references/routing.md
5. gin-api/references/middleware.md

No dependencies from previous phases. After each file, verify it passes the
relevant quality checklist items from SPECIFICATION.md. Report progress at the end.
```

### Fase 2 — Database (4 arquivos)

```
Read SPECIFICATION.md and GIN-API-REFERENCE.md. Files already created: LICENSE,
gin-api/SKILL.md, gin-api/references/*. Create these files for Phase 2:
1. gin-database/SKILL.md
2. gin-database/references/gorm-patterns.md
3. gin-database/references/sqlx-patterns.md
4. gin-database/references/migrations.md

Use the same User domain model and AppError pattern from gin-api. Verify quality
checklist items after each file. Report progress.
```

### Fase 3 — Auth (3 arquivos)

```
Read SPECIFICATION.md and GIN-API-REFERENCE.md. Files already created: LICENSE,
gin-api/*, gin-database/*. Create these files for Phase 3:
1. gin-auth/SKILL.md
2. gin-auth/references/jwt-patterns.md
3. gin-auth/references/rbac.md

Reference gin-api handler patterns and gin-database user repository. Verify
quality checklist. Report progress.
```

### Fase 4 — Testing (4 arquivos)

```
Read SPECIFICATION.md and GIN-API-REFERENCE.md. Files already created: LICENSE,
gin-api/*, gin-database/*, gin-auth/*. Create these files for Phase 4:
1. gin-testing/SKILL.md
2. gin-testing/references/unit-tests.md
3. gin-testing/references/integration-tests.md
4. gin-testing/references/e2e.md

Test examples should test the handlers/services from gin-api and repositories
from gin-database. Verify quality checklist. Report progress.
```

### Fase 5 — Deploy (4 arquivos)

```
Read SPECIFICATION.md and GIN-API-REFERENCE.md. Files already created: LICENSE,
gin-api/*, gin-database/*, gin-auth/*, gin-testing/*. Create these files for Phase 5:
1. gin-deploy/SKILL.md
2. gin-deploy/references/dockerfile.md
3. gin-deploy/references/docker-compose.md
4. gin-deploy/references/kubernetes.md

The Dockerfile should build the same project structure from gin-api. Include
migration running from gin-database. Verify quality checklist. Report progress.
```

### Fase 6 — README + Verificação Final (1 arquivo + checklist)

```
Read SPECIFICATION.md. All 21 other files are created. Create README.md following
the README.md Specifications section in SPECIFICATION.md.

After creating README.md, run the FULL quality checklist from SPECIFICATION.md:
- Per SKILL.md checks (all 5)
- Per reference file checks (all 14)
- Cross-skill consistency checks
- Repository-level checks

Report any issues found. List all 22 files with their line counts.
```

---

## Dicas

- Se o Claude Code perder contexto, repita: "Read CLAUDE.md, SPECIFICATION.md, and GIN-API-REFERENCE.md"
- Se um arquivo ficar com problema, peça revisão isolada: "Read gin-api/SKILL.md and verify against the quality checklist in SPECIFICATION.md. Fix any issues."
- Commit após cada fase para ter rollback seguro
