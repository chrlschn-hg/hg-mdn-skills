# Release Automation (step 13, optional)

Versioned releases of the container image: the next version is computed from commit types, the image is built with it, and a GitHub Release creates the tag.  **Skippable at scaffolding**: the base templates do not depend on it, the files and local tests need no GitHub, and "Once, on GitHub" lists what to finish after the repository exists.

## How it fits

```text
push to main   docker-publish.yml -> ghcr.io/<owner>/<repo>:latest and :sha-<commit>        reports 0.0.0-dev+<commit>
Actions -> Release
  plan         build/next-version.sh -> 1.4.0   (from the tags and commits; nothing is committed)
               stops here if the image or tag for 1.4.0 exists
  validate     ci.yml: build, tests, lint, image smoke test, at this commit
  ship         [approval gate] image :1.4.0, built with VERSION=1.4.0 GIT_SHA=<commit>; deploy
  publish      GitHub Release v1.4.0: creates the tag at the commit, notes from the merged PR titles
```

**The app version.**  `VERSION` becomes `-p:Version=1.4.0` in the Docker build, then the informational version, `BuildInfo`, `GET /api/health`, the startup log, `service.version`, and `BuildInfoLabel` (`hg-mdn-backend`, `references/build-info.md`).  The tag is created last, at the commit that was built, so a failed release leaves no tag.  Builds of main between releases report `0.0.0-dev` and their commit.  The version exists only in tags: no version file, no release commit, no changelog file.

## Policy

| Commits since the last stable release | Release |
|---|---|
| `type!:` subject, or a `BREAKING CHANGE:` footer in the commit body | major (**minor while the version is 0.x**) |
| `feat` | minor |
| `fix` | patch |
| `docs`, `chore`, `ci`, `build`, `refactor`, `test`, `perf`, `style`, `revert`, merge commits, non-conventional subjects | nothing |

- The highest bump wins.  Types match case-insensitively.  A revert does not cancel the commit it reverts.  Nothing releasable: the plan fails and says why; `version` (override) releases anyway.
- **The first release is `0.1.0`**, whatever the commits.  **1.0.0 is declared**, not computed: run the release with `version` = `1.0.0`.
- The baseline is the last stable `vX.Y.Z` tag reachable from `HEAD`; `-rc.N` tags never move it.  An rc is `X.Y.Z-rc.N` of the version the stable release would get, N counting up, published as a GitHub prerelease.  The stable release after it is `X.Y.Z` (more commits can raise the bump), built from the current tip of main rather than the rc image, so it includes anything merged since: the plan prints the commits since the last rc.
- An override must be `X.Y.Z` (stable channel) or `X.Y.Z-rc.N` (rc channel), without a `v`, greater than the last release, and not an existing tag.  A tag or image tag is never overwritten.
- It needs full history and tags (`fetch-depth: 0`) and at least one commit.  Releases run from `main` only; maintenance branches are not covered.  One version for the whole repository.
- **The commit subjects are the input.**  A squash merge takes the PR title as the commit message, so a title that is not `type(scope): Subject` silently releases nothing (`pr-title.yml` checks it), and with that default a `BREAKING CHANGE:` footer in the PR description never reaches the commit: mark breaking changes with `!` in the title.  The plan prints the counts and the commits (up to 50) for the approver to check.

## Files

