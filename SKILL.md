---
name: skill-releaser
description: Release skills to ClawhHub through the full publication pipeline — auto-scaffolding, OPSEC scan, dual review (agent + user), force-push release, security scan verification. Use when releasing a skill, preparing a skill for release, reviewing a skill for publication, or checking release readiness.
version: 1.4.0
triggers:
  - release skill
  - publish skill
  - prepare for clawhub
  - release readiness
  - skill release review
  - publish to clawhub
---

# Skill Releaser

Orchestrates the full skill publication pipeline from internal repo to ClawhHub.

## When to Use

- User says "release {skill}" or "publish {skill} to clawhub"
- User says "prepare {skill} for release" or "check release readiness"
- User says "review {skill} for publication"
- Cron-triggered release check during refactory pipeline

## Assumptions

**How OpenClaw and user interact during release:**
- Agent runs on a machine with shell access (exec tool) for git and CLI operations
- User communicates via messaging channel (Telegram, Discord, Signal, etc.) — likely on a phone
- User reviews the private GitHub repo directly in their browser/phone — the repo IS the review artifact, not a text summary
- User approves or rejects by replying to the agent's message (natural language: "approve", "revise: fix the readme", "reject")
- Agent can create and manage GitHub repos via `gh` CLI on behalf of the user's authenticated account
- Agent pushes to the private staging repo BEFORE requesting user review, so there is something to review
- Agent does NOT publish anything publicly without explicit user approval — this is a hard gate
- The repo starts private for staging and review. At release time, history is erased via orphan branch + force push (single clean commit), then flipped to public
- The full release can span multiple sessions — the private staging repo preserves state so any agent can resume
- Multiple skills can be in different stages of the pipeline simultaneously

## Prerequisites

- `gh` CLI authenticated (for repo creation and visibility changes)
- `clawhub` CLI installed (for ClawhHub publishing)
- A skill directory with at least a `SKILL.md` file

## Scope & Boundaries

**This skill handles:** The full release pipeline — structure scaffolding, OPSEC scanning, review, publishing.
**This skill does NOT handle:** Skill content creation or design. The SKILL.md must already describe what the skill does. Everything else (boilerplate, structure, scaffolding) is this pipeline's job.

A user with a finished SKILL.md should be able to say "release this skill" and this skill handles everything from there — including generating all missing structure files.

## Automation Model

The pipeline has two fully automated phases separated by one human gate:

```
Phase 1 (AUTO): Steps 1-7 — scaffold, validate, stage, scan, review, push
     ↓
  GATE: User reviews private repo, replies "approve" / "revise" / "reject"
     ↓
Phase 2 (AUTO): Steps 9-12 — erase history, flip public, publish, verify scan, deliver
```

**Design principles:**
- User says "release {skill}" once. Agent runs Phase 1 end-to-end without interruption.
- Agent sends ONE message: the review link + recommendation. Then waits.
- User replies with one word. Agent runs Phase 2 end-to-end without interruption.
- If any step fails, agent fixes it automatically and continues. Only report to user if unfixable.
- Rate limits, retries, and delays are handled silently (sleep + retry, not "rate limited, should I try again?")

**Anti-patterns (never do these):**
- Do not ask "should I create the repo?" — just create it
- Do not ask "should I run OPSEC scan?" — just run it
- Do not report intermediate steps — only report the final review request and final delivery
- Do not ask about rate limits or transient errors — retry silently
- Do not send multiple messages during a phase — batch everything into one

## Process

### Step 1: Structure Scaffolding (Auto-Generate Boilerplate)
Before any quality checks, generate all missing structure files from the existing SKILL.md:

**Auto-generate if missing:**

| File | Source | Generation Method |
|------|--------|-------------------|
| `skill.yml` | SKILL.md frontmatter + triggers | Extract name, description, version, triggers from SKILL.md |
| `README.md` | SKILL.md description + usage | GitHub landing page: what it does, when to use, example commands |
| `CHANGELOG.md` | Version from skill.yml + git log | `## v{version} — {date}` + summary of current state |
| `tests/test-triggers.json` | SKILL.md triggers + "When to Use" | `shouldTrigger` from triggers list, `shouldNotTrigger` from anti-patterns |
| `scripts/` | Create directory | Empty dir or placeholder README if no scripts needed |
| `references/` | Create directory | Empty dir or placeholder README if no references needed |
| `LICENSE` | Default MIT | Standard MIT license text |
| `.gitignore` | Standard | `node_modules/`, `.DS_Store`, `*.log` |

