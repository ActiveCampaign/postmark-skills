# Contributing to Postmark Skills

Thank you for helping improve the Postmark Skills library. This guide covers how skills are structured, how to add or update content, and the standards each contribution should meet.

## Repository Structure

```
postmark-skills/
├── SKILL.md                          # Root skill — routes to sub-skills
├── README.md                         # Repository overview and installation
├── CONTRIBUTING.md                   # This file
├── postmark-send-email/
│   ├── SKILL.md                      # Sub-skill entry point
│   └── references/                   # Detailed reference documentation
│       ├── installation.md
│       └── ...
├── postmark-inbound/
│   ├── SKILL.md
│   └── references/
│       └── ...
├── postmark-templates/
│   ├── SKILL.md
│   └── references/
│       └── ...
├── postmark-webhooks/
│   ├── SKILL.md
│   └── references/
│       └── ...
└── postmark-email-best-practices/
    ├── SKILL.md
    └── references/
        └── ...
```

Each skill is a directory containing a `SKILL.md` entry point and a `references/` folder with detailed documentation. The `SKILL.md` is what AI agents load first — it must be concise and actionable.

---

## SKILL.md Format

Every `SKILL.md` must begin with YAML frontmatter:

```yaml
---
name: postmark-skill-name
description: One or two sentences describing when an AI agent should load this skill. Start with "Use when..." or similar trigger language.
license: MIT
metadata:
  author: postmark
  version: "1.0.0"
---
```

### Frontmatter Fields

| Field | Required | Value |
|-------|----------|-------|
| `name` | Yes | Matches directory name exactly |
| `description` | Yes | Trigger language for the agent skills router |
| `license` | Yes | `MIT` |
| `metadata.author` | Yes | `postmark` |
| `metadata.version` | Yes | Semantic version string, quoted |

### SKILL.md Content Guidelines

- **Keep it short** — target 100–200 lines. If it exceeds this, move detail to `references/`.
- **Lead with a Quick Reference table** — what topics are covered, what to use each for.
- **Include at least one working code example** — the most common use case, ready to copy.
- **Link to reference files** — don't repeat content that lives in `references/`.
- **List Common Mistakes** — things developers frequently get wrong with this feature.
- **End with a Notes section** — key constraints, limits, and gotchas.

### Example SKILL.md Structure

```markdown
---
name: postmark-example
description: Use when...
license: MIT
metadata:
  author: postmark
  version: "1.0.0"
---

# Feature Name

One paragraph description.

## Quick Reference

| Topic | Use When |
|-------|----------|
| ... | ... |

## Quick Start

[Minimal working code example]

## [Topic 1]

[Short overview, key points, link to references/topic1.md for details]

## Common Mistakes

- [Specific mistake]: [Why it's wrong and what to do instead]

## Notes

- [Key constraint or gotcha]
```

---

## Reference File Guidelines

Reference files in `references/` contain detail that doesn't belong in the main `SKILL.md`. Each file should cover one topic thoroughly.

### File Naming

Use lowercase kebab-case matching the topic:

```
references/
├── payload-structure.md      # Good
├── handler-examples.md       # Good
├── PayloadStructure.md       # Bad — wrong case
├── handlers.md               # Too vague — be specific
```

### Reference File Content Standards

- **Start with a clear heading** — `# Topic Name`
- **Use tables** for API fields, parameters, options, and comparisons
- **Use working code examples** — every example should be copy-paste ready
- **Show multiple languages** when relevant — Node.js first, then Python, cURL
- **Link to Postmark docs** for anything that may change (rate limits, error codes, API endpoints)
- **Don't duplicate** content that exists in another reference file — link to it instead

### Code Example Standards

All code examples must:

- Use real Postmark API field names (correct casing: `From`, `To`, `HtmlBody`, not `from`, `to`, `htmlBody`)
- Use environment variables for credentials: `process.env.POSTMARK_SERVER_TOKEN`
- Be syntactically correct and runnable
- Show the install command or import at the top of the first example in each file
- Prefer async/await over callbacks

```javascript
// Good
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const result = await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'recipient@example.com',
  Subject: 'Hello',
  TextBody: 'Hello world.',
  MessageStream: 'outbound'
});
```

---

## Adding a New Skill

1. Create a directory: `postmark-new-feature/`
2. Create `postmark-new-feature/SKILL.md` with frontmatter and content following the format above
3. Create `postmark-new-feature/references/` with at least one reference file
4. Add the skill to the root `SKILL.md` Sub-Skills table and Quick Routing section
5. Add the skill to `README.md` Available Skills table and add a usage example prompt

---

## Updating an Existing Skill

When Postmark updates its API:

1. Update the relevant reference file(s) in `references/`
2. If it's a breaking change or significant addition, update the `SKILL.md` Quick Start example
3. Bump `metadata.version` in the skill's frontmatter (e.g., `1.0.0` → `1.1.0`)
4. Note the change in your PR description

---

## Pull Request Checklist

- [ ] Frontmatter is complete and correctly formatted on all modified `SKILL.md` files
- [ ] `SKILL.md` is under 200 lines
- [ ] All code examples use `process.env.POSTMARK_SERVER_TOKEN` (not hardcoded tokens)
- [ ] All code examples use correct Postmark API field casing (`From`, not `from`)
- [ ] Reference files linked from `SKILL.md` actually exist
- [ ] New skills are added to root `SKILL.md` and `README.md`
- [ ] No duplicate content — detail lives in `references/`, overview in `SKILL.md`
