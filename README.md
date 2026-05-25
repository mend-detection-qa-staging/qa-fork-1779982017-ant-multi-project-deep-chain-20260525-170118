# ant-multi-project-deep-chain

Project #9 (P1) from the Ant coverage plan. A 10-module Ant multi-project layout chained
via Ivy's filesystem resolver: `mod01 -> mod02 -> ... -> mod10`. Tests both catalog pattern
#8 (`multi-project-filesystem`) and catalog pattern #20 (`complex-deep-chain`).

---

## Catalog patterns exercised

| Catalog ID | Pattern name | What it exercises |
|---|---|---|
| #8 | `multi-project-filesystem` | 10 sub-projects linked via Ivy filesystem resolver; each module is a separate Mend project candidate |
| #20 | `complex-deep-chain` | Chain traversal depth >= 10; validates Mend does not truncate at any internal cap |

---

## Mode-3 reality (CRITICAL)

The bolt-4 SCM scanner does NOT invoke Mode-1 Ivy resolution (`ant.ivyResolveDependencies`).
It falls through to **Mode-3 default path scan**: SHA1 fingerprinting of binary files (`.jar`,
`.war`, `.dll`, etc.) found anywhere under the repo root.

For this probe to produce ANY Mode-3 output, each module hosts exactly one JAR in its `lib/`
directory. The 10 JARs are all real Maven Central artifacts, pre-resolved and committed.
Mode-3 fingerprints them and emits 10 flat direct dependencies with no transitive structure.

The Ivy configuration (`ivy.xml`, `ivysettings.xml`, `whitesource.config`) is aspirational:
it documents the intended Mode-1 chain structure and will activate if/when the SCM scanner
gains `ant.ivyResolveDependencies` support. Until then, the regression value is the 10-JAR
fingerprint set.

---

## Multi-project chain structure

```
ant-multi-project-deep-chain/
  build.xml                      (umbrella: <subant> to all 10 mod*/build.xml)
  ivysettings.xml                (shared: filesystem + ibiblio chain resolver)
  .whitesource                   (Bucket-A: java pinned to 17)
  whitesource.config             (aspirational Mode-1 flags)
  .gitignore                     (does NOT exclude lib/ — JARs must be committed)
  mod01/
    build.xml  ivy.xml  lib/commons-collections4-4.4.jar
  mod02/
    build.xml  ivy.xml  lib/commons-lang3-3.12.0.jar
  mod03/
    build.xml  ivy.xml  lib/commons-io-2.11.0.jar
  mod04/
    build.xml  ivy.xml  lib/junit-4.13.2.jar
  mod05/
    build.xml  ivy.xml  lib/hamcrest-core-1.3.jar
  mod06/
    build.xml  ivy.xml  lib/slf4j-api-1.7.36.jar
  mod07/
    build.xml  ivy.xml  lib/commons-text-1.10.0.jar
  mod08/
    build.xml  ivy.xml  lib/commons-codec-1.11.jar
  mod09/
    build.xml  ivy.xml  lib/httpcore-4.4.16.jar
  mod10/
    build.xml  ivy.xml  lib/httpclient-4.5.14.jar
```

Ivy chain (Mode-1, aspirational):

```
mod01 → mod02 → mod03 → mod04 → mod05
                                  ↓
                        mod10 ← mod09 ← mod08 ← mod07 ← mod06
```

`mod10` is the terminal leaf; it declares the Maven Central external dep (`httpclient`)
that would propagate back up the chain when Mode-1 resolution is active.

---

## JAR inventory

