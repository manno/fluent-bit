---
description: |
  Weekly health check for the fluent-bit SUSE rebuild pipeline.
  Monitors build status, auto-update-bci bot liveness, open security/dependency
  PRs, and BCI digest freshness for both bci-base (builder + debug) and
  bci-minimal (production runtime).

  Meta-monitor — flags when the rebuild automation stalls. Skips anything
  that doesn't speak to "is the rebuild pipeline producing fixed images?"

on:
  schedule: weekly on monday
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read
  actions: read

network: defaults

safe-outputs:
  create-issue:
    max: 1
    title-prefix: "[weekly-health-check] "
    labels: [weekly-health-check, automated]

tools:
  bash: ["git:*", "gh:*", "sed", "grep", "cat", "echo", "date", "curl", "jq", "docker:*", "awk", "head", "tail"]
  github:
    toolsets: [actions, pull_requests]

timeout-minutes: 20
---

# Weekly Health Check — Fluent Bit (SUSE Rebuild)

Generate the weekly health report for `${{ github.repository }}`.

This fork rebuilds upstream `fluent/fluent-bit` v3.1.8 with fresh SUSE BCI
base images to address CVEs. Application code and vendored C libraries are
**frozen** — we do NOT cherry-pick. The release pipeline:

- `auto-update-bci.yaml` — daily; opens a PR when either bci-base or bci-minimal digest changes
- `renovate.json5` — GitHub Actions updates (auto-merge on vuln + patch/minor)
- `cve-response.md` — agentic CVE fix for runtime zypper packages and vendored-lib tracking

**This report's job is to catch when that automation stalls.**

Note: unlike the Go forks, fluent-bit has no `auto-update-go.yaml` (C project),
no `go.mod`, and no `govulncheck` equivalent for in-tree code (the vendored C
libs are frozen). Image-level CVE scanning is delegated to the rancher
image-scanning team's pipeline (which triggers our `cve-response.md`).

---

## Step 1 — Build status on rancher-main

```bash
gh run list --repo ${{ github.repository }} --branch rancher-main --limit 30 \
  --json conclusion,status,name,workflowName,createdAt,event,headSha,url
```

For each distinct `workflowName`, find the most recent run on `rancher-main`.
Workflows to expect: `Build SUSE Image`, `Auto-Update SUSE BCI`,
`Weekly Health Check`. If a workflow you expect is missing entirely, treat
that as a finding.

A failing run on `rancher-main` blocks CVE-fix PRs from merging cleanly.

## Step 2 — Auto-update bot liveness

`auto-update-bci.yaml` IS the rebuild pipeline. If it stops, CVE rebuilds stop.

```bash
gh run list --repo ${{ github.repository }} --workflow auto-update-bci.yaml --limit 5 \
  --json conclusion,createdAt,event,url
```

Record:
- Age of the most recent successful run (expected: < 36 h)
- Conclusion of the last 5 runs (success rate)
- Whether there's an open PR from the bot:
  ```bash
  gh pr list --repo ${{ github.repository }} --state open --head auto-update-suse-bci \
    --json number,title,createdAt,url
  ```

**Escalation rules:**
- Last successful run > 36 h ago → bot stalled
- Most recent run failed → investigate
- Open PR from bot older than 7 days with green checks → why hasn't it merged?

## Step 3 — Open PR analysis

```bash
gh pr list --repo ${{ github.repository }} --state open --limit 50 \
  --json number,title,labels,createdAt,updatedAt,author,isDraft,url,headRefName,statusCheckRollup
```

Group by labels:

| Group | Label(s) | Source |
|---|---|---|
| CVE fixes | `cve-fix` | `cve-response.md` |
| CVE tracking (vendored libs) | `cve-tracking` | `cve-response.md` (frozen-vendored-lib strategy) |
| Security | `security`, `vulnerability` | Renovate vuln alerts |
| SUSE BCI | `suse-bci-update` | `auto-update-bci.yaml` |
| GitHub Actions | `ci`, `github-actions` | Renovate |
| Other deps | `dependencies` | Renovate |
| Untagged | (none of above) | Manual |

For each group: count, oldest PR/issue age in days, count with failing checks.

**Escalation rules:**
- Any `cve-fix` PR open > 3 days → review escalation
- Any `cve-tracking` issue open > 14 days with no upstream movement → flag for security team
- Any `security`/`vulnerability` PR present (should auto-merge) → investigate
- Any group with failing checks → list the PRs

