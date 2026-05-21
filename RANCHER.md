# Rancher Fluent Bit Fork

SUSE-based fork of `fluent/fluent-bit` with automated rebuilds and security workflows.

## Overview

This fork maintains a **frozen code version (v3.1.8)** for stability while providing
security updates through rebuilds. Application code (and the vendored C libraries
under `lib/`) is NOT modified — security comes from rebuilding with fresh SUSE BCI
base images and updated zypper runtime packages.

**Security Model:**
- ✅ Security through fresh builds with updated BCI base image + zypper packages
- ❌ We do NOT cherry-pick code changes from newer upstream versions
- ❌ Vendored C libs (`lib/ctraces`, `lib/cfl`, `lib/cmetrics`, etc.) stay frozen
- ❌ CMakeLists.txt build configuration stays frozen

## Build pipeline

| Layer | Mechanism | What it owns |
|---|---|---|
| Daily cron | `.github/workflows/auto-update-bci.yaml` (~12:13 UTC) | Bumps both bci-base + bci-minimal digests |
| Continuous | `renovate.json5` | GitHub Actions updates only |
| Triggered | `.github/workflows/cve-response.md` (agentic) | Long-tail CVE fixes (zypper package bumps, vendored lib documentation) |
| Weekly | `.github/workflows/weekly-health-check.md` (agentic) | Meta-monitor — catches when automation stalls |
| Push/Tag/PR | `.github/workflows/build.yaml` | Multi-arch SUSE image build (production + debug) |

## Why no auto-update-go / no goreleaser

Unlike the Go forks (`logging-operator`, `config-reloader`), fluent-bit is a C/CMake
project. The build happens entirely inside Docker (no pre-built binary copied in).
Go-specific automation (`auto-update-go.yaml`, `govulncheck`) is inapplicable here.

## Image variants

`Dockerfile.suse` produces two tagged variants:

| Variant | Base | Size | Use |
|---|---|---|---|
| `production` | `bci-minimal` | ~50-80 MB | Default runtime — minimal attack surface |
| `debug` | `bci-base` | ~250 MB | In-pod debugging (gdb, strace, tcpdump, vim, etc.) — referenced by `rancher-logging` chart as `images.fluentbit_debug` |

Both are built and pushed by `build.yaml` for every push to `rancher-main`.

## Coexistence with upstream

We deleted the upstream `.github/workflows/` (35 files of PR-gating, fuzz tests,
integration suites, perf tests, package builds, staging releases, etc.) that don't
apply to a rebuild fork. The upstream `dockerfiles/Dockerfile` is left in place
for reference; our `Dockerfile.suse` is the one our automation uses.

## Local build

```bash
# Production variant
docker buildx build -f Dockerfile.suse --target production \
  -t fluent-bit:dev .

# Debug variant
docker buildx build -f Dockerfile.suse --target debug \
  -t fluent-bit:dev-debug .

# Multi-arch
docker buildx build -f Dockerfile.suse --target production \
  --platform linux/amd64,linux/arm64 \
  -t fluent-bit:dev .
```

## Images

Built images push to GHCR:

- `ghcr.io/manno/fluent-bit:dev-<sha>` — production
- `ghcr.io/manno/fluent-bit:dev-<sha>-debug` — debug
- `ghcr.io/manno/fluent-bit:<version>-suse1` — production release tag
- `ghcr.io/manno/fluent-bit:<version>-suse1-debug` — debug release tag

## Chart consumption

The `rancher-logging` 4.10 chart references this image as `images.fluentbit` and
`images.fluentbit_debug` in `packages/rancher-logging/4.10/generated-changes/patch/values.yaml.patch`
(ob-team-charts repo). Once images are stable, that patch will be updated to
point at `ghcr.io/manno/fluent-bit` instead of the upstream mirror.

## Windows note

The chart's `images.nodeagent_fluentbit` is a Windows variant. SUSE BCI is
Linux-only — we keep using the existing Windows image and do NOT build a SUSE
variant for it.

## Upstream

- **Upstream**: https://github.com/fluent/fluent-bit
- **Fork point**: v3.1.8 (per `CMakeLists.txt`)
- **Sync strategy**: None — we do NOT sync code changes from upstream
- **Security strategy**: Rebuild with fresh BCI base + zypper packages

## References

- POC state: `docs/logging/fork/STATE.md` in `ob-team-charts`
- Sibling forks: `manno/logging-operator`, `manno/config-reloader`, `manno/fluentd`
