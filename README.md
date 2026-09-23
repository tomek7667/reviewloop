# reviewloop

A Claude Code skill: **review → fix → review again**, until a full pass finds nothing.

After implementing a feature or bugfix, `/reviewloop` pins what was asked for as a
checkable acceptance list, reviews the real git diff (including untracked files) with a
fresh-eyes subagent, fixes every finding, and starts a new pass from scratch. It exits
only on a clean pass — capped at 5 iterations, escalating on oscillating findings.

## Install

### As a plugin (Claude Code)

```
/plugin marketplace add tomek7667/reviewloop
/plugin install reviewloop@reviewloop
```

### With the skills CLI

```
npx skills add tomek7667/reviewloop
```

### Manually

```bash
git clone https://github.com/tomek7667/reviewloop
cp -r reviewloop/skills/reviewloop ~/.claude/skills/
```

## Usage

```
/reviewloop add a toggle for the new UI, keep the old one working
```

See [SKILL.md](skills/reviewloop/SKILL.md) and
[REFERENCE.md](skills/reviewloop/REFERENCE.md) for the full workflow and checklist.
