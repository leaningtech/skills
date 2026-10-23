# Leaning Technologies skills

Agent skills for [Leaning Technologies](https://leaningtech.com) products. Each
skill teaches coding agents how to work with one product and links to its docs
for current facts.

| Skill                                | Product                                                                                                                   |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| [`cheerpj`](skills/cheerpj/SKILL.md) | [CheerpJ](https://cheerpj.com): Java applications, applets, Java Web Start, Oracle Forms/EBS and libraries in the browser |

## Install

- **Skills CLI:** `npx skills add leaningtech/skills --skill cheerpj`
- **Claude Code:** `/plugin marketplace add leaningtech/skills`, then `/plugin install cheerpj@leaningtech-skills`
- **Codex:** `codex plugin marketplace add leaningtech/skills`, then install `cheerpj` from the plugin directory
- **Gemini CLI:** `gemini extensions install https://github.com/leaningtech/skills`
- **Any agent:** point it at the raw `SKILL.md`, or at https://cheerpj.com/skills/cheerpj/SKILL.md

## Layout

- `skills/<name>/SKILL.md`: the skills. This repo is their source of truth.

## Licence

MIT. See [LICENSE](LICENSE).
