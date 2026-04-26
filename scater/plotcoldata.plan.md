# Galaxy Tool: scater 1.38.0 — `plotColData`

## Approach

The tool:
- accepts a `SingleCellExperiment` object serialised as a loom file (Galaxy datatype `loom`)
- embeds all R code in a `<configfile name="script_file">` block using Cheetah template syntax
- exposes `@CMD@`, `@CMD_imports@`, `@CMD_read_inputs@` macro tokens for boilerplate — mirroring `tools/scuttle-1.20.0/macros.xml`
- one `.xml` file per tool function; shared macros live in `macros.xml`
- test data is fetched from Zenodo using `location=` attributes; no local files to commit

## Directory layout

```
tools/scater-1.38.0/
├── macros.xml
├── scater_plotcoldata.xml
└── .shed.yml
```

---

## Step 0 – Understand `plotColData`

`scater::plotColData()` creates a ggplot2 plot of column-level metadata from a
`SingleCellExperiment` object.

Key parameters to expose in Galaxy:

| R argument | Galaxy param type | Notes |
|---|---|---|
| `object` | `data` (loom) | input SCE |
| `y` | `text` | `colData` column for the y-axis; required |
| `x` | `text` (optional) | `colData` column for the x-axis; omit for a violin/beeswarm plot |

The function returns a `ggplot2` object. Save it as PNG using `ggplot2::ggsave()`.
ggplot2 is an indirect dependency of scater and will be present in the Conda
environment; it does not need to be declared as a separate requirement.

---

## Step 1 – Create the directory

```bash
mkdir -p tools/scater-1.38.0
```

No local test data to prepare: all test inputs are fetched from Zenodo via
`location=` attributes in the XML.

---

## Step 2 – Write `macros.xml`

Create `tools/scater-1.38.0/macros.xml`:

```xml
<?xml version="1.0"?>
<macros>
    <token name="@TOOL_VERSION@">1.38.0</token>
    <token name="@VERSION_SUFFIX@">0</token>
    <token name="@PROFILE@">23.0</token>

    <xml name="requirements">
        <requirements>
            <requirement type="package" version="@TOOL_VERSION@">bioconductor-scater</requirement>
            <requirement type="package" version="1.28.0">bioconductor-loomexperiment</requirement>
            <yield/>
        </requirements>
    </xml>

    <token name="@CMD@"><![CDATA[
cp '$input_loom' sce.loom &&
cat '$script_file' > '$hidden_output' &&
Rscript '$script_file' >> '$hidden_output'
    ]]></token>

    <token name="@CMD_imports@"><![CDATA[
library(scater)
library(LoomExperiment)
    ]]></token>

    <token name="@CMD_read_inputs@"><![CDATA[
sce <- import('sce.loom', format = "loom", type = "SingleCellLoomExperiment")
    ]]></token>

    <xml name="input_sce">
        <param name="input_loom" type="data" format="loom"
               label="Input SingleCellExperiment (loom)"/>
    </xml>

    <xml name="inputs_common_advanced">
        <section name="advanced_common" title="Advanced Output" expanded="false">
            <param name="show_log" type="boolean" checked="false"
                   label="Output log?"/>
        </section>
    </xml>

    <xml name="outputs_common_advanced">
        <data name="hidden_output" format="txt"
              label="${tool.name} on ${on_string}: log">
            <filter>advanced_common['show_log']</filter>
        </data>
    </xml>

    <xml name="citations">
        <citations>
            <citation type="doi">10.1093/bioinformatics/btw777</citation>
        </citations>
    </xml>
</macros>
```

---

## Step 3 – Write `scater_plotcoldata.xml`

Create `tools/scater-1.38.0/scater_plotcoldata.xml`:

```xml
<?xml version="1.0"?>
<tool id="scater_plotcoldata" name="Scater: Plot column data"
      version="@TOOL_VERSION@+galaxy@VERSION_SUFFIX@" profile="@PROFILE@">
    <description>from a SingleCellExperiment object</description>
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

p <- scater::plotColData(
    object = sce,
    y      = '$y'
#if $x
    , x = '$x'
#end if
)

ggplot2::ggsave('output.png', plot = p)
        ]]></configfile>
    </configfiles>
    <inputs>
        <expand macro="input_sce"/>
        <param name="y" type="text"
               label="Y-axis column"
               help="Column from colData to display on the y-axis (e.g. detected)"/>
        <param name="x" type="text" optional="true"
               label="X-axis column (optional)"
               help="Column from colData to display on the x-axis (e.g. sum). Leave empty for a violin/beeswarm plot."/>
        <expand macro="inputs_common_advanced"/>
    </inputs>
    <outputs>
        <data name="output_png" format="png" from_work_dir="output.png"
              label="${tool.name} on ${on_string}: plot"/>
        <expand macro="outputs_common_advanced"/>
    </outputs>
    <tests>
        <test expect_num_outputs="2">
            <!-- test1: y only -->
            <param name="input_loom" location="https://zenodo.org/records/19665848/files/sce_after_addpercellqcmetrics.loom" ftype="loom"/>
            <param name="y" value="detected"/>
            <section name="advanced_common">
                <param name="show_log" value="true"/>
            </section>
            <output name="hidden_output">
                <assert_contents>
                    <has_text_matching expression="plotColData"/>
                </assert_contents>
            </output>
            <output name="output_png" ftype="png">
                <assert_contents>
                    <has_size size="PLACEHOLDER" delta="10000"/>
                </assert_contents>
            </output>
        </test>
        <test expect_num_outputs="2">
            <!-- test2: y and x -->
            <param name="input_loom" location="https://zenodo.org/records/19665848/files/sce_after_addpercellqcmetrics.loom" ftype="loom"/>
            <param name="y" value="detected"/>
            <param name="x" value="sum"/>
            <section name="advanced_common">
                <param name="show_log" value="true"/>
            </section>
            <output name="hidden_output">
                <assert_contents>
                    <has_text_matching expression="plotColData"/>
                </assert_contents>
            </output>
            <output name="output_png" ftype="png">
                <assert_contents>
                    <has_size size="PLACEHOLDER" delta="10000"/>
                </assert_contents>
            </output>
        </test>
    </tests>
    <help><![CDATA[
**What it does**

Runs ``scater::plotColData()`` on a ``SingleCellExperiment`` object to visualise
column-level metadata. Without an x-axis variable, each column is shown as a
violin/beeswarm plot. With an x-axis variable, a scatter plot is produced.

**Inputs**

- A ``SingleCellExperiment`` object saved as a loom file.
- The name of a ``colData`` column to display on the y-axis (e.g. ``detected``).
- Optionally, the name of a ``colData`` column to display on the x-axis (e.g. ``sum``).

**Outputs**

- A PNG image of the plot.
    ]]></help>
    <expand macro="citations"/>
</tool>
```

---

## Step 4 – Lint and test with Planemo

```bash
# From the tools/scater-1.38.0/ directory:

# Lint XML
planemo lint scater_plotcoldata.xml

# Run tests (builds Conda env on first run – slow)
planemo test scater_plotcoldata.xml

# Serve locally to click through the Galaxy UI
# Requires a pre-existing Galaxy checkout; planemo cannot build the client from scratch.
planemo serve scater_plotcoldata.xml
```

---

## Step 5 – Harden size assertions

After the first successful `planemo test` run, inspect the reported output sizes
and replace both `size="PLACEHOLDER"` values in the `<has_size>` assertions of
`scater_plotcoldata.xml` with the actual observed byte counts.  Keep
`delta="10000"` — PNG file sizes vary slightly across platforms due to compression.

---

## Step 6 – Write `.shed.yml`

Create `tools/scater-1.38.0/.shed.yml`:

```yaml
categories:
  - Transcriptomics
  - Single-cell
description: Galaxy wrappers for the Bioconductor scater package (v1.38.0)
long_description: |
  scater provides tools for quality control, normalisation, and visualisation
  of single-cell RNA-seq data, built around the SingleCellExperiment class.
name: scater
owner: iuc
```