## Step 4 — BCI digest drift (cross-check against bot)

The point: **if drift exists AND no open auto-update-bci PR exists, the bot
failed silently.**

For each BCI variant used in `Dockerfile.suse`:

```bash
for variant in bci-base bci-minimal; do
  CURRENT=$(grep -E "^FROM registry\.suse\.com/bci/${variant}" Dockerfile.suse | head -1 \
            | sed -nE 's|.*@(sha256:[a-f0-9]+).*|\1|p')
  LATEST=$(docker buildx imagetools inspect "registry.suse.com/bci/${variant}:latest" \
            --format '{{json .}}' 2>/dev/null | jq -r '.manifest.digest' || echo unknown)
  AGE=$(git log -1 --format=%cr -- Dockerfile.suse)
  echo "${variant}: current=${CURRENT} latest=${LATEST} dockerfile_last_changed=${AGE}"
done
```

If either variant has drift AND no open `auto-update-suse-bci` PR exists, that
is a **High Priority** finding (bot is silently broken).

## Step 5 — Vendored library inventory (static)

These libraries are frozen at v3.1.8 — drift is **expected**. The point of
listing them is to give the security team a quick reference for what's pinned
when image scanning flags a CVE in a vendored lib.

```bash
ls -1 lib/ 2>/dev/null | grep -v -E '^(README|LICENSE)' | sort
```

Report as a static list — no comparison against upstream (we don't track that).

## Step 6 — Emit the report

**Emit the full markdown report as your final response.** The `safe-outputs`
handler will turn it into a GitHub issue.

```markdown
# Health Report — Week of <YYYY-MM-DD>

## Weekly Health Report — Fluent Bit (SUSE Rebuild)

**Repository**: `${{ github.repository }}`
**Report date**: <YYYY-MM-DD>
**Branch**: `rancher-main`
**Frozen at**: v3.1.8
**Pipeline status**: <one-line: 🟢 healthy / 🟡 drift / 🔴 broken>

---

### Build health (most recent run per workflow on `rancher-main`)

| Workflow | Conclusion | Age | Run |
|---|---|---|---|
| Build SUSE Image (production) | ✅/❌ | Xh | <url> |
| Build SUSE Image (debug) | ✅/❌ | Xh | <url> |
| Auto-Update SUSE BCI | ✅/❌ | Xh | <url> |

<If any failed, list one bullet per failure with run URL.>

---

### Auto-update bot liveness

| Bot | Last success | Last 5 runs | Open PR |
|---|---|---|---|
| auto-update-bci | Xh ago | ✅✅✅✅✅ | #N or none |

<If the bot stalled (no success in >36h, OR last run failed):>
> 🚨 **auto-update-bci stalled** — last successful run Xh ago. The rebuild
> pipeline is not running. See <run-url>.

---

### Open PRs / Issues

| Group | Count | Oldest | Failing checks |
|---|---|---|---|
| cve-fix | X | Nd | Y |
| cve-tracking (vendored) | X (issues) | Nd | n/a |
| security / vulnerability | X | Nd | Y |
| suse-bci-update | X | Nd | Y |
| ci / github-actions | X | Nd | Y |
| dependencies (other) | X | Nd | Y |
| untagged | X | Nd | Y |

---

### BCI digest drift

| Variant | Current | Latest | Status |
|---|---|---|---|
| bci-base (builder + debug) | `<short>` | `<short>` | ✅ in sync / ⚠️ drift / 🚨 drift + bot silent |
| bci-minimal (production runtime) | `<short>` | `<short>` | ✅ / ⚠️ / 🚨 |

`Dockerfile.suse` last changed: <relative-time>

---

### Vendored libraries (frozen at v3.1.8)

These are not auto-updated. CVEs here go through `cve-response.md` with the
`frozen-vendored-lib` strategy (opens a tracking issue, not a PR).

<bulleted list of `lib/*` directories>

---

### 🎯 Action items

Generate strictly from the data above. Empty sections → write "None."

**High priority** (rebuild pipeline broken or unaddressed CVE):
- ...

**Medium priority** (drift accumulating or stale CVE-tracking issues):
- ...

**Low priority** (nuisance / housekeeping):
- ...

---

📅 **Next report**: <next Monday date>
🤖 Generated by [Weekly Health Check](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})

<!-- gh-aw-workflow-id: weekly-health-check -->
```

That markdown block IS your final response — emit it verbatim with placeholders
filled in. Do not run further tool calls after emitting it.
