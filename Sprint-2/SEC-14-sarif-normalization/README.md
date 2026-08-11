# SEC-14 · SARIF/JSON Normalization

**Sprint:** 2 · **Epic:** E2 — Runner workflow, scanners & unified findings
**Repo implemented in:** [`Sentinel-AI-Sec/sentinelai-backend`](https://github.com/Sentinel-AI-Sec/sentinelai-backend) (`dev` branch)
**Layer:** `SentinelAI.Domain` / `SentinelAI.Application` / `SentinelAI.Infrastructure`

---

## What this task is

Four scanners run on the GitHub Actions runner — **Roslyn/SCS** (code), **OSV-Scanner**
(dependencies), **Trivy** and **Checkov** (infra) — and each emits its results in its own
shape. SEC-14 is the translator: it reads whatever each tool wrote and turns every single
result into one common `Finding` object (`SourceTool`, `Layer`, `Severity` 0–4, `CweId`,
`CveId`, `Message`, `Redacted`), so nothing downstream — the rule-mapper, the graph builder,
retrieval — ever has to know which tool produced a given result.

`ScanStage.Normalize` already existed as an enum value before this task ("SARIF/JSON parsed
into the unified Finding set, CWEs resolved") — nothing implemented it yet. This closes that
gap.

---

## What was built

- **`IFindingExtractor`** — one contract, four implementations (`RoslynSarifExtractor`,
  `OsvJsonExtractor`, `TrivySarifExtractor`, `CheckovSarifExtractor`).
- **A shared, tolerant SARIF reader** (`SarifReader` / `SarifFindings`) that reads both SARIF
  v1 and v2 shapes without crashing on the difference.
- **`SeverityScale`** — collapses each tool's own severity vocabulary onto the shared 0–4
  scale from `docs/Data_Contracts.md`.
- **`LinkingKeys`** — pulls CWE/CVE identifiers out of each tool's native fields.
- **OSV-Scanner reads native JSON, not a SARIF export** — this was the whole reason
  OSV-Scanner replaced Dependency-Check in the toolchain: the native shape keeps the
  CVE/GHSA ids that a SARIF export can drop.
- **Layer tagging:** Roslyn → `Code`, OSV → `Dep`, Checkov → `Infra`, Trivy → `Dep` when the
  finding carries a CVE, else `Infra`.
- **`NormalizationPipeline`** (Application layer) — reads a scan job's stored bundle via
  `IBundleStore`, routes each findings file to the right extractor by name, and returns one
  combined `IEnumerable<Finding>`. A single unreadable file is skipped and logged, not fatal
  to the whole scan.
- **`NodeRef` deliberately left empty at this stage.** Building it is the graph stage's job
  (SEC-16+); SEC-14's only rule here is *never* build a node key by string concatenation —
  only `NodeId` factory methods are allowed to do that, per `docs/Data_Contracts.md`.

---

## Problems hit (and what they taught me)

**1. SARIF isn't one format, it's two.**
SARIF v1 and v2 disagree on where a rule id and a CWE tag actually live in the result
object. A reader written against only one version silently returns nothing on the other —
no exception, just an empty finding list, which looks exactly like a clean scan. The fix was
a single tolerant reader (`SarifReader`) that both extractors call into, rather than four
copies of "guess the shape."

**2. The file-extension assumption I shipped was wrong, and my own tests didn't catch it.**
I built `OsvJsonExtractor` to read `osv.json`, on the stated rationale that OSV's SARIF
export drops CVE/GHSA ids — a fact that's actually true of **Dependency-Check**, the tool
OSV-Scanner replaced, not of OSV-Scanner itself. In reality neither our GitHub Action nor the
fixtures repo ever produces an `osv.json` file — `osv.sarif` is what actually lands in the
bundle, and OSV's own SARIF *does* carry the CVE (in `ruleId`) and the package coordinate (in
the message text). Because my unit tests supplied hand-crafted `osv.json` fixtures that
matched my own (wrong) assumption, everything passed while the real pipeline silently
discarded the entire dependency layer — a whole scanner's output vanishing at a `LogDebug`
line, with green tests the whole time. This was caught in a later pass that ran the pipeline
against the fixture's actual scanner output rather than hand-built samples, and is now fixed
by having the extractor detect and read either shape. The lesson: **a test fixture you wrote
yourself can only ever prove your assumption is internally consistent — it can't prove the
assumption is true.** Testing against one real captured scanner run per tool is what actually
would have caught this before merge.

**3. "Just skip a bad file" needed a severity, not just a try/catch.**
Early on, an unroutable or malformed findings file logged at `LogDebug` and moved on — correct
for resilience, wrong for visibility. A whole scanner going missing from a bundle must be loud
enough to notice; it was quietly downgraded to a warning so a disappearing tool doesn't read
as a clean scan.

**4. Deduplication has to happen one layer up, not here.**
It's tempting to dedupe inside each extractor, but Trivy in particular reports the same
underlying issue more than once per file in some configurations. Collapsing duplicates is
correctly SEC-16's job on the *unified* set (it needs cross-tool visibility to do it safely) —
SEC-14 stays a pure per-file translator and doesn't try to be clever about it.

---

## Files to push into `sprint2/SEC-14-sarif-normalization/`

Copy these from the `sentinelai-backend` repo, preserving their relative paths under a `src/`
and `tests/` split so the structure stays recognizable:

```
sprint2/SEC-14-sarif-normalization/
├── README.md                                          # this file
├── src/
│   ├── SentinelAI.Domain/
│   │   └── Abstractions/
│   │       ├── IFindingExtractor.cs
│   │       ├── FindingExtractionException.cs
│   │       ├── ScannerNames.cs
│   │       └── Repositories/
│   │           └── IBundleStore.cs                    # modified: + OpenFindingsAsync
│   ├── SentinelAI.Application/
│   │   ├── DependencyInjection.cs                     # modified: registers the pipeline
│   │   └── Features/Scan/Normalization/
│   │       └── NormalizationPipeline.cs
│   └── SentinelAI.Infrastructure/
│       ├── DependencyInjection.cs                      # modified: registers the extractors
│       ├── Implementation/Repositories/
│       │   └── FileSystemBundleStore.cs                # modified: + OpenFindingsAsync impl
│       └── Normalization/
│           ├── SarifReader.cs
│           ├── SarifFindings.cs
│           ├── SeverityScale.cs
│           ├── LinkingKeys.cs
│           ├── RoslynSarifExtractor.cs
│           ├── OsvJsonExtractor.cs                     # later renamed OsvExtractor.cs — see note below
│           ├── TrivySarifExtractor.cs
│           └── CheckovSarifExtractor.cs
└── tests/
    └── SentinelAI.Infrastructure.Tests/
        ├── SentinelAI.Infrastructure.Tests.csproj
        └── Normalization/
            ├── Fixtures.cs
            ├── RoslynSarifExtractorTests.cs
            ├── OsvJsonExtractorTests.cs                # later OsvExtractorTests.cs — see note below
            ├── TrivySarifExtractorTests.cs
            ├── CheckovSarifExtractorTests.cs
            ├── NormalizationPipelineTests.cs
            └── FileSystemBundleStoreFindingsTests.cs
```

Also worth including for context (small, modified rather than added):
- `SentinelAI.slnx` — registers the new `SentinelAI.Infrastructure.Tests` project
- `tests/SentinelAI.Integration.Tests/Scan/FakeScanInfrastructure.cs` — test double updated to match the new bundle-reading surface

> **Note on the OSV rename.** The file above is listed under its name at merge time
> (`OsvJsonExtractor.cs` / `OsvJsonExtractorTests.cs`). A later fix (documented in "Problems
> hit," item 2) renamed it to `OsvExtractor.cs` / `OsvExtractorTests.cs` once it was reading
> both `osv.json` and `osv.sarif`. If you're copying from the current `dev` branch rather than
> the original SEC-14 merge commit, use the current filename — it's the same component, just
> corrected and renamed to reflect what it actually does now.

---

## How to verify it works

1. `dotnet test tests/SentinelAI.Infrastructure.Tests` — one happy-path + one malformed-input
   case per extractor, plus a pipeline-level test asserting the combined count and correct
   `Layer` tagging.
2. Point `NormalizationPipeline` at the fixture's real bundle (not hand-built samples) and
   confirm every scanner that ran actually contributes at least one finding — this is the
   check that would have caught problem #2 above before merge.
3. Spot-check that dependency findings keep their `CveId` — the entire reason OSV-Scanner is
   read natively instead of via SARIF export.

---

## References

- `docs/Data_Contracts.md` — the `Finding` shape and the `NodeId` canonical-key rule
- `SentinelAI_Product_Backlog_v2_1.md` — SEC-14 acceptance criteria (Epic E2)
- `Sprint2.md` — the original task write-up and step-by-step plan
