# SEC-12: Scanner runners on the runner

**Task:** SentinelAI backlog, Epic E2 — Runner workflow, scanners & unified findings
**My role:** Sole owner
**Status:** ✅ Core validation complete — generalization to `action.yml` pending SEC-11

---

## Overview about SEC-11 Approach Before action.yml
![Workflow Diagram](/Sprint-2/SEC-12-scanner-validation/Related/images/test-flow.png)
## Context

SentinelAI scans a target repo across three layers — code, dependency,
infra — using four tools (Roslyn/Security Code Scan, OSV-Scanner, Trivy,
Checkov) inside a GitHub Actions composite action. SEC-11 builds that
action's skeleton; **SEC-12 is the task that makes each scanner inside it
actually work**: pinned versions, valid output, fault-tolerant to any one
tool failing.

## My approach

SEC-11's `action.yml` didn't exist yet, so I built a standalone workflow —
`local-test.yml` — directly in the fixture repo to validate all four
scanners independently, on a real `ubuntu-latest` runner, before any
integration point existed. Validated each tool locally on Windows first,
then ported to Linux, stripping Windows-only workarounds (App Execution
Alias path hacks, `.exe` extensions, manual binary placement) that don't
apply on the real runner.

## Toolchain decisions

| Decision | Reason |
|---|---|
| Dropped Semgrep | Thin C# coverage; Roslyn/SCS already covers the same categories |
| Replaced Dependency-Check → OSV-Scanner | Dependency-Check's CPE matching had a structural gap — zero matches even on known vulnerable packages |
| Pinned: Checkov 3.2.0, Trivy 0.72.0, OSV-Scanner 2.4.0, SCS 5.6.7 | Reproducibility |

Roslyn/SCS isn't a standalone CLI call — it's a NuGet `PackageReference`
that runs as a side effect of `dotnet build -p:ErrorLog=...`.

## Problems hit

**1. One scanner's exit code killed every scanner after it.**
Checkov, Trivy, and OSV-Scanner all exit non-zero when they *find* real
issues — expected, not a crash. `bash -e` (GitHub Actions' default) aborts
the whole step on the first non-zero exit, so Checkov finding 28 issues
silently cancelled Trivy, OSV-Scanner, and Roslyn.
→ Fix: `set +e` at the top of each block, `exit 0` at the end — preserves
fault tolerance while `echo "exit=$?"` still surfaces the real code.

**2. Checkov's `--output-file-path` creates a directory, not a file.**
It writes a fixed filename (`results_sarif.sarif`) inside whatever path
you give it.
→ Fix: `mv .../results_sarif.sarif bundle/findings/checkov-infra.sarif`.

**3. OSV-Scanner silently scanned the filesystem root.**
Log showed `Starting filesystem walk for root: /` — a leaked `cd` from an
earlier command in the same shell session.
→ Fix: split each scanner into its own step so each starts fresh in
`$GITHUB_WORKSPACE`.

**4. Trivy's `fs` subcommand rejected the flag its own deprecation
warning recommended.** `--output-file` doesn't exist for `fs` in v0.72.0,
despite `config`'s warning implying the migration was complete.
→ Fix: kept `--output` for `fs` specifically; documented rather than
silently worked around.

**5. `dotnet build: MSB1009 Project file does not exist`.**
Wrong assumed path (`OrderApp.csproj` at repo root vs. actual
`src/OrderApp/OrderApp.csproj`).
→ Fix: corrected path; added a `find . -name "*.csproj"` diagnostic so
the real path is confirmed, never assumed.

**6. Trivy double-counted every dependency finding.**
`dotnet build` (step 5) creates `bin/OrderApp.deps.json` — a second
manifest Trivy scans alongside `packages.lock.json`.
→ Fix: `trivy fs --skip-dirs "**/bin,**/obj" ...`.

## Evidence

Real SARIF output from a passing run, in [`evidence/`](./evidence/):

| File | Results | Flagship finding |
|---|---|---|
| `roslyn.sarif` | 5 | `SCS0028` — unsafe deserialization (CWE-502) |
| `osv-scanner.sarif` | 14 | `CVE-2024-21907` — Newtonsoft.Json |
| `trivy.sarif` | 48 | `AWS-0345` — S3 wildcard IAM policy |
| `checkov-infra.sarif` | 28 | `CKV_AWS_290/289/288/355` — same S3 policy, different rule logic |
| `checkov-docker.sarif` | 3 | `CKV_DOCKER_3` — container runs as root |

Trivy and Checkov flagging the same wildcard policy through entirely
different rule logic is a live example of the multi-scanner correlation
value the project is built around, not a defect.

Workflow: [`workflows/local-test.yml`](./workflows/local-test.yml)

---

## Next: generalizing into `action.yml`

## Overview general
![Workflow Diagram](/Sprint-2/SEC-12-scanner-validation/Related/images/general.png)

*This section will be completed once SEC-11 lands and this task closes.*

The validated logic above is fixture-specific (hardcoded paths like
`src/OrderApp/OrderApp.csproj`, `infra/`). Before merging into the real
`action.yml`, it needs to work on **any** .NET/Terraform/Docker repo, not
just this one — via auto-discovery instead of hardcoded paths:

- `.csproj`/`.sln` located via `find`, not assumed
- Terraform directories discovered via `find . -name "*.tf"`, scanned per
  directory rather than assuming a single `infra/` folder
- OSV-Scanner run in `--recursive` mode so it discovers lockfiles for any
  ecosystem, not just NuGet
- Each layer's scanner step conditioned on that layer actually being
  present (`if: steps.discover.outputs.dotnet_target != ''`), so a
  non-.NET repo skips cleanly instead of failing

A draft of this generalized version exists at
[`workflows/action.yml`](./workflows/action.yml),
tested against the same fixture to confirm it reproduces every finding
above through discovery rather than hardcoded paths. Final integration
into `action.yml` — with `shell: bash` added per step and the exit-code
vs. output-validity distinction from the production hardening notes above
— is pending SEC-11.