`build/next-version.sh` is the whole calculation, with no dependencies beyond `bash` and `git` (it runs on macOS's bash 3.2 and on Linux).  Run it locally with `mise run release-version [-- --channel rc]`.  Why a script: `svu` and `git-cliff` would do the same arithmetic, but `svu`'s `--tag.pattern` is a git glob and cannot exclude `-rc.N` tags, `git-cliff` bumps a patch for docs-only history (use it if a `CHANGELOG.md` is wanted), and a GitHub Action cannot be tested locally.

### `build/next-version.sh`

Make it executable (`chmod +x build/*.sh`).

```bash
#!/usr/bin/env bash
# Next release version from the Conventional Commits since the last stable release tag.
#   build/next-version.sh [--channel stable|rc] [--override VERSION]
#
# stdout: version=, tag=, previous=, bump=   (the release workflow appends it to $GITHUB_OUTPUT)
# stderr: the plan, as Markdown (the release workflow shows it in the job summary)
# Needs full history and tags (`fetch-depth: 0`).  The version lives in git tags only.
#
# breaking (`type!:` or a `BREAKING CHANGE:` footer) -> major, a minor while 0.x; feat -> minor;
# fix -> patch; nothing else releases.  The first release is 0.1.0; 1.0.0 is declared with
# `--override 1.0.0`; `rc` adds `-rc.N` to the version the stable release would get.
set -euo pipefail

channel=stable
override=""

while [ $# -gt 0 ]; do
  case "$1" in
    --channel) channel="${2:?--channel needs stable or rc}"; shift 2 ;;
    --override) override="${2:?--override needs a version}"; shift 2 ;;
    *) echo "usage: next-version.sh [--channel stable|rc] [--override VERSION]" >&2; exit 2 ;;
  esac
done

fail() { echo "::error::$*" >&2; exit 1; }

case "$channel" in stable | rc) ;; *) fail "channel must be stable or rc, not '$channel'" ;; esac

git rev-parse -q --verify HEAD > /dev/null || fail "the repository has no commits yet"

if [ "$(git rev-parse --is-shallow-repository)" = true ]; then
  fail "the history is shallow: check out with fetch-depth: 0 so the tags and commits are all there"
fi

# The last stable release reachable from HEAD: rc tags are ignored, so an rc never moves the baseline.
previous="$(git tag --merged HEAD --list 'v[0-9]*' | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | sort -t. -k1.2,1n -k2,2n -k3,3n | tail -n 1 || true)"
range="HEAD"
[ -n "$previous" ] && range="$previous..HEAD"

subjects="$(git log --no-merges --format=%s "$range")"
count() { { printf '%s\n' "$subjects" | grep -E -i -c "$1" || true; }; }

breaking_subjects="$(count '^[a-z]+(\([^)]*\))?!:')"
breaking_footers="$(git log --no-merges -E --grep='^BREAKING[ -]CHANGE:' --format=%h "$range" | grep -c . || true)"
features="$(count '^feat(\([^)]*\))?:')"
fixes="$(count '^fix(\([^)]*\))?:')"

if [ -z "$previous" ]; then
  base="0.1.0"
  bump="first release"
else
  IFS=. read -r major minor patch <<< "${previous#v}"

  if [ $((breaking_subjects + breaking_footers)) -gt 0 ]; then
    if [ "$major" -eq 0 ]; then
      bump="minor"
    else
      bump="major"
    fi
  elif [ "$features" -gt 0 ]; then
    bump="minor"
  elif [ "$fixes" -gt 0 ]; then
    bump="patch"
  else
    bump="none"
  fi

  case "$bump" in
    major) base="$((major + 1)).0.0" ;;
    minor) base="$major.$((minor + 1)).0" ;;
    patch) base="$major.$minor.$((patch + 1))" ;;
    none) base="" ;;
  esac
fi

if [ -n "$override" ]; then
  [[ "$override" =~ ^[0-9]+\.[0-9]+\.[0-9]+(-rc\.[0-9]+)?$ ]] || fail "override '$override' is not X.Y.Z or X.Y.Z-rc.N (no leading v)"
  case "$override" in
    *-rc.*) [ "$channel" = rc ] || fail "override $override is a release candidate: use --channel rc" ;;
    *) [ "$channel" = stable ] || fail "override $override is a stable version: use --channel stable" ;;
  esac
  base="${override%%-rc.*}"
  bump="override"

  if [ -n "$previous" ]; then
    highest="$(printf '%s\n%s\n' "${previous#v}" "$base" | sort -t. -k1,1n -k2,2n -k3,3n | tail -n 1)"
    if [ "$base" = "${previous#v}" ] || [ "$highest" != "$base" ]; then
      fail "override $base is not greater than the last release $previous"
    fi
  fi

  version="$override"
elif [ -z "$base" ]; then
  version=""
else
  version="$base"

  if [ "$channel" = rc ]; then
    last_rc="$(git tag --list "v$base-rc.*" | sed -E "s/^v$base-rc\\.([0-9]+)\$/\\1/" | grep -E '^[0-9]+$' | sort -n | tail -n 1 || true)"
    version="$base-rc.$(( ${last_rc:-0} + 1 ))"
  fi
fi

{
  echo "## Release plan"
  echo
  echo "- Last release: ${previous:-none}"
  total="$(git rev-list --no-merges --count "$range")"
  echo "- Commits since: $total"
  echo "- Breaking: $((breaking_subjects + breaking_footers)), feat: $features, fix: $fixes"
  echo "- Bump: $bump"
  last_rc_tag="$(git tag --merged HEAD --list "v${version%%-rc.*}-rc.*" 2> /dev/null | sort -t. -k4,4n | tail -n 1)"
  if [ -n "$version" ] && [ -n "$last_rc_tag" ]; then
    echo "- Commits since $last_rc_tag: $(git rev-list --no-merges --count "$last_rc_tag..HEAD")"
  fi
  echo "- Next version: ${version:-none}"
  echo
  git log --no-merges -n 50 --format='- %s (%h)' "$range"
  [ "$total" -le 50 ] || echo "- ... and $((total - 50)) more"
} >&2

if [ -z "$version" ]; then
  fail "nothing to release since $previous: no feat, fix, or breaking commits (docs, chore, ci, build, refactor, test, perf, style, and revert do not release); pass --override to release anyway"
fi

if git rev-parse -q --verify "refs/tags/v$version" > /dev/null; then
  fail "the tag v$version already exists"
fi

echo "version=$version"
echo "tag=v$version"
echo "previous=${previous:-}"
echo "bump=$bump"
```

### `build/next-version.test.sh`

Builds synthetic repositories and asserts the policy above: each bump, 0.x, rc numbering, ordering (`0.10.0` > `0.9.0`), tags outside the history, overrides, refusals, a shallow clone, no commits, and a history of thousands of commits (a pipe to `head` once killed `git log` with SIGPIPE).  `mise run release-test`.

```bash
#!/usr/bin/env bash
# Tests build/next-version.sh against synthetic git histories (no network, no GitHub).
#   build/next-version.test.sh        (also: mise run release-test)
set -uo pipefail

script="$(cd "$(dirname "$0")" && pwd)/next-version.sh"
tmp="$(mktemp -d)"
trap 'rm -rf "$tmp"' EXIT

passed=0
failed=0

# A fresh repository; each scenario starts in its own directory.
new_repo() {
  cd "$(mktemp -d "$tmp/repo.XXXXXX")" || exit 1
  git init -q -b main
  git config user.email test@example.com
  git config user.name test
  git config commit.gpgsign false
  git config tag.gpgsign false
}

commit() { echo "$1" >> file.txt && git add file.txt && git commit -q -m "$1"; }

# many_commits <count>: thousands of `fix` commits at once (git fast-import), on top of main
many_commits() {
  local i
  for ((i = 1; i <= $1; i++)); do
    [ "$i" -eq 1 ] && printf 'commit refs/heads/main\ncommitter t <t@example.com> %d +0000\ndata <<EOM\nfix(x): Fix %d\nEOM\nfrom refs/heads/main^0\n\n' "$((1700000000 + i))" "$i" && continue
    printf 'commit refs/heads/main\ncommitter t <t@example.com> %d +0000\ndata <<EOM\nfix(x): Fix %d\nEOM\n\n' "$((1700000000 + i))" "$i"
  done | git fast-import --quiet --force
  git reset -q --hard main
}

ok() { passed=$((passed + 1)); echo "ok   $1"; }
bad() { failed=$((failed + 1)); echo "FAIL $1: $2"; }

# expect <name> <version> [script args]: succeeds with `version=<version>`
expect() {
  local name="$1" want="$2" out
  shift 2
  out="$("$script" "$@" 2>/dev/null)" || { bad "$name" "exited $? (wanted $want)"; return; }
  local got
  got="$(printf '%s\n' "$out" | sed -n 's/^version=//p')"
  if [ "$got" = "$want" ]; then
    ok "$name"
  else
    bad "$name" "got '$got', wanted '$want'"
  fi
}

# expect_fail <name> <message fragment> [script args]: fails, prints no version, says why
expect_fail() {
  local name="$1" fragment="$2" out err
  shift 2
  err="$tmp/stderr"
  if out="$("$script" "$@" 2> "$err")"; then
    bad "$name" "succeeded with: $out"
  elif grep -q -- "$fragment" "$err"; then
    ok "$name"
  else
    bad "$name" "no '$fragment' in: $(tail -n 3 "$err" | tr '\n' ' ')"
  fi
}

# expect_plan <name> <fragment> [script args]: the plan on stderr contains the fragment
expect_plan() {
  local name="$1" fragment="$2" plan
  shift 2
  plan="$("$script" "$@" 2>&1 > /dev/null)"
  case "$plan" in
    *"$fragment"*) ok "$name" ;;
    *) bad "$name" "no '$fragment' in the plan" ;;
  esac
}

# ---- no commits (the normal state while scaffolding)
new_repo
expect_fail "a repository with no commits is refused with a clear message" "has no commits yet"

# ---- first release
new_repo; commit "chore(init): Initial commit"
expect "first release is 0.1.0, whatever the commits" 0.1.0
expect "first rc is 0.1.0-rc.1" 0.1.0-rc.1 --channel rc

# ---- bumps from a stable tag
new_repo; commit "chore(init): Initial commit"; git tag v0.1.0
commit "fix(api): Handle an empty list"
expect "fix bumps the patch" 0.1.1
commit "feat(api): Add search"
expect "feat bumps the minor (beats fix)" 0.2.0

new_repo; commit "feat(init): Start"; git tag v1.2.3
for type in docs chore ci build refactor test perf style revert; do
  commit "$type(x): Not a release"
done
expect_fail "docs, chore, ci, build, refactor, test, perf, style, revert do not release" "nothing to release since v1.2.3"
commit "Merge pull request #7 from someone/branch"
commit "a message with no type"
expect_fail "merges and non-conventional commits do not release" "nothing to release"
expect "--override releases anyway" 1.2.4 --override 1.2.4

# ---- breaking changes
new_repo; commit "feat(init): Start"; git tag v1.2.3
commit "feat(api)!: Drop the v1 route"
expect "breaking subject bumps the major at 1.x" 2.0.0
new_repo; commit "feat(init): Start"; git tag v1.2.3
commit "fix(api): Tweak

BREAKING CHANGE: the route moved"
expect "BREAKING CHANGE footer bumps the major at 1.x" 2.0.0

new_repo; commit "feat(init): Start"; git tag v0.3.0
commit "feat(api)!: Drop the v1 route"
expect "breaking subject bumps the minor while 0.x" 0.4.0
new_repo; commit "feat(init): Start"; git tag v0.3.0
commit "fix(api): Tweak

BREAKING CHANGE: the route moved"
expect "BREAKING CHANGE footer bumps the minor while 0.x" 0.4.0
expect "1.0.0 is declared with --override" 1.0.0 --override 1.0.0

# ---- rc channel
new_repo; commit "feat(init): Start"; git tag v0.3.0
commit "feat(api): Add search"
expect "first rc of the next minor" 0.4.0-rc.1 --channel rc
git tag v0.4.0-rc.1
expect "second rc" 0.4.0-rc.2 --channel rc
expect "stable after an rc is the same base" 0.4.0
commit "feat(api)!: Break"
expect "a new commit after rc.1 still bumps from the last stable tag" 0.4.0 --channel stable
git tag v0.4.0
expect_fail "nothing to release right after a release" "nothing to release since v0.4.0"

# ---- version ordering and the baseline
new_repo; commit "feat(init): Start"; git tag v0.9.0; git tag v0.10.0; git tag v0.10.1-rc.1
commit "fix(api): Tweak"
expect "versions sort numerically (0.10.0 > 0.9.0) and an rc is not a baseline" 0.10.1

# ---- tags that are not part of this history, and spelling
new_repo; commit "feat(init): Start"; git tag v0.3.0
git switch -q -c hotfix; commit "fix(api): Hotfix"; git tag v0.5.0; git switch -q main
commit "fix(api): Tweak"
expect "a stable tag on an unmerged branch is not the baseline" 0.3.1
new_repo; commit "feat(init): Start"; git tag v0.3.0
commit "Feat(api): Capitalized type"
expect "type case does not matter" 0.4.0
new_repo; commit "feat(init): Start"; git tag v0.3.0
commit "fix(api): Handle it"
commit 'Revert "fix(api): Handle it"'
expect "a revert does not cancel the commit it reverts" 0.3.1

# ---- the plan
new_repo; commit "feat(init): Start"; git tag v0.3.0; commit "feat(api): Add search"; git tag v0.4.0-rc.1; commit "fix(api): Tweak"
expect_plan "the plan counts the commits since the last rc" "Commits since v0.4.0-rc.1: 1" --channel rc
expect_plan "the plan lists the commits it counted" "feat(api): Add search"

# ---- override validation
new_repo; commit "feat(init): Start"; git tag v0.3.0; commit "feat(api): Add search"
expect_fail "override must be X.Y.Z" "is not X.Y.Z" --override v0.4.0
expect_fail "override must not be lower than the last release" "not greater" --override 0.2.0
expect_fail "override must not equal the last release" "not greater" --override 0.3.0
expect "override may be an rc" 0.4.0-rc.3 --channel rc --override 0.4.0-rc.3
git tag v0.4.0-rc.3
expect_fail "an existing tag is refused" "already exists" --channel rc --override 0.4.0-rc.3
expect_fail "an rc override needs the rc channel" "use --channel rc" --override 0.4.1-rc.1
expect_fail "a stable override needs the stable channel" "use --channel stable" --channel rc --override 1.0.0

# ---- a long history: the plan lists at most 50 commits, and the output is larger than a pipe
# buffer, so a pipe to `head` would kill `git log` with SIGPIPE and fail the script
new_repo; commit "feat(init): Start"; git tag v0.1.0
many_commits 4000
expect "a history of thousands of commits still releases" 0.1.1

# ---- channel and history
expect_fail "unknown channel" "channel must be stable or rc" --channel beta
git clone -q --depth 1 "file://$(pwd)" "$tmp/shallow" 2> /dev/null
if (cd "$tmp/shallow" && ! "$script" > /dev/null 2> "$tmp/stderr" && grep -q "shallow" "$tmp/stderr"); then
  ok "a shallow clone is refused"
else
  bad "a shallow clone is refused" "no shallow error"
fi

echo
echo "$passed passed, $failed failed"
[ "$failed" -eq 0 ]
```

### `.github/workflows/release.yml`

A manual release.  `dry_run` defaults to true, so a first run only plans and validates.  `validate` calls `ci.yml` (which needs `workflow_call`).  `ship` waits for the `production` environment's reviewers.  `latest` and `sha-<commit>` stay with main: deploy by the exact version tag.  `publish` runs last and starts the notes at the last stable tag, so a stable release lists everything since the previous stable, including what first shipped in an rc.  The image check is best effort: only a successful lookup counts as "exists" (a repository that does not exist yet answers "denied").

```yaml
# Release: computes the next version from Conventional Commits, validates, ships the image behind
# an approval, then creates the GitHub Release (which creates the tag).
#   Run: Actions -> Release -> Run workflow (start with `dry_run`).
# The version lives in git tags only; the image reports it (`VERSION` build arg -> `BuildInfo`).
# Needs the `production` environment (Settings -> Environments) with required reviewers.

name: Release

on:
  workflow_dispatch:
    inputs:
      channel:
        description: Release channel (rc publishes a prerelease)
        type: choice
        options: [stable, rc]
        default: stable
      version:
        description: Optional override, X.Y.Z or X.Y.Z-rc.N without a leading v (declares 1.0.0)
        required: false
      dry_run:
        description: Plan and validate only; push and tag nothing
        type: boolean
        default: true

# One real release at a time, never cancelled (one waiting for approval holds the group); dry runs
# change nothing, so they have their own group and do not queue behind it.
concurrency:
  group: release-${{ inputs.dry_run && 'dry-run' || 'release' }}
  cancel-in-progress: false

permissions:
  contents: read

env:
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  plan:
    name: Plan
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read
    outputs:
      version: ${{ steps.next.outputs.version }}
      tag: ${{ steps.next.outputs.tag }}
      previous: ${{ steps.next.outputs.previous }}
    steps:
      # Releases ship what is on main, even when dispatched from another ref.
      - name: Releases come from main
        run: |
          if [[ "$GITHUB_REF" != "refs/heads/main" ]]; then
            echo "::error::Run the release from main (ref: $GITHUB_REF)"
            exit 1
          fi

      # The version is computed from the tags and the commits since the last release.
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0

      # stdout (version, tag, previous) goes to the step outputs, the plan (Markdown) to the log
      # and the job summary.
      - name: Next version
        id: next
        env:
          CHANNEL: ${{ inputs.channel }}
          OVERRIDE: ${{ inputs.version }}
        run: |
          args=(--channel "$CHANNEL")
          if [[ -n "$OVERRIDE" ]]; then
            args+=(--override "$OVERRIDE")
          fi
          status=0
          build/next-version.sh "${args[@]}" >> "$GITHUB_OUTPUT" 2> "$RUNNER_TEMP/plan.md" || status=$?
          cat "$RUNNER_TEMP/plan.md"
          grep -v '^::' "$RUNNER_TEMP/plan.md" >> "$GITHUB_STEP_SUMMARY" || true
          exit "$status"

      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Release images are immutable: stop here, before anyone is asked to approve, if this
      # version's image already exists (a release that failed after the push).  Best effort: only a
      # successful lookup counts; a repository that does not exist yet answers "denied".
      - name: Image tag is new
        env:
          VERSION: ${{ steps.next.outputs.version }}
        run: |
          image="${IMAGE,,}"
          if docker manifest inspect "$image:$VERSION" > /dev/null 2>&1; then
            echo "::error::$image:$VERSION already exists, so that version was shipped: release the next one (version input)"
            exit 1
          fi

  # The same checks as a pull request (build, tests, lint, image smoke test), at this commit.
  validate:
    name: Validate
    needs: plan
    permissions:
      contents: read
      checks: write
      pull-requests: write
    uses: ./.github/workflows/ci.yml

  ship:
    name: Ship
    needs: [plan, validate]
    if: ${{ !inputs.dry_run }}
    runs-on: ubuntu-latest
    # Required reviewers on this environment are the approval gate.
    environment: production
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ github.sha }}

      - uses: docker/setup-buildx-action@v4

      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Again, so that re-running this job can never overwrite a pushed image.
      - name: Image tag is new
        env:
          VERSION: ${{ needs.plan.outputs.version }}
        run: |
          image="${IMAGE,,}"
          if docker manifest inspect "$image:$VERSION" > /dev/null 2>&1; then
            echo "::error::$image:$VERSION already exists: do not re-run a shipped release, release the next version"
            exit 1
          fi

      # One tag, the exact version (the commit is in the image's revision label).  `latest` and
      # `sha-<commit>` belong to main (`docker-publish.yml`).
      - id: meta
        uses: docker/metadata-action@v6
        with:
          images: ${{ env.IMAGE }}
          flavor: latest=false
          tags: type=raw,value=${{ needs.plan.outputs.version }}

      - uses: docker/build-push-action@v7
        with:
          context: .
          file: build/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            GIT_SHA=${{ github.sha }}
            VERSION=${{ needs.plan.outputs.version }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Deploy here, by the exact version tag (`$IMAGE:<version>`), using OIDC federation to
      # your cloud rather than long-lived credentials.  Deployment depends on the platform chosen
      # at scaffolding; add the steps (or call a deploy workflow) after the image is pushed.

  # Last, so a failed release leaves no tag: creating the release creates `vX.Y.Z` at this commit.
  publish:
    name: Publish GitHub Release
    needs: [plan, ship]
    if: ${{ !inputs.dry_run }}
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Create the release
        env:
          GH_TOKEN: ${{ github.token }}
          GH_REPO: ${{ github.repository }}
          TAG: ${{ needs.plan.outputs.tag }}
          VERSION: ${{ needs.plan.outputs.version }}
          PREVIOUS: ${{ needs.plan.outputs.previous }}
        run: |
          args=("$TAG" --target "$GITHUB_SHA" --title "$TAG" --generate-notes)
          # Notes cover everything since the last stable release, rc releases included.
          if [[ -n "$PREVIOUS" ]]; then
            args+=(--notes-start-tag "$PREVIOUS")
          fi
          if [[ "$VERSION" == *-rc.* ]]; then
            args+=(--prerelease)
          else
            args+=(--latest)
          fi
          gh release create "${args[@]}"
```

### `.github/workflows/pr-title.yml`

Optional.  Enforces `type(scope): Subject` on pull request titles, the input of the version calculation.

```yaml
# Release versions come from commit types, and a squash merge takes the pull request title as the
# commit message: a title that is not `type(scope): Subject` silently releases nothing.
# Optional; needs "Pull request title" as the default squash commit message (Settings -> General).

name: PR title

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

permissions:
  pull-requests: read

jobs:
  conventional-title:
    name: Conventional title
    runs-on: ubuntu-latest
    steps:
      - uses: amannn/action-semantic-pull-request@v6
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            docs
            chore
            refactor
            test
            ci
            build
            perf
            style
            revert
          # `feat(area): Sentence-case description`
          subjectPattern: ^[A-Z].*$
          subjectPatternError: "The subject must start with a capital letter: feat(area): Add a thing"
```

### `.github/release.yml`

Optional.  Shapes the generated release notes; add `categories` (by PR label) for headings.

```yaml
# Shapes the notes that `gh release create --generate-notes` writes (the release workflow).
# Notes are the merged pull request titles since the previous release.
changelog:
  exclude:
    labels:
      - ignore-for-release
    authors:
      - dependabot
```

## Edits to other files

Only with release automation; the base templates do not have them, so skipping leaves `mise run check` and CI untouched.

`.config/mise.toml`: the tasks, and the script tests in `check`:

```toml
[tasks.release-version]
description = "Print the next release version from the Conventional Commits since the last release (--channel rc, --override X.Y.Z)"
run = "build/next-version.sh"

[tasks.release-test]
description = "Test the release version script against synthetic git histories"
run = "build/next-version.test.sh"
```

```toml
[tasks.check]
run = [
  # ... the existing entries ...
  "build/next-version.test.sh",
]
```

`.github/workflows/ci.yml`: `workflow_call:` under `on:` (the release workflow runs the same checks), and a step in the backend job:

```yaml
      - name: Release version script tests
        run: build/next-version.test.sh
```

`AGENTS.md` (working agreements) and `README.md`:

```markdown
- Releases come from commit types: `feat` releases a minor, `fix` a patch, `type!:` (or a `BREAKING CHANGE:` footer in the commit body) a major (a minor while the version is 0.x); `docs`, `chore`, `ci`, `build`, `refactor`, `test`, `perf`, `style`, and `revert` release nothing.  A squash merge takes the PR title as the commit message, so PR titles are conventional too.  `mise run release-version` prints the next version; cut a release with Actions -> Release (start with `dry_run`)
```

```markdown
## Releasing

The version lives in git tags and is computed from the Conventional Commits since the last release (`mise run release-version` prints the next one).  Run Actions -> Release: `dry_run` first, then a real run (`channel` stable or rc).  It validates, pushes the image behind the `production` environment approval, and creates the GitHub Release, which creates the tag.
```

## Setup

**At scaffolding** (no GitHub needed):

1. Add the files above (`chmod +x build/*.sh`) and make the edits in "Edits to other files".  `docker-publish.yml` is already main-only (`references/containers-ci.md`).
2. Verify: `mise run release-test` passes.  `mise run release-version` prints `0.1.0` once the repository has a commit.
3. To skip: leave all of it out.  To release by pushing tags instead, use the tag-triggered variant of `docker-publish.yml` (last section).

**Once, on GitHub** (after the repository exists; in this order):

1. Merge the workflows to the default branch: a workflow can only be dispatched from there.
2. Settings -> Environments -> `production`: required reviewers (the approval gate) and deployment branches limited to `main`.
3. Settings -> General -> Pull Requests: allow squash merging with the default message "Pull request title"; consider disabling merge commits and rebase merging, so history is one conventional commit per PR.
4. A branch ruleset on `main`: require a pull request and the CI checks (`Backend`, `Front-end`, `Image`, and `Conventional title` when `pr-title.yml` is used).
5. The first image push creates `ghcr.io/<owner>/<repo>` (private) and links it to the repository through the image's source label; if the package already exists, add the repository under the package's "Manage Actions access" with write role.  Set the visibility you want.
6. Deployment: add the steps to `ship` after the push (OIDC federation to the cloud chosen in step 0, variables or secrets as needed), deploying by the exact version tag.  Until then a release publishes the image and the GitHub Release only.
7. First runs: `dry_run`, and read the plan in the job summary; then `channel` = rc for real (prerelease `v0.1.0-rc.1`: check the image, the release, the tag, and the version on `/api/health`); then stable.  The approval gate, the reusable `ci.yml` call, and the package permissions are first exercised here.

A release or tag created with `GITHUB_TOKEN` starts no other workflow: a later `on: release` or `on: push: tags` workflow needs a personal access token or a GitHub App for the release step.

## Releasing

Actions -> Release -> Run workflow: `channel` (stable or rc), `version` (blank, or an override), `dry_run` (uncheck to ship).  Dry runs have their own concurrency group, so they do not queue behind a release that waits for approval.

- **Check the plan** (job summary) before approving: the last release, the counts, the bump, and the commits.  An under-bump means a subject was not conventional: reject the approval, then fix it or override.
- **rc, then stable:** run rc until happy, then stable; the stable release is the same `X.Y.Z`.
- **Declare 1.0:** `version` = `1.0.0` on a stable run.
- **`publish` failed** (the image is pushed, the release is not): re-run the failed jobs, or create it by hand: `gh release create vX.Y.Z --target <commit> --generate-notes --notes-start-tag <previous>`.
- **`ship` failed after the push** (a deploy step): the version is spent, and the plan and `ship` refuse to overwrite its image.  Fix the deploy and deploy that image by its tag, then create the release by hand as above, or release the next version with the `version` input.
- **A bad release:** `gh release delete vX.Y.Z --cleanup-tag`; the image tag stays (immutable), so the fix is the next version.

Per project: the registry and image name, the `production` environment, the deployment, the types `pr-title.yml` accepts, and release-note categories.

## Without release automation

Release by tagging (`git tag v1.2.3 && git push origin v1.2.3`) with a tag-triggered `docker-publish.yml`.  The tag exists before the build, so a failed publish leaves a tag with no image.  Restore in `docker-publish.yml`:

```yaml
on:
  push:
    branches: [main]
    tags: ["v*"]
```

```yaml
      # Tags: `latest` on main, the version on `v*` tags, and the commit SHA on every push.
      - id: meta
        uses: docker/metadata-action@v6
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=semver,pattern={{version}}
            type=sha

      # What the app reports on /api/health: the commit always, the version on `v*` tags (`v1.2.3`
      # reports `1.2.3`); other builds report `0.0.0-dev`.
      - id: build-info
        run: |
          echo "commit=$GITHUB_SHA" >> "$GITHUB_OUTPUT"
          if [[ "$GITHUB_REF" == refs/tags/v* ]]; then echo "version=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"; fi
```

and in the build step: `build-args: |` with `GIT_SHA=${{ steps.build-info.outputs.commit }}` and `VERSION=${{ steps.build-info.outputs.version }}`.
