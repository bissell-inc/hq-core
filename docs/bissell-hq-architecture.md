# BISSELL HQ Architecture

Self-hosted version of Indigo HQ using BISSELL's existing AWS infrastructure.

---

## Architecture Overview

```mermaid
flowchart TD
    subgraph AWS["BISSELL AWS Account — Shared Services"]
        CUP["Cognito User Pool\n(existing)"] 
        AD["Microsoft AD SSO\n(existing)"]
        IP["Cognito Identity Pool\nIdentityPoolStack"]
        ADG["NEW AD Groups\nbissell-hq-admin\nbissell-hq-user"]
        S3["S3: bis-hq-vault-{env}\nNEW bucket\n─────────────────\ncompanies/bissell/\n  knowledge/  rw:admin\n  policies/   rw:admin\n  workers/    ro:user\n  skills/     ro:user\n  projects/"]
    end

    AD --> CUP
    ADG --> IP
    CUP --> IP
    IP -->|"temp STS credentials"| S3

    S3 -->|"bissell-hq-sync-runner\nreplaces @indigoai-us/hq-cloud"| SYNC

    SYNC[ ]
    style SYNC fill:none,stroke:none

    SYNC --> A
    SYNC --> B
    SYNC --> C

    subgraph A["Balu — admin"]
        A1["~/bissell-hq\n~/.claude/\npush + pull"]
    end
    subgraph B["Dev B — user"]
        B1["~/bissell-hq\n~/.claude/\npull only"]
    end
    subgraph C["Dev C — user"]
        C1["~/bissell-hq\n~/.claude/\npull only"]
    end
```

---

## AD Group → IAM Role Mapping

| AD Group | IAM Role | S3 Access | Who |
|----------|----------|-----------|-----|
| `bissell-hq-admin` | `adminRole` | read + write all `companies/bissell/*` | Balu, tech leads |
| `bissell-hq-user` | `userRole` | read only `companies/bissell/*` | All other devs |

Admins curate knowledge/policies → push to S3 → all users pull automatically.

---

## Repo Structure (`bissell-inc/bissell-hq-core`)

```mermaid
flowchart LR
    subgraph REPO["bissell-inc/bissell-hq-core"]
        direction TB
        subgraph ENGINE[".claude/  —  local only"]
            CMD["commands/\n53 slash commands"]
            HK["hooks/\nlifecycle hooks"]
            SK["skills/\nskill definitions"]
        end
        subgraph BISSELL["companies/bissell/  —  synced via S3"]
            KN["knowledge/\nnestjs-patterns.md\napi-design-standards.md\nair-freight-domain.md"]
            PO["policies/\nbranching-strategy.md\ntesting-standards.md"]
            WK["workers/\nbissell-backend-dev/"]
        end
        SR["scripts/\nbissell-hq-sync-runner.js"]
        CORE["core/\nbundled HQ workers & knowledge\n(local only)"]
    end
```

---

## What Syncs vs. What Stays Local

| Location | Synced to S3 | Description |
|----------|-------------|-------------|
| `companies/bissell/knowledge/` | ✅ Yes | BISSELL docs, domain knowledge |
| `companies/bissell/policies/` | ✅ Yes | Coding standards, governance rules |
| `companies/bissell/workers/` | ✅ Yes | Custom BISSELL AI agents |
| `companies/bissell/skills/` | ✅ Yes | Custom BISSELL skills |
| `companies/bissell/projects/` | ✅ Yes | PRDs, project state |
| `.claude/commands/` | ❌ No | HQ engine — updated via `git pull` |
| `.claude/hooks/` | ❌ No | Lifecycle hooks — local only |
| `core/workers/` | ❌ No | Bundled HQ workers — local only |
| `workspace/threads/` | ❌ No | Personal session state |
| `personal/` | ❌ No | Personal notes |

---

## CDK Changes Required (`feature/bissell-api-mcp`)

**`infrastructure/shared-cdk/config/dev-1/cdk.config.ts`** — update config:
```typescript
identityPoolName: 'bis-hq',
adminGroup: 'bissell-hq-admin',   // new AD group
userGroup: 'bissell-hq-user',     // new AD group
s3BucketName: 'bis-hq-vault',     // new dedicated bucket
```

Same pattern for `uat` and `prod` configs.

