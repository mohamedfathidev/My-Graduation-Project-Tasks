# SEC-14: SARIF/JSON normalization

**Task:** SentinelAI backlog, Epic E2
**My role:** Owner
**Status:** 🔄 In progress

## Context

Converts each scanner's raw output (SARIF v1/v2, plus Dependency-Check's
native JSON) into one common `Finding` shape — severity normalized to a
0–4 scale, tagged by `source_tool` and `layer`.

## Known input so far (from SEC-12)

- Roslyn/SCS → SARIF **v1.0.0** (`resultFile`, no `driver` object)
- OSV-Scanner, Trivy, Checkov → SARIF **v2.1.0** (`physicalLocation`, has `driver`)

The normalizer needs to tolerate both shapes — real samples of each are
already in [`SEC-12-scanner-validation/evidence/`](../SEC-12-scanner-validation/evidence/).

## Next

- [ ] Define severity mapping per tool
- [ ] Handle v1 vs v2 shape differences
- [ ] Preserve CWE/CVE as linking keys
- [ ] Write-up + evidence once complete