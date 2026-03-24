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

## Real root cause

The real problem encountered in the customer's run is *not* GitHub Actions itself
but the Maven ecosystem in combination with the SapMachine JDK build naming.
The `maven-enforcer-plugin` (and Maven's model parser in some configurations)
can fail to process the POM when a JDK with the SapMachine four-part version
string (e.g. `21.0.10.0.1`) is installed as the active JVM.

## How to reproduce locally

1. Download the SapMachine 21.0.10.0.1 build (the macOS .jdk bundle) and place it in `~/Downloads`.
2. Install it with SDKMAN (or set JAVA_HOME manually):

```bash
# install the local JDK with sdkman
sdk install java 21.0.10.0.1 ~/Downloads/sapmachine-jdk-21.0.10.0.1.jdk/Contents/Home

# switch to that JDK
sdk use java 21.0.10.0.1
```

3. From the repo root run:

```bash
mvn package
```

You should see the Maven model parse error similar to the one from the customer's run:

```
[INFO] Scanning for projects...
[ERROR] [ERROR] Some problems were encountered while processing the POMs:
[FATAL] Non-parseable POM /path/to/pom.xml: Unrecognised tag: 'java.version' (position: START_TAG seen ...</pluginManagement>\n      <java.version>... @28:21)  @ line 28, column 21
 @ 
[ERROR] The build could not read 1 project -> [Help 1]
[ERROR]   
[ERROR]   The project  (/path/to/pom.xml) has 1 error
[ERROR]     Non-parseable POM /path/to/pom.xml: Unrecognised tag: 'java.version' (position: START_TAG seen ...</pluginManagement>\n      <java.version>... @28:21)  @ line 28, column 21 -> [Help 2]
```

This demonstrates the issue occurs when the JVM's version string is the four-part
SapMachine name; Maven's POM parser (or a plugin) ends up reading the active
JDK information and fails to parse certain tags in some environments.

## Notes / mitigations

- Reverting to a simpler POM (remove enforcer/plugin sections that reference
  Java properties) avoids the immediate failure — this repository contains a
  simplified `pom.xml` you can use for debugging.
- Using a JDK whose reported version string is the conventional semver form
  (as `actions/setup-java` exposes it, e.g. `21.0.10+0.1`) or using a JDK
  distribution that reports a simpler version can avoid the problem.
- The CI repro in this repo focuses on `mvn package` with SapMachine `21.0.10+0.1`.

