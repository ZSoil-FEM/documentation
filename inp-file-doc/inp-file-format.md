# ZSoil `.inp` File Format Reference

This document describes the structure and syntax of the ZSoil `.inp` input file, as used by the current (v26) self-describing file format. It is written for engineers who need to read or hand-edit `.inp` files, and is based on direct inspection of real `.inp` files (not on any external specification document).

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

Field-level documentation quality varies by section, reflecting what could actually be confirmed from the available sources (real sample files, cross-checked in places against `zsoil_inp.py`). Where a field's exact meaning could not be determined this way — mostly opaque numeric solver-tuning parameters — it is marked explicitly rather than guessed, so treat unmarked fields as confirmed and marked ones as needing verification against the ZSoil GUI/help if precision matters.

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

The file is plain text. Records are whitespace-delimited (fields separated by spaces/tabs); most sections use one record per line, though multi-line records occur (e.g. materials, where several lines together form one logical record). Numbers are written as plain decimals or scientific notation (`1.000000000000e+000`); there is no fixed column width — parsers split on whitespace.

### 2.2 Section markers

Every data block in the body of the file is introduced by a marker line. Two marker styles are used:

- **Dot-prefixed markers**, e.g. `.ing`, `.i0g`, `.inb` — historically named after the companion file extension that used to hold that data. A full scan of a representative file found **91 distinct dot-prefixed markers**, always appearing in the same fixed order (see [§14 Quick Reference](#14-quick-reference) for the complete list). Every marker is present in every file, even when its block is empty — an empty block is simply the marker line followed immediately by a blank line (or, for count-terminated blocks, a `0` count).
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

### 2.6 Compact element-list notation

A few list fields (e.g. selecting a subset of elements) use a compact range notation instead of listing every id:

```
<start>A<end>            → every integer from start to end, inclusive, step 1
<start>A<end>P<stride>   → every integer from start to end, inclusive, step <stride>
<id>                      → a single element id, listed literally
```

Multiple tokens (space-separated) combine into one list, e.g. `12A45P2 90` expands to `12, 14, 16, ..., 44, 90` (up to and including 44, since 45 is not reached by the stride of 2 from 12) plus `90`.

---

## 3. File Header

### 3.1 Version line

The very first line of the file identifies the writing application/version:

```
@GeoDev_SARL_version: 26 9.05 26.03
```

Format: `@<vendor>_version: <majorVersion> <createdWithVersion> <lastOpenedWithVersion>`, confirmed: `<majorVersion>` (`26`) is the leading format-generation number identifying the current self-describing format documented here; `<createdWithVersion>` (`9.05` in this example) is the ZSoil version the file was **originally created with**; `<lastOpenedWithVersion>` (`26.03`) is the version of **ZSoil PrePro that last opened/saved** the file. This explains the sub-version discrepancies noted throughout this document (e.g. the `.inb` record-width difference in §8.1) — a file's on-disk record layout can reflect an older `createdWithVersion` even though the leading `26` and `lastOpenedWithVersion` look current.

### 3.2 Header count block

Lines 2–136 are a fixed, always-136-line sequence of `<count> # <description>` lines in fixed order. Each count tells the reader how many records to expect later in the corresponding dot-marker section. 

Example:

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

*(Field indices above are relative to this table, not raw file line numbers; the block spans file lines 2–136 in every sample inspected — a small number of labels are truncated/duplicated oddly in the source format itself, see entries 33/49/88 above.)*

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

Each `HEAT:`/`HUMIDITY:`/`FREE_FILED_MOTION:`/`FLOW:` line names an associated companion project file, or `-` if none is linked.
`UnitSystemName` is a free-text label (e.g. `STANDARD`, `EXAMPLE UNITS`). The two unit lines are **not a duplicate** — confirmed: the first line is the unit system used **in the preprocessor** (model input/editing), the second is the unit system used **in the postprocessor** (results display); they're usually identical but can in principle differ if a user changes display units after building the model. Observed unit values include `kN`, `m`, `deg`/`rad`, `year`/`day`/`h`, `C`.

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

**`CONTROL <n>` — nonlinear solver settings.** Each set's first line holds 18 fields, confirmed field-by-field:

| # | Field | Example value (`boxd1.inp`, "Default" set) |
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

The second line holds the "sharpened" tolerances used for kinematic (displacement-controlled) loading, plus a solver-switching strategy flag 

| # | Field | Example value |
|---|---|---|
| 1 | Sharpening of tolerance for kinematic loading, flag | `1` |
| 2 | (Sharpened) tolerance for solid-phase RHS | `0.01` |
| 3 | (Sharpened) tolerance for solid-phase energy | `0.01` |
| 4 | Strategy for divergent steps: automatic solver switching flag | `1` |
| 5–10 | More parameters for dealing with lack of convergence | `1 5 5 0 0 0` |

The same "named-set" pattern (keyword, count, then that many named sets) applies to:

- **`DYN_CONTROL <n>`** — dynamic-analysis solver control (same general kind of tolerance/iteration settings, extended for time integration; field-by-field layout not further decoded here).
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

Later in the header (after materials and functions — see §4–5), a `DRIVERS <n>` block (same named-set pattern as §3.4) is followed by a long sequence of simple `KEYWORD` / `<value>` pairs — each keyword on its own line, its value(s) on the line(s) immediately after. These are largely self-explanatory from their names and are documented here as a flat reference rather than field-by-field prose:

| Keyword | Purpose (as inferred from name; value meaning not further decoded) |
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
 <sub-tag blocks, see 4.2>
NUM_MATERIALS= 0             <- terminates the block (0 = no more standard materials; a second
                                 NUM_MATERIALS= <n> immediately after, if present, defines FIBER
                                 materials for layered shells, using the same record shape — see
                                 §4.2 GEOM-> "Shell Layered" and §4.3 for the confirmed mechanism)
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

### 4.2 Sub-tag blocks

Each sub-tag line starts with the tag (some indented by a leading space, inconsistently — e.g. ` ELAS->` vs `CREEP->`) followed by one or more numeric fields; several tags (`NONL->`, `ELAS->`, `INIS->`) continue onto additional lines until the *next* expected tag is reached, rather than having a fixed line count. Field layout differs by material type — the tables below show what could be confirmed by comparing multiple materials; unconfirmed fields are marked accordingly.

**`ELAS->` — elasticity.** Continuum materials: `ELAS-> <E> <nu>` plus 3 more lines of secondary elastic parameters *(meaning unclear from available sources — likely anisotropy/Biot or damping-related extensions, all `0` in inspected samples)*. Beam materials add a third value: `ELAS-> <E> <nu> <1>`, followed by 2 more lines.

```
Continuum (Elastic soil): ELAS-> 25000  0.3
                           0  0
                           0  0
                           0  0

Beam (Elastic Beam nail):  ELAS-> 200000000  0.3  1
                            0  0
                            0  0
```

**`GEOM->` — geometry/cross-section.** Layout depends on material type:
- **Beam**: `GEOM-> <type>` where `type` selects how the cross-section is defined: `0` = "Profiles" (extra lines: blank, profile name, dimension values), `1` = "User": explicit dimensions + computed section values (2 lines: dimensions, then computed values), `2` = "Values": raw section values only (1 line). 
Example (`type=1`, rectangular): `GEOM-> 1` / `2  0.05  0.3  0.5  0.1  0.1  0.1` (dimensions) / `0.00196... 0.00168... ...` (area, Iyy, Izz, etc. — computed section properties) / `1  -1  0`.
  - **Layered/composite beam variant** — defines two otherwise-identical beam materials differing only in `BUTTONS[6]` (0-indexed) and the line that follows the dimensions/section-values pair: a plain `"Elastic beam"` (`BUTTONS[6]=0`) ends that pair with `1  -1  0` (as above, no further lines); a `"Layered beam"` (`BUTTONS[6]=1`, same material type string `Elastic Beam`, same `type=1` dimensions/values pair) instead ends it with `<1+n_reinf_fibers>  1  1`, followed by **one line per embedded reinforcement fiber**, blank-line-terminated:
    ```
     GEOM-> 1
    0  1  0.3  1  0.1  0.1  0.1
    1  0.833333  0.833333  0.140583  0.0833333  0.0833333  1
    3  1  1
    0  2  0.4  0.1  0.1  0  180  10  0.005  0  1  2
    0  2  -0.4  0.1  0.1  0  180  10  0.005  0  1  2
    ```
    The two fiber lines that follow are symmetric (`+0.4`/`-0.4`) rebar layers. Per fiber line `<?> <offsetType> <yOffset> <zOffsetL> <zOffsetR> <?> <?> <n_reinf_bars> <total_area> <prestress> <?> <componentId>`:
   - `offsetType`: `0` = "From top"; `1` = "From bot."; `2` = "From center"
   - `yOffset`: (3rd field, `±0.4`) is the fiber's *relative*, vertical position within the section. 
   - `zOffsetL` and `zOffsetR`: (4th and 5th fields, `0.1`) are the left and right-most fiber's *relative*, horizontal position within the section. 
   - `n_reinf_bars`: (8th field, `10`) is the number of bars in the fiber. This value has no influence on the physics, as only the total area matters. The value is used for the section figure only.
   - `total_area`: The total area of all `n_reinf_bars` in the fiber.
   - `prestress`: The prestress in the fiber.
   - `componentId`: references a `LAYERED_BEAM_COMPONENTS` entry by number (§4.3) — here component `2` = "steel"
- **Truss**: `GEOM-> <area>` — a single cross-sectional area value.
- **Continuum**: `GEOM-> <value>` — a single value, unclear meaning.
- **Shell Layered**: multi-fiber section definition:
  ```
   GEOM-> 1  10  2  0.005  0.4  2    0.005  -0.4  2    1  2  2
  ```
  Fields (0-indexed after the `GEOM->` token itself is `v[0]`): 
 - `v[1]`=`type`=1
 - `v[2]`=`10` *(meaning unclear from available sources)*
 - `v[3]`=`nFibers`=2
 - then per fiber `k:`
  - `<area> <distance> <distanceFrom>` at `v[4+3k]`/`v[5+3k]`/`v[6+3k]`: fiber 1: area=`0.005`, distance=`0.4` (above mid-plane), `distanceFrom`=`2`; fiber 2: area=`0.005`, distance=`-0.4` (below mid-plane), `distanceFrom`=`2`
 - then `v[4+3·nFibers]`=`core_material`=`1` (the shell's core/matrix material id — here material `1`, "concrete"), followed by one material-id field per fiber at `v[5+3·nFibers+k]` — both fibers reference material `2` ("steel"). This is the shell equivalent of the layered-beam pattern above: a concrete core (`core_material`) with steel reinforcement fibers symmetrically placed top and bottom. The fiber/core material ids reference a **second, separate `NUM_MATERIALS=` block** of type `"Fiber Shell"` that immediately follows this material's own block — see §4.3.

  **`distanceFrom` meaning**: The fiber position for a shell layer is a coordinate `ξ ∈ [-1, 1]` normalized to half-thickness (`y_physical = ξ · thickness/2`), and `distanceFrom` selects how the entered `distance` maps onto that normalized `ξ`:
  - `distanceFrom=0`: `distance` is a fraction of *full* thickness measured inward from the top surface, giving `ξ = 1 - 2·distance`.
  - `distanceFrom=1`: mirror of `distanceFrom=0`, measured from the bottom surface, giving `ξ = 2·distance - 1`.
  - `distanceFrom=2`: `ξ = distance` directly — i.e. `distance` *is* already the normalized mid-plane-relative coordinate.

**`DENS->` — density/consolidation/permeability-adjacent parameters.** Example (continuum): `DENS-> 20  10  0.6  1  1  16.7` followed by 3 more lines (largely `0`s in samples, plus a line like `-1  26.5  20  10  0`). First value looks like unit weight (`20`); remaining fields *(meaning unclear from available sources beyond the first value)*.

**`FLOW->` — hydraulic/permeability parameters** (large parameter set, ~28 values on the first line plus 3 more lines): e.g. `FLOW-> 1  1  1  0  1  0  0  0  1  0  2000000  0  20  1  0  1  1000000  100  1  1  100  1  1  0  1e+38  1  2  0`. *(Field-by-field meaning not confirmed from available sources — this is a large solver/seepage parameter block; cross-check against the ZSoil GUI's flow tab if precision matters.)* Beam/truss materials have an effectively empty `FLOW->` (no values, just the tag).

**`CREEP->`**: e.g. `CREEP-> 0  0  0  0  0  0  0  0  4  0  500  50  5  0  0  0  0` — identical across every inspected material regardless of type, suggesting this is a shared default block, populated only when creep is actually enabled (via `BUTTONS=` flags). *(Field meaning not confirmed.)*

**`NONL->` — nonlinear (plasticity) parameters.** Empty (`NONL->` alone) for plain "Elastic"/"Elastic Beam" materials, since neither uses a nonlinear model. **Populated example confirmed** from the `"Fiber Shell"` materials in `NL_shell_traction.inp` (see §4.3 — these are the fiber-reinforcement sub-materials referenced from a `Shell Layered` section's `GEOM->`):
```
 NONL-> FIRE_DATABASE 0 OFF
1000  500000  1  1  0.02  0.15  0.2
0  0  0  0
```
concrete fiber material: `1000`, `500000` (tension/compression strength-like values, first-field magnitude matching the `ft`-position value seen in the analogous `LAYERED_BEAM_COMPONENTS` concrete entry, §4.3); steel fiber material: `235000  235000` in the same position (symmetric tension/compression — matches structural steel's symmetric yield behavior, and the identical `235000` value seen in the beam-layer case). `FIRE_DATABASE <flag> <ON|OFF>` prefixes the line — a fire-resistance-analysis toggle, unconfirmed beyond its evident purpose from the name. Remaining fields (`1  1  0.02  0.15  0.2` and the trailing `0 0 0 0` line) *(meaning unclear from available sources — plausibly softening/regularization parameters, by analogy with `LAYERED_BEAM_COMPONENTS`' `reg_soft`/`char_len` fields)*.

**`HEAT->`**: e.g. `HEAT-> 1e-05  207.36  2000  0  105000  1.5  0.2083333333333333  4000  20  0  1` plus 2 more lines — thermal conductivity/capacity-type parameters. *(Field-by-field meaning not confirmed.)*

**`HUMID->`**: e.g. `HUMID-> 0  0.05  0.75  0.031104  0` — humidity/moisture-transport parameters. *(Field-by-field meaning not confirmed.)*

**`INIS->` — initial-state parameters**: e.g. `INIS-> 0.385  0.385  0  1  0  0  0  1  0  0  0  1` (continuum) vs `INIS-> 1  1  0  1  0  0  0  1  0  0  0  1` (beam) — first two values differ meaningfully between the two examples (`0.385 0.385` vs `1 1`), possibly initial void ratio or saturation for continuum vs. a not-applicable default for beams. *(Not fully confirmed.)*

**`STAB->`**: e.g. `STAB-> 2  0  0  0  1  1.2  0.2  1  1.5  0.5` (continuum, non-trivial) vs `STAB-> 0  0  0  0  0  0  0  0  0  0` (beam, all zero) — stabilization/regularization parameters, active for continuum materials only in inspected samples.

**`DISC->`**: observed empty (bare tag) in every sample — discontinuity/localization-band parameters, presumably populated only for softening models.

**`DAMP->`**: e.g. `DAMP-> 0  0  0  0  0  0  0` — Rayleigh/material damping parameters, all-zero in every inspected (non-dynamic-focused) sample.

**`MAIN->`** (beam/truss materials only): e.g. `MAIN-> 0` or `MAIN-> 1  0  0  0  0  0` — a small flag set, first value possibly a material-model sub-type selector. *(Not fully confirmed.)*

### 4.3 Other material-related blocks

- **`LAYERED_BEAM_COMPONENTS <n>`**: components for layered/composite beam sections (the fiber materials referenced by a `"Layered beam"` material's `GEOM->` block, §4.2). Record: number, name, then one line of parameters — type (`0`=elastic, `1`=elastic-plastic, `2`=user), E, nu, and further softening/creep parameters (`E0_setup`, regularization flag, characteristic length, coupled softening flag, creep type/A/B); if `type==2` (user model), two further lines name the tension/compression stress-strain functions (`ft_fun`, `fc_fun`).

  **Populated example confirmed** from `NL_beam_traction.inp` / `NL_beam_traction_forceCtrl.inp` (identical in both — a reinforced-concrete beam modeled with a concrete matrix and steel reinforcement fibers):
  ```
  LAYERED_BEAM_COMPONENTS 2
    1
  concrete
  2e+07  0.3  25  1  1000  10000  0  0.005  0  0  -1  0  1  
    2
  steel
  2e+08  0.3  25  1  235000  235000  0  0.005  0  0  -1  0  1  
  ```
  Field-by-field for the parameter line (`v[0]`-indexed): 
 - `E`=`v[0]` (`2e+07` concrete, `2e+08` steel)
 - `nu`=`v[1]` (`0.3` both)
 - `v[2]`=`25` *(meaning unclear — identical for both materials despite very different real-world unit weights, so probably not unit weight; possibly a shared default such as a reference temperature)*, `type`=`v[3]`=`1` (elastic-plastic, both)
 - `ft`=`v[4]`: tension strength
 - `fc`=`v[5]`: compression strength
 - `reg_soft`=`v[6]`=`0`
 - `char_len`=`v[7]`=`0.005` (both)
 - `E0_setup`=`v[8]`=`0`
 - `coupTC_soft`=`v[9]`=`0`
 - `creep_type`=`v[10]`=`-1` (both — creep disabled)
 - `creep_A`=`v[11]`=`0`
 - `creep_B`=`v[12]`=`1` (both).
- **`SIG_EPS_FUN <n>`**: user-defined stress-strain functions, referenced from `LAYERED_BEAM_COMPONENTS`/nonlinear materials by name. *(full field layout not confirmed)*.
- **Fiber-material sub-blocks for layered shells**: unlike layered beams (which use the dedicated `LAYERED_BEAM_COMPONENTS` keyword above), a `"Shell Layered"` material's fiber/core materials are defined as a **second `NUM_MATERIALS= <n>` block**, placed immediately after the layered shell material's own single-material block, using material type `"Fiber Shell"` — confirmed from `NL_shell_traction.inp`:
  ```
  NUM_MATERIALS= 1
  MATERIAL  26.00 1
  Shell Layered
  No name
  BUTTONS= 0 1 0 0 1 0 1 0 0 0 0 0 0 0
  ...                                    <- this material's own ELAS->/GEOM->/etc., see §4.2
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
  This matches `zsoil_inp.py`'s own two-pass materials logic (`if len(self.materials)==0: ... else: # fiber materials`), confirming that a second `NUM_MATERIALS=` block is specifically how fiber/layer sub-materials are encoded, distinct from the beam mechanism.

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

Confirmed field-by-field against `zsoil_inp.py` (`ExistFun` class): `number` (int), `name` (rest of the line after the number), `nInstances` (how many active time-intervals follow), then one line listing `nInstances` `[start, end]` pairs — the element/entity referencing this EF exists only during these time interval(s). EF number `0` is never defined in the file — it's implicit, meaning "permanent" (active for the interval `[0, 1e36]`, per the parser's built-in default).

Example (`boxd1.inp`):
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

Confirmed against `zsoil_inp.py` (`LoadFun` class, which also exposes `.at(t)` to interpolate a value at a given analysis time): `number`, `nSteps`, `name` on the first line; a `flags` line (free text/flag string, e.g. `No flags`); an `options` line of `[t0, scale, type]` (`t0` = time origin, `scale` = multiplier applied to all step values, `type` = interpolation/repeat-mode selector — exact codes for `type` not confirmed); then `nSteps` lines of `[time, value]` pairs defining the piecewise function. LF number `0` is implicit — a constant multiplier of `1` for the whole analysis, never defined explicitly in the file.

Example (`boxd1.inp`):
```
LOAD_FUN 1
1 1 No name
No flags
0 1 0
0 0
```
(here `nSteps=1`, giving one `[time,value]` pair: `time=0, value=0` — a trivial/placeholder function, since this file has no active nodal loads.)

---

## 6. Geometry

### 6.1 Nodes (`.ing`)

The finite-element mesh nodes:

```
<id> <x> <y> <z> <flag>
```

One line per node, count taken from the header (§3.2, "number of nodes"). Confirmed against `zsoil_inp.py`, which reads `id, x, y, z` into `self.coords[0/1/2]`. The trailing `<flag>` (observed always `0` in inspected samples) is not read by the parser — *(meaning unclear from available sources)*.

Example (`boxd1.inp`):
```
.ing
1 0.000000000000e+000 0.000000000000e+000 0.000000000000e+000 0
2 1.000000000000e+000 0.000000000000e+000 0.000000000000e+000 0
```

### 6.2 Geometry points (`.pob`)

Points used to build geometric (sketch-level) construction lines/arcs, distinct from mesh nodes — same record shape as `.ing` (`<id> <x> <y> <z> <flag>`), count from the header ("number of geometrical points"). Example (`3D_anchor_disp.inp`):

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

Example (`3D_anchor_disp.inp`, 5 lines forming a closed boundary from 6 points):
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

`GEOMARC` and `GEOMPLINE` variants (for arcs and polylines) were not found non-empty in the inspected sample set, but follow the same `<id> <TYPE> ... / <name> / <point references>` shape; their point-reference line likely differs (an arc needs 3 points or a center+radius, a polyline needs a variable-length point list) — *(exact field layout for these two variants not confirmed from available sources)*.

### 6.4 Subdomains (`.sdm`) and related meshing metadata

Subdomains (corresponding to header counts "number of continuum 2D/3D subdomains", "beam/truss/membrane/shell subdomains") capture the GUI's meshing region definition — including, in the inspected example, a full sub-mesh boundary and interior node listing. This is normally GUI-generated and not something to hand-edit; documented here at a structural level only:

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

Example header line (`3D_anchor_disp.inp`, one 2D subdomain): `1 128 SUBD_2D 4 4 0 4 1 4 0 1 0 0`. *(Full field-by-field meaning of the header/meshing-parameter lines is not confirmed from available sources — treat this block as opaque GUI-managed data unless specifically working on subdomain/staging definitions.)* Related markers `.psd`, `.goa`, `.sg0` appear immediately after `.sdm` in the marker sequence and were empty (`0`) in every sample inspected; likely auxiliary subdomain data (`.psd` possibly "point-subdomain" cross-references) — *(not confirmed)*.

### 6.5 Local bases, axes, and auxiliary geometry

The following markers were found empty (`0` / blank) in every sample inspected, so only their header-count purpose can be documented, not their populated syntax:

| Marker | Header count label | Notes |
|---|---|---|
| `.glk` | number of Local bases | Custom local coordinate systems, referenced by element records' `rm1`/`rm2` fields. |
| `.axs` | *(not directly named in header; likely tied to "Local bases" or a display/axis setting)* | *(meaning unclear from available sources)* |
| `.apl` | number of auxiliary planes | Construction planes for the GUI's sketch tools. |
| `.igl` | number of auxiliary points | Auxiliary sketch points, distinct from `.pob`. |
| `.isd` | number data super elements | Reusable geometry/mesh templates; `zsoil_inp.py` defines a `Thickness`-adjacent concept but does not parse `.isd` itself. |

If a hand-edit needs to touch any of these, populate them by building the corresponding geometry in the ZSoil GUI on a minimal test file and diffing the resulting `.inp`, rather than authoring them from this reference alone.

---

## 7. Elements

This chapter documents the element-definition blocks of the `.inp` file. Each block is introduced by a dot-prefixed marker and is either count-terminated (the count comes from the global header-count block, §3.2) or blank-line-terminated. All examples below are quoted verbatim from real files in the `zsoil_inp_files` sample set and, where available, cross-checked against the field indices used by the working parser `zsoil_inp.py`.

A recurring pattern across nearly every element record is:

```
<idx> <number> <TAG> <node1> ... <nodeN> <...type-specific fields...> <mat> <rm1> <rm2> <EF> <LF> [<extra>]
```

where `<idx>` is the 1-based sequential position of the record within the block (matches array order) and `<number>` is the user-visible element number (may or may not coincide with `<idx>`). Several trailing integer fields recur across element families whose exact purpose could not be pinned down from the parser or samples; these are called out per-section as *(meaning unclear from available sources)*.

### 7.1 `.i0g` — Volumic/continuum elements

Count-terminated: the record count equals `nVolumics3D + nVolumics2D` (header fields "number of continuum 3D elements" / "number of continuum 2D elements"). One line per element. Type tag in field 3 selects the node count and the position of the trailing field group.

Confirmed via the parser (`zsoil_inp.py`, `.i0g` branch): node indices follow the tag, then — at an offset `pos` that depends on the tag (`pos=5` for `Q4`, `pos=10` for `B8`) — the fields `mat` (`v[pos+4]`), `rm1` (`v[pos+5]`), `rm2` (`v[pos+6]`), `EF` (`v[pos+7]`), `LF` (`v[pos+8]`) appear, followed by one trailing field whose meaning is unclear.

**Q4 (4-node quad, 2D)** — `boxd1.inp`:
```
.i0g
1 1 Q4 1 2 3 4 1 1 1 0 0 0 0 0
```
Field breakdown (0-indexed after split): `v0`=1 (idx), `v1`=1 (element number), `v2`=`Q4`, `v3..v6`=nodes 1,2,3,4, `v7..v8`=1,1 *(meaning unclear)*, `v9`=1 (mat), `v10`=0 (rm1), `v11`=0 (rm2), `v12`=0 (EF), `v13`=0 (LF), `v14`=0 (trailing, unclear).

**B8 (8-node hex)** — `el_simple_load_3D_brick.inp`:
```
.i0g
1 1 B8 1 2 4 3 5 6 8 7 1 1 1 1 0 0 0 0 0
```
`v0`=1, `v1`=1, `v2`=`B8`, `v3..v10`=nodes (1,2,4,3,5,6,8,7), `v11..v13`=1,1,1 *(meaning unclear — one more flag field than `Q4`, plausibly a 3D-specific integration/formulation flag)*, `v14`=1 (mat), `v15`=0 (rm1), `v16`=0 (rm2), `v17`=0 (EF), `v18`=0 (LF), `v19`=0 (trailing, unclear). This layout is identical for the EAS and B-bar formulation variants (`el_simple_load_3D_brick_eas.inp`, `el_simple_load_3D_brick_bbar.inp`) — the formulation choice is not encoded in this record.

**B8 face numbering** — relevant wherever a `faceId` targets a `B8` element (`.gsl` `UNI_LOAD`, §9.3; `.ple` `POINT_LOAD`/`SURF_LOAD` with `targetType 8`, §9.4). Confirmed by cross-referencing `.i0g` connectivity against `.ing` node coordinates and the `.ple` `SURF_LOAD` face targets in `3D_with_loads.inp` (a cube meshed into 216 `B8` elements, with a surface load applied over the `x=0.5` boundary via 36 explicit `eleId faceId` pairs). Local nodes 1–4 form one quad face and 5–8 the opposite quad face, connected edge-to-edge (1–5, 2–6, 3–7, 4–8); the 6 face ids are assigned so that **opposite faces sum to 7**:

| Face id | Local nodes |
|---|---|
| 1 | 1-2-3-4 |
| 2 | 1-2-6-5 |
| 3 | 4-1-5-8 *(inferred, see note below)* |
| 4 | 2-3-7-6 |
| 5 | 3-4-8-7 *(inferred, see note below)* |
| 6 | 5-6-7-8 |

Faces 1, 2, 4, and 6 were each independently confirmed against a distinct element in the sample (elements 215, 214, 181/216, and 213 respectively): for each, the element's `x=0.5` face was identified from node coordinates, mapped to its local node quadruplet, and matched against the recorded `faceId`. Faces 3 and 5 were not directly observed in any sample — they are inferred by elimination and by the "opposite faces sum to 7" pattern the other four establish; treat them as unconfirmed if precision matters.

**T3 (3-node tri, 2D)** — `el_simple_load_2D_tri.inp`:
```
.i0g
1 1 T3 2 1 3 1 1 0 0 0 0 0
2 2 T3 1 4 3 1 1 0 0 0 0 0
```
`v0`=1, `v1`=1, `v2`=`T3`, `v3..v5`=nodes (2,1,3), then `v6..v12`=`1 1 0 0 0 0 0` (7 trailing fields vs. 8 for `Q4`/`B8` — one fewer). The parser's `T3` branch reads and discards these fields (its `mat`/`EF`/`LF`/`rm1`/`rm2` assignment lines are commented out as unimplemented), so the exact trailing-field-to-attribute mapping for `T3` is *(meaning unclear from available sources)* — by position it would plausibly be 2 unclear ints, then mat, rm1, rm2, EF, LF, but this is not confirmed by working code.

A `W6` (6-node wedge/prism) tag was also observed in `el_simple_load_3D_wedge.inp` with the same layout pattern as `B8` (nodes then `1 1 1` then mat/rm1/rm2/EF/LF/trailing):
```
1 1 W6 1 4 6 2 3 5 1 1 1 1 0 0 0 0 0
```

### 7.2 `.ibg` — Beam elements

Count-terminated: count = "number of Beams (*.ibg),(*ibm)". Each `BEL2` (2-node beam) element occupies 5 physical lines. Confirmed via parser: line 1 gives number/nodes; the next line is skipped by the parser (unread/unused); the following line is the orientation-vector record; then a material line (`mat`, `EF` at `v[3]`); then a single flag line — if it equals `1`, two more lines follow (not observed in any sample, so their content is undocumented here).

`loads_on_beams_2D_auto_select.inp`:
```
.ibg
411 1 BEL2 1 3
0  0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00 0.000000e+00
0  -0.000000e+00 1.000000e+00 -0.000000e+00 -0.000000e+00 1.000000e+00 -0.000000e+00
 11 0 0 2 0 0
0
```
Line 1: `<idx> <number> BEL2 <node1> <node2>` = `411 1 BEL2 1 3`.
Line 2: unused/skipped by parser — six floats, all zero in every sample seen *(meaning unclear from available sources)*.
Line 3: orientation vector, six floats (two triplets) defining the beam's local axis; the parser stores this whole line as `beam.dir`.
Line 4: ` 11 0 0 2 0 0` → `mat`=11 (`v0`), then `v1`,`v2`=0,0 *(unclear, plausibly rm1/rm2)*, `EF`=2 (`v3`), `v4`,`v5`=0,0 *(unclear, plausibly LF and a trailing flag)*.
Line 5: end-release/hinge flag, `0` in every sample. Per the parser, if this were `1`, two additional (undocumented) lines would follow.

### 7.3 `.itg` — Truss elements

Count-terminated: count = "number of truss elements (*.itg),(*.itm)". Each record is `TRS2` (truss, `type=0`) or `LNK2` (anchor/link, `type=1`) — only `TRS2` was observed in the sample set; no `LNK2` example exists in any inspected file. Confirmed via parser: element line, then a material line (`mat`, `rm1`, `rm2`, `EF`, `LF`), then a prestress-record count line, then that many prestress-record lines (variable-length).

**No prestress** — `2D_anchor_disp_s1m.inp`:
```
.itg
126 1 TRS2 158 157 0 0 0 0
 13 0 0 1 0 0
0
```
Line 1: `<idx> <number> TRS2 <node1> <node2> <4 trailing zeros>` — the four trailing fields after the node pair are *(meaning unclear from available sources)*, always `0 0 0 0` in the samples inspected.
Line 2 (material line): ` 13 0 0 1 0 0` → `mat`=13 (`v0`), `rm1`=0 (`v1`), `rm2`=0 (`v2`), `EF`=1 (`v3`), `LF`=0 (`v4`), trailing `v5`=0 *(unclear)*.
Line 3: prestress record count = `0` → no prestress lines follow.

**With prestress** — `2D_anchor_disp_s2_5m_P0100.inp` (and identically in `3D_anchor_disp_P0100.inp`, `truss_axisym_P0.inp`):
```
.itg
126 1 TRS2 158 157 0 0 0 0
 13 0 0 1 0 0
1
1.000000000000e+02 0 2  1
```
Prestress count = `1`, followed by one record: `1.000000000000e+02 0 2  1` → prestress force = `1.0e+02` (first field, confirmed by position and by the file name suggesting a 100-unit prestress case), followed by three integer fields (`0`, `2`, `1`) whose individual meanings are *(meaning unclear from available sources)* — plausibly a load-function reference, a prestress-type code, and a stage/sequence index; the parser stores the whole split line verbatim as `truss.prestress[i]` without further interpretation.

### 7.4 `.iff` — Fixed Anchor Zones

Header count field: `"# number of Fixed Anchor Zones"` (confirmed present, e.g. `2D_anchor_disp_s1m.inp`: `1 # number of Fixed Anchor Zones`). Despite a non-zero header count in every anchor sample file, the `.iff` block itself was found completely blank (zero records) in **every** inspected file, including all four anchor-displacement samples. (The "Fixed anchor zone interface" text that does appear in these files is a material-catalog entry name, unrelated to this marker.) No populated `.iff` record syntax could be confirmed from the available sample set.

### 7.5 Shell elements: `.ilg` + `.ilt` (thickness) + `.ily` (layers) + `.ish` (hinges)

**`.ilg`** — count-terminated. The count that governs this block is **not** the generic header field `"number of shell elements (*.ilg),(*.ilm)"` (which was `0` in every sample checked, including files with real shell elements) but the separate field `"number of Shell one layer elements"` (e.g. `UL_shell.inp`: `1`; `bridge_loads_on_shells_withVols.inp`: `966`). Hand-editors adding/removing `SXQ4`/`SHQ4` records must update this second count field, not the first.

Type tags: `SXQ4` (4-node thin/one-layer shell, `pos=6`) and `SHQ4` (8-node thick shell, `pos=10`) — confirmed via parser. Fields `mat`, `rm1`, `rm2`, `EF`, `LF`, `thick` (thickness-table index, resolved against `.ilt`) sit at `v[pos+4..pos+9]` respectively.

`UL_shell.inp`:
```
.ilg
2 1 SXQ4 1 5 6 2 1 1 1 2 2 2 3 1 1 0
```
`v0`=2 (idx), `v1`=1 (number), `v2`=`SXQ4`, `v3..v6`=nodes (1,5,6,2), `v7..v9`=1,1,1 *(meaning unclear, same pattern as B8's three extra ints)*, `v10`=2 (mat), `v11`=2 (rm1), `v12`=2 (rm2), `v13`=3 (EF), `v14`=1 (LF), `v15`=1 (thickness-table index → `.ilt` record 1), `v16`=0 (trailing, unclear). No `SHQ4` example was found in the sample set.

**`.ilt`** (thickness table) — blank-line-terminated list of thickness definitions referenced by the `thick` field above. Confirmed via parser: each entry starts with a `type` code; `type=0` is followed by one line giving a single float (uniform thickness); `type=1` ("not tested" per the parser's own comment) would be followed by two float lines but no such example exists in the samples.

`UL_shell.inp`:
```
.ilt
0  
0.5
```
Record 1: `type`=0, thickness = `0.5`.

The `type` field can also carry an optional trailing free-text label after it, confirmed from `NL_shell_traction.inp`:
```
.ilt
0  thickness=1m
1
```
Here `type`=0 with label `thickness=1m` (a human-readable echo of the value on the next line, `1` — i.e. this specific shell's thickness table entry is 1 m); the parser only reads the numeric `type` token and ignores the rest of that line, so the label is cosmetic/GUI-generated rather than functionally required.

**`.ily`** (layers) and **`.ish`** (hinges, header count "number of shell hinges") — both blank/empty in every inspected file; no populated example exists for either.

### 7.6 `.ipg` — Seepage elements

Count-terminated: count = `"number of water Seepage's (*.ipg) (*.ipm)"`, `0` in every inspected file — no `SPL2` element or other `.ipg` content exists anywhere in the sample set. Based on the parser's (unexercised) `.ipg` branch, the expected record layout would be:
```
<idx> <number> SPL2 <elem> <face> <node1> <node2> <mat> <EF> <LF>
```
each followed by one skipped line — but since no real example exists, this layout is *(inferred from parser code only, not confirmed against sample data)*.

### 7.7 `.ivg` — Convection elements

Header count field `"# number of convection element (*.ivg)"`, `0` in every inspected file. Block is blank in every sample; no populated example exists.

### 7.8 `.icg` — Contacts on continuum (volumic-volumic interfaces)

Count-terminated: count = `"number of contact lines (*.icg), (*.icm)"`. Both `C_L2` (2D line contact) and `C_Q4` (3D quad-face contact) tags were observed. Interface `type` codes, confirmed by position (first field of the trailing data line): `0` = continuity with pressure, `1` = contact, `2` = continuity without pressure.

**`C_L2` (2D)** — `interface2D_shear.inp`:
```
.icg
1 1 C_L2 2 1 1 3
 5 8 4 7 1 1 0
0.000000e+00 0
No name
1 2 0 0 0.000000e+00
```
Line 1: `<idx> <number> C_L2 <volele> <volface> <?> <?>` → `volele`=2, `volface`=1 (`v[3]`, `v[4]`, confirmed by parser); `v5`,`v6`=1,3 *(meaning unclear)*.
Line 2: 4-node connectivity ` 5 8 4 7` (the two coincident node pairs across the interface), followed by `1 1 0` — the second of these three trailing fields (here `1`) is the **count of trailing type/mat/EF/LF records** that follow (see the multi-record example below); the other two are *(meaning unclear)*.
Line 3: `0.000000e+00 0` — skipped by parser, *(meaning unclear, plausibly initial gap + a flag)*.
Line 4: interface name (`No name`).
Line 5 (repeated per the record count from line 2): `<type> <mat> <EF> <LF> <trailing float>` = `1 2 0 0 0.000000e+00` → type=1 (contact), mat=2, EF=0, LF=0, trailing=0.0 *(unclear)*.

**`C_Q4` (3D)** — `interface3D_shear.inp`: same field structure as `C_L2` but with an 8-node connectivity line (two 4-node faces):
```
.icg
1 1 C_Q4 1 5 2 2
 4 7 15 12 5 8 16 13 1 1 0
0.000000e+00 0
No name
1 2 0 0 0.000000e+00
```

**Multiple staged records on one contact** — `UL_vol_with_cnt2.inp` shows the same geometry re-used across load stages by bumping the trailing record count on the node line from `1` to `3`:
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
Here the geometry is fixed but `EF` cycles 1→2→3 across the three trailing records — i.e. this is how a contact's activation is staged across construction phases without redefining its geometry (compare the single-record version of the same model, `UL_vol_with_cnt1.inp`, whose node line ends `...1 1 0` with one trailing record).

### 7.9 Contacts on structures: `.ics` + `.scs` + `.ims`

**`.ics`** — count-terminated: count = `"number of inetrfaces (*.ics)"` [sic]. Two forms confirmed: `C_Q4` (contact paired with a volumic-element face or shell-element face) and `C_L2` (contact paired with a beam element).

**`C_Q4` paired with a shell element** — `UL_shell_with_cnt1.inp`:
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
Line 1: `<idx> <number> C_Q4 <cnt-number> <?> <?> <nsides> <side>` → `v3`=2 is the contact element number (confirmed: parser reads `self.cnt.number.append(int(v[3]))`); `v6`=1 → single-sided; `v7`=2 → negative side active (per parser: `v[6]=='1'` and `v[7]=='2'` ⇒ negative side only; `v[6]=='2'` ⇒ double-sided, both sides active).
Line 2: interface name. Line 3: `<paired-elem> <paired-face>` = `1 3` (shell element 1, face 3). Line 4: 8-node connectivity. Line 5–6: skipped by parser, *(meaning unclear)*. Line 7: `<?> <mat> <EF> <LF> <trailing float>` = `1 3 3 0 0.000000e+00` → mat=3, EF=3, LF=0.
If double-sided (`v[6]=='2'`), lines 3–7 repeat once more for the negative side (confirmed by parser branch, no sample with a double-sided `.ics C_Q4` record was found).

**`C_L2` paired with beam elements** — `loads_on_beams_2D_auto_select.inp`:
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
Same 7-line-per-record structure as the `C_Q4` case: header line (here `v3`=482 is the beam element number the contact rides on — confirmed by parser, `beamele = int(v[3])`), name, a `<?> <?>` line, 4-node connectivity, two skipped lines, and a final `<?> <mat> <EF> <LF> <trailing>` line. Note the parser's own `.ics`/`C_L2` handling is explicitly marked "not yet implemented" (it only skips the right number of lines without decoding them) — the field interpretation above is inferred by direct analogy with the `C_Q4` record shape, not confirmed by working code.

**`.scs`** — bare marker with no header count and no content in every one of the 71 inspected files. **`.ims`** — count-terminated, value `0` in every inspected file. No populated example exists for either.

### 7.10 `.ikg` — Kinematic constraints, `.gnl` — Nodal links, `.ijg` — Joints

All three are blank in every inspected file (`.ikg` header: `"# number of kinematic constrains (*.ikg)"`; `.gnl` header: `"# number of Nodal Links (*.gnl)"`; `.ijg` has no dedicated header comment identified). No populated examples exist for any of the three.

### 7.11 `.pil` — Piles

Count-terminated: count = `"number of Piles"`. Confirmed via parser field-by-field. Structure: header line, then 4 always-skipped lines (a flag, a second flag, and a 2-line orientation vector — same shape as the beam orientation record), then a material line (`mat`, `qsmat`, `qpmat`), then `nSeg+1` point lines tracing the pile axis.

`pile.inp`:
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
Header line: `<idx> <number> PILE <EF> <nSeg> <diam?> <?> ...` = `12 1 PILE 3 8 1.000000 0.250000 20 0 1 1 126 0 1 2` → `EF`=3 (`v3`), `nSeg`=8 (`v4`, confirmed: the pile axis is traced by `nSeg+1`=9 points below). `v5`=1.0, `v6`=0.25 are plausibly diameter/perimeter-related; remaining fields *(meaning unclear from available sources)*.
Material line: ` 11 12 0 13 0` → `mat`=11 (`v0`), `qsmat`=12 (`v1`, skin-friction material), `qpmat`=13 (`v3`, tip-resistance material).
Points: `nSeg+1`=9 lines of `<x> <y> <z> <flags>` tracing the pile axis; endpoints have one trailing flag, interior points have two — plausibly a segment-count/segment-index pair, but *(meaning unclear)*.

**Qs/Qp material selection variants** — comparing `pile_Qstanphi.inp` against `pile_constQsQp.inp` shows the tip-resistance-model toggle lives in the header line's `v9` field and the material line's `qpmat` field: `v9`=0 (Qstanphi, `qpmat`=0) vs `v9`=1 (const Qs/Qp, `qpmat` populated).

### 7.12 `.gbh` — Boreholes

Count-terminated: header field `"# number of boreholes"`, value `0` in **every one of the 71 inspected sample files** — no populated `.gbh` block exists anywhere in the sample set. Even with 0 boreholes, default interpolation-method records persist:
```
.gbh
0
KRIGING 1
2 0.200000 0.800000 200.000000 0.250000
SEDIMENTATION 0
```
The per-borehole record layout (`name` / `x y z nLayers` / `nLayers × [top bottom material]`) is inferred from parser code only and *(not confirmed against sample data)*.

### 7.13 `.nil` — Nails

Count-terminated: header field `"# number of Nails"`. Structure parallels `.pil`: header line, a `<count> <beam-element-number>` line referencing an already-defined `.ibg` `BEL2` element, two orientation-vector lines, a material line, then `nSeg+1` axis points.

`2D_nail_disp_s1m.inp`:
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
Header line: `EF`=3 (`v3`), `nSeg`=23 (`v4`; confirmed — 24 axis points follow). Line 2 (`1 126`): the number of the `.ibg BEL2` beam element this nail's structural bar is defined on (126, matches the `.ibg` example in §7.2 with `mat`=11). Material line ` 11 12 0 0 0`: `mat`=11 (matches the structural beam's material), `qsmat`=12; no separate tip material (consistent with nails having no tip resistance).

### 7.14 `.anh` — Anchors

Count-terminated, no dedicated header comment found. Simplified version of `.pil`/`.nil` — header line, a short material line, then `nSeg+1` axis points (no beam/orientation-vector lines).

`2D_anchor_disp_s1m.inp`:
```
.anh
127 1 ANCH_HEAD 126 3 20 1.000000 0.050000 4.000000 1 0 0.150000
 14 0
4.400000000000e+00 5.000000000000e-01 0.000000000000e+00 1 53
4.200000000000e+00 5.000000000000e-01 0.000000000000e+00 2 54 55
...
4.000000000000e-01 5.000000000000e-01 0.000000000000e+00 1 74
```
`v3`=126 references the `.itg` `TRS2` truss element this anchor's free length is built on (same element number as the §7.3 truss example); `v5`=20 is plausibly `nSeg` (21 axis points follow, matching). Material line ` 14 0`: single `mat`=14, no separate `qsmat`/`qpmat`.

### 7.15 Cables/tendons: `.cbl` + `.bcb` + `.bcl`

**`.cbl`** (cable/tendon element instances) — blank in every inspected file. **`.bcb`** (cable *geometry* catalog, referenced by name from `.cbl` records) — count-terminated, and notably present with the same **built-in default catalog of 2 entries in every single sample file** (even files with zero actual cable elements), suggesting these are ZSoil-authored default templates rather than user data:
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
Structure per entry: a name line, a flag line whose last field is the control-point count (`1 1 1 2` → 2 points; `1 1 2 3` → 3 points), then that many `<relative-position 0..1> <offset>` point lines describing the cable profile along the host beam/truss (a straight profile for entry 1; a parabolic sag profile for entry 2). A trailing `0 0` line follows the last entry — *(meaning unclear from available sources)*.

**`.bcl`** — blank in every inspected file; likely a duct/tendon-loss-coefficient catalog analogous to `.bcb`, but unconfirmed.

### 7.16 Heat exchangers (`.hex`, `.hef`), Groups (`.grp`)

All blank/`0` in every one of the 71 inspected files. `.hex`/`.hef` header: `"# number of Heat exchangers"`. No populated examples exist for any of the three.

### 7.17 `.gsh` — General shell

Count-terminated; the count matches `"number of Shell one layer elements"` (the same field that drives `.ilg`, §7.5) — confirmed by cross-checking `bridge_loads_on_shells_withVols.inp` (count = 966) and `UL_shell.inp` (count = 1).

**Large model, one line per shell** — `bridge_loads_on_shells_withVols.inp`:
```
.gsh
966
65 0
66 0
67 0
...
1030 0
```
966 lines, each `<element-number> <flag>`, flag = `0` throughout (default/inactive) for shell elements 65–1030.

**Small model, inconsistent line count** — `UL_shell.inp` (a single-shell model): declared count is `1`, but **two** data lines follow (`2 1` and `1 3`) — inconsistent with the one-line-per-shell pattern above.
```
.gsh
1
2 1
1 3
```
It is unclear whether this is a single two-line record or the count has a different meaning here; *(meaning unclear from available sources)* — hand-editors relying on this marker should verify against a known-good ZSoil-exported file rather than this documentation alone.

---

## 8. Boundary Conditions

### 8.1 Nodal boundary conditions (`.inb`)

One line per constrained node, count from the header ("number of boundary conditions"). The record is `<idx> <nodeId> <flag>` followed by **one fixity block per translational DOF** — `[fixedFlag, prescribedValue, ?, ?, (?)]`:

- **3D** (confirmed against `zsoil_inp.py`, which reads flags at token indices 3, 8, 13 — i.e. block width 5, 19 tokens total): `<idx> <nodeId> <flag> [xFixed xVal ? ? ?] [yFixed yVal ? ? ?] [zFixed zVal ? ? ?] <trailingFlag>`.

  Example (`3D_anchor_disp.inp`, node 1 fixed in x and z):
  ```
  .inb
  1 1 1 1 0.000000000000e+00 0 0 0 0 0.000000000000e+00 0 0 0 1 0.000000000000e+00 0 0 0 0
  ```

- **2D — width varies by file, not reliably by dimension alone.** Two different 2D samples (`el_simple_load_2D_tri.inp`, format sub-version `24.06`; `2D_anchor_disp_s1m.inp`) use the **same 19-token / width-5 layout as 3D** (flags at indices 3, 8, 13):
  ```
  .inb                                    <- el_simple_load_2D_tri.inp
  1 1 1 1 0.000000000000e+00 0 0 0 1 0.000000000000e+00 0 0 0 0 0.000000000000e+00 0 0 0 0
  ```
  but `boxd1.inp` (format sub-version `9.05`, older) uses a **narrower 16-token / width-4 layout**, flags at indices 3, 7, 11:
  ```
  .inb                                    <- boxd1.inp
  1 1 1 1 0.000000000000e+000 0 0 1 0.000000000000e+000 0 0 0 0.000000000000e+000 0 0 0
  ```
  **This discrepancy is not fully explained from available sources.** The most likely explanation, based on the differing format sub-version numbers on the `@GeoDev_SARL_version:` line, is that the `.inb` record gained an extra field per DOF block somewhere between sub-version `9.05` and `24.06` — i.e. it may be a **format evolution within v26**, not a 2D/3D distinction. `zsoil_inp.py`'s `.inb` reader unconditionally uses the 3D/newer-shaped indices `[3,8,13]`, which would misread an older-style 16-token file like `boxd1.inp` (a commented-out alternate branch, `indices = [3,7,11]`, hints the older layout used to be handled explicitly). **Practical guidance**: when hand-editing `.inb`, always match the token width already used elsewhere in the *same* file rather than assuming a fixed layout from this reference.

### 8.2 `.inc` — named constraint/support groups

Appears immediately immediately after `.inb` in every sample; **not parsed by `zsoil_inp.py`**, so this section is derived purely from direct file inspection and should be treated as inferred rather than confirmed. Structure observed: one block per node (same count as `.inb`), each block being a header line, a 6-flag line, then a run of numeric lines, terminated by a free-text **name** whose content is informative — e.g. `Box UXUZ` (suggesting "constrain Ux, Uz") in one sample.

```
<recordIdx> <nodeId> <flag> <flag> <flag>
<flag>[6]
<numeric line>                 <- repeated N times (N varied: 18 lines in one boxd1.inp
...                                block, fewer in others — possibly a per-DOF curve
                                   point count driven by a preceding flag)
<name>                          <- e.g. "Box UXUZ", "No name"
```

**Best inference (not confirmed): named/grouped nodal constraint sets**, layered on top of the plain `.inb` fixity — possibly nonlinear spring-support curves (the repeated `[0, value, 0, 0]`-shaped lines resemble force/displacement curve points) or a GUI "boundary condition group" feature that lets multiple `.inb` entries share a name and extra per-DOF data. If a hand-edit needs to add/modify `.inc` entries, cross-check against a GUI-exported file with the same named BC group rather than authoring from this description alone.

### 8.3 Periodic boundary conditions (`.pbc`)

Per `zsoil_inp.py`: node-pair count, an EF line, a name, then a list of node-id pairs:

```
.pbc
<pairCount>
<EF>
<name>
<nodeIdA1> <nodeIdB1>
...                     <- pairCount lines
```

No sample in the inspected corpus had a non-zero `.pbc` block (all `0`) — the structure above is transcribed from `zsoil_inp.py`'s reader, not independently confirmed against a populated file.

### 8.4 Field (water/heat/humidity) boundary conditions

The following markers cover boundary conditions for seepage/thermal/humidity analyses. **No sample in the available corpus (the `zsoil_inp_files` set, nor the wider `Benchmarking\Unittests` and `v26\manual` trees) had a non-zero count for any of these** — they are documented here only by their confirmed header-count purpose, since inventing record syntax without a real example would risk being wrong:

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

> **Note on `.isg` and `.isl`**: these two markers sit near `.iph`/`.iwh` in the marker sequence and were initially suspected to be BC-related, but direct inspection shows `.isl`'s header label is "number of surface loads" — i.e. it is a **load**, not a boundary condition, and is documented in [§9 Loads](#9-loads) instead. `.isg` has no directly-matching header label; it most likely corresponds to "number of Surface load defined by Pressure head" (shared with `.iwh`, or an adjacent unlabeled load-related count) and is also treated as load-related — see §9.

---

## 9. Loads

### 9.1 Nodal loads (`.inl`)

Count-terminated (from the header "number of nodal forces"). Confirmed against `zsoil_inp.py`: node id, 6 load components, LF, then a name line:

```
<idx> <nodeId> ? ? <Fx> <Fy> <Fz> <Mx> <My> <Mz> <LF>
<name>
```

No populated (non-empty) `.inl` example was found in the inspected corpus despite several files declaring a non-zero header count for it (e.g. `el_simple_load_2D_tri.inp` declares `1` but the `.inl` block itself is blank) — this looks like a quirk of those specific test files (count not matching actual content) rather than a format issue. The field layout above is transcribed directly from `zsoil_inp.py`'s reader, not independently confirmed against a populated file; treat it as reliable (the parser is actively used against production files) but unverified visually.

### 9.2 Beam loads (`.ibf`)

Per `zsoil_inp.py`: for each beam, a line giving the beam id and how many loads apply to it, followed by that many `[6 values, LF]` + name records:

```
<beamId> <nLoadsOnThisBeam>
<Fx> <Fy> <Fz> <Mx> <My> <Mz> ? <LF>
<name>
...                              <- repeated nLoadsOnThisBeam times, then next beam
```

Every sample inspected — including files named `loads_on_beams_*.inp` — has an **all-zero `.ibf` block** (every beam declares 0 loads), e.g.:
```
.ibf
1 0
2 0
3 0
```
This strongly suggests `.ibf` is a legacy mechanism, superseded in current-version files by loads-on-elements (`.ple`, §9.4) even for beam-targeted loads — the "loads_on_beams" sample files put their actual load data in `.ple` instead (see §9.4). Treat `.ple` as the primary mechanism for beam loads in new files.

### 9.3 Surface loads (`.gsl`)

Blank-line-terminated (no separate count field consumed here; the block simply ends at the first blank line). Two record types, confirmed against `zsoil_inp.py`'s `SurfaceLoad` class:

**`UNI_LOAD`** — uniform pressure/traction load over a set of element faces:
```
<idx> UNI_LOAD <?> <LF> <?> <nFaces> <?>
<name>
<Vx> <Vy> <Vz>                  <- load vector/magnitude data
<eleId1> <faceId1>
...                              <- nFaces lines of (element id, local face id)
```
(the local `faceId` numbering for `B8` volumic targets follows §7.1's B8 face-numbering convention; the example below targets a 2D `Q4` element instead, whose edge-numbering convention was not independently derived.)
Example (`el_simple_load_2D_quad.inp`, 4 separate uniform loads on individual element faces):
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

**`GRAD_LOAD`** — a gradient (linearly-varying) load, referencing a reference node and direction, per `zsoil_inp.py`; no populated example was found in the inspected corpus, so its record fields beyond "reference node, direction, data" are not independently confirmed.

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
`targetType` (shared header field, same position in `POINT_LOAD`/`LINE_LOAD`/`SURF_LOAD`):
    - `1`: automatically to any structural element
    - `2`: automatically to Shell elements
    - `3`: automatically to Beam elements
    - `4`: automatically to Truss elements
    - `5`: automatically to Membrane elements
    - `8`: to previously selected faces (3D) / edges (2D). In this case, the selected faces/edges are saved in `<eleId>`/`<faceId>` — for a `B8` volumic target, `<faceId>` follows the face-numbering convention in §7.1. `<nTargets>` then lists either bare node/element ids or `eleId faceId` pairs accordingly, and can also use the [compact element-list notation](#26-compact-element-list-notation), e.g. `569A611P...` style ranges seen in the `LINE_LOAD` element list below.

The <LF2> tag is unconfirmed for now.

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

**`SURF_LOAD`** : a gradient (linearly-varying) surface load applied over a polygon, similar to `.gsl`'s `GRAD_LOAD`. When `targetType=8`, `<faceId>` follows the same B8 face-numbering convention documented in §7.1.

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

Referenced by `Move_Path:` in `.ple` records for moving-load analyses (e.g. vehicle loads crossing a bridge). Empty in every inspected sample (`Move_Path: 0` everywhere, meaning "no path" / not a moving load) — *(record syntax not confirmed from available sources)*.

---

## 10. Masses

Separate from loads (§9) — these define additional lumped/distributed mass, primarily for dynamic (seismic/vibration) analyses. **No sample in the available corpus (`zsoil_inp_files`, nor the wider `Benchmarking\Unittests` / `v26\manual` trees) had a non-zero count for either marker below** — both are documented purely from `zsoil_inp.py`'s confirmed reader logic, not independently verified against a populated file. If a hand-edit needs to add mass data, generate a minimal example from the GUI and inspect it directly rather than authoring from this reference alone.

### 10.1 Nodal masses (`.inm`)

Count-terminated (header "number of nodal masses"). Per `zsoil_inp.py`:

```
<idx> <nodeId> ? ? <massValue>
<line 2 — not read by the parser, purpose unclear>
<name>
```

### 10.2 Element masses (`.iem`)

Note this section is **blank-line/self-count-terminated with its own leading count line** (not solely driven by the global header count), per `zsoil_inp.py`:

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

The header also carries a second, related pair of counts — "number of Line masses on elements" and "number of Surface masses on elements" — which correspond to the `.pme` marker (loads-on-elements-style masses, analogous to how `.ple` extended `.ibf`/`.gsl` for loads). No populated example of either `.iem` or `.pme` was found; `.pme`'s record shape is *(not confirmed from available sources — likely mirrors `.ple`'s `LINE_LOAD`/`SURFACE_LOAD` shape but for mass instead of force, given its position immediately after `.ple` in the marker sequence)*.

---

## 11. Initial Conditions

Initial-state data applied to elements/nodes before the first analysis step (e.g. geostatic stress, phreatic surface). `zsoil_inp.py` does **not** parse any of these markers — everything below is derived from direct file inspection.

### 11.1 Initial stress conditions (`.izg`)

Count-terminated (header "number of ini. stress cond."). Assigns an explicit initial stress tensor to a list of elements:

```
<idx> <nElements> <eleId1> <eleId2> ... <eleIdN> <flag> <flag>
<name>
<sxx> <syy> <szz> <sxy> <syz> <sxz>       <- one line per element in the list, in order
...
```

Example (`boxd2.inp`, initial vertical stress of -10 applied to elements 1–4):
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

Example (`casea.inp`, initial condition value assigned to elements 3, 1, 2, 4):
```
.iig
1 4 3 1 2 4 0 0
-4.000000000000e+000
-3.400000000000e+001
-3.400000000000e+001
-4.000000000000e+000
No name
```
(values `-4, -34, -34, -4` for elements `3, 1, 2, 4` respectively — note the `<name>` line comes *after* the value list here, unlike `.izg` where the name precedes the data; order was confirmed by direct inspection of this sample, not assumed.)

### 11.3 Heat / humidity / strain initial conditions

The remaining initial-condition markers had **no non-zero header count anywhere in the inspected corpus**, so only their header-count purpose is documented, not populated syntax:

| Marker | Header count label |
|---|---|
| `.iag` / `.iav` | number of Heat IC / number of Heat IC values |
| `.img` / `.imv` | number of Humidity IC / number of Humidity IC values |
| `.ieg` | number of ini. strains cond. (Imposed strains) |
| `.ist` | number of constant eps0 |

Given the `.izg`/`.iig` pattern above (id + element list + flags, then per-element data, with name position varying), these likely follow a similar "element list header, then per-element data block" shape — but this is an **extrapolation, not a confirmed fact**, since no populated sample exists to verify field count or name-line placement for these specific markers.

### 11.4 Initial velocities/accelerations (`.ivd`, `.svd`)

Header count "number of initial velocities/accelerations" was `0` in every inspected sample. *(Record syntax not confirmed from available sources — relevant only for dynamic/seismic analyses with non-zero initial velocity fields.)*

---

## 12. Reinforcement

### 12.1 Reinforcement sets and members (`.brc`)

No populated (non-empty) `.brc` block was found anywhere in the inspected corpus (typically used for reinforced-concrete beam/shell rebar layouts, which none of the available sample files define) — the structure below is transcribed directly and in detail from `zsoil_inp.py`'s reader, which is unusually thorough for this section, so it is presented with confidence despite lacking a raw-file cross-check:

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

Key confirmed fields per layer (from `zsoil_inp.py`'s `ReinfLayer`): `length_type` (how the layer's extent along the beam is defined), `dist_l`/`dist_r` (left/right distances), `ypos_type`/`ydist` (position across the section), `zoffset_l`/`zoffset_r`, `diam` (bar diameter), `nBars` (bar count), `prestress`, `enabled` (layer active flag, `False` when the corresponding field is `'0'`), and `material` (material id for the rebar itself, separate from the host beam's material). A `ReinfMember` links a named reinforcement set to a specific list of beam elements — i.e. the same rebar layout can be reused across many beams by reference name.

---

## 13. Mesh Tying & Domain Reduction

### 13.1 Face groups / mesh tying (`.fac`, `.mrt`)

Per `zsoil_inp.py` (no populated example found in the inspected corpus):

```
.fac
<nFaceGroups>
<nFaces>                    <- per face group
<name>
<eleId1> <faceId1>
...                          <- nFaces lines
...                          <- repeated per group
```

`.mrt` sits immediately after `.fac` in the marker sequence and was empty (`0`) in every sample; not parsed by `zsoil_inp.py`. *(Purpose inferred as a related mesh-tying setting from its position — not confirmed.)*

### 13.2 Domain Reduction Method (`.drz`, `.dre`)

Per `zsoil_inp.py`: a simple element-id list for each of the DRM domain's interior and exterior element sets (used for seismic wave-propagation analyses with a reduced/absorbing domain boundary):

```
.drz
<nElements>
<eleId1> <eleId2> ...        <- nElements ids (interior elements)

.dre
<nElements>
<eleId1> <eleId2> ...        <- nElements ids (exterior elements)
```

Both were empty (`0`, no id list) in every inspected sample.

### 13.3 Other structural-mesh markers (unconfirmed)

The following markers appear in the sequence but were empty in every sample inspected and are not parsed by `zsoil_inp.py`; only their header-count purpose (where one exists) is documented:

| Marker | Notes |
|---|---|
| `.eie` | Position corresponds to no directly-matching header label found; *(purpose unclear from available sources)*. |
| `.crc` | Same — *(purpose unclear)*. |
| `.idv` | Same — *(purpose unclear)*. |
| `.spg`, `.svg`, `.scg` | Cluster of three markers near `.sdm`/`.psd` — likely further subdomain/staging metadata given their position in the sequence (§6.4); *(purpose unclear)*. |
| `.gos` | **Write-only in `zsoil_inp.py`**: not read by `read_inp()`, but written by `write_new_inp()` as "geometrical surfaces (element connectivity + boundary line list)" — confirming it exists and has a defined purpose in the format even though this reference can't show real populated syntax for it. |
| `.cld` | Empty (`0` × 3 lines) in every sample; *(purpose unclear)*. |
| `.igl` | See §6.5 (auxiliary points). |

If a hand-edit needs to touch any of these, the most reliable path is still to reproduce the change in the ZSoil GUI on a minimal file and diff the result, rather than authoring from this reference.

---

## 14. Quick Reference

### 14.1 All 91 dot-prefixed markers, in file order

Compiled from a full scan of `boxd1.inp`; the same 91 markers, in the same order, were confirmed present in every other v26 sample inspected (only their contents vary). "Doc §" is where each marker is documented; a dash means the marker was found empty/`0` in every inspected sample and could not be documented beyond its position in the sequence — treat these as confirmed-to-exist-but-unpopulated-in-samples, not as unused/removable.

| # | Marker | Purpose | Doc § |
|---|---|---|---|
| 1 | `.icf` | Contact evolution function references | §15 (mention only) |
| 2 | `.ing` | Nodes | §6.1 |
| 3 | `.inl` | Nodal loads | §9.1 |
| 4 | `.inb` | Nodal boundary conditions | §8.1 |
| 5 | `.inc` | Named constraint/support groups (inferred) | §8.2 |
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
| 25 | `.isl` | Surface loads (header-confirmed; content not independently shown) | §8.4 note, §9 |
| 26 | `.iwf` | Water flux (BC) | §8.4 |
| 27 | `.iab` | Heat boundary conditions | §8.4 |
| 28 | `.imb` | Humidity boundary conditions | §8.4 |
| 29 | `.iwb` | Water boundary conditions | §8.4 |
| 30 | `.ibf` | Beam loads (legacy — superseded by `.ple`) | §9.2 |
| 31 | `.glk` | Local bases | §6.5 |
| 32 | `.ilb` | *(purpose unclear — empty in all samples)* | — |
| 33 | `.ihf` | Heat flux (BC) | §8.4 |
| 34 | `.iuf` | Humidity flux (BC) | §8.4 |
| 35 | `.ghf` | *(likely macro/geometry-level heat flux — unconfirmed)* | — |
| 36 | `.guf` | *(likely macro/geometry-level humidity flux — unconfirmed)* | — |
| 37 | `.gab` | *(likely macro/geometry-level heat BC — unconfirmed)* | — |
| 38 | `.gmb` | *(likely macro/geometry-level humidity BC — unconfirmed)* | — |
| 39 | `.gsl` | Surface loads (UNI_LOAD, GRAD_LOAD) | §9.3 |
| 40 | `.gwb` | *(likely macro/geometry-level water BC — unconfirmed)* | — |
| 41 | `.gwf` | *(likely macro/geometry-level water flux — unconfirmed)* | — |
| 42 | `.idg` | *(purpose unclear — empty in all samples)* | — |
| 43 | `.isd` | Data super elements | §6.5 |
| 44 | `.ily` | Shell layers | §7.5 |
| 45 | `.inm` | Nodal masses | §10.1 |
| 46 | `.iem` | Element masses | §10.2 |
| 47 | `.cld` | *(purpose unclear — empty in all samples)* | §13.3 |
| 48 | `.igl` | Auxiliary points | §6.5 |
| 49 | `.axs` | Axes | §6.5 |
| 50 | `.pob` | Geometry points | §6.2 |
| 51 | `.gob` | Geometrical objects (GEOMLINE/ARC/PLINE) | §6.3 |
| 52 | `.sdm` | Subdomains | §6.4 |
| 53 | `.psd` | Subdomain-related (unconfirmed) | §6.4 |
| 54 | `.fac` | Face groups / mesh tying | §13.1 |
| 55 | `.mrt` | Mesh-tying related (unconfirmed) | §13.1 |
| 56 | `.idv` | *(purpose unclear — empty in all samples)* | §13.3 |
| 57 | `.crc` | *(purpose unclear — empty in all samples)* | §13.3 |
| 58 | `.spg` | *(subdomain/staging-related, unconfirmed)* | §13.3 |
| 59 | `.svg` | *(subdomain/staging-related, unconfirmed)* | §13.3 |
| 60 | `.scg` | *(subdomain/staging-related, unconfirmed)* | §13.3 |
| 61 | `.gos` | Geometrical surfaces (write-only in `zsoil_inp.py`) | §13.3 |
| 62 | `.ish` | Shell hinges | §7.5 |
| 63 | `.ist` | Constant eps0 (initial strain) | §11.3 |
| 64 | `.goa` | Subdomain-related (unconfirmed) | §6.4 |
| 65 | `.sg0` | Subdomain-related (unconfirmed) | §6.4 |
| 66 | `.gnl` | Nodal links | §7.10 |
| 67 | `.pil` | Piles | §7.11 |
| 68 | `.eie` | *(purpose unclear — empty in all samples)* | §13.3 |
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
| 81 | `.scs` | Structural-contact related (unconfirmed) | §7.9 |
| 82 | `.ims` | Structural-contact related (unconfirmed) | §7.9 |
| 83 | `.brc` | Reinforcement sets and members | §12.1 |
| 84 | `.cbl` | Cable/tendon element instances | §7.15 |
| 85 | `.bcb` | Cable geometry catalog | §7.15 |
| 86 | `.bcl` | Cable loss-coefficient catalog (unconfirmed) | §7.15 |
| 87 | `.hex` | Heat exchangers | §7.16 |
| 88 | `.ple` | Loads on elements (POINT_LOAD/LINE_LOAD/SURF_LOAD) | §9.4 |
| 89 | `.pme` | Masses on elements (unconfirmed) | §10.2 |
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

`GEOM->`, `DENS->`, `FLOW->`, `CREEP->`, `NONL->`, `ELAS->`, `HEAT->`, `HUMID->`, `INIS->`, `STAB->`, `DISC->`, `DAMP->`, `MAIN->` — see §4.2 for full field-level detail per tag.

---

## 15. Worked Example

This walks the complete file `boxd1.inp` (616 lines, `zsoil_inp_files\boxd1.inp`) end to end — a minimal 2D model: a single `Q4` continuum element on 4 nodes, 1 material, 4 boundary conditions, and every other block empty. It's small enough to read in full, and every non-empty line in it is explained below. Line numbers refer to the original file.

| Lines | Content | Explanation |
|---|---|---|
| 1 | `@GeoDev_SARL_version: 26 9.05 26.03` | Version line (§3.1). |
| 2–136 | `2 # dimension 2 or 3` / `1 # number of materials` / `0 # number of Existence functions` / `0 # number of nodal forces(*.inl)` / `4 # number of nodes (*.ing)` / `1 # number of continuum 2D elements` / `4 # number of boundary conditions (*.inb)` / ... (rest `0`) | The fixed 135-line header count block (§3.2) — this file is 2D, 1 material, 0 EF/LF, 4 nodes, 1 `Q4` element, 4 BCs, everything else zero. |
| 137–141 | `ASSOCIATED_PROJECTS:` / `HEAT: -` / `HUMIDITY: -` / `FREE_FILED_MOTION: -` / `FLOW: -` | No linked companion projects (§3.3). |
| 142–145 | `STANDARD` / `kN  m  deg  year  C` (×2) / *(blank)* | Unit system: kN, m, degrees, years, °C. |
| 146 | `4 0` | Un-named flag pair preceding `CONTROL` — *(meaning unclear from available sources)*. |
| 147–150 | `CONTROL 1` / `Default` / 18-value line / 10-value line | One named solver-control set, "Default" (§3.4). |
| 151–160 | `DYN_CONTROL 1` ... `PSH_CONTROL 1` ... | Dynamic/pushover control, one "Default" set each. |
| 161–166 | `NONL_GEOM 0` / `CONSTRUCTION 0  0` / `PROJECT_PRESELECTION` / `5 0 0 0 0 1` | Simple analysis flags (§3.4) — no large-deformation, no construction staging. |
| 167 | `NUM_MATERIALS= 1` | Begin materials block (§4). |
| 168–171 | `MATERIAL  26.00 1` / `Elastic` / `No name` / `BUTTONS= 1 1 0 0 1 0 0 0 0 0 0 0 0 0` | One material: id 1, type "Elastic", unnamed. |
| 173–196 | ` ELAS-> 100000  0.3` ... ` DAMP-> 0  0  0  0  0  0  0` | Full sub-tag sequence for this material (E=100000, ν=0.3, thickness 0.1, unit weight 20 — see §4.2 for the full field-by-field breakdown of this exact example). |
| 197 | `NUM_MATERIALS= 0` | No fiber materials follow. |
| 198 | `LAYERED_BEAM_COMPONENTS 0` | None defined. |
| 199–203 | `SIG_EPS_FUN 1` / `1 2 Default` / ... | One (unused, since no nonlinear material references it) stress-strain function. |
| 204–207 | `EXIST_FUNC 1` / `1 No name` / `1` / `0 3.37e+38` | One EF defined but never referenced by any element in this file (§5.1 example). |
| 208–212 | `LOAD_FUN 1` / `1 1 No name` / `No flags` / `0 1 0` / `0 0` | One trivial LF (§5.2 example) — unused, since this file has no active loads. |
| 213–289 | `DRIVERS 0` ... `PARALLEL_SETTINGS` / `1  20  0.0625` / `ARC_LENGTH` / `0  1` / `0` / `0` | Output/solver/project-metadata settings (§3.5) — all defaults, empty project metadata. |
| 291–306 | `AUGMENTED_LAGRANGIAN` ... `CONTACT_PARAMETERS 1` / `1 Base table` / ... `EVOLUTION_FUN 0` | Default contact-solver settings; one unused default contact material table. |
| 308 | `.icf` | Empty marker (contact evolution function list — not otherwise documented, empty here). |
| 310–314 | `.ing` / `1 0.0 0.0 0.0 0` / `2 1.0 0.0 0.0 0` / `3 1.0 1.0 0.0 0` / `4 0.0 1.0 0.0 0` | 4 nodes forming a unit square (§6.1). |
| 316–317 | `.inl` / *(blank)* | No nodal loads. |
| 318–322 | `.inb` / 4 lines | 4 boundary conditions — nodes 1–2 fixed in x&y, nodes 3–4 fixed in y only (16-token / older-sub-version layout, §8.1). |
| 324–408 | `.inc` / 4 named blocks (`No name` ×4) | Per-node constraint-group data mirroring the 4 `.inb` entries (§8.2 — inferred, not fully confirmed). |
| 410–411 | `.i0g` / `1 1 Q4 1 2 3 4 1 1 1 0 0 0 0 0` | The single `Q4` continuum element, connecting nodes 1-2-3-4, material 1 (§7 — see the Elements chapter for the full field breakdown of this record). |
| 413–607 | `.ibg` through `.hex` — **81 markers, every one empty** | Every element/BC/load/geometry marker not otherwise used by this minimal model — each present as a bare marker line followed by a blank line (or `0`), per the "every marker always present" rule of §2.2. See [§14 Quick Reference](#14-quick-reference) for what each of these markers is for. |
| 609–616 | `.ple` / *(blank)* / `.pme` / *(blank)* / `.pth` / *(blank)* / `.hef` / *(blank)* | Final markers in the fixed sequence, all empty — end of file. |

**Takeaway**: even a maximally minimal model touches all 91 dot-markers — the file format doesn't omit unused sections, it just leaves them empty. When hand-editing an existing file, the safest strategy is exactly what this table demonstrates: locate the marker for the section you need by name, and edit only its block, leaving the surrounding empty markers untouched.

---

