# ZSoil `.inp` File Format Reference

This document describes the structure and syntax of the ZSoil `.inp` input file, as used by the current (v26) self-describing file format. It is written for engineers who need to read or hand-edit `.inp` files.

> **Scope note:** this reference covers the current v26 format only (files beginning with a `@<vendor>_version: 26 ...` header line and using `#`-commented header count lines). Older ZSoil file versions use a different, fixed-column header and are out of scope here.

## Table of Contents

1. [Introduction](#1-introduction)
2. [General File Conventions](#2-general-file-conventions)
3. [File Header](#3-file-header)
4. [Materials](#4-materials)
5. [Existence & Load Functions](#5-existence--load-functions)
6. [Geometry](#6-geometry)
7. [Elements](#7-elements)
8. [Boundary Conditions](#8-boundary-conditions)
9. [Loads](#9-loads)
10. [Masses](#10-masses)
11. [Initial Conditions](#11-initial-conditions)
12. [Reinforcement](#12-reinforcement)
13. [Mesh Tying & Domain Reduction](#13-mesh-tying--domain-reduction)
14. [Quick Reference](#14-quick-reference)
15. [Worked Example](#15-worked-example)

Field-level documentation quality varies by section. Where a field's exact meaning is unknown — mostly opaque numeric solver-tuning parameters — it is marked explicitly rather than guessed; treat unmarked fields as reliable and marked ones as needing independent verification if precision matters.

---

## 1. Introduction

A ZSoil `.inp` file is a single plain-text file that fully describes a model: analysis settings, materials, geometry, mesh, boundary conditions, loads, and more. Despite the presence of short dot-prefixed tokens throughout the file (e.g. `.ing`, `.i0g`, `.inb`) that look like file extensions, **these are not separate files** — they are historical section markers, left over from an earlier version of the format that did split data across companion files by extension. In the current format, every section lives in the one `.inp` file, introduced by its marker line.

### Notation conventions used in this document

| Notation | Meaning |
|---|---|
| `<value>` | A placeholder for a single field whose actual value depends on the model. |
| `value[N]` | The field `value` repeats `N` times, normally one repetition per line, in a block whose length is `N` (`N` is read from an earlier count field). |
| `<A \| B>` | Either `A` or `B` is valid at this position, depending on a preceding flag or element type tag. |
| *(optional, if ...)* | The field or line is only present under the stated condition (e.g. a following flag being non-zero). |
| `# ...` | Text after `#` on a header count line is a human-readable label, not part of the file syntax proper — it exists only in the current (v26) format. |
| blank line | An actual empty line in the file, used as a block terminator for several sections. |

---

## 2. General File Conventions

### 2.1 Text format

The file is plain text. Records are whitespace-delimited (fields separated by spaces/tabs); most sections use one record per line, though multi-line records occur (e.g. materials, where several lines together form one logical record). Numbers are written as plain decimals or scientific notation (`1.000000000000e+000`); there is no fixed column width.

### 2.2 Section markers

Every data block in the body of the file is introduced by a marker line. Two marker styles are used:

- **Dot-prefixed markers**, e.g. `.ing`, `.i0g`, `.inb` — historically named after the companion file extension that used to hold that data. There are **91 distinct dot-prefixed markers**, always appearing in the same fixed order (see [§14 Quick Reference](#14-quick-reference) for the complete list). Every marker is present in every file, even when its block is empty — an empty block is simply the marker line followed immediately by a blank line (or, for count-terminated blocks, a `0` count).
- **Plain-keyword markers**, e.g. `NUM_MATERIALS=`, `EXIST_FUNC`, `LOAD_FUN`, `CONTROL`, `LAYERED_BEAM_COMPONENTS` — used in the header/materials portion of the file, before the dot-marker sequence begins.

### 2.3 Block termination styles

Two patterns are used to determine how many lines belong to a section:

- **Count-terminated**: a header count (read earlier, in the file's global count block, §3.2) tells the reader exactly how many records follow. Most element and entity blocks (`.ing`, `.i0g`, `.inb`, …) work this way — the global header is the single authority on block length, not a per-block repeated count.
- **Blank-line-terminated**: the block continues until an empty line is encountered, independent of any global count (e.g. `.gsl` surface loads). This is used where entries have variable-length, self-delimiting records.

### 2.4 Element type tags

Several element sections carry a short type tag identifying the element's shape/formulation, which in turn determines how many connectivity/property fields follow on that record. These are introduced in detail per-section in [§7 Elements](#7-elements); as a quick preview:

| Tag | Meaning |
|---|---|
| `B8` | 8-node hexahedral (brick) continuum element |
| `Q4` | 4-node quadrilateral continuum element (2D) |
| `T3` | 3-node triangular continuum element (2D) |
| `BEL2` | 2-node beam element |
| `TRS2` | 2-node truss element |
| `LNK2` | 2-node link/anchor element |
| `SXQ4` | 4-node shell element |
| `SHQ4` | 8-node thick shell element |
| `C_Q4` | Contact element on a structural (quad) face |
| `C_L2` | Contact/interface element on a line |
| `SPL2` | 2-node seepage element |

### 2.5 Existence functions (EF) and Load functions (LF)

Nearly every element, load, and mass record references an **existence function number (EF)** and, where relevant, a **load function number (LF)** — these are defined once (in `EXIST_FUNC` / `LOAD_FUN`, [§5](#5-existence--load-functions)) and referenced by number everywhere else in the file:

- **EF 0** is implicit and always active — the "permanent" existence function, used by anything that exists for the whole analysis.
- **LF 0** is implicit and equal to a constant multiplier of 1 — used by anything not subject to a time-varying load history.
- Any other EF/LF number must be defined in the corresponding `EXIST_FUNC`/`LOAD_FUN` block earlier in the file.
- An EF reference isn't only used to control whether an *element* exists — it can also gate a boundary condition's fixity, letting a `.inb` record stay nominally fixed while only actually being enforced during that EF's active interval (see §8.1 for how a `.inb` fixity block references an EF this way).

### 2.6 Compact element-list notation

A few list fields (e.g. selecting a subset of elements) use a compact range notation instead of listing every id:

```
<start>A<end>            → every integer from start to end, inclusive, step 1
<start>A<end>P<stride>   → every integer from start to end, inclusive, step <stride>
<id>                      → a single element id, listed literally
```

Multiple tokens (space-separated) combine into one list, e.g. `12A45P2 90` expands to `12, 14, 16, ..., 44, 90` plus `90`.

---

## 3. File Header

### 3.1 Version line

The very first line of the file identifies the writing application/version:

```
@GeoDev_SARL_version: 26 9.05 26.03
```

Format: `@<vendor>_version: <majorVersion> <createdWithVersion> <lastOpenedWithVersion>`. `<majorVersion>` (`26`) is the leading format-generation number identifying the current self-describing format documented here; `<createdWithVersion>` (`9.05` in this example) is the ZSoil version the file was **originally created with**; `<lastOpenedWithVersion>` (`26.03`) is the version of **ZSoil PrePro that last opened/saved** the file. A file's on-disk record layout (e.g. the `.inb` record width, §8.1) can reflect an older `createdWithVersion` even though the leading `26` and `lastOpenedWithVersion` look current — when hand-editing, match the layout already used elsewhere in the *same* file rather than assuming a fixed one.

### 3.2 Header count block

Lines 2–136 are a fixed, always-135-line sequence of `<count> # <description>` lines in fixed order. Each count tells the reader how many records to expect later in the corresponding dot-marker section.

| # | Count field (label as it appears in the file) |
|---|---|
| 1 | dimension 2 or 3 |
| 2 | number of materials |
| 3 | number of Existence functions |
| 4 | number of nodal forces (`.inl`) |
| 5 | number of nodes (`.ing`) |
| 6 | number of continuum 2D elements |
| 7 | number of continuum 3D elements |
| 8 | number of shell elements (`.ilg`, `.ilm`) |
| 9 | number of Shell one layer elements |
| 10 | number of Beams (`.ibg`, `.ibm`) |
| 11 | number of Beams relaxation codes (`.ibl`) |
| 12 | number of truss elements (`.itg`, `.itm`) |
| 13 | number of Fixed Anchor Zones |
| 14 | number of loads for trusses (`.itl`) |
| 15 | number of Piles |
| 16 | number of Nails |
| 17 | number of Membrane elements |
| 18 | number of Infinite elements |
| 19 | number of contact lines (`.icg`, `.icm`) |
| 20 | number of interfaces (`.ics`) |
| 21 | number of water Seepage's (`.ipg`, `.ipm`) |
| 22 | number of convection elements (`.ivg`) |
| 23 | number of viscous dampers (`.vsd`) |
| 24 | number of boundary conditions (`.inb`) |
| 25 | number of surface elements (`.pbc`) |
| 26 | number of water bound. cond. (`.iwb`) |
| 27 | number of Heat BC (`.iab`) |
| 28 | number of Humidity BC (`.imb`) |
| 29 | number of pressure Heads loads (`.iph`) |
| 30 | number of water flux (`.iwf`) |
| 31 | number of heat flux (`.ihf`) |
| 32 | number of humidity flux (`.iuf`) |
| 33 | *not used* |
| 34 | number of surface loads (`.isl`) |
| 35 | number of Surface load defined by Pressure head |
| 36 | number of init. cond. geom. (`.iig`) |
| 37 | number of ini. stress cond. (`.izg`) |
| 38 | number of Heat IC (`.iag`) |
| 39 | number of Heat IC values (`.iav`) |
| 40 | number of Humidity IC (`.img`) |
| 41 | number of Humidity IC values (`.imv`) |
| 42 | number of Imposed strains |
| 43 | number of ini. strains cond. (`.ieg`) |
| 44 | number of constant eps0 (`.ist`) |
| 45 | number of Nodal Links (`.gnl`) |
| 46 | number of kinematic constraints (`.ikg`) |
| 47 | number of nodal masses (`.inm`) |
| 48 | number of element masses (`.iem`) |
| 49 | *not used* |
| 50 | number of Local bases |
| 51 | number of geometrical points |
| 52 | number of polylines |
| 53 | number of lines |
| 54 | number of arcs |
| 55 | number of circles |
| 56 | number of surfaces |
| 57 | number of auxiliary planes |
| 58 | number of continuum 2D subdomains |
| 59 | number of continuum 3D subdomains |
| 60 | number of beam subdomains |
| 61 | number of truss subdomains |
| 62 | number of membrane subdomains |
| 63 | number of shell subdomains |
| 64 | number of macro seepages |
| 65 | number of macro convections |
| 66 | number of macro interfaces |
| 67 | number of macro interfaces on structural subdomains |
| 68 | number of macro dampers |
| 69 | number of point loads |
| 70 | number of macro surface loads |
| 71 | number of macro heat fluxes |
| 72 | number of macro humidity fluxes |
| 73 | number of macro fluid fluxes |
| 74 | number of macro solid BC |
| 75 | number of macro pressure BC |
| 76 | number of macro pressure heads |
| 77 | number of macro heat BC |
| 78 | number of macro humidity BC |
| 79 | number of mesh tying elements |
| 80 | number of beam loads |
| 81 | number of boreholes |
| 82 | number of initial velocities/accelerations |
| 83 | number of shell hinges |
| 84 | number of beam reinforcements |
| 85 | number of data super elements |
| 86 | number of Tendons |
| 87 | number of auxiliary points |
| 88 | *not used* |
| 89 | number of Heat exchangers |
| 90 | number of movement paths |
| 91 | number of Point loads on elements |
| 92 | number of Line loads on elements |
| 93 | number of Surface loads on elements |
| 94 | number of Line masses on elements |
| 95 | number of Surface masses on elements |
| 96 | number of Splines |
| 97 | number of Heat Exchanger heat fluxes |
| 98–135 | *"Not used" — reserved* |

*(Field indices above are relative to this table, not raw file line numbers. Entries 33, 49, and 88 have no functional label but are followed by an unrelated, oddly duplicated-looking label in some files — a quirk of the format itself.)*

### 3.3 Associated projects and units

Immediately after the count block:

```
ASSOCIATED_PROJECTS:
HEAT: -
HUMIDITY: -
FREE_FILED_MOTION: - 
FLOW: -
<UnitSystemName>
<UnitForce> <UnitLength> <UnitAngle> <UnitTime> <UnitTemperature>
<UnitForce> <UnitLength> <UnitAngle> <UnitTime> <UnitTemperature>
                                 <- blank line
```

Each `HEAT:`/`HUMIDITY:`/`FREE_FILED_MOTION:`/`FLOW:` line names an associated companion project file, or `-` if none is linked. `UnitSystemName` is a free-text label (e.g. `STANDARD`, `EXAMPLE UNITS`). The two unit lines are **not a duplicate**: the first is the unit system used **in the preprocessor** (model input/editing), the second is the unit system used **in the postprocessor** (results display) — usually identical, but can differ if display units were changed after the model was built. Observed unit values include `kN`, `m`, `deg`/`rad`, `year`/`day`/`h`, `C`.

### 3.4 Analysis control blocks

After the units block, a fixed sequence of named control blocks follows. The general pattern for the solver-control blocks (`CONTROL`, `DYN_CONTROL`, `PSH_CONTROL`) is:

```
<KEYWORD> <n>
<name of set 1>
<numeric parameter line(s)>
<name of set 2>
<numeric parameter line(s)>
...                              <- n named sets total
```

`<n>` is the number of named parameter sets that follow.

**`CONTROL <n>` — nonlinear solver settings.** Each set's first line holds 18 fields:

| # | Field | Example |
|---|---|---|
| 1 | Tolerance for solid-phase RHS | `0.01` |
| 2 | Tolerance for solid-phase energy | `0.001` |
| 3 | Tolerance for fluid-phase RHS | `0.01` |
| 4 | Tolerance for fluid-phase energy | `0.001` |
| 5 | Nonlinear solver: `1`=Full Newton-Raphson, `2`=BFGS, `3`=Initial stiffness, `4`=Modified N-R, `5`=Initial stiffness (accelerated) | `1` |
| 6 | Max. number of iterations per step | `15` |
| 7 | Reform stiffness every *n* steps (Modified N-R) | `1` |
| 8 | Reform stiffness every *n* iterations (Modified N-R) | `1` |
| 9 | *(unknown)* | `1` |
| 10 | *(unknown)* | `1` |
| 11 | *(unknown)* | `1` |
| 12 | Absolute max. number of iterations | `200` |
| 13 | Reduce-step-size flag | `0` |
| 14 | Max. reduction trials | `3` |
| 15 | Step reduction factor | `0.5` |
| 16 | Increase dt for time-dependent/dynamic drivers flag | `1` |
| 17 | dt increase criterion | `0.25` |
| 18 | Step amplification factor | `1.25` |

The second line holds the "sharpened" tolerances used for kinematic (displacement-controlled) loading, plus a solver-switching strategy flag:

| # | Field | Example |
|---|---|---|
| 1 | Sharpening of tolerance for kinematic loading, flag | `1` |
| 2 | (Sharpened) tolerance for solid-phase RHS | `0.01` |
| 3 | (Sharpened) tolerance for solid-phase energy | `0.01` |
| 4 | Strategy for divergent steps: automatic solver switching flag | `1` |
| 5–10 | More parameters for dealing with lack of convergence | `1 5 5 0 0 0` |

The same "named-set" pattern (keyword, count, then that many named sets) applies to:

- **`DYN_CONTROL <n>`** — dynamic-analysis solver control (same general kind of tolerance/iteration settings, extended for time integration; field-by-field layout not decoded here).
- **`PSH_CONTROL <n>`** — **pushover-analysis solver control**: governs the nonlinear static (pushover) solver settings specifically, distinct from the general `CONTROL` block used for standard nonlinear static/staged analyses.

Following the solver-control blocks, a series of simple flag/value lines:

```
NONL_GEOM <0|1>              <- large-displacement/nonlinear-geometry flag
CONSTRUCTION <flag> <n>
<n lines, one per multi-initial-state definition, if n>0>
THM_SETTINGS
BISHOP_FLAG <0|1>            <- unsaturated-soil Bishop's effective stress flag
PROJECT_PRESELECTION
<6 flags>
```

### 3.5 Driver / output settings and project metadata

Later in the header (after materials and functions — see §4–5), a `DRIVERS <n>` block (same named-set pattern as §3.4) is followed by a long sequence of simple `KEYWORD` / `<value>` pairs — each keyword on its own line, its value(s) on the line(s) immediately after. These are largely self-explanatory from their names:

| Keyword | Purpose |
|---|---|
| `RAM_MAXIMUM` | Solver memory limit |
| `SOLVER_TYP` | Solver type selector |
| `SOLVER_OOC` | Out-of-core solver flag |
| `PRINTSTR` | Print structural results flag |
| `PRINTDISP` | Print displacements flag |
| `VISPL_REGUL_CONTINUUM` | Visco-plastic regularization flag (continuum) |
| `NROFPOINT_BINRESELE` | Result-output point count, binary results (element) |
| `NROFPOINT_BINRES_SHELL` | Result-output point count, binary results (shell) |
| `NROFPOINT_BINRES_BEAM` | Result-output point count, binary results (beam) |
| `ELEM_BIN_RSL` | Element binary result-output selector code |
| `ELEM_LAY_RSL` | Element layer result-output selector code |
| `RSLT_IN_DEFORMED_CFG` | Report results in deformed configuration flag |
| `PRINTTHS` | Print time-history flag |
| `THS_D`, `THS_V`, `THS_A` | Time-history output flags: displacement / velocity / acceleration |
| `THS_DOF_CODE` | Time-history DOF selector code |
| `THS_NODES_GROUP` | Time-history node group name (`EMPTY` if none) |
| `THS_T1`, `THS_T2` | Time-history output time window start/end |
| `ELEMENTS_CFG` | Element configuration code |
| `STAB_CONSOL` | Stability/consolidation analysis flags block |
| `VERSION_TYPE` | Format/version-type flag |
| `PRJ_COMPANY`, `PRJ_AUTHOR`, `PRJ_TITLE`, `PRJ_DESCRIPTION` | Free-text project metadata (may be blank) |
| `UPD_COORD_CONSTRUCTION_LARGE_DEF` | Update-coordinates-on-construction-stage flag (large deformations) |
| `PARALLEL_SETTINGS` | Parallel-solver settings (thread count, etc.) |
| `ARC_LENGTH` | Arc-length method settings/flags |
| `AUGMENTED_LAGRANGIAN` | Augmented Lagrangian contact-solver settings |
| `SEISMIC_DATA <n>` | Seismic-analysis data block |
| `PARAMETRIC_ANALYSIS <n>` | Parametric-analysis settings |
| `CONTACT_PARAMETERS <n>` | Contact-material default parameter set(s), same named-set pattern as §3.4 |

These blocks are safe to leave untouched when hand-editing a file for a different purpose (e.g. adding a load or boundary condition) — they only need attention if the change is specifically to solver/output configuration.

---

## 4. Materials

### 4.1 Material record structure

Materials are introduced by `NUM_MATERIALS= <n>`, followed by `<n>` material records. Each record starts with 4 fixed lines, then a variable sequence of sub-tag blocks that runs until `DAMP->` is reached:

```
MATERIAL  <formatVersion> <number>
<type>                       <- e.g. "Elastic", "Elastic Beam", "Mohr-Coulomb"
<name>                       <- free text, or "No name"
BUTTONS= <flag>[14]          <- 14 flags selecting which sub-blocks are meaningful/enabled
                              <- blank line
 <sub-tag blocks, see 4.3>
NUM_MATERIALS= 0             <- terminates the block (0 = no more standard materials; a second
                                 NUM_MATERIALS= <n> immediately after, if present, defines FIBER
                                 materials for layered shells, using the same record shape — see
                                 §4.3.2 GEOM-> "Shell Layered" and §4.4)
```

Example (continuum "Elastic" soil material):

```
MATERIAL  26.00 1
Elastic
Soil
BUTTONS= 1 1 0 0 1 0 0 0 0 1 0 0 0 0
```

Example (beam "Elastic Beam" material):

```
MATERIAL  26.00 11
Elastic Beam
nail
BUTTONS= 1 1 1 1 0 0 0 0 0 0 0 0 0 0
```

### 4.2 Material formulations by element type

ZSoil organizes every material by **Continuum/Structure type** (which element group can use it) and then **Material formulation** (the specific model within that group) — this is the natural grouping for the `.inp` format too, since it determines which sub-tags (§4.3) are meaningful and how they're laid out.

| Continuum/Structure type | Material formulation | Sub-tags present (order as in file) | Notes |
|---|---|---|---|
| Continuum (`Q4`/`B8`/`T3`, §7.1) | `Elastic` | `ELAS->` `GEOM->` `DENS->` `FLOW->` `CREEP->` `NONL->` `HEAT->` `HUMID->` `INIS->` `STAB->` `DISC->` `DAMP->` | `Elastic`, `Unit weights`, `Flow`, and `Initial Ko State` checkboxes present in the GUI; `Creep` present but unchecked; no `Non linear` option (no plasticity model). |
| Continuum | *(nonlinear soil models, e.g. Mohr-Coulomb)* | same tag set, `NONL->` populated | Exact `<type>` string and a populated `NONL->` example not available. |
| Beams (`BEL2`, §7.2) | `Elastic Beam` (plain) | `ELAS->` `GEOM->` `MAIN->` `DENS->` `FLOW->` `CREEP->` `NONL->` `HEAT->` `HUMID->` `INIS->` `STAB->` `DISC->` `DAMP->` | `GEOM->` selects Profiles/User/Values cross-section (§4.3.2). |
| Beams | `Elastic Beam` (layered/composite) | same tag set; `GEOM->` carries embedded reinforcement fibers | Same type *string* as plain `Elastic Beam` — distinguished only by `BUTTONS[6]` (§4.3.2). Fibers reference `LAYERED_BEAM_COMPONENTS` (§4.4). |
| Trusses (`TRS2`/`LNK2`, §7.3) | *(type string not available)* | `ELAS->` `GEOM-><area>` `MAIN->` `DENS->` `FLOW->` `CREEP->` `NONL->` `HEAT->` `HUMID->` `INIS->` `STAB->` `DISC->` `DAMP->` | Only the numeric sub-tag content is documented, via the element's `mat` reference. |
| Shells (`SXQ4`/`SHQ4`, §7.5) | `Shell` (plain, single/one-layer) | `Unit weights`(`DENS->`) `Flow`(`FLOW->`) `Non linear`(`NONL->`) `Heat`(`HEAT->`) `Humidity`(`HUMID->`) `Damping`(`DAMP->`) | This formulation's own `ELAS->`/`GEOM->` content is not documented here. |
| Shells | `Shell Layered` | `ELAS->` (own, likely vestigial) `GEOM->` (core+fiber layer table) `DENS->` `FLOW->` (shell variant, §4.3.5) `CREEP->` `NONL->` (own, likely unused) `HEAT->` `HUMID->` `INIS->` `STAB->` `DISC->` `DAMP->` | `GEOM->` references a **second `NUM_MATERIALS=` pass** (§4.4) for its fiber/core materials, rather than embedding them the way a layered beam does. |
| *(fiber/core layer material, 2nd `NUM_MATERIALS=` pass — §4.4)* | `Fiber Shell` | `Elastic`(`ELAS-><E>` only) `Geometry`(`GEOM->`, type+direction) `Non linear`(`NONL->`, `ft`/`fc`) `MAIN->` `DENS->` `FLOW->` `CREEP->` `HEAT->` `HUMID->` `INIS->` `STAB->` `DISC->` `DAMP->` | This is where a layered shell's real tension/compression capacity and stiffness actually live (§4.3.1, §4.3.7). |
| *(fiber/core layer material, 2nd pass)* | `Orthotropic shell` (lower-case `"shell"`) | `Elastic`(`ELAS->`, E1/E2/ν12/G0) `Unit weights`(`DENS->`) `Geometry`(`GEOM->`, direction vector only) `Flow`(`FLOW->`) `MAIN->` `CREEP->` `HEAT->` `HUMID->` `INIS->` `STAB->` `DISC->` `DAMP->` | **No `Non linear` option at all** — always linear-elastic, e.g. for a directional mesh/stiffener layer rather than a rebar fiber. |

### 4.3 Tags

This subchapter documents each sub-tag once, covering every formulation-specific variant found for it — cross-reference the table in §4.2 to see which variants apply to which element type. Each sub-tag line starts with its tag (some indented by a leading space, inconsistently — e.g. ` ELAS->` vs `CREEP->`) followed by one or more numeric fields; several tags (`NONL->`, `ELAS->`, `INIS->`) continue onto additional lines until the *next* expected tag is reached, rather than having a fixed line count.

#### 4.3.1 `ELAS->` — elasticity

| Formulation | Fields | Example |
|---|---|---|
| Continuum `Elastic` | `<E> <nu>` + 3 more lines, unknown | `ELAS-> 25000  0.3` / `0  0` / `0  0` / `0  0` |
| Beam `Elastic Beam` (plain or layered) | `<E> <nu> <1>` + 2 more lines, unknown | `ELAS-> 200000000  0.3  1` / `0  0` / `0  0` |
| `Fiber Shell` | `<E>` only — the material only takes a single Young's modulus; the file still stores a second value (`nu`), presumably unused for this uniaxial fiber model | `ELAS-> 2e+08  0.3` / `0  0` ×2 |
| `Orthotropic shell` | `<E1> <E2> <nu12> <G0>` — `E1`, `E2` `[kN/m²]`, `ν12`, `G0` `[kN/m²]` | `ELAS-> 2.7e+07  2.7e+07  0.2  1.1e+07` / `0 0 0 0` ×2 |

The 3 unknown secondary lines for Continuum/Beam are *(meaning unclear — likely anisotropy/Biot or damping-related extensions, all `0` in every model seen)*. Note `"Orthotropic shell"`'s material type string is lower-case `"shell"`, inconsistent with `"Shell Layered"`/`"Fiber Shell"`'s capitalization elsewhere in the format.

#### 4.3.2 `GEOM->` — geometry/cross-section

**Beam** (plain `Elastic Beam`): `GEOM-> <type>` where `type` selects the cross-section definition: `0` = "Profiles" (extra lines: blank, profile name, dimension values), `1` = "User" (2 lines: dimensions, then computed section values), `2` = "Values" (raw section values only, 1 line). Example (`type=1`, rectangular): `GEOM-> 1` / `2  0.05  0.3  0.5  0.1  0.1  0.1` (dimensions) / `0.00196... 0.00168... ...` (area, Iyy, Izz, etc.) / `1  -1  0`.

**Beam, layered/composite variant**: two otherwise-identical beam materials differ only in `BUTTONS[6]` (0-indexed) and the line following the dimensions/section-values pair — plain (`BUTTONS[6]=0`) ends with `1  -1  0` (no further lines); layered (`BUTTONS[6]=1`, same `Elastic Beam` type string, same `type=1` dimensions/values pair) ends with `<1+n_reinf_fibers>  1  1`, followed by **one line per embedded reinforcement fiber**, blank-line-terminated:
```
 GEOM-> 1
0  1  0.3  1  0.1  0.1  0.1
1  0.833333  0.833333  0.140583  0.0833333  0.0833333  1
3  1  1
0  2  0.4  0.1  0.1  0  180  10  0.005  0  1  2
0  2  -0.4  0.1  0.1  0  180  10  0.005  0  1  2
```
The two fiber lines here are symmetric (`+0.4`/`-0.4`) rebar layers. Per fiber line `<?> <offsetType> <yOffset> <zOffsetL> <zOffsetR> <?> <?> <n_reinf_bars> <total_area> <prestress> <?> <componentId>`:
- `offsetType`: `0` = "From top"; `1` = "From bot."; `2` = "From center"
- `yOffset` (3rd field, `±0.4`): the fiber's *relative*, vertical position within the section
- `zOffsetL`/`zOffsetR` (4th/5th fields, `0.1`): the left/right-most fiber's *relative*, horizontal position within the section
- `n_reinf_bars` (8th field, `10`): number of bars in the fiber — no influence on the physics (only `total_area` matters), used for the section figure only
- `total_area`: total area of all `n_reinf_bars` in the fiber
- `prestress`: prestress in the fiber
- `componentId`: references a `LAYERED_BEAM_COMPONENTS` entry by number (§4.4) — here `2` = "steel"

**Truss**: `GEOM-> <area>` — a single cross-sectional area value.

**Continuum**: `GEOM-> <value>` — a single value, unclear meaning.

**`Shell Layered`**: multi-fiber section definition —
```
 GEOM-> 1  10  2  0.005  0.4  2    0.005  -0.4  2    1  2  2
```
Fields (0-indexed after the `GEOM->` token itself is `v[0]`): `v[1]`=`type`=1; `v[2]`=`10` *(meaning unclear from available sources)*; `v[3]`=`nFibers`=2; then per fiber `k`, `<area> <distance> <distanceFrom>` at `v[4+3k]`/`v[5+3k]`/`v[6+3k]` (fiber 1: area=`0.005`, distance=`0.4`, `distanceFrom`=`2`; fiber 2: area=`0.005`, distance=`-0.4`, `distanceFrom`=`2`); then `v[4+3·nFibers]`=`core_material`=`1` (the shell's core/matrix material id, resolved against the 2nd `NUM_MATERIALS=` pass, §4.4), followed by one material-id field per fiber at `v[5+3·nFibers+k]` (both fibers here reference material `2`) — a concrete core with steel reinforcement fibers symmetrically placed top and bottom.

`distanceFrom` meaning: the fiber position is a coordinate `ξ ∈ [-1, 1]` normalized to half-thickness (`y_physical = ξ · thickness/2`); `distanceFrom` selects how `distance` maps onto `ξ`:
- `distanceFrom=0`: `distance` is a fraction of *full* thickness measured inward from the top, `ξ = 1 - 2·distance`
- `distanceFrom=1`: mirror of `0`, measured from the bottom, `ξ = 2·distance - 1`
- `distanceFrom=2`: `ξ = distance` directly — already the normalized mid-plane-relative coordinate

A `"Shell Layered"` material's cross-section is a table of layers: `Core` for the base/matrix layer, `Reinforcement` for each fiber, each with an **area**, a **distance** (a value plus a from-top/from-bottom/relative choice, matching `distanceFrom` 0/1/2 above), and a **material** (by number, resolved against the 2nd `NUM_MATERIALS=` pass). There is also a top-level **shear correction factor** (location within the raw `GEOM->` record not identified — possibly stored under `MAIN->`).

**`Fiber Shell`** (2nd-pass layer material): `GEOM-> <type> <vx> <vy> <vz>` — a leading type code followed by a 3-component direction vector, e.g. `GEOM-> 1  1  0  0`.

**`Orthotropic shell`** (2nd-pass layer material): `GEOM-> <vx> <vy> <vz>` — **no leading type code**, just the 3-component direction vector directly, e.g. `GEOM-> 1  0  0` (`vx=1, vy=0, vz=0`). Compare `Fiber Shell`'s extra leading `type` field above.

#### 4.3.3 `MAIN->` (beam/truss materials only)

E.g. `MAIN-> 0` or `MAIN-> 1  0  0  0  0  0` — a small flag set, first value possibly a material-model sub-type selector. *(Not fully known.)*

#### 4.3.4 `DENS->` — density/consolidation/permeability-adjacent parameters

Example (continuum): `DENS-> 20  10  0.6  1  1  16.7` followed by 3 more lines (largely `0`s, plus a line like `-1  26.5  20  10  0`). Field 1 (`20`) = **γ, weight/unit volume** `[kN/m³]`; field 2 (`10`) = **ρ, mass/unit volume** `[kg/m³]`. A "Data mode" setting can switch γ/ρ from a plain constant to a value driven by an `EXIST_FUNC`/`LOAD_FUN`-style function instead — this likely accounts for the remaining trailing fields being non-zero in some models. Remaining fields beyond γ/ρ *(meaning unclear)*.

#### 4.3.5 `FLOW->` — hydraulic/permeability parameters

Two distinct layouts, depending on material type:
- **Continuum**: a fixed 28-field record on the first line, `v[0]`–`v[27]` (0-indexed after the `FLOW->` token), plus 3 more lines. Field positions below are pinned down exactly (except where marked), by editing a live model to give each field a distinct value, saving, and comparing against the unmodified record:

  | Index | Field | Notes |
  |---|---|---|
  | `v0` | Darcy's coefficient (1st) `[m/day]` | e.g. **kx** |
  | `v1` | Darcy's coefficient (2nd) `[m/day]` | e.g. **kz**; equal to `v0` when "isotropic flow" is on |
  | `v2` | *(unclear)* | always equal to `v0` in every model seen — possibly an unexposed 3rd permeability component, or an internal duplicate of `v0` |
  | `v3` | Inclination angle `[deg]` | |
  | `v4`, `v8` | flags, unclear | always `1` |
  | `v5`, `v6`, `v7`, `v9` | flags, unclear | always `0` |
  | `v10` | Fluid bulk modulus **Kf** `[kN/m²]` | |
  | `v11` | van Genuchten residual saturation ratio **Sᵣ** | |
  | `v12` | van Genuchten parameter related to the air entry suction **α** `[1/m]` | |
  | `v13`, `v15`, `v19` | flags, unclear | always `1` |
  | `v14` | Permeability-function-for-unsaturated-medium selector | `0`=Irmay, `1`=Mualem |
  | `v16` | Undrained penalty factor **K^F/K** | |
  | `v17` | Suction pressure cut-off `[kN/m²]` | |
  | `v18`, `v21`, `v22` | Bishop's-effective-stress section: "use global setting" flag and pore-pressure weighting-term selector (`S` vs `Seff^(1/(n·m))`) | which of these 3 positions is which of the 2 settings not fully separated — toggling both together shifted all three from `1`→`0` |
  | `v20` | Air stiffness bulk modulus **Ka** `[kN/m²]` | |
  | `v23` | Biot coefficient "Enable" flag | `0`=disabled (default), `1`=enabled |
  | `v24` | Biot: solid grains stiffness bulk modulus **Ks** `[kN/m²]` | |
  | `v25` | Biot: current value of **α** (elastic range) | |
  | `v26` | van Genuchten measure of the pore-size distribution **n** | |
  | `v27` | "Skip gravity term" flag | `0`=off (default), `1`=on |

  Example (defaults): `FLOW-> 1  1  1  0  1  0  0  0  1  0  2000000  0  20  1  0  1  1000000  100  1  1  100  1  1  0  1e+38  1  2  0`.
- **Shell**: a much shorter record, e.g. `FLOW-> 1  1  1  1  0  0  1  0  0  2  2e+06`. Fields, in order of appearance (exact token positions not independently verified, but the field set and meaning are known): a "fully permeable" flag, an "anisotropic flow in shell" flag, permeabilities **kx′**/**ky′** (only if anisotropic)/**kz′** `[m/s]`, an **orientation vector for the x′ axis** (`vx`,`vy`,`vz`, only meaningful if anisotropic), and a **van Genuchten soil-water-retention curve**: **residual saturation Sᵣ** and **parameter related to the air entry α** `[1/m]`.

Beam/truss materials have an effectively empty `FLOW->` (no values, just the tag).

#### 4.3.6 `CREEP->`

E.g. `CREEP-> 0  0  0  0  0  0  0  0  4  0  500  50  5  0  0  0  0` — identical across every inspected material regardless of type, suggesting this is a shared default block, populated only when creep is actually enabled (via `BUTTONS=` flags). *(Field meaning not confirmed.)*

#### 4.3.7 `NONL->` — nonlinear (plasticity) parameters

Empty (`NONL->` alone) for plain Continuum `Elastic`/Beam `Elastic Beam` materials (no plasticity model), and **entirely absent as an option** for `Orthotropic shell` (always linear-elastic, no `Non linear` checkbox in the GUI at all). Populated example confirmed for `Fiber Shell`:
```
 NONL-> FIRE_DATABASE 0 OFF
1000  500000  1  1  0.02  0.15  0.2
0  0  0  0
```
The first two numeric fields are **`ft`** (tension strength) then **`fc`** (compression strength), in that order — e.g. concrete `1000`(ft)/`500000`(fc); steel `235000`(ft)/`235000`(fc), consistent with steel's symmetric tension/compression yield. `FIRE_DATABASE <flag> <mode>` prefixes the line and toggles fire-resistance design checks; `<mode>` is `Off` or `EC2:2008` (Eurocode 2 fire-design method); `<flag>` is `0` for `Off`. Remaining fields (`1  1  0.02  0.15  0.2` and the trailing `0 0 0 0` line) *(meaning unclear — plausibly softening/regularization parameters, by analogy with `LAYERED_BEAM_COMPONENTS`' `reg_soft`/`char_len` fields, §4.4)*.

#### 4.3.8 `HEAT->`

E.g. `HEAT-> 1e-05  207.36  2000  0  105000  1.5  0.2083333333333333  4000  20  0  1` plus 2 more lines. Field 1 (`1e-05`) = heat dilatancy (thermal expansion coefficient) `[1/°C]`. Remaining fields *(meaning unclear)*.

#### 4.3.9 `HUMID->`

E.g. `HUMID-> 0  0.05  0.75  0.031104  0`. Field 1 = hygral dilatancy (e.g. `0.01` in the beam example `HUMID-> 0.01`). Remaining fields *(meaning unclear)*.

#### 4.3.10 `INIS->` — initial-state parameters

**Continuum**: e.g. `INIS-> 0.385  0.385  0  1  0  0  0  1  0  0  0  1`. The first 3 fields are the **initial Ko (at-rest earth pressure) state**: **Ko(x′)**, **Ko(z)** (the two assumed lateral-earth-pressure coefficients), then the **inclination angle ⟨x′,x⟩** `[deg]` (here `0.385`, `0.385`, `0` — a 2D model with equal, isotropic Ko in both directions and no inclination). Remaining fields *(meaning unclear)*.

**Beam**: e.g. `INIS-> 1  1  0  1  0  0  0  1  0  0  0  1` — same 12-field shape as continuum, but the first 3 fields (Ko-related for continuum) are presumably not applicable/ignored for a beam; not independently confirmed. Remaining fields *(meaning unclear)*.

#### 4.3.11 `STAB->`

E.g. `STAB-> 2  0  0  0  1  1.2  0.2  1  1.5  0.5` (continuum, non-trivial) vs `STAB-> 0  0  0  0  0  0  0  0  0  0` (beam, all zero) — stabilization/regularization parameters, active for continuum materials only.

#### 4.3.12 `DISC->`

Always empty (bare tag) — discontinuity/localization-band parameters, presumably populated only for softening models.

#### 4.3.13 `DAMP->`

E.g. `DAMP-> 0  0  0  0  0  0  0`. Fields 1–2: **α₀** `[1/s]` (mass-proportional coefficient) and **β₀** `[s]` (stiffness-proportional coefficient) — the classic Rayleigh damping pair `C = α₀M + β₀K`. Remaining 5 fields *(meaning unclear)*.

### 4.4 Layer/fiber sub-material mechanisms

Two distinct mechanisms exist for attaching reinforcement/layer sub-materials to a host material, matching the two rows in §4.2's table that reference "fiber materials":

**Layered beams** use a dedicated **`LAYERED_BEAM_COMPONENTS <n>`** keyword block: number, name, then one line of parameters — type (`0`=elastic, `1`=elastic-plastic, `2`=user), E, nu, and further softening/creep parameters (`E0_setup`, regularization flag, characteristic length, coupled softening flag, creep type/A/B); if `type==2` (user model), two further lines name the tension/compression stress-strain functions (`ft_fun`, `fc_fun`). Example (a reinforced-concrete beam, concrete matrix + steel reinforcement fibers):
```
LAYERED_BEAM_COMPONENTS 2
  1
concrete
2e+07  0.3  25  1  1000  10000  0  0.005  0  0  -1  0  1  
  2
steel
2e+08  0.3  25  1  235000  235000  0  0.005  0  0  -1  0  1  
```
Field-by-field for the parameter line (`v[0]`-indexed): `E`=`v[0]` (`2e+07` concrete, `2e+08` steel); `nu`=`v[1]` (`0.3` both); `v[2]`=`25` *(meaning unclear — identical for both materials despite very different real-world unit weights, so probably not unit weight; possibly a shared default such as a reference temperature)*; `type`=`v[3]`=`1` (elastic-plastic, both); `ft`=`v[4]` (tension strength); `fc`=`v[5]` (compression strength); `reg_soft`=`v[6]`=`0`; `char_len`=`v[7]`=`0.005` (both); `E0_setup`=`v[8]`=`0`; `coupTC_soft`=`v[9]`=`0`; `creep_type`=`v[10]`=`-1` (both — creep disabled); `creep_A`=`v[11]`=`0`; `creep_B`=`v[12]`=`1` (both).

**`SIG_EPS_FUN <n>`**: user-defined stress-strain functions, referenced from `LAYERED_BEAM_COMPONENTS`/nonlinear materials by name. *(Full field layout not confirmed.)*

**Layered shells** instead use a **second `NUM_MATERIALS= <n>` block**, placed immediately after the layered shell material's own single-material block — confirmed from `NL_shell_traction.inp`. **A layer's material is not restricted to `Fiber Shell`** — a layer can instead reference an `Orthotropic shell` material — both formulations coexist in the same second pass, distinguished by their `<type>` line:
```
NUM_MATERIALS= 1
MATERIAL  26.00 1
Shell Layered
No name
BUTTONS= 0 1 0 0 1 0 1 0 0 0 0 0 0 0
...                                    <- this material's own ELAS->/GEOM->/etc., see §4.2/§4.3
NUM_MATERIALS= 2
MATERIAL  26.00 1
Fiber Shell
concrete
...                                    <- core material, id 1, referenced by GEOM->'s core_material field
MATERIAL  26.00 2
Fiber Shell
steel
...                                    <- fiber material, id 2, referenced by GEOM->'s per-fiber material fields
```

---

## 5. Existence & Load Functions

### 5.1 Existence functions (`EXIST_FUNC`)

```
EXIST_FUNC <n>
<number> <name>
<nInstances>
<start1> <end1> <start2> <end2> ...   <- nInstances pairs of [start,end] time intervals
...                                     <- repeated for each of the n functions
```

`number` (int), `name` (rest of the line after the number), `nInstances` (how many active time-intervals follow), then one line listing `nInstances` `[start, end]` pairs — the element/entity referencing this EF exists only during these time interval(s). EF number `0` is never defined in the file — it's implicit, meaning "permanent" (active for the interval `[0, 1e36]`).

An EF reference isn't only used to control whether an *element* exists — it can also gate a boundary condition's fixity, letting a `.inb` record stay nominally fixed while only actually being enforced during that EF's active interval (see §8.1 for how a `.inb` fixity block references an EF this way).

Example:
```
EXIST_FUNC 1
1 No name
1
0 3.37e+38 
```

### 5.2 Load functions (`LOAD_FUN`)

```
LOAD_FUN <n>
<number> <nSteps> <name>
<flags>
<t0> <scale> <type>
<time1> <value1>
<time2> <value2>
...                    <- nSteps [time,value] pairs
...                    <- repeated for each of the n functions
```

`number`, `nSteps`, `name` on the first line; a `flags` line (free text/flag string, e.g. `No flags`); an `options` line of `[t0, scale, type]` (`t0` = time origin, `scale` = multiplier applied to all step values, `type` = interpolation/repeat-mode selector, exact codes unknown); then `nSteps` lines of `[time, value]` pairs defining the piecewise function. LF number `0` is implicit — a constant multiplier of `1` for the whole analysis, never defined explicitly in the file.

Example:
```
LOAD_FUN 1
1 1 No name
No flags
0 1 0
0 0
```
(here `nSteps=1`, giving one `[time,value]` pair: `time=0, value=0` — a trivial/placeholder function, unused since this example has no active nodal loads.)

---

## 6. Geometry

### 6.1 Nodes (`.ing`)

The finite-element mesh nodes:

```
<id> <x> <y> <z> <flag>
```

One line per node, count taken from the header (§3.2, "number of nodes"). The trailing `<flag>` (always observed as `0`) *(meaning unclear)*.

**Hand-editing gotcha**: `<flag>` must be written as a plain integer literal (`0`), not a float — writing it as `0.000000000000e+00` (e.g. by careless string-formatting when programmatically rewriting node lines) produces a file ZSoil silently refuses to open. Confirmed by reproducing the failure: a batch coordinate-update script that reused the line's float formatter for every field, including this one, broke the file; reverting just this field to a bare `0` fixed it.

Example:
```
.ing
1 0.000000000000e+000 0.000000000000e+000 0.000000000000e+000 0
2 1.000000000000e+000 0.000000000000e+000 0.000000000000e+000 0
```

### 6.2 Geometry points (`.pob`)

Points used to build geometric (sketch-level) construction lines/arcs, distinct from mesh nodes — same record shape as `.ing` (`<id> <x> <y> <z> <flag>`), count from the header ("number of geometrical points"):

```
.pob
1 1.526556658860e-15 6.383782391595e-16 0.000000000000e+00 0
2 1.526556658860e-15 1.000000000000e+00 0.000000000000e+00 0
```

### 6.3 Geometrical objects (`.gob`)

Sketch-level lines/arcs/polylines built from the `.pob` points, used by the GUI's mesh generator. Blank-line-terminated list, preceded by a count line:

```
.gob
<count>
<id> GEOMLINE <flag> <flag> <colorCode>
<name>
<pointId1> <pointId2>
...                          <- repeated per object
```

Example (5 lines forming a closed boundary from 6 points):
```
.gob
5
1 GEOMLINE 0 2 13107200
No name
1 2 
2 GEOMLINE 0 2 13107200
No name
2 3 
```

`GEOMARC` and `GEOMPLINE` variants (for arcs and polylines) follow the same `<id> <TYPE> ... / <name> / <point references>` shape; their point-reference line likely differs (an arc needs 3 points or a center+radius, a polyline needs a variable-length point list) *(exact field layout unknown)*.

### 6.4 Subdomains (`.sdm`) and related meshing metadata

Subdomains (corresponding to header counts "number of continuum 2D/3D subdomains", "beam/truss/membrane/shell subdomains") capture the GUI's meshing region definition, including a full sub-mesh boundary and interior node listing. This is normally GUI-generated and not something to hand-edit; documented here at a structural level only:

```
.sdm
<count>
<id> <flag> SUBD_2D <...>            <- subdomain header/meshing-parameter line
<name>
<flags line>
<meshing parameters line>
EXCAV_DATA <n>                       <- excavation-stage boundary data, n groups of point-index lines
<boundary index lines...>
SURFACE <n>
<n boundary point indices>
NODES <count>
<id> <x> <y> <z>                     <- one line per interior/boundary node of the subdomain's own sub-mesh
...
```

Example header line (one 2D subdomain): `1 128 SUBD_2D 4 4 0 4 1 4 0 1 0 0`. *(Full field-by-field meaning of the header/meshing-parameter lines is unknown — treat this block as opaque GUI-managed data unless specifically working on subdomain/staging definitions.)* Related markers `.psd`, `.goa`, `.sg0` appear immediately after `.sdm` in the marker sequence and are typically empty; likely auxiliary subdomain data *(unknown)*.

### 6.5 Local bases, axes, and auxiliary geometry

The following markers are typically empty, so only their header-count purpose can be documented, not their populated syntax:

| Marker | Header count label | Notes |
|---|---|---|
| `.glk` | number of Local bases | Custom local coordinate systems, referenced by element records' `rm1`/`rm2` fields. |
| `.axs` | *(no direct header label)* | *(meaning unclear)* |
| `.apl` | number of auxiliary planes | Construction planes for the GUI's sketch tools. |
| `.igl` | number of auxiliary points | Auxiliary sketch points, distinct from `.pob`. |
| `.isd` | number data super elements | Reusable geometry/mesh templates. |

If a hand-edit needs to touch any of these, populate them by building the corresponding geometry in the ZSoil GUI on a minimal test file and diffing the resulting `.inp`, rather than authoring them from this reference alone.

---

## 7. Elements

This chapter documents the element-definition blocks of the `.inp` file. Each block is introduced by a dot-prefixed marker and is either count-terminated (the count comes from the global header-count block, §3.2) or blank-line-terminated.

A recurring pattern across nearly every element record is:

```
<idx> <number> <TAG> <node1> ... <nodeN> <...type-specific fields...> <mat> <rm1> <rm2> <EF> <LF> [<extra>]
```

where `<idx>` is a **global counter shared across every element block in the file**, incrementing continuously in file order regardless of type — e.g. in a model with 3943 `.i0g` continuum elements followed by 32 `.ibg` beams, the first beam's `<idx>` is `3944`, not `1`. `<number>` is a **per-block-type counter that restarts at 1** for each new element section (`.i0g`, `.ibg`, `.itg`, `.anh`, ...) — it only happens to equal `<idx>` for continuum elements because `.i0g` is the first element block in the file. Confirmed empirically: `<idx>` (not `<number>`) is the value used whenever one element cross-references another — `.ics`/`.icg` contact records name their paired element by `<idx>`, and `.anh` anchor headers name their `.itg` truss by `<idx>`. When hand-inserting new elements, give them fresh `<idx>` values continuing past the current file-wide maximum (gaps in the sequence appear tolerated — ordering, not contiguity, is what other records seem to rely on) and let `<number>` continue whatever count is natural for that element's own block. Several trailing integer fields recur across element families whose exact purpose is unknown; these are called out per-section.

### 7.1 `.i0g` — Volumic/continuum elements

Count-terminated: the record count equals `nVolumics3D + nVolumics2D` (header fields "number of continuum 3D elements" / "number of continuum 2D elements"). One line per element. Type tag in field 3 selects the node count and the position of the trailing field group.

Node indices follow the tag, then — at an offset `pos` that depends on the tag (`pos=5` for `Q4`, `pos=10` for `B8`) — the fields `mat` (`v[pos+4]`), `rm1` (`v[pos+5]`), `rm2` (`v[pos+6]`), `EF` (`v[pos+7]`), `LF` (`v[pos+8]`) appear, followed by one trailing field whose meaning is unclear.

**Q4 (4-node quad, 2D)**:
```
.i0g
1 1 Q4 1 2 3 4 1 1 1 0 0 0 0 0
```
Field breakdown (0-indexed after split): `v0`=1 (idx), `v1`=1 (element number), `v2`=`Q4`, `v3..v6`=nodes 1,2,3,4, `v7..v8`=1,1 *(unclear)*, `v9`=1 (mat), `v10`=0 (rm1), `v11`=0 (rm2), `v12`=0 (EF), `v13`=0 (LF), `v14`=0 (trailing, unclear).

**Node order / face numbering (confirmed empirically)**: `v3..v6` must be listed counter-clockwise (positive-area shoelace sum) for the element to be valid — a clockwise or self-intersecting listing produces a degenerate/inverted element. Local face (edge) numbering, otherwise undocumented by ZSoil, follows directly from this listed order: **face `k` is the edge from `node_k` to `node_{k+1}`** (1-indexed, wrapping — so face 1 = n1→n2, face 2 = n2→n3, face 3 = n3→n4, face 4 = n4→n1). This is what `.ics`/`.icg` "paired-elem face" fields (§7.8, §7.9) and `.gsl` `UNI_LOAD` face references (§9.3) actually mean; confirmed by cross-checking multiple real `.ics`/`.icg` records against the coordinates of the nodes on the named face.

**B8 (8-node hex)**:
```
.i0g
1 1 B8 1 2 4 3 5 6 8 7 1 1 1 1 0 0 0 0 0
```
`v0`=1, `v1`=1, `v2`=`B8`, `v3..v10`=nodes (1,2,4,3,5,6,8,7), `v11..v13`=1,1,1 *(unclear — one more flag field than `Q4`, plausibly a 3D-specific integration/formulation flag)*, `v14`=1 (mat), `v15`=0 (rm1), `v16`=0 (rm2), `v17`=0 (EF), `v18`=0 (LF), `v19`=0 (trailing, unclear). This layout is identical for the EAS and B-bar formulation variants — the formulation choice is not encoded in this record.

**B8 face numbering** — relevant wherever a `faceId` targets a `B8` element (`.gsl` `UNI_LOAD`, §9.3; `.ple` `POINT_LOAD`/`SURF_LOAD` with `targetType 8`, §9.4). Local nodes 1–4 form one quad face and 5–8 the opposite quad face, connected edge-to-edge (1–5, 2–6, 3–7, 4–8); the 6 face ids are assigned so that **opposite faces sum to 7**:

| Face id | Local nodes |
|---|---|
| 1 | 1-2-3-4 |
| 2 | 1-2-6-5 |
| 3 | 4-1-5-8 *(inferred)* |
| 4 | 2-3-7-6 |
| 5 | 3-4-8-7 *(inferred)* |
| 6 | 5-6-7-8 |

Faces 3 and 5 are inferred by elimination and by the "opposite faces sum to 7" pattern the other four establish; treat them as unconfirmed if precision matters.

**T3 (3-node tri, 2D)**:
```
.i0g
1 1 T3 2 1 3 1 1 0 0 0 0 0
2 2 T3 1 4 3 1 1 0 0 0 0 0
```
`v0`=1, `v1`=1, `v2`=`T3`, `v3..v5`=nodes (2,1,3), then `v6..v12`=`1 1 0 0 0 0 0` (7 trailing fields vs. 8 for `Q4`/`B8` — one fewer). The trailing-field-to-attribute mapping for `T3` is *(unclear)* — by position it would plausibly be 2 unclear ints, then mat, rm1, rm2, EF, LF, but this is not confirmed.

A `W6` (6-node wedge/prism) tag exists, with the same layout pattern as `B8` (nodes then `1 1 1` then mat/rm1/rm2/EF/LF/trailing):
```
1 1 W6 1 4 6 2 3 5 1 1 1 1 0 0 0 0 0
```

### 7.2 `.ibg` — Beam elements

Count-terminated: count = "number of Beams (*.ibg),(*ibm)". Each `BEL2` (2-node beam) element occupies 5 physical lines.

```
.ibg
411 1 BEL2 1 3
0  0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00
0  -0.000000e+00 1.000000e+00 -0.000000e+00 -0.000000e+00 1.000000e+00 -0.000000e+00
 11 0 0 2 0 0
0
```
Line 1: `<idx> <number> BEL2 <node1> <node2>` = `411 1 BEL2 1 3`.
Line 2: six floats, always zero *(meaning unclear)*.
Line 3: orientation vector, six floats (two triplets) defining the beam's local axis.
Line 4: ` 11 0 0 2 0 0` → `mat`=11 (`v0`), then `v1`,`v2`=0,0 *(unclear, plausibly rm1/rm2)*, `EF`=2 (`v3`), `v4`,`v5`=0,0 *(unclear, plausibly LF and a trailing flag)*.
Line 5: end-release/hinge flag, `0` typically; if `1`, two additional (undocumented) lines follow.

### 7.3 `.itg` — Truss elements

Count-terminated: count = "number of truss elements (*.itg),(*.itm)". Each record is `TRS2` (truss, `type=0`) or `LNK2` (anchor/link, `type=1`).

**No prestress**:
```
.itg
126 1 TRS2 158 157 0 0 0 0
 13 0 0 1 0 0
0
```
Line 1: `<idx> <number> TRS2 <node1> <node2> <4 trailing zeros>` — the four trailing fields after the node pair are *(meaning unclear)*, always `0 0 0 0` in every model seen.
Line 2 (material line): ` 13 0 0 1 0 0` → `mat`=13 (`v0`), `rm1`=0 (`v1`), `rm2`=0 (`v2`), `EF`=1 (`v3`), `LF`=0 (`v4`), trailing `v5`=0 *(unclear)*.
Line 3: prestress record count = `0` → no prestress lines follow.

**With prestress**:
```
.itg
126 1 TRS2 158 157 0 0 0 0
 13 0 0 1 0 0
1
1.000000000000e+02 0 2  1
```
Prestress count = `1`, followed by one record: `1.000000000000e+02 0 2  1` → prestress force = `1.0e+02` (first field), followed by three integer fields (`0`, `2`, `1`) whose individual meanings are *(unclear)* — plausibly a load-function reference, a prestress-type code, and a stage/sequence index.

### 7.4 `.iff` — Fixed Anchor Zones

Header count field: `"# number of Fixed Anchor Zones"`. Despite a non-zero header count in anchor models, the `.iff` block itself is typically completely blank (zero records). (The "Fixed anchor zone interface" text that does appear in some files is a material-catalog entry name, unrelated to this marker.) No populated `.iff` record syntax is available.

### 7.5 Shell elements: `.ilg` + `.ilt` (thickness) + `.ily` (layers) + `.ish` (hinges)

**`.ilg`** — count-terminated. The count that governs this block is **not** the generic header field `"number of shell elements (*.ilg),(*.ilm)"` (typically `0` even in files with real shell elements) but the separate field `"number of Shell one layer elements"`. Hand-editors adding/removing `SXQ4`/`SHQ4` records must update this second count field, not the first.

Type tags: `SXQ4` (4-node thin/one-layer shell, `pos=6`) and `SHQ4` (8-node thick shell, `pos=10`). Fields `mat`, `rm1`, `rm2`, `EF`, `LF`, `thick` (thickness-table index, resolved against `.ilt`) sit at `v[pos+4..pos+9]` respectively.

```
.ilg
2 1 SXQ4 1 5 6 2 1 1 1 2 2 2 3 1 1 0
```
`v0`=2 (idx), `v1`=1 (number), `v2`=`SXQ4`, `v3..v6`=nodes (1,5,6,2), `v7..v9`=1,1,1 *(unclear, same pattern as B8's three extra ints)*, `v10`=2 (mat), `v11`=2 (rm1), `v12`=2 (rm2), `v13`=3 (EF), `v14`=1 (LF), `v15`=1 (thickness-table index → `.ilt` record 1), `v16`=0 (trailing, unclear). No `SHQ4` example is available.

**`.ilt`** (thickness table) — blank-line-terminated list of thickness definitions referenced by the `thick` field above. Each entry starts with a `type` code; `type=0` is followed by one line giving a single float (uniform thickness); `type=1` would be followed by two float lines but no example is available.

```
.ilt
0  
0.5
```
Record 1: `type`=0, thickness = `0.5`.

The `type` field can also carry an optional trailing free-text label after it:
```
.ilt
0  thickness=1m
1
```
Here `type`=0 with label `thickness=1m` (a human-readable echo of the value on the next line, `1`); the label is cosmetic/GUI-generated rather than functionally required.

**`.ily`** (layers) and **`.ish`** (hinges) — both typically blank/empty; no populated example is available for either. **Note**: despite the name, `.ily` is *not* where a `"Shell Layered"` material's core/reinforcement layers live — those are fully defined within that material's own `GEOM->` block (§4.3.2/§4.4). `.ily`'s actual purpose is unknown.

### 7.6 `.ipg` — Seepage elements

Count-terminated: count = `"number of water Seepage's (*.ipg) (*.ipm)"`, typically `0`. Expected record layout:
```
<idx> <number> SPL2 <elem> <face> <node1> <node2> <mat> <EF> <LF>
```
each followed by one skipped line — but no real example is available to confirm this.

### 7.7 `.ivg` — Convection elements

Header count field `"# number of convection element (*.ivg)"`, typically `0`. No populated example is available.

### 7.8 `.icg` — Contacts on continuum (volumic-volumic interfaces)

Count-terminated: count = `"number of contact lines (*.icg), (*.icm)"`. Both `C_L2` (2D line contact) and `C_Q4` (3D quad-face contact) tags exist. Interface `type` codes (first field of the trailing data line): `0` = continuity with pressure, `1` = contact, `2` = continuity without pressure.

**`C_L2` (2D)**:
```
.icg
1 1 C_L2 2 1 1 3
 5 8 4 7 1 1 0
0.000000e+00 0
No name
1 2 0 0 0.000000e+00
```
Line 1: `<idx> <number> C_L2 <elem1> <face1> <elem2> <face2>` — confirmed: `v3`/`v4` and `v5`/`v6` are each an `(element idx, local face)` pair (face numbering per §7.1), naming the **two element edges being tied together**. Here `volele`=2 face=1, and `elem2`=1 face=3.
Line 2: 4-node connectivity ` 5 8 4 7` (the two coincident node pairs across the interface), followed by `1 1 0` — the second of these three trailing fields (here `1`) is the **count of trailing type/mat/EF/LF records** that follow (see the multi-record example below); the other two are *(unclear)*.
Line 3: `0.000000e+00 0` *(unclear, plausibly initial gap + a flag)*.
Line 4: interface name (`No name`).
Line 5 (repeated per the record count from line 2): `<type> <mat> <EF> <LF> <trailing float>` = `1 2 0 0 0.000000e+00` → type=1 (contact), mat=2, EF=0, LF=0, trailing=0.0 *(unclear)*.

**Connectivity node order (confirmed)**: on line 2, the 4 nodes are `<elem1_faceNode_k+1> <elem1_faceNode_k> <elem2_faceNode_k> <elem2_faceNode_k+1>` — i.e. `elem1`'s face-node pair is listed **reversed**, `elem2`'s **forward**. E.g. for a tie where `elem1` face 3 runs `n3=1806 → n4=1802` and `elem2` face 1 runs `n1=1801 → n2=1805`, the connectivity line reads ` 1802 1806 1801 1805` (elem1 reversed, elem2 forward). This opposite winding is consistent with the two faces having opposite outward normals.

**Real-world use beyond staged construction**: `type`=2 ("continuity without pressure") is used not only at documented material/staging boundaries but also as a generic **rigid tie between two independently-numbered mesh regions that are geometrically coincident but don't share node IDs** — e.g. where ZSoil's mesher stitched two separately-generated subdomains together at a seam with no wall, contact, or staging involved. Encountered in a real model at a plain soil-soil seam ~30 m from any structural or staging feature: two nodes at identical coordinates but different IDs (no shared node, no `.ikg` kinematic constraint) were tied only by a `.icg type=2` record. **Do not assume a duplicated-position, differently-numbered node pair is disconnected/erroneous** — check `.icg` for a tie before concluding the mesh has a crack.

**When splitting/refining an element that has a `.icg` tie on one of its edges**: the tie's `elem`/`face` reference must be updated to whichever new sub-element now owns that edge (the *original* element idx no longer exists) — and remember to also search for `.icg`/`.ics` records referencing that element from other, unrelated ties (an element can be named by more than one contact/tie record, e.g. one tie on its near edge from the feature you're editing, and an unrelated tie on a different edge from something else entirely).

**`C_Q4` (3D)**: same field structure as `C_L2` but with an 8-node connectivity line (two 4-node faces):
```
.icg
1 1 C_Q4 1 5 2 2
 4 7 15 12 5 8 16 13 1 1 0
0.000000e+00 0
No name
1 2 0 0 0.000000e+00
```

**Multiple staged records on one contact** — the same geometry can be re-used across load stages by bumping the trailing record count on the node line:
```
.icg
1 1 C_Q4 2 3 1 4
 4 14 17 8 3 13 16 7 1 3 0
0.000000e+00 0
No name
1 2 1 1 0.000000e+00
1 2 2 1 0.000000e+00
1 2 3 1 0.000000e+00
```
Here the geometry is fixed but `EF` cycles 1→2→3 across the three trailing records — this is how a contact's activation is staged across construction phases without redefining its geometry.

### 7.9 Contacts on structures: `.ics` + `.scs` + `.ims`

**`.ics`** — count-terminated: count = `"number of inetrfaces (*.ics)"` [sic]. Two forms: `C_Q4` (contact paired with a volumic-element face or shell-element face) and `C_L2` (contact paired with a beam element).

**`C_Q4` paired with a shell element**:
```
.ics
1 1 C_Q4 2 1 0 1 2
No name
1 3
 2 8 10 4 1 7 9 3
1 1
0.000000e+00 0
1 3 3 0 0.000000e+00
```
Line 1: `<idx> <number> C_Q4 <cnt-number> <?> <?> <nsides> <side>` → `v3`=2 is the contact element number; `v6`=1 → single-sided; `v7`=2 → negative side active (`v6`=`2` ⇒ double-sided, both sides active).
Line 2: interface name. Line 3: `<paired-elem> <paired-face>` = `1 3` (shell element 1, face 3). Line 4: 8-node connectivity. Line 5–6: *(unclear)*. Line 7: `<?> <mat> <EF> <LF> <trailing float>` = `1 3 3 0 0.000000e+00` → mat=3, EF=3, LF=0.
If double-sided, lines 3–7 repeat once more for the negative side.

**`C_L2` paired with beam elements**:
```
.ics
1 1 C_L2 482 1 0 1 2
No name
356 1
 146 144 145 143
1 1
0.000000e+00 0
1 18 2 0 0.000000e+00
```
Same 7-line-per-record structure as the `C_Q4` case: header line (here `v3`=482 is the beam element number the contact rides on), name, a `<?> <?>` line, 4-node connectivity, two skipped lines, and a final `<?> <mat> <EF> <LF> <trailing>` line. `v6` (here `1`) is `nsides`: `1` = single-sided (one `<paired-elem face>` block, as above); `2` = double-sided — a beam embedded in soil on both faces (e.g. a wall) gets **two** `<paired-elem face> / <connectivity> / <skip> / <skip> / <data>` blocks back to back, one per face, sharing the one header/name pair.

**Double-sided example** (a wall's bottom beam segment, contacted on both its west and east faces):
```
.ics
32 32 C_L2 3944 1 0 2 3
No name
1480 2
 1806 14 1804 14
1 1
0.000000e+00 0
1 7 5 0 0.000000e+00
1833 2
 14 1807 14 1804
1 1
0.000000e+00 0
1 7 5 0 0.000000e+00
```
Beam `3944` connects nodes `14`(lower) → `1804`(upper). West side: paired-elem `1480` face `2`; connectivity ` 1806 14 1804 14` = `<soil@upper> <soil@lower> <beam@upper> <beam@lower>`. East side: paired-elem `1833` face `2`; connectivity ` 14 1807 14 1804` = `<soil@lower> <soil@upper> <beam@lower> <beam@upper>` — **the two sides use opposite node order** (west reversed relative to the beam's own node1→node2 direction, east forward), matching the `.icg` reversed/forward convention (§7.8) and presumably serving the same opposite-outward-normal purpose.

**Node reuse gotcha**: where two beam elements share an endpoint (e.g. beam A ends and beam B starts at the same beam node), ZSoil's mesher does **not** reuse one soil-side contact node across both beams' contact records — each beam segment gets its **own**, separately-numbered soil-side node at that shared point (per side). Don't assume the soil-side node from an adjacent beam segment's contact can be reused when hand-building a new one; always allocate a fresh coincident node.

**`.scs`** and **`.ims`** — typically empty; no populated example is available for either.

### 7.10 `.ikg` — Kinematic constraints, `.gnl` — Nodal links, `.ijg` — Joints

All three are typically blank (`.ikg` header: `"# number of kinematic constrains (*.ikg)"`; `.gnl` header: `"# number of Nodal Links (*.gnl)"`; `.ijg` has no dedicated header label). No populated examples are available for any of the three.

### 7.11 `.pil` — Piles

Count-terminated: count = `"number of Piles"`. Structure: header line, then 4 always-skipped lines (a flag, a second flag, and a 2-line orientation vector — same shape as the beam orientation record), then a material line (`mat`, `qsmat`, `qpmat`), then `nSeg+1` point lines tracing the pile axis.

```
.pil
12 1 PILE 3 8 1.000000 0.250000 20 0 1 1 126 0 1 2
1
11 
0  0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00
0  1.000000e+00 0.000000e+00 -0.000000e+00 1.000000e+00 0.000000e+00 -0.000000e+00
 11 12 0 13 0
5.000000000000e-01 0.000000000000e+00 5.000000000000e-01 1 1
5.000000000000e-01 -1.000000000000e+00 5.000000000000e-01 2 1 2
...
5.000000000000e-01 -7.500000000000e+00 5.000000000000e-01 1 8
```
Header line: `<idx> <number> PILE <EF> <nSeg> <diam?> <?> ...` = `12 1 PILE 3 8 1.000000 0.250000 20 0 1 1 126 0 1 2` → `EF`=3 (`v3`), `nSeg`=8 (`v4` — the pile axis is traced by `nSeg+1`=9 points below). `v5`=1.0, `v6`=0.25 are plausibly diameter/perimeter-related; remaining fields *(unclear)*.
Material line: ` 11 12 0 13 0` → `mat`=11 (`v0`), `qsmat`=12 (`v1`, skin-friction material), `qpmat`=13 (`v3`, tip-resistance material).
Points: `nSeg+1`=9 lines of `<x> <y> <z> <flags>` tracing the pile axis; endpoints have one trailing flag, interior points have two — plausibly a segment-count/segment-index pair, but *(unclear)*.

**Qs/Qp material selection**: the tip-resistance-model toggle lives in the header line's `v9` field and the material line's `qpmat` field: `v9`=0 (Qstanphi model, `qpmat`=0) vs `v9`=1 (constant Qs/Qp model, `qpmat` populated).

### 7.12 `.gbh` — Boreholes

Count-terminated: header field `"# number of boreholes"`, typically `0` — no populated `.gbh` block is available. Even with 0 boreholes, default interpolation-method records persist:
```
.gbh
0
KRIGING 1
2 0.200000 0.800000 200.000000 0.250000
SEDIMENTATION 0
```
The per-borehole record layout (`name` / `x y z nLayers` / `nLayers × [top bottom material]`) is inferred, *(not confirmed against real data)*.

### 7.13 `.nil` — Nails

Count-terminated: header field `"# number of Nails"`. Structure parallels `.pil`: header line, a `<count> <beam-element-number>` line referencing an already-defined `.ibg` `BEL2` element, two orientation-vector lines, a material line, then `nSeg+1` axis points.

```
.nil
127 1 NAIL 3 23 1.000000 0.050000 20 0 1 0 14 6 5
1 126
0  0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00
0  0.000000e+00 -1.000000e+00 0.000000e+00 0.000000e+00 -1.000000e+00 0.000000e+00
 11 12 0 0 0
5.000000000000e+00 5.000000000000e-01 0.000000000000e+00 1 51
4.800000000000e+00 5.000000000000e-01 0.000000000000e+00 2 51 52
...
4.000000000000e-01 5.000000000000e-01 0.000000000000e+00 1 74
```
Header line: `EF`=3 (`v3`), `nSeg`=23 (`v4`; 24 axis points follow). Line 2 (`1 126`): the number of the `.ibg BEL2` beam element this nail's structural bar is defined on. Material line ` 11 12 0 0 0`: `mat`=11 (matches the structural beam's material), `qsmat`=12; no separate tip material (consistent with nails having no tip resistance).

### 7.14 `.anh` — Anchors

Count-terminated, no dedicated header label. Simplified version of `.pil`/`.nil` — header line, a short material line, then `nSeg+1` axis points (no beam/orientation-vector lines).

```
.anh
127 1 ANCH_HEAD 126 3 20 1.000000 0.050000 4.000000 1 0 0.150000
 14 0
4.400000000000e+00 5.000000000000e-01 0.000000000000e+00 1 53
4.200000000000e+00 5.000000000000e-01 0.000000000000e+00 2 54 55
...
4.000000000000e-01 5.000000000000e-01 0.000000000000e+00 1 74
```
`v3`=126 references the `.itg` `TRS2` truss element this anchor's free length is built on; `v5`=20 is plausibly `nSeg` (21 axis points follow, matching). Material line ` 14 0`: single `mat`=14, no separate `qsmat`/`qpmat`.

**Axis-point trailing fields (confirmed)**: each axis-point line's trailing `<count> <id...>` is **not** node indices (despite superficially resembling the `.pil`/`.nil` point-flag pattern) — it is `<count of .i0g continuum elements> <element idx...>` identifying which continuum element(s) the fixed-anchor-zone point falls in: `1 <id>` when the point lies strictly inside element `<id>`, `2 <id1> <id2>` when it lies on the shared edge between two elements. Confirmed by checking that the referenced elements' node coordinates bound the axis point's own `(x,y)`. This is presumably how ZSoil interpolates/distributes the bond-length load transfer into the surrounding soil mesh. Practical consequence: if an anchor's fixed zone is moved (e.g. to a different elevation) without remeshing, these element references go stale and must be recomputed by point-locating the new trajectory against the current `.i0g` mesh — they will not simply carry over.

### 7.15 Cables/tendons: `.cbl` + `.bcb` + `.bcl`

**`.cbl`** (cable/tendon element instances) — typically blank. **`.bcb`** (cable *geometry* catalog, referenced by name from `.cbl` records) — count-terminated, and notably present with a **built-in default catalog of 2 entries** in most files (even those with zero actual cable elements), suggesting these are ZSoil-authored default templates rather than user data:
```
.bcb
2
Straight cable
1 1 1 2
0.000000e+00 2.000000e-01
1.000000e+00 2.000000e-01
Cable geom. 1
1 1 2 3
0.000000e+00 2.000000e-01
5.000000e-01 1.000000e-01
1.000000e+00 2.000000e-01
0 0
```
Structure per entry: a name line, a flag line whose last field is the control-point count (`1 1 1 2` → 2 points; `1 1 2 3` → 3 points), then that many `<relative-position 0..1> <offset>` point lines describing the cable profile along the host beam/truss (a straight profile for entry 1; a parabolic sag profile for entry 2). A trailing `0 0` line follows the last entry *(meaning unclear)*.

**`.bcl`** — typically blank; likely a duct/tendon-loss-coefficient catalog analogous to `.bcb`, but unconfirmed.

### 7.16 Heat exchangers (`.hex`, `.hef`), Groups (`.grp`)

Typically blank/`0`. `.hex`/`.hef` header: `"# number of Heat exchangers"`. No populated examples are available for any of the three.

### 7.17 `.gsh` — General shell

Count-terminated; the count matches `"number of Shell one layer elements"` (the same field that drives `.ilg`, §7.5).

**Large model, one line per shell**:
```
.gsh
966
65 0
66 0
67 0
...
1030 0
```
One line per shell, each `<element-number> <flag>`, flag = `0` throughout (default/inactive).

**Small model, inconsistent line count**: a declared count of `1` can be followed by **two** data lines, inconsistent with the one-line-per-shell pattern above:
```
.gsh
1
2 1
1 3
```
It is unclear whether this is a single two-line record or the count has a different meaning here *(meaning unclear)* — hand-editors relying on this marker should verify against a known-good ZSoil-exported file rather than this documentation alone.

---

## 8. Boundary Conditions

### 8.1 Nodal boundary conditions (`.inb`)

One line per constrained node, count from the header ("number of boundary conditions"). The record is `<idx> <nodeId> <flag>` followed by **one fixity block per translational DOF** — `[fixedFlag, prescribedValue, EF, LF, ?]` — and ends with a **trailing flag** selecting the local basis the fixity directions are expressed in: `0` = the model's global Cartesian axes, `1` = the local basis defined in `.ilb` (see "Local basis definitions" below).

Within each block, **EF** (token 3) is an Existence Function reference: `0` = none (ordinary constant fixity); otherwise the fixity is only actually *enforced* while that EF's interval is active — outside it, the DOF is effectively free even though `fixedFlag` still reads `1`. **LF** (token 4) is a Load Function reference: `0` = none, `prescribedValue` stays constant; otherwise the prescribed value follows that function's time history instead. This is confirmed for 3D beam (6-DOF) nodes — see "3D beam nodes" below — but not independently checked for solid/continuum nodes, which may use a simpler convention. Token 5 remains unclear (always `0` in every example seen).

- **3D** (block width 5, 19 tokens total, flags at token indices 3, 8, 13): `<idx> <nodeId> <flag> [xFixed xVal EF LF ?] [yFixed yVal EF LF ?] [zFixed zVal EF LF ?] <basisFlag>`.

  Example (node 1 fixed in x and z):
  ```
  .inb
  1 1 1 1 0.000000000000e+00 0 0 0 0 0.000000000000e+00 0 0 0 1 0.000000000000e+00 0 0 0 0
  ```

- **2D — width varies by file, not by dimension alone.** Some 2D files use the **same 19-token / width-5 layout as 3D** (flags at indices 3, 8, 13):
  ```
  .inb
  1 1 1 1 0.000000000000e+00 0 0 0 1 0.000000000000e+00 0 0 0 0 0.000000000000e+00 0 0 0 0
  ```
  while others use a **narrower 16-token / width-4 layout**, flags at indices 3, 7, 11:
  ```
  .inb
  1 1 1 1 0.000000000000e+000 0 0 1 0.000000000000e+000 0 0 0 0.000000000000e+000 0 0 0
  ```
  This is a **format evolution within v26** rather than a 2D/3D distinction — older files (older `createdWithVersion`, §3.1) use the narrower layout. **Practical guidance**: when hand-editing `.inb`, always match the token width already used elsewhere in the *same* file rather than assuming a fixed layout from this reference.

**3D beam nodes (6 DOF): two `.inb` records per node.** A 3D beam node has 6 DOF — 3 translations and 3 rotations — so it gets **two** `.inb` records instead of one, both sharing the same `<nodeId>`: one with `<flag>=1` (governs X/Y/Z translation fixity), the other with `<flag>=2` (governs RX/RY/RZ rotation fixity).

Example — a 3-phase isostatic beam mechanism (node 1 = fixed end, node 2 = driven end), where each rotational DOF's fixity is toggled on/off over time via `EXIST_FUNC`, and translations/rotations are driven via `LOAD_FUN`:
```
.inb
1 1 1 1 0.000000000000e+00 0 0 0 1 0.000000000000e+00 0 0 0 1 0.000000000000e+00 0 0 0 0
2 1 2 1 0.000000000000e+00 1 0 0 1 0.000000000000e+00 2 0 0 1 0.000000000000e+00 3 0 0 0
3 2 1 1 1.000000000000e+00 0 1 0 0 0.000000000000e+00 0 0 0 1 1.000000000000e+00 0 2 0 0
4 2 2 0 0.000000000000e+00 0 0 0 1 1.000000000000e+00 0 3 0 0 0.000000000000e+00 0 0 0 0
```
Node 1's rotation record (line 2) has all three rotational DOF nominally fixed (`fixedFlag=1`, `prescribedValue=0`), but each block references a *different* EF (`1`, `2`, `3` for RX, RY, RZ respectively) — matching an `EXIST_FUNC 3` block whose entries are named e.g. `1 BC ROTX` / `2 BC ROTY` / `3 BC ROTZ`. Since each EF is active over a different, non-overlapping time interval, this is how a model achieves "exactly one rotational DOF free at a time" using only three static-looking fixity lines — outside its own EF's interval, a nominally-fixed DOF is actually free.

Node 2's records (lines 3–4) reference LF numbers instead of EF numbers in the same token position (`1`, `2`, `3`, matching `LOAD_FUN 3` entries such as `1 "dx"` / `2 "dz"` / `3 "roty"`) — each fixed DOF's prescribed value follows that load function's ramp over time, rather than staying constant. `EF 0` / `LF 0` in this position (as in the other `.inb` examples in this section) means no gating / no imposed ramp — ordinary constant fixity.

**Local basis definitions (`.ilb`)**, referenced by the trailing flag above: when that flag is `1`, the fixity directions are expressed in a local basis defined by an `.ilb` record instead of the global axes. `.ilb` records come in 3 modes, selected by a `<id> <MODE>` header line:
```
1   CARTESN_3D          <- 3×3 rotation matrix (3 lines), explicit local x/y/z axes
1   VECTOR_3D           <- 1 line: normal vector to the local y-z plane
1  2  2                    (e.g. this vector); local y/z axes within that plane
                            are then chosen arbitrarily
2   CYLINDR_3D          <- 2 lines: a direction and a point defining a cylinder axis
1  0  0                    direction (local z is parallel to it)
2  1  2                    point (local x points radially outward from the axis
                            through this point)
```
*(Field layout not independently verified against a populated `.ilb` example — treat as reliable but unconfirmed if precision matters.)*

### 8.2 `.inc` — named constraint/support groups

Appears immediately after `.inb` in every file. Structure: one block per node (same count as `.inb`), each block being a header line, a 6-flag line, then a run of numeric lines, terminated by a free-text **name** whose content is informative — e.g. `Box UXUZ` (suggesting "constrain Ux, Uz").

```
<recordIdx> <nodeId> <flag> <flag> <flag>
<flag>[6]
<numeric line>                 <- repeated N times (N varies — possibly a per-DOF curve
...                                point count driven by a preceding flag)
<name>                          <- e.g. "Box UXUZ", "No name"
```

Best interpretation: named/grouped nodal constraint sets, layered on top of the plain `.inb` fixity — possibly nonlinear spring-support curves (the repeated `[0, value, 0, 0]`-shaped lines resemble force/displacement curve points) or a GUI "boundary condition group" feature that lets multiple `.inb` entries share a name and extra per-DOF data. If a hand-edit needs to add/modify `.inc` entries, cross-check against a GUI-exported file with the same named BC group rather than authoring from this description alone.

### 8.3 Periodic boundary conditions (`.pbc`)

Node-pair count, an EF line, a name, then a list of node-id pairs:

```
.pbc
<pairCount>
<EF>
<name>
<nodeIdA1> <nodeIdB1>
...                     <- pairCount lines
```

No populated example is available.

### 8.4 Field (water/heat/humidity) boundary conditions

The following markers cover boundary conditions for seepage/thermal/humidity analyses. No populated example is available for any of them — they are documented here only by their header-count purpose:

| Marker | Header count label |
|---|---|
| `.iwb` | number of water bound. cond. |
| `.iph` | number of pressure Heads loads |
| `.iwh` | number of Surface load defined by Pressure head |
| `.iab` | number of Heat BC |
| `.imb` | number of Humidity BC |
| `.iwf` | number of water flux |
| `.ihf` | number of heat flux |
| `.iuf` | number of humidity flux |

If a hand-edit needs to add one of these, the safest approach is to build a minimal seepage/thermal model in the ZSoil GUI and inspect the resulting block directly, rather than relying on this reference.

> **Note on `.isg` and `.isl`**: these two markers sit near `.iph`/`.iwh` in the marker sequence. `.isl`'s header label is "number of surface loads" — i.e. it is a **load**, not a boundary condition, and is documented in [§9 Loads](#9-loads) instead. `.isg` has no directly-matching header label; it most likely corresponds to "number of Surface load defined by Pressure head" and is also treated as load-related — see §9.

---

## 9. Loads

### 9.1 Nodal loads (`.inl`)

Count-terminated (from the header "number of nodal forces"). Node id, 6 load components, LF, then a name line:

```
<idx> <nodeId> ? ? <Fx> <Fy> <Fz> <Mx> <My> <Mz> <LF>
<name>
```

### 9.2 Beam loads (`.ibf`)

For each beam, a line giving the beam id and how many loads apply to it, followed by that many `[6 values, LF]` + name records:

```
<beamId> <nLoadsOnThisBeam>
<Fx> <Fy> <Fz> <Mx> <My> <Mz> ? <LF>
<name>
...                              <- repeated nLoadsOnThisBeam times, then next beam
```

This block is typically all-zero (every beam declares 0 loads), e.g.:
```
.ibf
1 0
2 0
3 0
```
This strongly suggests `.ibf` is a legacy mechanism, superseded in current-version files by loads-on-elements (`.ple`, §9.4) even for beam-targeted loads. Treat `.ple` as the primary mechanism for beam loads in new files.

### 9.3 Surface loads (`.gsl`)

Blank-line-terminated (no separate count field consumed here; the block simply ends at the first blank line). Two record types:

**`UNI_LOAD`** — uniform pressure/traction load over a set of element faces:
```
<idx> UNI_LOAD <?> <LF> <?> <nFaces> <?>
<name>
<Vx> <Vy> <Vz>                  <- load vector/magnitude data
<eleId1> <faceId1>
...                              <- nFaces lines of (element id, local face id)
```
(the local `faceId` numbering for `B8` volumic targets follows §7.1's B8 face-numbering convention; for a 2D `Q4` element, `faceId` `k` = the edge from the element's `node_k` to `node_{k+1}`, 1-indexed and wrapping — see the confirmed convention in §7.1.)

Example (4 separate uniform loads on individual element faces):
```
.gsl
1 UNI_LOAD 0 1 0 1 0
No name
-1.000000000000e+00 -1.000000000000e+00 0.000000000000e+00
1 1
2 UNI_LOAD 0 1 0 1 0
No name
-1.000000000000e+00 -1.000000000000e+00 0.000000000000e+00
1 4
```
(each `UNI_LOAD` here has exactly 1 face; `1 1` = element 1, local face 1, etc.)

**`GRAD_LOAD`** — a gradient (linearly-varying) load, referencing a reference node and direction; no populated example is available, so its record fields beyond "reference node, direction, data" are unconfirmed.

### 9.4 Loads on elements (`.ple`)

Count-terminated by the sum of the three "...loads on elements" header counts (point + line + surface). Each record opens with a type tag:

**`POINT_LOAD`**:
```
<idx> POINT_LOAD <LF> <EF> <targetType> <?>
<name>
POSITION: <x> <y> <z>
FORCE: <Fx> <Fy> <Fz>
MOMENT: <Mx> <My> <Mz>
COORD_SYSTEM: <code>
VECTOR: <vx> <vy> <vz>
<nTargets>
<eleId> [<faceId>]              <- nTargets lines; a bare element id, or "eleId faceId" if on a face
MOVING_LOAD: <0|1>
Move_Path: <id>
Load_Function: <LF2>
Velocity: <value>
Acceleration: <value>
```
`targetType` (shared field, same position in `POINT_LOAD`/`LINE_LOAD`/`SURF_LOAD`):
- `1`: automatically to any structural element
- `2`: automatically to Shell elements
- `3`: automatically to Beam elements
- `4`: automatically to Truss elements
- `5`: automatically to Membrane elements
- `8`: to previously selected faces (3D) / edges (2D). In this case, the selected faces/edges are saved in `<eleId>`/`<faceId>` — for a `B8` volumic target, `<faceId>` follows the face-numbering convention in §7.1. `<nTargets>` then lists either bare node/element ids or `eleId faceId` pairs accordingly, and can also use the [compact element-list notation](#26-compact-element-list-notation), e.g. `569A611P...` style ranges (seen in the `LINE_LOAD` element list below).

`<LF2>`'s exact meaning is unconfirmed.

**`LINE_LOAD`**:
```
<idx> LINE_LOAD <LF> <EF> <targetType> <?>
<name>
POINT: <x1> <y1> <z1>
FORCE: <Fx1> <Fy1> <Fz1>
MOMENT: <Mx1> <My1> <Mz1>
POINT: <x2> <y2> <z2>
FORCE: <Fx2> <Fy2> <Fz2>
MOMENT: <Mx2> <My2> <Mz2>
COORD_SYSTEM: <code>
VECTOR: <vx> <vy> <vz>
<nTargets>
<eleId or compact-range>        <- e.g. "808A866P2" (see §2.6)
MOVING_LOAD: <0|1>
Move_Path: <id>
Load_Function: <LF>
Velocity: <value>
Acceleration: <value>
```

**`SURF_LOAD`**: a gradient (linearly-varying) surface load applied over a polygon, similar to `.gsl`'s `GRAD_LOAD`. When `targetType=8`, `<faceId>` follows the same B8 face-numbering convention documented in §7.1.

```
<idx> SURF_LOAD <LF> <EF> <targetType> <c>
<name>
P0: <value>
LOAD_DIRECTION: <code>
GRADIENTS: <gx> <gy> <gz>
REF_POINT: <x> <y> <z>
NO_POINT: <n>
POINT: <x> <y> <z>            <- n lines, defines the loaded polygon
COORD_SYSTEM: <code>
VECTOR: <vx> <vy> <vz>
<nTargets>
<eleId> <faceId>              <- nTargets lines; entire block omitted if nTargets = 0
MOVING_LOAD: <0|1>
Move_Path: <id>
Load_Function: <LF>
Velocity: <value>
Acceleration: <value>
```

### 9.5 Movement paths (`.pth`)

Referenced by `Move_Path:` in `.ple` records for moving-load analyses (e.g. vehicle loads crossing a bridge). Typically empty (`Move_Path: 0` everywhere, meaning "no path" / not a moving load) — record syntax unconfirmed.

---

## 10. Masses

Separate from loads (§9) — these define additional lumped/distributed mass, primarily for dynamic (seismic/vibration) analyses. If a hand-edit needs to add mass data, generate a minimal example from the GUI and inspect it directly rather than authoring from this reference alone.

### 10.1 Nodal masses (`.inm`)

Count-terminated (header "number of nodal masses"):

```
<idx> <nodeId> ? ? <massValue>
<line 2 — purpose unclear>
<name>
```

### 10.2 Element masses (`.iem`)

Blank-line/self-count-terminated with its own leading count line (not solely driven by the global header count):

```
.iem
<count>
<idx> ? <massValue> <LF> <EF> ? <nElements>
<filterFlag1> <filterFlag2> <filterFlag3>
<name>
<eleId1>
...                        <- nElements lines, one element id each
...                        <- repeated <count> times
```

The header also carries a second, related pair of counts — "number of Line masses on elements" and "number of Surface masses on elements" — which correspond to the `.pme` marker (loads-on-elements-style masses, analogous to how `.ple` extended `.ibf`/`.gsl` for loads). No populated example of either `.iem` or `.pme` is available; `.pme`'s record shape is unconfirmed, likely mirroring `.ple`'s `LINE_LOAD`/`SURFACE_LOAD` shape but for mass instead of force.

---

## 11. Initial Conditions

Initial-state data applied to elements/nodes before the first analysis step (e.g. geostatic stress, phreatic surface).

### 11.1 Initial stress conditions (`.izg`)

Count-terminated (header "number of ini. stress cond."). Assigns an explicit initial stress tensor to a list of elements:

```
<idx> <nElements> <eleId1> <eleId2> ... <eleIdN> <flag> <flag>
<name>
<sxx> <syy> <szz> <sxy> <syz> <sxz>       <- one line per element in the list, in order
...
```

Example (initial vertical stress of -10 applied to elements 1–4):
```
.izg
1 4 1 2 3 4 0 0
No name
 0.000000000000e+000 -1.000000000000e+001 0.000000000000e+000 0.000000000000e+000 0.000000000000e+000 0.000000000000e+000
 0.000000000000e+000 -1.000000000000e+001 0.000000000000e+000 0.000000000000e+000 0.000000000000e+000 0.000000000000e+000
```
(6-component stress tensor per line: `sxx syy szz sxy syz sxz` — here `syy = -10` for every listed element, all other components zero.)

### 11.2 Initial geometric conditions (`.iig`)

Count-terminated (header "number of init. cond. geom."). Assigns a single scalar value (e.g. an initial pressure-head / phreatic-surface elevation) per element:

```
<idx> <nElements> <eleId1> <eleId2> ... <eleIdN> <flag> <flag>
<value1>
<value2>
...                                        <- one line per element in the list, in order
<name>
```

Example (a value assigned to elements 3, 1, 2, 4):
```
.iig
1 4 3 1 2 4 0 0
-4.000000000000e+000
-3.400000000000e+001
-3.400000000000e+001
-4.000000000000e+000
No name
```
(values `-4, -34, -34, -4` for elements `3, 1, 2, 4` respectively — note the `<name>` line comes *after* the value list here, unlike `.izg` where the name precedes the data.)

### 11.3 Heat / humidity / strain initial conditions

The remaining initial-condition markers had no non-zero example available, so only their header-count purpose is documented, not populated syntax:

| Marker | Header count label |
|---|---|
| `.iag` / `.iav` | number of Heat IC / number of Heat IC values |
| `.img` / `.imv` | number of Humidity IC / number of Humidity IC values |
| `.ieg` | number of ini. strains cond. (Imposed strains) |
| `.ist` | number of constant eps0 |

Given the `.izg`/`.iig` pattern above (id + element list + flags, then per-element data, with name position varying), these likely follow a similar "element list header, then per-element data block" shape — but this is an extrapolation, not confirmed.

### 11.4 Initial velocities/accelerations (`.ivd`, `.svd`)

Header count "number of initial velocities/accelerations" is typically `0`. Record syntax unconfirmed — relevant only for dynamic/seismic analyses with non-zero initial velocity fields.

---

## 12. Reinforcement

### 12.1 Reinforcement sets and members (`.brc`)

Typically used for reinforced-concrete beam/shell rebar layouts:

```
.brc
<nReinfSets> <nReinfMembers>
<nLayers> <nMaterials> <?>                      <- per reinforcement set
<setName>
<materialId1> <materialId2> ...                  <- nMaterials material ids used by this set
<?> <?> <lengthType> <distL> <distR> <yposType> <yDist> <zOffsetL> <zOffsetR> <?> <?> <diam> <nBars> <?> <prestress> <enabledFlag> <material>
...                                                <- nLayers layer lines, repeated per set
<?> <nBeams> <?> <?>                              <- per reinforcement member
<reinfSetName>                                    <- name of the .brc set this member uses (matched by name, not id)
<beamId1> <beamId2> ... <beamIdN>                 <- nBeams beam element ids carrying this reinforcement
...                                                <- repeated per member
```

Per layer: `length_type` (how the layer's extent along the beam is defined), `dist_l`/`dist_r` (left/right distances), `ypos_type`/`ydist` (position across the section), `zoffset_l`/`zoffset_r`, `diam` (bar diameter), `nBars` (bar count), `prestress`, `enabled` (layer active flag, off when the corresponding field is `0`), and `material` (material id for the rebar itself, separate from the host beam's material). A reinforcement member links a named reinforcement set to a specific list of beam elements — i.e. the same rebar layout can be reused across many beams by reference name.

---

## 13. Mesh Tying & Domain Reduction

### 13.1 Face groups / mesh tying (`.fac`, `.mrt`)

```
.fac
<nFaceGroups>
<nFaces>                    <- per face group
<name>
<eleId1> <faceId1>
...                          <- nFaces lines
...                          <- repeated per group
```

`.mrt` sits immediately after `.fac` in the marker sequence and is typically empty; likely a related mesh-tying setting, purpose unconfirmed.

### 13.2 Domain Reduction Method (`.drz`, `.dre`)

A simple element-id list for each of the DRM domain's interior and exterior element sets (used for seismic wave-propagation analyses with a reduced/absorbing domain boundary):

```
.drz
<nElements>
<eleId1> <eleId2> ...        <- nElements ids (interior elements)

.dre
<nElements>
<eleId1> <eleId2> ...        <- nElements ids (exterior elements)
```

### 13.3 Other structural-mesh markers (unconfirmed)

The following markers appear in the sequence but are typically empty:

| Marker | Notes |
|---|---|
| `.eie` | No matching header label found; purpose unclear. |
| `.crc` | Purpose unclear. |
| `.idv` | Purpose unclear. |
| `.spg`, `.svg`, `.scg` | Cluster of three markers near `.sdm`/`.psd` — likely further subdomain/staging metadata given their position in the sequence (§6.4). |
| `.gos` | Geometrical surfaces (element connectivity + boundary line list) — a write-only marker, confirming it exists and has a defined purpose in the format even though no populated syntax is available. |
| `.cld` | Typically empty; purpose unclear. |
| `.igl` | See §6.5 (auxiliary points). |

If a hand-edit needs to touch any of these, the most reliable path is still to reproduce the change in the ZSoil GUI on a minimal file and diff the result, rather than authoring from this reference.

---

## 14. Quick Reference

### 14.1 All 91 dot-prefixed markers, in file order

The same 91 markers, in the same order, appear in every v26 file (only their contents vary). "Doc §" is where each marker is documented; a dash means the marker is typically empty and could not be documented beyond its position in the sequence — treat these as confirmed-to-exist-but-unpopulated, not as unused/removable.

| # | Marker | Purpose | Doc § |
|---|---|---|---|
| 1 | `.icf` | Contact evolution function references | §15 (mention only) |
| 2 | `.ing` | Nodes | §6.1 |
| 3 | `.inl` | Nodal loads | §9.1 |
| 4 | `.inb` | Nodal boundary conditions | §8.1 |
| 5 | `.inc` | Named constraint/support groups | §8.2 |
| 6 | `.i0g` | Volumic/continuum elements (B8, Q4, T3) | §7.1 |
| 7 | `.ibg` | Beam elements (BEL2) | §7.2 |
| 8 | `.ijg` | Joints | §7.10 |
| 9 | `.itg` | Truss elements (TRS2, LNK2) | §7.3 |
| 10 | `.iff` | Fixed Anchor Zones | §7.4 |
| 11 | `.ilg` | Shell elements (SXQ4, SHQ4) | §7.5 |
| 12 | `.ilt` | Shell thickness table | §7.5 |
| 13 | `.ipg` | Seepage elements (SPL2) | §7.6 |
| 14 | `.ivg` | Convection elements | §7.7 |
| 15 | `.icg` | Contacts on continuum (volumic-volumic) | §7.8 |
| 16 | `.ikg` | Kinematic constraints | §7.10 |
| 17 | `.iag` | Heat initial conditions | §11.3 |
| 18 | `.iig` | Initial geometric conditions | §11.2 |
| 19 | `.img` | Humidity initial conditions | §11.3 |
| 20 | `.izg` | Initial stress conditions | §11.1 |
| 21 | `.ieg` | Initial strain conditions | §11.3 |
| 22 | `.iph` | Pressure head loads (BC) | §8.4 |
| 23 | `.iwh` | Surface load via pressure head | §8.4 |
| 24 | `.isg` | Load-related (surface load, unconfirmed shape) | §8.4 note |
| 25 | `.isl` | Surface loads (content not shown) | §8.4 note, §9 |
| 26 | `.iwf` | Water flux (BC) | §8.4 |
| 27 | `.iab` | Heat boundary conditions | §8.4 |
| 28 | `.imb` | Humidity boundary conditions | §8.4 |
| 29 | `.iwb` | Water boundary conditions | §8.4 |
| 30 | `.ibf` | Beam loads (legacy — superseded by `.ple`) | §9.2 |
| 31 | `.glk` | Local bases | §6.5 |
| 32 | `.ilb` | Local basis definitions (CARTESN_3D/VECTOR_3D/CYLINDR_3D), referenced by `.inb`'s trailing flag | §8.1 |
| 33 | `.ihf` | Heat flux (BC) | §8.4 |
| 34 | `.iuf` | Humidity flux (BC) | §8.4 |
| 35 | `.ghf` | *(likely macro/geometry-level heat flux)* | — |
| 36 | `.guf` | *(likely macro/geometry-level humidity flux)* | — |
| 37 | `.gab` | *(likely macro/geometry-level heat BC)* | — |
| 38 | `.gmb` | *(likely macro/geometry-level humidity BC)* | — |
| 39 | `.gsl` | Surface loads (UNI_LOAD, GRAD_LOAD) | §9.3 |
| 40 | `.gwb` | *(likely macro/geometry-level water BC)* | — |
| 41 | `.gwf` | *(likely macro/geometry-level water flux)* | — |
| 42 | `.idg` | *(purpose unclear)* | — |
| 43 | `.isd` | Data super elements | §6.5 |
| 44 | `.ily` | Shell layers | §7.5 |
| 45 | `.inm` | Nodal masses | §10.1 |
| 46 | `.iem` | Element masses | §10.2 |
| 47 | `.cld` | *(purpose unclear)* | §13.3 |
| 48 | `.igl` | Auxiliary points | §6.5 |
| 49 | `.axs` | Axes | §6.5 |
| 50 | `.pob` | Geometry points | §6.2 |
| 51 | `.gob` | Geometrical objects (GEOMLINE/ARC/PLINE) | §6.3 |
| 52 | `.sdm` | Subdomains | §6.4 |
| 53 | `.psd` | Subdomain-related | §6.4 |
| 54 | `.fac` | Face groups / mesh tying | §13.1 |
| 55 | `.mrt` | Mesh-tying related | §13.1 |
| 56 | `.idv` | *(purpose unclear)* | §13.3 |
| 57 | `.crc` | *(purpose unclear)* | §13.3 |
| 58 | `.spg` | *(subdomain/staging-related)* | §13.3 |
| 59 | `.svg` | *(subdomain/staging-related)* | §13.3 |
| 60 | `.scg` | *(subdomain/staging-related)* | §13.3 |
| 61 | `.gos` | Geometrical surfaces (write-only) | §13.3 |
| 62 | `.ish` | Shell hinges | §7.5 |
| 63 | `.ist` | Constant eps0 (initial strain) | §11.3 |
| 64 | `.goa` | Subdomain-related | §6.4 |
| 65 | `.sg0` | Subdomain-related | §6.4 |
| 66 | `.gnl` | Nodal links | §7.10 |
| 67 | `.pil` | Piles | §7.11 |
| 68 | `.eie` | *(purpose unclear)* | §13.3 |
| 69 | `.gbh` | Boreholes | §7.12 |
| 70 | `.ivd` | Initial velocities | §11.4 |
| 71 | `.svd` | Initial accelerations | §11.4 |
| 72 | `.pbc` | Periodic boundary conditions | §8.3 |
| 73 | `.drz` | DRM domain, interior elements | §13.2 |
| 74 | `.dre` | DRM domain, exterior elements | §13.2 |
| 75 | `.nil` | Nails | §7.13 |
| 76 | `.anh` | Anchors | §7.14 |
| 77 | `.grp` | Groups | §7.16 |
| 78 | `.apl` | Auxiliary planes | §6.5 |
| 79 | `.gsh` | General shell (per-shell flag list) | §7.17 |
| 80 | `.ics` | Contacts on structures | §7.9 |
| 81 | `.scs` | Structural-contact related | §7.9 |
| 82 | `.ims` | Structural-contact related | §7.9 |
| 83 | `.brc` | Reinforcement sets and members | §12.1 |
| 84 | `.cbl` | Cable/tendon element instances | §7.15 |
| 85 | `.bcb` | Cable geometry catalog | §7.15 |
| 86 | `.bcl` | Cable loss-coefficient catalog | §7.15 |
| 87 | `.hex` | Heat exchangers | §7.16 |
| 88 | `.ple` | Loads on elements (POINT_LOAD/LINE_LOAD/SURF_LOAD) | §9.4 |
| 89 | `.pme` | Masses on elements | §10.2 |
| 90 | `.pth` | Movement paths (moving loads) | §9.5 |
| 91 | `.hef` | Heat exchanger heat fluxes | §7.16 |

### 14.2 Element type tags

| Tag | Meaning | Marker |
|---|---|---|
| `B8` | 8-node hexahedral continuum | `.i0g` |
| `Q4` | 4-node quadrilateral continuum (2D) | `.i0g` |
| `T3` | 3-node triangular continuum (2D) | `.i0g` |
| `W6` | 6-node wedge/prism continuum (3D) | `.i0g` |
| `BEL2` | 2-node beam | `.ibg` |
| `TRS2` | 2-node truss | `.itg` |
| `LNK2` | 2-node link/anchor | `.itg` |
| `SXQ4` | 4-node thin/one-layer shell | `.ilg` |
| `SHQ4` | 8-node thick shell | `.ilg` |
| `C_Q4` | Contact on quad face | `.icg`, `.ics` |
| `C_L2` | Contact/interface on line | `.icg`, `.ics` |
| `SPL2` | 2-node seepage | `.ipg` |
| `PILE` | Pile axis element | `.pil` |
| `NAIL` | Nail axis element | `.nil` |
| `ANCH_HEAD` | Anchor axis element | `.anh` |
| `GEOMLINE` / `GEOMARC` / `GEOMPLINE` | Geometric line / arc / polyline | `.gob` |
| `UNI_LOAD` / `GRAD_LOAD` | Uniform / gradient surface load | `.gsl` |
| `POINT_LOAD` / `LINE_LOAD` / `SURF_LOAD` | Loads on elements | `.ple` |

### 14.3 Material sub-tags

`GEOM->`, `DENS->`, `FLOW->`, `CREEP->`, `NONL->`, `ELAS->`, `HEAT->`, `HUMID->`, `INIS->`, `STAB->`, `DISC->`, `DAMP->`, `MAIN->` — see §4.3 for full field-level detail per tag, and §4.2 for which formulations use which tags.

---

## 15. Worked Example

This walks a complete, minimal 2D file end to end — a single `Q4` continuum element on 4 nodes, 1 material, 4 boundary conditions, and every other block empty. It's small enough to read in full, and every non-empty line in it is explained below. Line numbers refer to this file.

| Lines | Content | Explanation |
|---|---|---|
| 1 | `@GeoDev_SARL_version: 26 9.05 26.03` | Version line (§3.1). |
| 2–136 | `2 # dimension 2 or 3` / `1 # number of materials` / `0 # number of Existence functions` / `0 # number of nodal forces(*.inl)` / `4 # number of nodes (*.ing)` / `1 # number of continuum 2D elements` / `4 # number of boundary conditions (*.inb)` / ... (rest `0`) | The fixed 135-line header count block (§3.2) — this file is 2D, 1 material, 0 EF/LF, 4 nodes, 1 `Q4` element, 4 BCs, everything else zero. |
| 137–141 | `ASSOCIATED_PROJECTS:` / `HEAT: -` / `HUMIDITY: -` / `FREE_FILED_MOTION: -` / `FLOW: -` | No linked companion projects (§3.3). |
| 142–145 | `STANDARD` / `kN  m  deg  year  C` (×2) / *(blank)* | Unit system: kN, m, degrees, years, °C. |
| 146 | `4 0` | Un-named flag pair preceding `CONTROL` — meaning unclear. |
| 147–150 | `CONTROL 1` / `Default` / 18-value line / 10-value line | One named solver-control set, "Default" (§3.4). |
| 151–160 | `DYN_CONTROL 1` ... `PSH_CONTROL 1` ... | Dynamic/pushover control, one "Default" set each. |
| 161–166 | `NONL_GEOM 0` / `CONSTRUCTION 0  0` / `PROJECT_PRESELECTION` / `5 0 0 0 0 1` | Simple analysis flags (§3.4) — no large-deformation, no construction staging. |
| 167 | `NUM_MATERIALS= 1` | Begin materials block (§4). |
| 168–171 | `MATERIAL  26.00 1` / `Elastic` / `No name` / `BUTTONS= 1 1 0 0 1 0 0 0 0 0 0 0 0 0` | One material: id 1, type "Elastic", unnamed. |
| 173–196 | ` ELAS-> 100000  0.3` ... ` DAMP-> 0  0  0  0  0  0  0` | Full sub-tag sequence for this material (E=100000, ν=0.3, thickness 0.1, unit weight 20 — see §4.3 for the full field-by-field breakdown). |
| 197 | `NUM_MATERIALS= 0` | No fiber materials follow. |
| 198 | `LAYERED_BEAM_COMPONENTS 0` | None defined. |
| 199–203 | `SIG_EPS_FUN 1` / `1 2 Default` / ... | One (unused, since no nonlinear material references it) stress-strain function. |
| 204–207 | `EXIST_FUNC 1` / `1 No name` / `1` / `0 3.37e+38` | One EF defined but never referenced by any element in this file (§5.1). |
| 208–212 | `LOAD_FUN 1` / `1 1 No name` / `No flags` / `0 1 0` / `0 0` | One trivial LF (§5.2) — unused, since this file has no active loads. |
| 213–289 | `DRIVERS 0` ... `PARALLEL_SETTINGS` / `1  20  0.0625` / `ARC_LENGTH` / `0  1` / `0` / `0` | Output/solver/project-metadata settings (§3.5) — all defaults, empty project metadata. |
| 291–306 | `AUGMENTED_LAGRANGIAN` ... `CONTACT_PARAMETERS 1` / `1 Base table` / ... `EVOLUTION_FUN 0` | Default contact-solver settings; one unused default contact material table. |
| 308 | `.icf` | Empty marker (contact evolution function list — not otherwise documented). |
| 310–314 | `.ing` / `1 0.0 0.0 0.0 0` / `2 1.0 0.0 0.0 0` / `3 1.0 1.0 0.0 0` / `4 0.0 1.0 0.0 0` | 4 nodes forming a unit square (§6.1). |
| 316–317 | `.inl` / *(blank)* | No nodal loads. |
| 318–322 | `.inb` / 4 lines | 4 boundary conditions — nodes 1–2 fixed in x&y, nodes 3–4 fixed in y only (16-token / older layout, §8.1). |
| 324–408 | `.inc` / 4 named blocks (`No name` ×4) | Per-node constraint-group data mirroring the 4 `.inb` entries (§8.2). |
| 410–411 | `.i0g` / `1 1 Q4 1 2 3 4 1 1 1 0 0 0 0 0` | The single `Q4` continuum element, connecting nodes 1-2-3-4, material 1 (§7). |
| 413–607 | `.ibg` through `.hex` — **81 markers, every one empty** | Every element/BC/load/geometry marker not otherwise used by this minimal model — each present as a bare marker line followed by a blank line (or `0`), per the "every marker always present" rule of §2.2. See [§14 Quick Reference](#14-quick-reference) for what each of these markers is for. |
| 609–616 | `.ple` / *(blank)* / `.pme` / *(blank)* / `.pth` / *(blank)* / `.hef` / *(blank)* | Final markers in the fixed sequence, all empty — end of file. |

**Takeaway**: even a maximally minimal model touches all 91 dot-markers — the file format doesn't omit unused sections, it just leaves them empty. When hand-editing an existing file, the safest strategy is exactly what this table demonstrates: locate the marker for the section you need by name, and edit only its block, leaving the surrounding empty markers untouched.