**Rules:**
- Never overwrite existing files — only generate what's missing
- All generated content derives from SKILL.md — no hallucinated features
- If SKILL.md lacks enough info to generate a file, flag it as a content gap (user must fix SKILL.md first)
- Generated README.md must make sense to a stranger who has never seen the skill before

**Validation after scaffolding:**
- Run `scripts/validate-structure.sh` — must score 8/8
- If not 8/8, identify what's still missing and fix it

### Step 1.5: Version Bump (updates only)
If this skill has been published before, bump the version before proceeding:

1. **Check current published version:**
```bash
clawhub search {name}
```

2. **Bump version** in both `skill.yml` and `SKILL.md` frontmatter:
   - Patch (1.0.0 → 1.0.1): bug fixes, typos, minor doc updates
   - Minor (1.0.0 → 1.1.0): new features, new sections, structural changes
   - Major (1.0.0 → 2.0.0): breaking changes, full rewrites

3. **Update CHANGELOG.md** with new version entry describing what changed

Skip this step for first-time releases.

### Step 2: Readiness Check
Verify the skill directory is complete:
- `SKILL.md` exists with description and usage instructions
- `skill.yml` exists with name, description, triggers
- Structure score 8/8 (from Step 1)
- No obvious OPSEC violations (quick scan)

If any check fails, report what needs fixing. Do not proceed.

### Step 3: Create Private Staging Repo
```bash
# Check if repo already exists
gh repo view your-org/openclaw-skill-{name} 2>/dev/null

# If not, create it
gh repo create your-org/openclaw-skill-{name} --private --description "{skill description from skill.yml}"
```

### Step 4: Prepare Release Content
Copy sanitized skill content to a clean directory:
```bash
mkdir -p /tmp/skill-release-{name}
cp -r skills/{name}/* /tmp/skill-release-{name}/

# Remove internal-only files
rm -f /tmp/skill-release-{name}/WORKSPACE.md
rm -f /tmp/skill-release-{name}/.gitignore
rm -rf /tmp/skill-release-{name}/_meta.json
```

Add release files if missing:
- `LICENSE` (MIT by default)
- `README.md` (must work as GitHub landing page for strangers)
- `.gitignore`

### Step 5: OPSEC Deep Scan
```bash
bash scripts/opsec-scan.sh /tmp/skill-release-{name}
```
Must return CLEAN (exit 0). If violations found, fix them in the release copy. Do NOT modify the source in openclaw-knowledge — keep the internal version as-is.

### Step 6: Agent Review
Generate review document:
```markdown
# Release Review: {skill-name}

## Checklist
- [ ] SKILL.md clear and useful to a stranger
- [ ] README.md works as GitHub landing page
- [ ] skill.yml triggers accurate and complete
- [ ] Scripts work without hardcoded dependencies
- [ ] Tests present and described
- [ ] CHANGELOG.md current
- [ ] LICENSE present
- [ ] No references to internal repos, infrastructure, or personal info
- [ ] OPSEC scan: CLEAN
- [ ] Competitive position: {novel|ahead}

## OPSEC Scan Output
{paste scan output}

## Competitive Summary
{from audits/{name}-competitive.md}

## Recommendation
APPROVE / REVISE: {reasons}
```

Save to `openclaw-knowledge/reviews/{name}-release-review.md`

### Step 7: Push to Private Staging Repo
Push sanitized content so user can review the actual repo on any device (phone, laptop):
```bash
cd /tmp/skill-release-{name}
git init
git config user.email "agent@localhost"
git config user.name "SkillEngineer"
git add .
git commit -m "v{version}: Initial release of {name}"
git remote add origin https://github.com/your-org/openclaw-skill-{name}.git
git branch -M main
git push -u origin main
```

### Step 8: User Review
Send repo link + brief summary to user via Telegram:
```
RELEASE REVIEW: {skill-name}

Score: {X}/24 | OPSEC: CLEAN | Position: {novel/ahead}
Review here: https://github.com/your-org/openclaw-skill-{name}

{1-2 sentence summary of what the skill does}

Agent recommendation: {APPROVE/REVISE}

Reply: approve / revise:{feedback} / reject
```

