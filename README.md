# skill-releaser

Release agent skills to ClawhHub through a full publication pipeline.

## What It Does

Takes any skill with a finished SKILL.md and handles everything needed to publish it:

1. **Scaffolds** missing structure files (README, CHANGELOG, tests, skill.yml) from SKILL.md
2. **Scans** for OPSEC violations (secrets, personal info, hardcoded paths)
3. **Reviews** via dual process — agent checklist + user approval
4. **Stages** in a private repo for user review on any device
5. **Publishes** by erasing history (force push) and flipping repo to public, then ClawhHub

## When to Use

- "Release {skill-name} to ClawhHub"
- "Is {skill-name} ready for release?"
- "Which skills are ready to publish?"
- "Prepare {skill-name} for release"

## Key Design Decisions

- **History erased at release** — orphan branch + force push gives a single clean commit
- **One repo per skill** — starts private for staging, flipped to public at release
- **User approval is a hard gate** — nothing goes public without explicit "approve"
- **Boilerplate is auto-generated** — only SKILL.md content needs to be written by hand
- **Security scan verified** — uses browser to check VirusTotal + OpenClaw scan results on ClawhHub before declaring release complete

## Prerequisites

- `gh` CLI authenticated
- `clawhub` CLI installed
- Shell access for git operations
- Browser tool available (for ClawhHub security scan verification)

## License

MIT
