---
name: implement-spec-stacked
description: "Implement a spec and its tickets as a stack of PRs with gh stack, one ticket per layer."
disable-model-invocation: true
---

You have been provided a spec. It should have tickets describing how to implement it.

The goal is the entire spec implemented as a **stack** of pull requests on GitHub: one ticket per **layer**, layers chained bottom-up from a **base branch**, so each PR shows only its own layer's diff.

The tickets are a **task graph** with blocking relationships, so there is always a **frontier** of tickets ready to be grabbed. A stack is strictly linear, so the plan flattens the graph into a layer order. Adjacent layers that don't depend on each other form a **parallel group**: they may be built at the same time.

Keep messages to and from subagents to **context pointers**: the spec, tickets, research notes, commits. The pointers hold the detail.

Load the `gh-stack` skill before the first `gh stack` command. It owns the CLI mechanics: non-interactive flags, editing a lower layer, conflict recovery, PR edits.

## Steps

1. **Read** the spec and tickets and build the task graph. Done when you know every ticket's blockers.

2. **Preflight.** Run every check. Each one either passes or stops the run with its fix shown to the user:

   | Check | Pass | Fix to show |
   | --- | --- | --- |
   | Forge | `git remote get-url origin` is on github.com | The stacked flow needs GitHub; use `/implement-spec` for a single PR. |
   | Extension | `gh extension list` shows `github/gh-stack` at v0.3.0 or newer | `gh extension install github/gh-stack`, or `gh extension upgrade stack` |
   | Skill | `gh-stack` is in your available skills | `gh skill install github/gh-stack gh-stack --agent claude-code --scope user` |
   | Main checkout | `git status --porcelain` is empty | Commit or stash local changes first. |
   | Issue tracker | The repo documents its tracker (e.g. `docs/agents/issue-tracker.md`) | Run `/setup-matt-pocock-skills` |

   GitHub is the only forge supported today. Support for another forge starts at the Forge check.

3. (optional) Use an **exploration subagent** for any exploration the tickets need: relevant codebase files or external docs. It saves markdown notes in a directory outside the repo that every later subagent can read, so implementers focus on implementing.

4. **Plan.** Present the plan and revise it until the user approves it. It contains:
   - **Base branch**: the repo's default branch (`gh repo view --json defaultBranchRef`). If the spec builds on an open, unmerged branch or stack, propose that instead as an explicit question.
   - **Layers**, bottom to top: branch name and ticket. Branch names follow the repo's convention (see recent PRs). One ticket per layer; when you fold trivially small tickets into one layer, name the fold and why.
   - **Parallel groups**.
   - **Mode**: _serial_ (one layer at a time, no rebases) or _parallel_ (one worktree per layer in a group, a rebase when the group lands). Recommend parallel only when a group holds two or more layers; otherwise recommend serial.

   ```
   Base: develop
   1. WEB-101-store-model    WEB-101 Store model
   2. WEB-102-store-api      WEB-102 Store API           ┐ parallel group
   3. WEB-103-store-search   WEB-103 Store search index  ┘
   4. WEB-104-store-page     WEB-104 Store page
   Mode: parallel (recommended: layers 2-3 share no dependency)
   ```

   Done when the user approves the plan.

5. **Build** the layers bottom-up, following the plan's mode. You, the orchestrator, run every `gh stack` command from the main checkout; implementers only commit.
   - Create a layer's branch only when its turn comes: `gh stack top` (or `gh stack init --base <base> <branch>` for the first layer), then `gh stack add <branch>`. `add` runs only from the top branch, so the top branch must be free of worktrees. A parallel group adds all its branches at once, each on top of the previous one. Then switch the main checkout back to the base branch so the new branches are free for worktrees.
   - Give each layer an **implementer subagent** in its own worktree (`git worktree add <path> <branch>`), run in the background where possible. Each implementer confirms its worktree has its layer branch checked out, calls the Skill tool with `tdd` to build the ticket, and commits on that branch following the repo's commit conventions.
   - Remove an implementer's worktree as soon as it reports done. Its branch is then free for `gh stack top`, `add`, and `rebase`.
   - Run `gh stack rebase` only when every layer above the rebase point is done. In a parallel group that means one rebase, once the whole group is done.
   - On a rebase conflict, dispatch a **resolver subagent** with pointers to both tickets and both commit ranges. It resolves in the worktree gh-stack names, continues the rebase, then runs the repo's checks (typecheck, tests, build) on every layer above the conflict: a clean rebase can still break the build.
   - When a layer or group lands, the frontier advances: start the next.

   Done when every ticket has its layer with commits, the stack is rebased, and the repo's checks pass on the top layer.

6. **Review.** Call the Skill tool with `code-review` over base..top. Make each fix in the layer that owns the code, then `gh stack rebase` to cascade it upward. Done when every issue is fixed or listed for the user, and checks pass on the top layer.

7. **Submit.** Show the user the branches about to be pushed and PRs about to be opened. Then run `gh stack submit --auto --open`, and set each PR's real title and body with `gh pr edit <number> --title … --body-file …`, following the repo's PR conventions (title prefixed with the ticket key, body pointing at its ticket and the spec). Done when every layer has an open PR with its real title and body.

8. **Tickets.** Comment each PR's link on its ticket, the way the repo's tracker doc says to comment. Leave ticket status as it is: tickets close when their PRs merge.

9. **Report** the base branch and the PRs bottom to top, after confirming no implementer worktrees remain.
