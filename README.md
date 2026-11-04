# skills

Agent skills by [@arnivuo](https://github.com/arnivuo).

| Skill | What it does |
| --- | --- |
| [`implement-spec-stacked`](skills/implement-spec-stacked/SKILL.md) | Implements a spec and its tickets as a stack of PRs with [`gh stack`](https://github.com/github/gh-stack), one ticket per layer. A stacked sibling of [`implement-spec`](https://github.com/mattpocock/skills). Needs the `gh-stack` extension (v0.3.0+) and skill. |

## Install

```sh
gh skill install arnivuo/skills implement-spec-stacked --agent claude-code --scope user
# or
npx skills add arnivuo/skills
```

`implement-spec-stacked` also needs:

```sh
gh extension install github/gh-stack
gh skill install github/gh-stack gh-stack --agent claude-code --scope user
```
