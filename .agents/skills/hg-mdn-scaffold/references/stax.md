# Stacked PRs with Stax (step 2 tool, step 3 skill)

[Stax](https://github.com/cesarferreira/stax) manages **stacked branches and PRs**: small, reviewable PRs that build on each other, restacked and submitted together.  It installs two binaries, `stax` and the short alias `st`.  It wraps `git` and the GitHub CLI; nothing in the repo depends on it, so a teammate without it can still use plain `git`.

Full command reference: <https://raw.githubusercontent.com/cesarferreira/stax/refs/heads/main/docs/commands/reference.md>.

## Install (Mise)

Pin it in `.config/mise.toml` with the other tools (verified: the GitHub backend downloads the release binary and exposes both `st` and `stax`):

```toml
[tools]
"github:cesarferreira/stax" = "0.115.0"   # stacked PRs: `st` / `stax`; check the latest release when scaffolding
```

Then, once per developer machine:

```shell
mise install
st setup --yes        # shell integration (for `st wt go`), git rerere, the agent skill for detected agents, auth from `gh`
st doctor             # repo health; `st doctor --fix` applies the safe repairs
```

- **Auth** comes from the GitHub CLI (`gh auth login` first; `st auth --from-gh` to import later).  `gh` is the only prerequisite.
- **Upgrades:** bump the version in `mise.toml` (`mise latest github:cesarferreira/stax`), not `st update`, so every machine and agent runs the same version.  Then refresh the skill (below).
- `st init` once per repo if `st` asks for the trunk (`main`).

## The agent skill

Stax ships an official agent skill (all commands, flags, and agent best practices).  Vendor it **in the repo**, so every agent and every clone gets the same version as the pinned binary:

```shell
mkdir -p .agents/skills/stax && st --skill > .agents/skills/stax/SKILL.md
```

- `.agents/skills` is linked into `.claude/skills` (and `.codex/skills`) in step 3, so one file serves every agent.  List it in `AGENTS.md` under Skills.
- **Refresh it whenever the pinned version changes** (`mise run stax-skill`, a task in the template); the file records `stax_version`.
- `st setup --skills ...` / `st skills update` instead install per-user copies into each agent's home directory.  That is fine for personal use, but the repo copy is what keeps a team (and CI agents) consistent.

## Working with stacks

One branch per logical change, each branch one PR, each based on the one below.  `st` commits and pushes, so the repository rule still applies: **agents run `st create`, `st modify`, and `st submit` only when asked to commit or push.**

```shell
# Start a stack from trunk: each `create` is a new branch + commit on top of the current one
st create -am "feat(notes): Add notebooks store"
st create -am "feat(notes): Show notebooks in the sidebar"

st modify -a          # amend the current branch with all changes (`-ar` also restacks what is above)
st submit             # push the whole stack and create/update its PRs (`--draft`, `--publish`, `--ai` for titles/bodies)
st ls                 # the stack: branches, PRs, and what needs restacking (`st ll` adds PR URLs)
```

Commit messages follow the repo convention (`feat(area): Sentence-case description`; also `fix`, `chore`, `refactor`, `docs`, `test`).  Before submitting, run `mise run check` on each branch, or across the stack: `st run --stack --fail-fast -- mise run check`.

**Navigating:** `st up` / `st down` (one branch), `st top` / `st bottom`, `st co` (picker), `st trunk`.

**Keeping current:**

```shell
st rs                 # sync: pull trunk, delete merged branches, reparent children
st rs --restack       # ...and rebase the stack onto the updated parents
st refresh            # sync + restack + push + update PRs, in one step
st restack            # rebase the current stack locally only
```

On a conflict: resolve, then `st continue` (or `st abort`).  Undo a stax operation with `st undo`.

## Watching CI and reviews

```shell
st ci                 # checks for the current PR, per check, with elapsed time and ETA
st ci --stack -1      # one line per PR in the stack
st ci --watch --strict  # watch until done; fail fast on the first failing check (`--alert` for a sound)
st watch --current    # live stack status with CI and PR state
st ready              # interactive readiness view: CI, approvals, mergeability (`--plain` / `--json` for agents)
st comments --stack   # review comments across the stack, with file/line locations (`--json` for agents)
st pr                 # open the current PR in the browser
```

For agents: prefer the non-interactive forms (`--json`, `--plain`, `-1`) and `st ci --watch --strict`, which exits non-zero when a check fails, as the signal to fix and resubmit.

## Merging

```shell
st merge --when-ready # wait for CI, approvals, and mergeability, then merge from the bottom up to the current branch
st merge --dry-run    # see what would merge first
st rs                 # afterwards: clean up merged branches and move to the next one
```

Merging is a human decision unless the user asks an agent to do it.

## Parallel agents: worktree lanes

`st lane <name> "<prompt>"` creates (or reuses) a worktree lane on its own branch and can launch an agent in it (`--agent claude`); `st wt ls` lists lanes, `st wt go <name>` switches (needs the shell integration), `st wt rm` removes the current one.  Use lanes when several agents work on independent changes at once; each lane is a normal stax branch, submitted the same way.

## Health

`st validate` (metadata), `st fix --dry-run` / `st fix` (repair), `st sweep` (classify stale branches; `--delete` to clean up), `st doctor`.
