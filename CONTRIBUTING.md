# Contributing to Railway Deploy Skill

Contributions are welcome! Here's how to help:

## Add a New Failure Pattern

1. Fork the repo
2. Add your pattern to `skills/railway-deploy/SKILL.md` following the existing format:
   - Error signature
   - Likely cause
   - Fix instructions
3. Submit a PR

## Pattern Format

```markdown
### Pattern X: Your Pattern Name
**Error:** `error 1`, `error 2`

**Likely cause:** Explanation.

**Fix:**
1. Step one
2. Step two
```

## CLI Commands

All CLI commands must be verified against actual `railway --help` output before submitting.

## PR Checklist

- [ ] Pattern includes real error signatures (not made up)
- [ ] Fix steps are actionable and tested
- [ ] CLI commands verified against `railway --help`
- [ ] No sensitive data in examples