The `IdentityPoolStack` CDK code already supports these as config properties — no structural changes needed.

---

## Developer Daily Workflow

### First-time setup (once per machine)
```bash
git clone https://github.com/bissell-inc/bissell-hq-core ~/bissell-hq
cp -r ~/bissell-hq/.claude/* ~/.claude/
```

### Start of day
```bash
hq-login   # browser opens → Microsoft AD login → closes
/hq-sync   # pulls latest company knowledge from S3
```

```mermaid
sequenceDiagram
    participant Dev
    participant Browser
    participant Cognito as Cognito (BISSELL)
    participant AD as Microsoft AD
    participant S3 as S3 Vault

    Dev->>Browser: hq-login
    Browser->>Cognito: OAuth2 PKCE
    Cognito->>AD: SSO
    AD-->>Cognito: token
    Cognito-->>Dev: ~/.hq/cognito-tokens.json
    Dev->>S3: /hq-sync (pull)
    S3-->>Dev: knowledge/ policies/ workers/
    Note over Dev: Claude Code now has full BISSELL context
```

### Working in any repo
```bash
cd ~/code/bissell/bissell-air-freight-service
claude     # open Claude Code — full BISSELL context available

/run backend-dev implement-feature
/quality-gate
/prd "add carrier quote comparison"
/review
```

### When you add/update knowledge or policies
```bash
# Edit files in ~/bissell-hq/companies/bissell/knowledge/ or policies/
/hq-sync   # pushes changes to S3
# All other devs get it on their next /hq-sync
```

```mermaid
sequenceDiagram
    participant Admin as Balu (admin)
    participant S3 as S3 Vault
    participant DevB as Dev B
    participant DevC as Dev C

    Admin->>Admin: adds nestjs-tracing.md to policies/
    Admin->>S3: /hq-sync (push)
    Note over S3: policy stored in S3
    DevB->>S3: /hq-sync (pull) — next morning
    S3-->>DevB: nestjs-tracing.md
    DevC->>S3: /hq-sync (pull) — next morning
    S3-->>DevC: nestjs-tracing.md
    Note over DevB,DevC: Claude Code now enforces tracing policy automatically
```

---

## Enterprise AI Governance

| Concern | How It's Handled |
|---------|-----------------|
| Consistent coding standards | `policies/` files injected into every Claude Code session |
| Change control on standards | PRs to `bissell-inc/bissell-hq-core` — reviewed like any code change |
| Audit trail | Git history on the fork — every policy change is a commit |
| Access control | AD groups → Cognito → IAM roles → S3 prefix permissions |
| New hire onboarding | Clone repo, copy `.claude/`, run `/hq-sync` — full BISSELL context on day 1 |
| Secrets | Never in knowledge base — stays in AWS Secrets Manager |
| Data residency | All data stays in BISSELL's own AWS account — nothing goes to Indigo |

---

## Implementation Tasks

| Task | Depends On | Effort |
|------|-----------|--------|
| Create AD groups (`bissell-hq-admin`, `bissell-hq-user`) in Azure AD | Dan/IT | ~1 hr |
| Update CDK config with new AD groups + bucket name | AD groups created | ~2 hrs |
| Merge `feature/bissell-api-mcp` and deploy to dev-1 | CDK config updated | ~2 hrs |
| Write `bissell-hq-sync-runner.js` | S3 bucket deployed | ~1-2 days |
| Fork hq-core → `bissell-inc/bissell-hq-core`, patch sync + login commands | Sync runner done | ~2 hrs |
| Populate initial BISSELL knowledge base | Fork ready | ~4-8 hrs |
| End-to-end test with 2-3 devs | All above done | ~4 hrs |

**Total: ~3-4 days of engineering work.**

---

## Key Decision: Self-Hosted vs. Indigo Commercial

| | Self-Hosted (this doc) | Indigo Commercial |
|--|----------------------|-------------------|
| Data residency | BISSELL AWS account only | Indigo's AWS account |
| Auth | BISSELL Cognito + AD | Indigo's Cognito pool |
| Cost | AWS S3 + infra (~$10-20/mo) | Per-seat SaaS pricing (unknown) |
| Control | Full — you own everything | Vendor dependency |
| Maintenance | You maintain sync runner | Indigo maintains |
| Enterprise security | BISSELL's existing controls | Indigo's (unverified) |