The repo IS the review artifact. User reviews actual files, not a summary.
Wait for user response. Do not proceed without explicit approval.

### Step 9: Erase History & Flip to Public (after user approval)
Erase git history (may contain OPSEC fixes from earlier revisions) and make the repo public:
```bash
cd /tmp/skill-release-{name}
# Orphan branch erases all history
git checkout --orphan clean
git add -A
git commit -m "v{version}: {name}"
git branch -D main
git branch -m main
git push -f origin main

# Flip visibility
gh repo edit your-org/openclaw-skill-{name} --visibility public
```

Single commit, clean history, one repo. No dual-repo complexity.

### Step 10: Publish to ClawhHub
```bash
clawhub publish /tmp/skill-release-{name} \
  --slug {name} \
  --name "{Display Name}" \
  --version {version} \
  --changelog "{summary of changes}"
```

### Step 11: Verify Security Scan (Browser Required)
ClawhHub automatically scans all published skills via VirusTotal (Code Insight) and OpenClaw's own scanner. **Do not consider the release complete until scans are reviewed.**

**Use the browser tool to check scan results — ClawhHub pages require JS rendering:**

1. **Open the skill detail page with browser:**
```
browser start (profile=openclaw)
browser navigate → https://clawhub.ai/{username}/{slug}
browser snapshot (refs=aria)
```

2. **Find the "Security Scan" section** in the snapshot. It shows:
   - **VirusTotal verdict:** Benign / Suspicious / Malicious / Pending
   - **OpenClaw verdict:** Benign / Suspicious / Malicious with confidence level
   - **Detail text:** Explanation of what was flagged (expand "Details" if collapsed)
   - **VirusTotal report link:** Direct URL to full analysis

3. **Interpret results and act:**

| Verdict | Meaning | Action |
|---------|---------|--------|
| Benign (both) | Clean, auto-approved | Proceed to Step 12 |
| Pending | Still processing | Wait 2 minutes, re-snapshot |
| Suspicious (undeclared permissions) | Skill needs privileged access not in metadata | Add `permissions` to skill.yml, bump version, re-publish |
| Suspicious (other) | Flagged behavior | Review detail text. If false positive, contact OpenClaw security team. If real, fix and re-publish |
| Malicious | Blocked from download | Fix immediately, bump version, re-run from Step 1.5 |

4. **Common fix — undeclared permissions:**
   If flagged for privileged CLI access (gh, clawhub, git, filesystem), add a `permissions` field to `skill.yml`:
   ```yaml
   permissions:
     - exec: git, gh CLI (repo creation, visibility changes)
     - exec: clawhub CLI (publishing)
     - filesystem: read/write skill directories
     - browser: verify scan results on ClawhHub
   ```
   Then bump version and re-publish. This declares intent and resolves the flag.

5. **If VirusTotal is still Pending** after 5 minutes, proceed to Step 12 but note it in the delivery. The scan completes asynchronously.

### Step 12: Deliver
Confirm the release is live and deliver all links and scan status to the user:

```
RELEASED: {skill-name} v{version}

GitHub: https://github.com/your-org/openclaw-skill-{name}
ClawhHub: https://clawhub.ai/{username}/{slug}
VirusTotal: {verdict} — {report link}
OpenClaw Scan: {verdict} ({confidence})

{1-line description}
```

**Update tracking** — record in memory or STATUS.json:
- GitHub URL (public)
- ClawhHub URL and slug
- Release date and version
- Security scan verdicts (VirusTotal + OpenClaw)
- Review dates (agent and user)

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| Readiness check fails | Score too low or OPSEC dirty | Complete refactoring first |
| OPSEC scan finds violations in release copy | Sanitization incomplete | Fix in release copy, re-scan |
| gh repo create fails | Auth issue or name taken | Check `gh auth status`, try different name |
| clawhub publish fails | CLI not installed or auth | Run `npm install -g clawhub`, authenticate |
| User rejects | Feedback provided | Address feedback, restart from Step 4 |

## Examples

**Release a specific skill:**
"Release skill-engineer to clawhub"

**Check readiness without releasing:**
"Is evidence-based-investigation ready for release?"

**Batch readiness check:**
"Which skills are ready to publish?"