| Module | JAR | Version | SHA1 | Source |
|---|---|---|---|---|
| mod01 | commons-collections4-4.4.jar | 4.4 | `62ebe7544cb7164d87e0637a2a6a2bdc981395e8` | Maven Central |
| mod02 | commons-lang3-3.12.0.jar | 3.12.0 | `c6842c86792ff03b9f1d1fe2aab8dc23aa6c6f0e` | Maven Central |
| mod03 | commons-io-2.11.0.jar | 2.11.0 | `a2503f302b11ebde7ebc3df41daebe0e4eea3689` | Maven Central |
| mod04 | junit-4.13.2.jar | 4.13.2 | `8ac9e16d933b6fb43bc7f576336b8f4d7eb5ba12` | Maven Central |
| mod05 | hamcrest-core-1.3.jar | 1.3 | `42a25dc3219429f0e5d060061f71acb49bf010a0` | Maven Central |
| mod06 | slf4j-api-1.7.36.jar | 1.7.36 | `6c62681a2f655b49963a5983b8b0950a6120ae14` | Maven Central |
| mod07 | commons-text-1.10.0.jar | 1.10.0 | `3363381aef8cef2dbc1023b3e3a9433b08b64e01` | Maven Central |
| mod08 | commons-codec-1.11.jar | 1.11 | `3acb4705652e16236558f0f4f2192cc33c3bd189` | Maven Central |
| mod09 | httpcore-4.4.16.jar | 4.4.16 | `51cf043c87253c9f58b539c9f7e44c8894223850` | Maven Central |
| mod10 | httpclient-4.5.14.jar | 4.5.14 | `1194890e6f56ec29177673f2f12d0b8e627dec98` | Maven Central |

---

## Expected dependency tree

Mode-3 flattens all JARs as direct dependencies (no transitive structure):

- 10 direct dependencies, 0 transitive
- All 10 JAR filenames appear as `artifactId` in Mend's output
- Mend emits `sha1` for each entry (Mode-3 SHA1 fingerprint)
- `source: "local"` for all entries; `source_detail.path` encodes the sub-project origin

Regression signals:
- **#8 multi-project-filesystem**: all 10 JARs detected across all 10 sub-directories
- **#20 complex-deep-chain**: depth assertion `gte: 10` on both total and direct dep counts

---

## Mend config

**Bucket A** — default-emit with `java` pinned to `17`. Ant itself is NOT pinnable via
`install-tool`; the Ant binary version comes from whatever the operator installs out-of-band.
This is a partial reproducibility limitation. Downstream comparators should treat the Ant
version as uncontrolled.

**`.whitesource`**: `configMode: "LOCAL"`, `versioning.java: "17"`. No additional dimensions
(branch scoping, project-token routing, or suppression) are needed for this probe.

**`whitesource.config`**: `ant.resolveDependencies=true` + `ant.ivyResolveDependencies=true`.
These flags are aspirational — the bolt-4 SCM scanner does not honor them at scan time (Mode-3
fallback is mandatory in the current SCM integration stack). They are present so the repo is
immediately usable for pipeline-mode `mend ua` testing when Mode-1 support is verified.

**`lib/` not in `.gitignore`**: JARs must be committed. Mode-3 has nothing to fingerprint
if `lib/` is gitignored. The `.gitignore` explicitly lists only build outputs.

---

## UA resolver notes (from java-jvm.md § 3)

- Mend's `IvyDependenciesAntTask` parses the Ivy `ResolveReport` for the dependency graph.
  This only activates in pipeline-mode `mend ua` with `ant.ivyResolveDependencies=true`.
- In SCM integration (bolt-4), the Ant resolver falls back to Mode-3 binary path scan.
  The `whitesource.config` flags are silently ignored.
- SHA1 in Mode-3 is the file fingerprint of the committed JAR, not recomputed from Maven Central.
  The `expected_dependency_checksums` entries use the actual SHA1 of the committed files.
- **Known limitation:** No EUA support for Ant — reachability analysis is disabled per resolver docs.
- `ant.ivyIgnoredConfigurations` can suppress configurations. Set to empty to include all confs.

---

## Probe metadata

```
probe_id:           ant-multi-project-deep-chain-20260525-170118
pm:                 ant
pm_version_tested:  ivy-2.5.x (Ivy used by UA's in-process library)
patterns:           [multi-project-filesystem, complex-deep-chain]
catalog_ids:        [#8, #20]
priority:           P1
generated:          2026-05-25
bucket:             A (java pinned to 17; Ant version uncontrolled)
target:             remote
schema_version:     1.0 (flat packages map)
```
