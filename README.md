# SapMachine 21.0.10.0.1 demo

Reproducer for a customer issue where `actions/setup-java` fails (or installs an
unexpected version) when `distribution: sapmachine` is used with a bare version
string like `"21.0.10"`.

**References:**
- Failing workflow run: https://github.com/cap-java/cds-feature-console/actions/runs/23295977729/workflow
- Customer PR: https://github.com/cap-java/cds-feature-console/pull/12

## The problem

SapMachine publishes Java releases with a **four-part version number**:

```
21.0.10.0.1   ← SapMachine release name (OpenJDK 21.0.10, SapMachine build 0.1)
```

`actions/setup-java` resolves SapMachine versions using the
[GitHub Releases API](https://api.github.com/repos/SAP/SapMachine/releases) and
stores them internally as SemVer.  The SapMachine-specific build qualifier
(`0.1`) is encoded as **SemVer build metadata**:

```
21.0.10+0.1   ← SemVer representation understood by actions/setup-java
```

When a workflow specifies `java-version: "21.0.10"` (no build metadata), there
is **no SapMachine release that exactly matches** — the only available build is
`21.0.10+0.1`.  This causes the setup step to either fail outright or fall back
to an unintended version.

## Affected customer workflow

Source: [cap-java/cds-feature-console – workflow run #23295977729](https://github.com/cap-java/cds-feature-console/actions/runs/23295977729/workflow)
and [PR #12](https://github.com/cap-java/cds-feature-console/pull/12).

```yaml
# cap-java/cds-feature-console – .github/workflows/build.yml
strategy:
  matrix:
    java-version: ["17.0.18", "21.0.10"]   # ← "21.0.10" does NOT match any SapMachine release
steps:
  - uses: ./.github/actions/build
    with:
      java-version: ${{ matrix.java-version }}
```

## Fix

Append the SapMachine build qualifier as SemVer build metadata:

| SapMachine release name | Correct `java-version` value |
|-------------------------|------------------------------|
| `17.0.14.0.1`           | `17.0.14+0.1`                |
| `21.0.10.0.1`           | `21.0.10+0.1`                |

```yaml
strategy:
  matrix:
    java-version: ["17.0.14+0.1", "21.0.10+0.1"]   # ✅
```

## Repo layout

| Path | Purpose |
|------|---------|
| `.github/workflows/customer-replica.yml` | **Direct replica** of the failing customer workflow (broken `"21.0.10"` matrix) |
| `.github/workflows/ci.yml` | Side-by-side comparison — broken matrix vs. fixed matrix |
| `.github/actions/build/action.yml` | Composite action mirroring the customer's setup (setup-java + Maven) |
| `pom.xml` | Minimal Maven project so `mvn verify` has something to execute |

