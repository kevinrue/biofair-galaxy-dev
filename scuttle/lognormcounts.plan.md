# Implementation plan: `scuttle_lognormcounts` Galaxy tool wrapper

## Overview

Wrap `scuttle::logNormCounts()` as a Galaxy tool. It accepts a
`SingleCellExperiment` (SCE) loom file, applies library-size normalisation and
log-transformation, and writes the updated SCE back as a loom file. The only
optional user-facing argument is the standard "Output log?" toggle already
present in the package macros.

No new macros, no new dependencies, and no new test-data files are needed —
everything required is already declared in `tools/scuttle-1.20.0/macros.xml`.

---

## Step 1 — Create the tool XML file

Create `tools/scuttle-1.20.0/scuttle_lognormcounts.xml` with the following
content:

```xml
<?xml version="1.0"?>
<tool id="scuttle_lognormcounts" name="Scuttle: Log-normalise counts"
      version="@TOOL_VERSION@+galaxy@VERSION_SUFFIX@" profile="@PROFILE@">
    <description>of a SingleCellExperiment object</description>
    <macros>
        <import>macros.xml</import>
    </macros>
    <expand macro="requirements"/>
    <command detect_errors="exit_code"><![CDATA[
@CMD@
    ]]></command>
    <configfiles>
        <configfile name="script_file"><![CDATA[
@CMD_imports@
@CMD_read_inputs@

sce <- scuttle::logNormCounts(x = sce)

@CMD_loom_write_outputs@
        ]]></configfile>
    </configfiles>
    <inputs>
        <expand macro="input_sce"/>
        <expand macro="inputs_common_advanced"/>
    </inputs>
    <outputs>
        <expand macro="output_sce"/>
    </outputs>
    <tests>
        <test expect_num_outputs="2">
            <param name="input_loom"
                   location="https://zenodo.org/records/19665848/files/sce_after_addpercellqcmetrics.loom"
                   ftype="loom"/>
            <section name="advanced_common">
                <param name="show_log" value="true"/>
            </section>
            <output name="hidden_output">
                <assert_contents>
                    <has_text_matching expression="logNormCounts"/>
                </assert_contents>
            </output>
            <output name="output_loom" ftype="loom">
                <assert_contents>
                    <has_size size="50000" delta="10"/>
                </assert_contents>
            </output>
        </test>
    </tests>
    <help><![CDATA[
**What it does**

Runs ``scuttle::logNormCounts()`` on a ``SingleCellExperiment`` object.
Each count is divided by the library size of its cell (or by pre-computed
size factors when present), multiplied by a pseudo-count scale factor (default
1e6 when no size factors exist, otherwise 1), and then log-transformed with
``log1p``. The result is stored as a new ``logcounts`` assay in the returned
object.

**Inputs**

- A ``SingleCellExperiment`` object saved as a loom file (e.g. produced by
  ``scuttle::addPerCellQCMetrics``).

**Outputs**

- The same ``SingleCellExperiment`` with a ``logcounts`` assay added, saved
  as a loom file.
    ]]></help>
    <expand macro="citations"/>
</tool>
```

**Notes on the test:**

- `expect_num_outputs="2"` because `show_log = true` enables the
  `hidden_output` in addition to `output_loom`.
- The `has_size size="50000" delta="10"` value is a placeholder. Run the test
  once (Step 3), observe the reported actual size in the failure message, then
  replace `50000` with that value.
- The `has_text_matching expression="logNormCounts"` assertion checks that the
  R script echoed to the log contains the function name, confirming the script
  ran correctly.

---

## Step 2 — Verify no macros.xml changes are needed

Open `tools/scuttle-1.20.0/macros.xml` and confirm that all tokens used by
the new tool are already present:

| Token / macro | Purpose |
|---|---|
| `@TOOL_VERSION@` | `1.20.0` |
| `@VERSION_SUFFIX@` | `0` |
| `@PROFILE@` | `23.0` |
| `requirements` xml | declares `bioconductor-scuttle` and `bioconductor-loomexperiment` |
| `@CMD@` | copies input loom, echoes script, runs Rscript |
| `@CMD_imports@` | `library(scuttle)` + `library(LoomExperiment)` |
| `@CMD_read_inputs@` | reads `sce.loom` into `sce` |
| `@CMD_loom_write_outputs@` | writes `sce` to `output.loom` |
| `input_sce` xml | `input_loom` param |
| `output_sce` xml | `output_loom` data + `outputs_common_advanced` |
| `inputs_common_advanced` xml | `advanced_common` section with `show_log` |
| `citations` xml | DOI for scater/scuttle paper |

No edits to `macros.xml` are required.

---

## Step 3 — Run planemo lint

```bash
planemo lint tools/scuttle-1.20.0/scuttle_lognormcounts.xml
```

All checks should pass. Fix any reported issues before proceeding.

---

## Step 4 — Run the test (first pass — expect size assertion failure)

```bash
planemo test \
    --biocontainers \
    tools/scuttle-1.20.0/scuttle_lognormcounts.xml
```

The test will fail on the `has_size` assertion and report the actual file size,
for example:

```
Expected file size 50000 (+-10), got 68432
```

---

## Step 5 — Fix the size assertion

Open `tools/scuttle-1.20.0/scuttle_lognormcounts.xml` and replace the
placeholder `size="50000"` with the actual value reported in Step 4:

```xml
<has_size size="68432" delta="10"/>
```

(Use whatever value planemo reported.)

---

## Step 6 — Run the test (second pass — expect full pass)

```bash
planemo test \
    --biocontainers \
    tools/scuttle-1.20.0/scuttle_lognormcounts.xml
```

All assertions should now pass. Review `tool_test_output.html` if any
unexpected failure occurs.

---

## Step 7 — Serve the tool locally (optional smoke test)

```bash
planemo serve \
    --biocontainers \
    tools/scuttle-1.20.0/scuttle_lognormcounts.xml
```

Open the Galaxy UI at `http://localhost:9090`, upload (or provide a URL for)
`sce_after_addpercellqcmetrics.loom`, run the tool, and inspect the output loom
and log.

---

## Summary of files touched

| Action | Path |
|---|---|
| **Create** | `tools/scuttle-1.20.0/scuttle_lognormcounts.xml` |
| No change | `tools/scuttle-1.20.0/macros.xml` |
| No change | `tools/scuttle-1.20.0/.shed.yml` |
| No change | `tools/scuttle-1.20.0/test-data/` |
