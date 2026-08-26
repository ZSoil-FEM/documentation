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
14. [Other Data](#14-other-data)
15. [Quick Reference](#15-quick-reference)
16. [Worked Example](#16-worked-example)

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

- **Dot-prefixed markers**, e.g. `.ing`, `.i0g`, `.inb` — historically named after the companion file extension that used to hold that data. There are **91 distinct dot-prefixed markers**, always appearing in the same fixed order (see [§15 Quick Reference](#15-quick-reference) for the complete list). Every marker is present in every file, even when its block is empty — an empty block is simply the marker line followed immediately by a blank line (or, for count-terminated blocks, a `0` count). **Mechanism, confirmed from source**: each marker's data is actually assembled from a separate per-extension fragment file (e.g. `<basename>.idg`), all listed by name in an external config file (`CFG\prepro.fil`, loaded once at startup into an in-memory list) and concatenated in that fixed order into the final `.inp`. A step that runs before every save (`ZMATE_CreateNoExistingPreproFile`) blanket-creates an *empty* fragment file for any extension in that list that no other code path has already written to — which is exactly why some markers (e.g. `.idg`, see §15.1) are present in every real file yet have no dedicated writer/reader anywhere in the source: nothing in the current codebase actually populates them, they just always exist as empty placeholders.
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

### 2.5 Existence functions (EF), Load functions (LF), and Unloading functions (ULF)

Nearly every element, load, and mass record references an **existence function number (EF)** and, where relevant, a **load function number (LF)** — these are defined once (in `EXIST_FUNC` / `LOAD_FUN`, [§5](#5-existence--load-functions)) and referenced by number everywhere else in the file:

- **EF 0** is implicit and always active — the "permanent" existence function, used by anything that exists for the whole analysis.
- **LF 0** is implicit and equal to a constant multiplier of 1 — used by anything not subject to a time-varying load history.
- Any other EF/LF number must be defined in the corresponding `EXIST_FUNC`/`LOAD_FUN` block earlier in the file.
- An EF reference isn't only used to control whether an *element* exists — it can also gate a boundary condition's fixity, letting a `.inb` record stay nominally fixed while only actually being enforced during that EF's active interval (see §8.1 for how a `.inb` fixity block references an EF this way).

Every element/material record's own trailing block also carries an **unloading function (ULF)** field alongside its `EF` — this references a `LOAD_FUN` entry by number too, the same list as `LF`, just used in a different *role*: while `LF` drives a quantity's value as it's applied/loaded, `ULF` governs its value during an unloading/deactivation transition (e.g. when the element or BC leaves its `EF` interval). `0` means no unloading function assigned. No sample file inspected so far uses a non-zero `ULF`, so its behavior in practice isn't documented here beyond this.

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
| 23 | number of viscous dampers (`.vsd`) — the label's own cited extension is stale; the actual markers written are `.ivd`/`.svd` (§13.4) |
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
| 82 | number of initial velocities/accelerations (`.idv`, §11.4) |
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

*(Field indices above are relative to this table, not raw file line numbers. Entries 33, 49, and 88 are genuine upstream labeling bugs, not fields with an undocumented separate meaning: the writer prints one description string per field from a fixed array, and for these three the array slot was left at its default "Not used" (entry 33) or accidentally duplicates a neighboring field's description (entry 49 duplicates entry 48's "number of element masses" label; entry 88 duplicates entry 86's "number of Tendons" label) — the underlying count fields are real and populated, only their printed labels are wrong.)*

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
| 5 | Nonlinear solver `ALGOID`: `1`=Full Newton-Raphson, `2`=BFGS, `3`=Initial stiffness, `4`=Modified N-R, `5`=Symmetric Full N-R, `6`=Initial stiffness (accelerated) | `1` |
| 6 | Max. number of iterations per step | `15` |
| 7 | Reform stiffness every *n* steps (Modified N-R) | `1` |
| 8 | Reform stiffness every *n* iterations (Modified N-R) | `1` |
| 9 | Restart-file save interval, `RESTART` | `1` |
| 10 | Print-output step interval, `IPRINTSTEP` | `1` |
| 11 | Plot-output step interval, `IPLOTSTEP` | `1` |
| 12 | Absolute max. number of iterations, `ABSMAX` | `200` |
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
| 4 | Strategy for divergent steps: automatic solver switching flag, `TryOtherNLsolvers` | `1` |
| 5 | Auto-increase-max-iterations flag, `AutoIncMaxitFlag` | `1` |
| 6 | Auto-increase amount, `AutoIncreaseIterByValue` | `5` |
| 7 | Amplification factor on RHS non-convergence, `AmplFactorForRHSNonconvergence` | `5` |
| 8 | Line-search flag, `LineSearch` (ver≥14.12) | `0` |
| 9 | Skip-diverged-steps flag, `SkipDivergedSteps` (ver≥16.98) | `0` |
| 10 | Available-solvers bitmask, `AvailableSolvers` (ver≥17.00): bit1=FNR, bit2=BFGS, bit4=IS, bit8=MNR, bit16=IS-accelerated, `31`=all, `23`=all except MNR | `0` |

The same "named-set" pattern (keyword, count, then that many named sets) applies to:

- **`DYN_CONTROL <n>`** — dynamic-analysis solver control (same general kind of tolerance/iteration settings, extended for time integration). Each named set is 5 lines:

  ```
  <name>
  <?> <?> <?> <?> <scheme> <alpha> <beta> <gamma>
  <?> <g> <?> <?> <?>
  <?>
  <alpha2> <?> <?>
  <?> <?> <?> <?> <?>
  ```

  Line 2, fields 6–8 are the **HHT-α time-integration parameters**: `alpha`, then the dependent Newmark `beta`/`gamma` — confirmed via the standard identities `beta = (1-alpha)^2/4`, `gamma = 0.5-alpha` (e.g. `alpha=-0.3` gives `beta=0.4225`, `gamma=0.8`, matching a real example exactly). Field 5 (`scheme`, `3` in every example seen) is presumably an integration-scheme selector; fields 1–4 unclear. Line 3, field 2 is gravitational acceleration `g` (`9.80655` in every example seen); the rest of that line and the following two lines are unclear, except line 5's **first field, which is `alpha` again** — the same value as line 2 field 6, apparently a redundant/secondary copy rather than an independent override (in every example checked so far the two agree; if a real file ever shows them *disagreeing*, that would need investigating — which one wins is not established).
- **`PSH_CONTROL <n>`** — **pushover-analysis solver control**, distinct struct from `CONTROL`. Each named set is 2 lines: `<name>` then `<TypeMass> <RotInertia> <MassFilter> <ForcePattern> <Acc> <DirPsh_x> <DirPsh_y> <DirPsh_z>` (`ForcePattern`: `0`=auto, `1`=constant, `2`=linear — only if `2`, a further line `<DForce_x,y,z> <Xo_x,y,z>` follows).

Following the solver-control blocks:

```
NONL_GEOM <GeomNonl>              <- large-displacement/nonlinear-geometry flag
CONSTRUCTION <n> <Active>
<tbeg> <tend> <mult1> <mult2> <inct>     <- n lines, one per multi-initial-state definition
THM_SETTINGS
BISHOP_FLAG <Bishop_flag>         <- 0=saturation ratio S, 1=effective saturation Seff
PROJECT_PRESELECTION
<MAX> <SelTab[0]> ... <SelTab[MAX-1]>    <- MAX=5: FRAMES_ONLY,STR_ONLY,DYNAMICS,PUSHOVER,SHOW_ALL preselection flags;
                                             SHOW_ALL uses 0=meaningful-only,1=all-in-color,2=all-in-black
```

### 3.5 Driver / output settings and project metadata

Later in the header (after materials and functions — see §4–5), a `DRIVERS <n>` block is followed by a long sequence of simple `KEYWORD` / `<value>` pairs — each keyword on its own line, its value(s) on the line(s) immediately after. These are largely self-explanatory from their names.

**`DRIVERS <n>` does *not* follow §3.4's simple 2-line (name + numeric) pattern** — each driver record is **3 lines**, and `<n>` counts drivers **after** a mandatory, always-present `init` driver that isn't itself counted:

```
DRIVERS <n>
<name>                                    <- init driver (always present, not counted in <n>)
<type> <?> <t_start> <t_end> <t_incr> <mult> <?>
<solver-settings-name>
<name>                                    <- driver 1 of <n>
<type> <?> <t_start> <t_end> <t_incr> <mult> <?>
<solver-settings-name>
...                                        <- <n> driver records total, 3 lines each
```

`<name>` (line 1 of each record) is a free-form label — it does **not** need to match anything defined elsewhere (in a real example it was literally `N-R`, coincidentally the same string as one of `CONTROL`'s named settings sets, but functioning as an independent label, not a reference). `<solver-settings-name>` (line 3) **does** need to match a name defined in the `CONTROL` block (e.g. `Default`) — using an unrecognized name here is what produces a "DRIVER ... not listed under CONTROL" failure.

The numeric line (line 2) is `<type> <?> <t_start> <t_end> <t_incr> <mult> <?>`, with `<type>` a **0-indexed** `DriversType` code (`C_DriversType.py` in ZSoilPy3: `INITIAL_STATE=1, STABILITY=2, TIME_DEPENDENT=3, ARC_LENGTH=4, DYNAMICS=5, PUSHOVER=6, EIGENMODES=7`, so 0-indexed **DYNAMICS = 4**, confirmed against real examples using exactly that value for a genuine dynamic phase) — the `init` driver's own `<type>` is `0` (0-indexed `INITIAL_STATE`).

After all `<n>+1` driver records, **4 more lines appear before the `RAM_MAXIMUM`/`SOLVER_TYP`/... keyword sequence starts**, not accounted for by the driver-record pattern above and not yet decoded — e.g.:
```
0 0 0 0 -1 0
0 0 -1 0
0
0  0  0  1  0  0  0  1  0  0  0  1
```
Present, unchanged, in every example seen so far; treat as required boilerplate until their purpose is established.

The following keywords, one per line with value(s) immediately after:

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

`DRIVERS <n>`'s 3-line record (§3.5 above) has one version-gated trailing field not otherwise noted: `<SaveRestart>` after `<mult>`/`<?>`, present only for files saved with ZSoil ≥17.00.

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

`BUTTONS=`'s 14 flags, in order: `Elastic, Unit weights(Density), Geometry, Main, Flow, Creep, Non linear, Heat, Humidity, Initial Ko State, <unused — legacy Impermeability slot>, Stability, Discontinuity, Damping`. Each toggles whether the corresponding sub-tag block is meaningful for this material (an unset flag's block is still written, just with default/inert content). For a plain vs. layered/composite `Elastic Beam`, it's the `Non linear` flag (index 6) that's repurposed as the "layered cross-section" selector — see §4.3.2.

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
| Continuum | `HS-small strain stiffness` (short name `HSS` in the `<name>` line's own conventional use, though `<name>` is free text) | same tag set as `Elastic`, `NONL->` populated | Hardening-Soil-type nonlinear/stress-dependent stiffness model with a small-strain plateau. See §4.3.1/§4.3.7 for confirmed `ELAS->`/`NONL->` field detail. |
| Continuum | `Drucker Prager` | same tag set, `NONL->` populated | Fits a DP cone to the Mohr-Coulomb hexagon — see §4.3.7 for the fitting-mode field. |
| Continuum | `True Mohr-Coulomb` | same tag set, `NONL->` populated | The actual MC hexagonal criterion (not a DP approximation) — see §4.3.7. |
| Continuum | `Multilaminate` / `Multilam. Menetrey` | same tag set, `NONL->` populated | Discrete-lamination-plane model — up to 3 planes, each with its own friction/dilatancy/cohesion/tension strength; the `Menetrey` variant adds a combined global failure criterion on top. See §4.3.7. |
| Continuum | `Mohr-Coulomb` / `Hoek-Brown` / `Rankine` / `Huber-Mises` (the "Menetrey-Willam" family) | same tag set, `NONL->` populated | **All four share the literal `<type>` string `"Menetrey"`** (DAT code `PLAS_ME_V`) — the GUI's friendly catalog names ("Mohr-Coulomb (M-W)" etc.) are display-only and never written to the file. Which of the 4 a given material actually is lives entirely inside its `NONL->` block's own `Mentype` field — see §4.3.7. |
| Continuum | *(other nonlinear soil models: Duncan-Chang, Cap model, Cam-Clay, Hujeux)* | same tag set, `NONL->` populated | Distinct model classes, not yet field-mapped. |
| Continuum / shell fiber layer | `Concrete elastic plastic damage` / `Concrete elastic plastic damage for shell` | same tag set, `NONL->` populated | Lee-Fenves concrete damage-plasticity model (DAT code `CDPM_1_V`) — see §4.3.7. |
| Beams (`BEL2`, §7.2) | `Elastic Beam` (plain) | `ELAS->` `GEOM->` `MAIN->` `DENS->` `FLOW->` `CREEP->` `NONL->` `HEAT->` `HUMID->` `INIS->` `STAB->` `DISC->` `DAMP->` | `GEOM->` selects Profiles/User/Values cross-section (§4.3.2). |
| Beams | `Elastic Beam` (layered/composite) | same tag set; `GEOM->` carries embedded reinforcement fibers | Same type *string* as plain `Elastic Beam` — distinguished only by `BUTTONS=`'s `Non linear` flag, index 6, repurposed as an `IsLayeredCrossSection` selector (§4.3.2). Fibers reference `LAYERED_BEAM_COMPONENTS` (§4.4). |
| Trusses (`TRS2`/`LNK2`, §7.3) | `Truss/Cable` | `ELAS->` `GEOM-><area>` `MAIN->` `DENS->` `FLOW->` `CREEP->` `NONL->` `HEAT->` `HUMID->` `INIS->` `STAB->` `DISC->` `DAMP->` | Only the numeric sub-tag content is documented, via the element's `mat` reference. |
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
| Continuum `HS-small strain stiffness` | see field table below | `ELAS-> 80000  0.2  0.5  100  10  0  1  193766  0.0002  2  1  0  90  0  2  1.6` |

The secondary lines for Continuum/Beam are not anisotropy or damping data — they're a load-function ref, a spatial-data ref, and (ver≥16.95) an evolution-function ref for `E`/`nu`, i.e. `E.Lf nu.Lf` / `E.Sdata nu.Sdata` / `E.EvolFun nu.EvolFun`; all `0`/unset in every model seen so far. Note `"Orthotropic shell"`'s material type string is lower-case `"shell"`, inconsistent with `"Shell Layered"`/`"Fiber Shell"`'s capitalization elsewhere in the format.

**`HS-small strain stiffness` `ELAS->` — fully field-mapped**:

| # | Field | Notes |
|---|---|---|
| 1 | `E_ur_ref` | Unload-reload reference modulus |
| 2 | `v_ur` | Unload-reload Poisson's ratio |
| 3 | `m` | Stress-dependency exponent — stiffness scales as `(σ'/pref)^m` |
| 4 | `sig_ref` | Reference stress `pref` |
| 5 | `sigL` | |
| 6 | `explicit_mode` | Unused |
| 7 | `ELAS_advanced_setup` | Flag |
| 8 | `E_o_ref` | Small-strain reference modulus `E0ref`/`G0ref` |
| 9 | `gamma_07` | Small-strain threshold shear strain `γ0.7` |
| 10 | `small_strain_mode` | `0`=disabled, `1`=Benz formulation, `2`=Brick formulation |
| 11 | `stress_dependency` | Stiffness stress-dependency basis (ver≥12.15 only); older files: `0`=σ₃-based, `1`=p-based |
| 12–16 | anisotropy block (ver≥23.5 only) | `Aniso_theta`, `Aniso_phi` (orientation angles, ver≥24.05) or a legacy placeholder pair; `Anisotropy` (enable flag); `GhhByGvh`; `Beta` |

Matches the example exactly: `E_ur_ref`=80000, `v_ur`=0.2, `m`=0.5, `sig_ref`=100, `sigL`=10, `explicit_mode`=0, `advanced_setup`=1, `E_o_ref`=193766, `gamma_07`=0.0002, `small_strain_mode`=2 (Brick), `stress_dependency`=1, then the anisotropy block `0 90 0 2 1.6` (angles 0°/90°, anisotropy disabled, `GhhByGvh`=2, `Beta`=1.6).

**`NONL->` for `HS-small strain stiffness` — fully field-mapped**:

| Line | Fields | Notes |
|---|---|---|
| 1 | `GlobPhi GlobPsi GlobCoh E_50_ref Rf D D_small_strain DilatancyCutOff eMax explicit_sin_psi` | friction angle φ, dilatancy angle ψ, cohesion c, secant reference modulus `E50ref`, failure ratio `Rf`, Rowe's-dilatancy-law multipliers `D`/`D_small_strain`, a cutoff flag, max void ratio, and an unused legacy field |
| 2 | `H M KoNC sig_ref_oed E_oed` | **cap parameters** `H`/`M` — auto-derived from the `ELAS->` block unless overridden (this is what the A/B test below actually observed changing) — then `K0^NC`, reference oedometer stress, oedometer tangent modulus `Eoed` |
| 3 | `tensile_cut_off CutOff_Value` | flag + value |
| 4 | `init_state_def_method pco_min POP OCR KoSR NONL_advanced_setup automatic_H_M_eval [KoSR_Setup (ver≥13.02)] [ApplyM1ForCapHardening CutOffUndrainedShearStrength (ver≥24.05)]` | initial-state method (`0`=OCR-based, `1`=qPOP-based), min preconsolidation stress, preoverburden pressure, OCR, `K0` stress ratio, and flags |
| 5–6 (ver-gated) | `Lf` refs then `Sdata` refs for `GlobPhi/GlobPsi/GlobCoh/E_50_ref/H/M/E_oed` | only on ver≥14.14/16.01 (Lf line) and ver≥19.01 (Sdata line) |

`H`/`M` (line 2) are the two auto-derived values an earlier A/B test observed changing when `ELAS->`'s `m` changed (`35940.9 1.028...` → `44429.5 1.265...`) — now confirmed as the model's cap-yield-surface parameters, auto-estimated whenever `automatic_H_M_eval` (line 4) is on (the GUI default).

**`NONL->` for `Drucker Prager` and `True Mohr-Coulomb` — fully field-mapped** (both share one underlying record shape; True Mohr-Coulomb adds a few fields of its own):

| Line | Fields | Notes |
|---|---|---|
| 1 | `DPAdjust GlobPhi GlobPsi GlobCoh tensile_cut_off CutOff_Value [DPxsi]` | `DPAdjust` — the DP-cone-to-MC-hexagon fitting mode: `0`=external edges, `1`=internal edges, `2`=plane strain, `3`=elastic (associated?), `-1`=intermediate (user-defined, only case where the trailing `DPxsi` field is present, range `[-1.5, 1.0]`). Then friction angle φ, dilatancy angle ψ, cohesion c, a tension-cutoff flag, and its value. |
| 1 (True MC only, appended) | `DilatancyCutOff eMax NonLocal [nonlocal-continuum block if NonLocal≠0]` | dilatancy-cutoff flag + max void ratio (mutually exclusive with `DPAdjust`/`DPxsi` in the GUI, but both field groups are still present in the record for this formulation), then a nonlocal/regularization enable flag and its data block |
| 2 | `GlobPhi.Lf GlobPsi.Lf GlobCoh.Lf CutOff_Value.Lf` | load-function refs |
| 3 | `GlobPhi.Sdata GlobPsi.Sdata GlobCoh.Sdata CutOff_Value.Sdata` | spatial-data refs |
| 4 (Drucker Prager only, ver≥13.08) | `GlobPhi.EvolFun GlobPsi.EvolFun GlobCoh.EvolFun CutOff_Value.EvolFun` | evolution-function refs |

Since `Drucker Prager` is a DP-cone fit to the MC hexagon, its `DPAdjust` field is the practically important one (which fitting scheme the cone uses is a real modeling choice); `True Mohr-Coulomb` implements the actual hexagonal criterion directly, so its own extra fields are about dilatancy limiting and nonlocal regularization instead.

**`NONL->` for `Multilaminate` / `Multilam. Menetrey` — fully field-mapped**: a **discrete-lamination-plane** model — instead of one global yield surface, up to 3 independent planes at fixed orientations, each able to yield/slip on its own. One continuous space-separated sequence, no embedded line breaks, of 3 plane records back to back:

```
<alpha1> <beta1> <phi1> <psi1> <coh1> <DefFlags1> <Ft1>  <alpha2> <beta2> ... (×3 planes total)
```

Per plane (7 fields): `alpha`/`beta` — the plane's orientation angles `[deg]`; `phi`/`psi`/`coh` — friction angle, dilatancy angle, cohesion; `DefFlags` — a bitmask, bit `1`=tension-cutoff active, bit `2`=this plane is active/enabled (only plane 1 is active by default); `Ft` — tension strength. **If the material's `AdvancedMLModel` setting is on** (matching the `"Multilam. Menetrey"` type string rather than plain `"Multilaminate"`) **a full Menetrey-family block is appended right after** the 21-field plane sequence — the same block documented next, letting a multilaminate material combine its discrete planes with one global continuum failure criterion.

**`NONL->` for `Mohr-Coulomb` / `Hoek-Brown` / `Rankine` / `Huber-Mises` (Menetrey-Willam family) — fully field-mapped.** One continuous space-separated line, 35 fields, no embedded line breaks:

```
Aphi Apsi fb_by_fc Fc Ft K Cutoffft Cutoffi1 Cutoffftflag Cutoffi1flag GlobCoh GlobPhi GlobPsi Sizeadj Soft Softa Softb Softwr Mentype Dpadjust Xsi PlasticFlow Af Bf Cf mf ef cf Ag Bg Cg mg eg TypeMaterial TypeAdj
```

Most of this record is **internal Menetrey-Willam failure-surface shape coefficients auto-derived from a handful of real inputs**, not independent user data — which fields are the meaningful inputs depends on which sub-model (`TypeMaterial`/`Mentype`) is active:

| Field(s) | Meaning |
|---|---|
| `Aphi`, `Apsi` | internal shape parameters derived from friction/dilatancy angle |
| `fb_by_fc` | biaxial-to-uniaxial compressive strength ratio (files saved with ZSoil <9.03 store `E` — Young's modulus — in this position instead) |
| `Fc`, `Ft` | compressive / tensile strength (meaningful for Hoek-Brown, Huber-Mises; Rankine uses `Ft` only) |
| `K` | deviatoric shape/roundness factor |
| `Cutoffft`, `Cutoffi1`, `Cutoffftflag`, `Cutoffi1flag` | tension cutoff and I₁ (mean-stress) cutoff values + their enable flags |
| `GlobCoh`, `GlobPhi`, `GlobPsi` | cohesion, friction angle, dilatancy angle (meaningful for Mohr-Coulomb) |
| `Sizeadj`, `Soft`, `Softa`, `Softb`, `Softwr` | size-adjustment flag, softening-enable flag, and softening curve parameters |
| `Mentype` | **which of the 4 sub-models this material actually is** — `0`=Huber-Mises, `1`=Drucker-Prager (a 5th Menetrey-internal option, distinct from the standalone `Drucker Prager` formulation above), `2`=Rankine, `3`=Mohr-Coulomb, `4`=Hoek-Brown |
| `Dpadjust`, `Xsi` | same DP-cone fitting mode / intermediate-mode parameter as standalone `Drucker Prager`'s `DPAdjust`/`DPxsi` (§4.3.7 above), reused here since some Menetrey sub-models are internally DP-based |
| `PlasticFlow` | flow-rule selector: `0`=deviatoric, `1`=Drucker-Prager, `2`=Rankine, `3`=tensile meridian, `4`=Hoek-Brown |
| `Af,Bf,Cf,mf,ef,cf` | Menetrey-Willam shape coefficients for the **failure** surface (auto-derived) |
| `Ag,Bg,Cg,mg,eg` | Menetrey-Willam shape coefficients for the **plastic-potential** surface (auto-derived) |
| `TypeMaterial` | the effective/active sub-model selector (same 5-value set as `Mentype` — the two aren't fully disambiguated from source alone; see `verification-notes.md`) |
| `TypeAdj` | adjustment-type selector, overlapping in role with `Dpadjust` |

**`NONL->` for `Concrete elastic plastic damage` (Lee-Fenves model) — fully field-mapped.** Starts with the same `FIRE_DATABASE <flag> <mode>` prefix as `Fiber Shell` (§4.3.7 above), then (ver≥17.50) a `UseSecantSitiffness` flag line, then a **24-field parameter array**, then the same 24 fields repeated 3 more times for `Lf`, `Sdata`, and (ver≥16.96) `EvolFun` references — every one of the 4 repetitions wrapped **10 fields per line**:

```
NONL-> FIRE_DATABASE 0 OFF
1
<24 values, 10 per line>
<24 Lf refs, 10 per line>
<24 Sdata refs, 10 per line>
<24 EvolFun refs, 10 per line>
```

The 24 parameters, in order (index 0–23), with GUI tooltip text quoted where the name alone isn't self-explanatory:

| # | Field | Notes |
|---|---|---|
| 0 | `eps_c1` | strain at peak on the compressive σ-ε curve |
| 1 | `fcm` | uniaxial compressive strength |
| 2 | `fc0_by_fcm` | initial (yield) compressive strength ratio `fc0/fc` |
| 3 | `sig_dam_by_fcm` | compressive damage-activation stress level |
| 4 | `fcbo_by_fco` | biaxial-to-uniaxial compressive strength ratio |
| 5 | `Gc` | compressive fracture energy |
| 6 | `sig_c_bar_by_fc` | reference stress level for compressive calibration |
| 7 | `Dc_bar` | damage value at that reference stress level |
| 8 | `c_estim_option` | compressive-parameter estimation procedure (combo: preserve `Gc`+`Dc_bar` / `Gc`+`eps_c1` / `eps_c1`+`Dc_bar`) |
| 9 | `ft` | uniaxial tensile strength |
| 10 | `Gt` | tensile fracture energy |
| 11 | `sig_t_bar_by_ft` | reference stress level for tensile calibration |
| 12 | `Dt_bar` | damage value at that reference stress level |
| 13 | `so` | stiffness-recovery factor |
| 14 | `dilatancy` | dilatancy type: `1`=constant, `2`=variable |
| 15 | `sig_dil_by_fcm` | compressive dilatancy-activation stress level |
| 16 | `alpha_po` | tensile dilatancy parameter |
| 17 | `alpha_p` | compressive dilatancy parameter |
| 18 | `epso_betaH` | yield-surface apex smoothing parameter |
| 19 | `vp_flag` | viscoplasticity enable flag |
| 20 | `viscosity` | viscous (relaxation-time) parameter |
| 21 | `lc_c` | characteristic length / sample size, compression |
| 22 | `lc_RC_flag` | "enforce characteristic length for RC structures" flag |
| 23 | `lc_RC` | that enforced characteristic length value |

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
The two fiber lines here are symmetric (`+0.4`/`-0.4`) rebar layers. Per fiber line `<Orientation> <offsetType> <yOffset> <zOffsetL> <zOffsetR> <alphaStart> <alphaEnd> <n_reinf_bars> <total_area> <prestress> <ReinforcementType> <componentId>`:
- `Orientation`: `0` = radial, `1` = circumferential — only meaningful for axisymmetric beams
- `offsetType`: `0` = "From top"; `1` = "From bot."; `2` = "From center"
- `yOffset` (3rd field, `±0.4`): the fiber's *relative*, vertical position within the section
- `zOffsetL`/`zOffsetR` (4th/5th fields, `0.1`): the left/right-most fiber's *relative*, horizontal position within the section
- `alphaStart`/`alphaEnd` (6th/7th fields, `0`/`180`): circular-section start/end angle in degrees — irrelevant (default `0°`/`180°`) for a straight rectangular beam like this example
- `n_reinf_bars` (8th field, `10`): number of bars in the fiber — no influence on the physics (only `total_area` matters), used for the section figure only
- `total_area`: total area of all `n_reinf_bars` in the fiber
- `prestress`: prestress in the fiber
- `ReinforcementType`: `0` = total area, `1` = density (area per unit width/length)
- `componentId`: references a `LAYERED_BEAM_COMPONENTS` entry by number (§4.4) — here `2` = "steel"

**Truss**: `GEOM-> <area>` — a single cross-sectional area value.

**Continuum**: `GEOM-> <value>` — a single value, unclear meaning.

**`Shell Layered`**: multi-fiber section definition —
```
 GEOM-> 1  10  2  0.005  0.4  2    0.005  -0.4  2    1  2  2
```
Fields (0-indexed after the `GEOM->` token itself is `v[0]`): `v[1]`=`ShearFactor`=1 (the shear correction factor itself — this **is** the "top-level shear correction factor" mentioned below, not a separate hidden field); `v[2]`=`nlayer`=10 (through-thickness numerical-integration layer count, default 10 — a real solver parameter, not an opaque constant); `v[3]`=`nFibers`=2; then per fiber `k`, `<area> <distance> <distanceFrom>` at `v[4+3k]`/`v[5+3k]`/`v[6+3k]` (fiber 1: area=`0.005`, distance=`0.4`, `distanceFrom`=`2`; fiber 2: area=`0.005`, distance=`-0.4`, `distanceFrom`=`2`); then `v[4+3·nFibers]`=`core_material`=`1` (the shell's core/matrix material id, resolved against the 2nd `NUM_MATERIALS=` pass, §4.4), followed by one material-id field per fiber at `v[5+3·nFibers+k]` (both fibers here reference material `2`) — a concrete core with steel reinforcement fibers symmetrically placed top and bottom.

`distanceFrom` meaning: the fiber position is a coordinate `ξ ∈ [-1, 1]` normalized to half-thickness (`y_physical = ξ · thickness/2`); `distanceFrom` selects how `distance` maps onto `ξ`:
- `distanceFrom=0`: `distance` is a fraction of *full* thickness measured inward from the top, `ξ = 1 - 2·distance`
- `distanceFrom=1`: mirror of `0`, measured from the bottom, `ξ = 2·distance - 1`
- `distanceFrom=2`: `ξ = distance` directly — already the normalized mid-plane-relative coordinate

A `"Shell Layered"` material's cross-section is a table of layers: `Core` for the base/matrix layer, `Reinforcement` for each fiber, each with an **area**, a **distance** (a value plus a from-top/from-bottom/relative choice, matching `distanceFrom` 0/1/2 above), and a **material** (by number, resolved against the 2nd `NUM_MATERIALS=` pass). There is also a top-level **shear correction factor** — this is `v[1]` above, the record's very first field.

**`Fiber Shell`** (2nd-pass layer material): `GEOM-> <Area_M> <vx> <vy> <vz>` — the leading field is **not a type code**, it's `Area_M` ("used only for membranes", default `1.0`), followed by a 3-component direction vector, e.g. `GEOM-> 1  1  0  0`.

**`Orthotropic shell`** (2nd-pass layer material): `GEOM-> <vx> <vy> <vz>` — no leading `Area_M` field, just the 3-component direction vector directly, e.g. `GEOM-> 1  0  0` (`vx=1, vy=0, vz=0`).

#### 4.3.3 `MAIN->` (beam/truss materials only)

Class-dependent, not one fixed shape: plain `Elastic Beam` writes a single field `BeamsType` (`0`=displacement formulation, `1`=flexibility formulation); `Truss/Cable` writes a single field `NonLinearGeomFlag`. The 6-field form sometimes seen (`MAIN-> 1  0  0  0  0  0` = `BbarModes, UStatesInc, NonLinear, Stab1, Stab2, Stab3`) belongs to a different, more generic material class (`ElasticStr`/"Elastic Structure"), not a beam or truss — if a real beam/truss sample shows 6 fields rather than 1, re-check which material class it actually is before trusting this mapping.

#### 4.3.4 `DENS->` — density/consolidation/permeability-adjacent parameters

4 lines. Line 1: `<UnitWeight> <WaterSpecWeight> <VoidRatio> <ForceMult> <MassMult> <UnitWeightDry>` — e.g. `20  10  0.6  1  1  16.7` (γ=20, γ_fluid=10, e0=0.6, mult=1, mult=1, γ_dry=16.7). Line 2: `UnitWeight.Lf WaterSpecWeight.Lf VoidRatio.Lf UnitWeightDry.Lf` (load-function refs). Line 3: `UnitWeight.Sdata WaterSpecWeight.Sdata VoidRatio.Sdata UnitWeightDry.Sdata` (spatial-data refs). Line 4: `calc_gamma_1ph_type calc_gamma_s calc_gamma calc_gamma_f calc_w` — specimen data for the GUI's Unit-Weight-Calculator tool (mode, γs, γ, γfluid, water content w); cosmetic/tool-state, not used by the solver.

#### 4.3.5 `FLOW->` — hydraulic/permeability parameters

Two distinct layouts, depending on material type:
- **Continuum**: a fixed 28-field record on the first line, `v[0]`–`v[27]` (0-indexed after the `FLOW->` token), plus 3 more lines. Field positions below are pinned down exactly (except where marked), by editing a live model to give each field a distinct value, saving, and comparing against the unmodified record:

  | Index | Field | Notes |
  |---|---|---|
  | `v0` | Darcy's coefficient **Kx** `[m/day]` | |
  | `v1` | Darcy's coefficient **Ky** `[m/day]` | equal to `v0` when "isotropic flow" is on |
  | `v2` | Darcy's coefficient **Kz** `[m/day]` | 3D only — not shown/editable in the 2D dialogs, hence equals the shared default whenever a 2D sample is inspected |
  | `v3` | Inclination angle `[deg]` | |
  | `v4`–`v6` | Orientation vector **m** (`mx,my,mz`) | default `1,0,0` — not flags |
  | `v7`–`v9` | Orientation vector **v** (`vx,vy,vz`) | default `0,1,0` — not flags |
  | `v10` | Fluid bulk modulus **Kf** `[kN/m²]` | |
  | `v11` | van Genuchten residual saturation ratio **Sᵣ** | |
  | `v12` | van Genuchten parameter related to the air entry suction **α** `[1/m]` | |
  | `v13` | hardcoded constant | always `1` — not user data |
  | `v14` | "Skip gravity term in Darcy's law" flag | `0`=off (default), `1`=on |
  | `v15` | "Undrained behavior" flag | |
  | `v16` | Undrained penalty factor **K^F/K** | |
  | `v17` | Suction pressure cut-off `[kN/m²]` | |
  | `v18` | "Isotropic flow" flag | unrelated to Bishop despite sitting inside this range |
  | `v19` | "Include air compressibility" flag | |
  | `v20` | Air stiffness bulk modulus **Ka** `[kN/m²]` | |
  | `v21` | "Use Bishop's global settings" flag | |
  | `v22` | Bishop pore-pressure weighting-term selector | `0`=`S`, `1`=`Seff^(1/(n·m))` |
  | `v23` | Biot coefficient "Enable" flag | `0`=disabled (default), `1`=enabled |
  | `v24` | Biot: solid grains stiffness bulk modulus **Ks** `[kN/m²]` | |
  | `v25` | Biot: current value of **α** (elastic range) | |
  | `v26` | van Genuchten measure of the pore-size distribution **n** | |
  | `v27` | Permeability-function-for-unsaturated-medium selector | `0`=Irmay, `1`=Mualem |

  Example (defaults): `FLOW-> 1  1  1  0  1  0  0  0  1  0  2000000  0  20  1  0  1  1000000  100  1  1  100  1  1  0  1e+38  1  2  0`.
- **Shell**: a much shorter record, e.g. `FLOW-> 1  1  1  1  0  0  1  0  0  2  2e+06`. Fields, in order of appearance (exact token positions not independently verified, but the field set and meaning are known): a "fully permeable" flag, an "anisotropic flow in shell" flag, permeabilities **kx′**/**ky′** (only if anisotropic)/**kz′** `[m/s]`, an **orientation vector for the x′ axis** (`vx`,`vy`,`vz`, only meaningful if anisotropic), and a **van Genuchten soil-water-retention curve**: **residual saturation Sᵣ** and **parameter related to the air entry α** `[1/m]`.

Beam/truss materials have an effectively empty `FLOW->` (no values, just the tag).

#### 4.3.6 `CREEP->`

`<TypeCurve> <ExFcDev> <ExFcVol> <ADev> <BDev> <AVol> <BVol> <NLA> <NLB> <kappa> <sig0> <sigC> <Alpha_s> <Ko> ...` — e.g. `0  0  0  0  0  0  0  0  4  0  500  50  5  0  0  0  0` (`TypeCurve=0`=power-law creep, `NLB` default `4.0`, `sig0` default `500`, `sigC` default `50`, `Alpha_s` default `5.0`; identical across every inspected material since none have creep actually enabled via `BUTTONS=`). `TypeCurve` selects which further conditional fields follow (Norton/double-power, EC2:2008(-enhanced), anisotropic-swell, or fire-creep variants each add their own named trailing fields) — only the base 14-field shape is documented here.

#### 4.3.7 `NONL->` — nonlinear (plasticity) parameters

Empty (`NONL->` alone) for plain Continuum `Elastic`/Beam `Elastic Beam` materials (no plasticity model), and **entirely absent as an option** for `Orthotropic shell` (always linear-elastic, no `Non linear` checkbox in the GUI at all). Populated example confirmed for `Fiber Shell`:
```
 NONL-> FIRE_DATABASE 0 OFF
1000  500000  1  1  0.02  0.15  0.2
0  0  0  0
```
Line 1 is `<ft> <fc> <ft0/ft> <fc0/fc> <eps_y> <eps_u> <eps_e>`: **`ft`** (tension strength) then **`fc`** (compression strength) — e.g. concrete `1000`(ft)/`500000`(fc); steel `235000`(ft)/`235000`(fc), consistent with steel's symmetric tension/compression yield — then `ft0/ft`/`fc0/fc` (ratios locating the start of nonlinearity relative to `ft`/`fc`, both default `1.0`), and `eps_y`/`eps_u`/`eps_e` (yield/ultimate/elastic-limit strain, defaults `0.02`/`0.15`/`0.2` — matching this example exactly). `FIRE_DATABASE <flag> <mode>` prefixes the line and toggles fire-resistance design checks; `<mode>` is `Off` or `EC2:2008` (Eurocode 2 fire-design method); `<flag>` is `0` for `Off`. The trailing `0 0 0 0` line is 4 evolution-function references (one per line-1 strength/ratio field), unset in this example.

#### 4.3.8 `HEAT->`

`<HeatDilat> <Lambda> <HCapacity> <HeatSource> <HInf> <Heata> <Heattd> <HeatQbyR> <HeatTf> <FluidHeatDilat> [<Heatb> if ver≥16.98] ...` — e.g. `1e-05  207.36  2000  0  105000  1.5  0.2083333333333333  4000  20  0  1`: **`HeatDilat`** (solid thermal dilatancy `[1/°C]`), **`Lambda`** (thermal conductivity), **`HCapacity`** (heat capacity c*), **`HeatSource`** ("source term" flag), then `HInf`/`Heata`/`Heattd`/`HeatQbyR`/`HeatTf` (heat-source model parameters, meaningful only if `HeatSource` is on), **`FluidHeatDilat`** (fluid thermal dilatancy), and `Heatb`. Further version-gated fields follow (advection-term flag, fluid conductivity/capacity, and load/evolution-function refs for several of the above) — not detailed here.

#### 4.3.9 `HUMID->`

`<HumDilat> <Huma> <Humwl> <HumDl> <Humkappa>` — e.g. `0  0.05  0.75  0.031104  0`: **`HumDilat`** (hygral dilatancy — e.g. `0.01` in the beam example `HUMID-> 0.01`), **`Huma`** (moisture-conductivity parameter "a", default `0.05`), **`Humwl`** (W1 parameter, default `0.75`), **`HumDl`** (D1 diffusivity), **`Humkappa`** (coupling term `[m³/J]`, default `0`).

#### 4.3.10 `INIS->` — initial-state parameters

**Continuum**: `<Ko(x')> <Ko(z)> <angle> <mx> <my> <mz> <vx> <vy> <vz> <EvaluateKo> <KoCutOffFlag> <KoCutOff>` — e.g. `0.385  0.385  0  1  0  0  0  1  0  0  0  1`. Fields 1–3 are the **initial Ko (at-rest earth pressure) state**: **Ko(x′)**, **Ko(z)**, then the **inclination angle ⟨x′,x⟩** `[deg]` (here `0.385`, `0.385`, `0` — a 2D model with equal, isotropic Ko in both directions and no inclination). Fields 4–9 are the same `m`/`v` orientation-vector pair seen in `FLOW->` (§4.3.5), defaults `(1,0,0)`/`(0,1,0)`. Field 10 is `EvaluateKo` (ver≥16.04, auto-Ko-from-OCR flag), field 11 `KoCutOffFlag` (ver≥23.53, "use upper limit" flag), field 12 `KoCutOff` (ver≥24.08, the limit value) — here `0`, `0`, `1`.

Same field shape seen in a `HS-small strain stiffness` continuum material — consistent with the same Ko(x′)/Ko(z)/inclination convention by structural match, though not independently re-confirmed via the GUI for this specific formulation. Notably, a model using this `INIS->` line ran its dynamic step directly from this initial state with **no separate `*Geostatic`-type consolidation driver** beforehand (unlike numgeo, which needs one for its own stress-dependent-stiffness material) — consistent with (but not proof of) `INIS->`'s Ko values being applied as the starting stress state automatically, without a dedicated solve step. Worth a GUI-confirmation pass if this matters for a future model.

**Beam**: e.g. `INIS-> 1  1  0  1  0  0  0  1  0  0  0  1` — same 12-field shape as continuum, but the first 3 fields (Ko-related for continuum) are presumably not applicable/ignored for a beam; not independently confirmed. Remaining fields *(meaning unclear)*.

#### 4.3.11 `STAB->`

`<StabType> <SL_start> <SL_end> <SL_inc> <tgPhi_start> <tgPhi_end> <tgPhi_inc> <c_start> <c_end> <c_inc>` — e.g. `2  0  0  0  1  1.2  0.2  1  1.5  0.5` (continuum, non-trivial) vs `0  0  0  0  0  0  0  0  0  0` (beam, all zero). `StabType`: `0`=disabled, others select which of the three safety-factor ranges (stress-level `SL`, `tan(φ)`, cohesion `c`) drive the local-stability check; the three ranges are each `[start, end, increment]`. Active for continuum materials only.

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
Field-by-field for the parameter line (`v[0]`-indexed): `E`=`v[0]` (`2e+07` concrete, `2e+08` steel); `nu`=`v[1]` (`0.3` both); `g`=`v[2]`=`25` — unit weight `[kN/m³]`; it reads `25` for both concrete and steel here only because this field defaults to `25` and wasn't edited for either component in this sample, not because it's unused; `type`=`v[3]`=`1` (elastic-plastic, both); `ft`=`v[4]` (tension strength); `fc`=`v[5]` (compression strength); `reg_soft`=`v[6]`=`0`; `char_len`=`v[7]`=`0.005` (both); `E0_setup`=`v[8]`=`0`; `coupTC_soft`=`v[9]`=`0`; `creep_type`=`v[10]`=`-1` (both — creep disabled); `creep_A`=`v[11]`=`0`; `creep_B`=`v[12]`=`1` (both).

**`SIG_EPS_FUN <n>`**: user-defined stress-strain functions, referenced from `LAYERED_BEAM_COMPONENTS`/nonlinear materials by name. Two version-gated formats: files saved with ZSoil <14.03 use a legacy shape (`<nPts> <FunName>` then `nPts` lines of `<x> <y>`); ≥14.03 uses `<Number> <size> <FunName>` / `<shift> <scaleFactor> <FlagBitCode> <UnitX> <UnitY>` / then `size` lines of `<x> <y>`.

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
<shift> <scale> <?>
<time1> <value1>
<time2> <value2>
...                    <- nSteps [time,value] pairs
...                    <- repeated for each of the n functions
```

`number`, `nSteps`, `name` on the first line; then a **flags line, a bit-coded `CString`**: the only currently-meaningful bit is bit 1 (`FC_FLAG_SKIP_TIME_STEPS`) — `#1` means the solver interpolates the table at its own time increments rather than being forced to step through the LTF's own listed time points; `#0`/`No flags` is the opposite (solver increments are forced to match the LTF's own points). Format is `"No flags"` or `"#<bitmask>"`. The next line is `<shift> <scale> <FlagBitCode>`: `shift` is a time shift applied to the function's time axis, `scale` is a multiplier applied to all step values, and `FlagBitCode` (always `0` in every example seen) selects which of three auto-generated, time-shifted copies of this function a construction-stage re-application belongs to (`1`/`2`/`4` = first/second/third shift group; `0` = not a shifted copy). Then `nSteps` lines of `[time, value]` pairs defining the piecewise function. LF number `0` is implicit — a constant multiplier of `1` for the whole analysis, never defined explicitly in the file.

Example (trivial, `No flags` variant):
```
LOAD_FUN 1
1 1 No name
No flags
0 1 0
0 0
```
(here `nSteps=1`, giving one `[time,value]` pair: `time=0, value=0` — a trivial/placeholder function, unused since this example has no active nodal loads.)

Example (`#N` interpolation-flag variant, a real time-history table — 2001-point base-velocity signal for a dynamic analysis, truncated here):
```
LOAD_FUN 1
1 2001 horizontal base velocity
#1
0 1 0
0.000000 -1.25114008e-35
0.000100 -8.25345046e-09
...
0.200000 -1.25114008e-35
```

---

## 6. Geometry

### 6.1 Nodes (`.ing`)

The finite-element mesh nodes:

```
<id> <x> <y> <z> <flag>
```

One line per node, count taken from the header (§3.2, "number of nodes"). The trailing `<flag>` (always observed as `0`) *(meaning unclear)*.

**Hand-editing gotcha**: `<flag>` must be a plain integer literal (`0`), not a float (`0.000000000000e+00`) — ZSoil silently refuses to open a file where this field is float-formatted.

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

| Marker | Header count label | Notes |
|---|---|---|
| `.glk` | number of Local bases | Custom local coordinate systems. How `.glk` bases are referenced from elements is unconfirmed. |
| `.axs` | *(no direct header label)* | *(meaning unclear)* |
| `.apl` | number of auxiliary planes | **Not purely a GUI construction plane** — see below; a populated `.pbc` (§8.3) needs a matching `.apl` entry. |
| `.igl` | number of auxiliary points | Auxiliary sketch points, distinct from `.pob`. |
| `.isd` | number data super elements | Reusable geometry/mesh templates. |

**`.apl` is functionally required by periodic BCs (`.pbc`, §8.3)**, not
just a sketch aid — a `.pbc` block ties nodes across a plane, and that
plane's geometry is defined here. Populated example (one plane, the one a
`.pbc` block above ties nodes across):

```
.apl
1
1 No name
3 1.000000e+00 0.000000e+00 0.000000e+00 -5.000000e-01
5.000000e-01 0.000000e+00 0.000000e+00
```

Format: `<count>`, then per plane: `<idx> <name>` / `<type> <nx> <ny> <nz>
<d>` (plane equation `nx·x + ny·y + nz·z + d = 0`; `<type>` meaning
unclear, `3` in every example seen) / `<px> <py> <pz>` (a point on the
plane — sanity-checks against the equation above). The `(nx,ny,nz,d)` here
matches a `.pbc` block's own plane-definition line numerically (§8.3); no
confirmed index field ties the two records together explicitly, but they
are created together by the GUI whenever a periodic BC is defined.

For the other markers in this section, if a hand-edit needs to touch them, populate them by building the corresponding geometry in the ZSoil GUI on a minimal test file and diffing the resulting `.inp`, rather than authoring them from this reference alone.

---

## 7. Elements

This chapter documents the element-definition blocks of the `.inp` file. Each block is introduced by a dot-prefixed marker and is either count-terminated (the count comes from the global header-count block, §3.2) or blank-line-terminated.

A recurring pattern across nearly every element record is:

```
<idx> <number> <TAG> <node1> ... <nodeN> <splitPar...> <mat1> <mat2> <mat3> <EF> <ULF> <iLayer>
```

where `<idx>` is a **global counter shared across every element block in the file**, incrementing continuously in file order regardless of type. `<number>` is a **per-block-type counter that restarts at 1** for each new element section. `<idx>` — not `<number>` — is what other records use to cross-reference an element (e.g. `.ics`/`.icg` paired-element fields, `.anh` truss references). When hand-inserting elements, assign fresh `<idx>` values past the current file-wide maximum.

The trailing block is uniform across element types: `<splitPar...>` is one mesh-subdivision count per local axis (1 value for `T3`, 2 for `Q4`, 3 for `B8`/`W6`, `1`=unsplit; absent — 0 values — for beams/trusses/shells), then `<mat1> <mat2> <mat3>` are **`InitialMaterial`**, **`ReplacementMaterial1`**, **`ReplacementMaterial2`** — a material-replacement mechanism. `mat1`/`InitialMaterial` is the element's material for the rest of the analysis unless replaced; `mat2`/`mat3` are `0` unless the element is set up to swap to a different material (e.g. at a construction stage or other trigger — the exact replacement-trigger mechanism isn't confirmed here). `<EF>` the existence function, `<ULF>` an **unloading-function** reference — a `LOAD_FUN` entry number, same list `LF` draws from, just used in the unloading role rather than the loading one (§2.5, and same concept as `.inb`'s `ULF` field, §8.1) — and the record always ends in `<iLayer>`, the GUI's construction-layer grouping index (unrelated to `EF`/staging) — the layer names it indexes into are listed in `.ily`, §14.1.

### 7.1 `.i0g` — Volumic/continuum elements

Count-terminated: the record count equals `nVolumics3D + nVolumics2D` (header fields "number of continuum 3D elements" / "number of continuum 2D elements"). One line per element. Type tag in field 3 selects the node count and the position of the trailing field group.

Node indices follow the tag, then the `splitPar` mesh-subdivision counts (2 values for `Q4`, 3 for `B8`), then `mat1 mat2 mat3 EF ULF iLayer` (§7 intro).

**Q4 (4-node quad, 2D)**:
```
.i0g
1 1 Q4 1 2 3 4 1 1 1 0 0 0 0 0
```
Field breakdown (0-indexed after split): `v0`=1 (idx), `v1`=1 (element number), `v2`=`Q4`, `v3..v6`=nodes 1,2,3,4, `v7..v8`=1,1 (`splitPar`, unsplit), `v9`=1 (`mat1`), `v10`=0 (`mat2`), `v11`=0 (`mat3`), `v12`=0 (`EF`), `v13`=0 (`ULF`), `v14`=0 (`iLayer`).

**Node order / face numbering**: `v3..v6` must be listed counter-clockwise (positive-area shoelace sum) — a clockwise or self-intersecting listing produces a degenerate/inverted element. Local face (edge) numbering follows directly from this order: **face `k` is the edge from `node_k` to `node_{k+1}`** (1-indexed, wrapping — face 1 = n1→n2, ..., face 4 = n4→n1). This is what `.ics`/`.icg` "paired-elem face" fields (§7.8, §7.9) and `.gsl` `UNI_LOAD` face references (§9.3) mean.

**B8 (8-node hex)**:
```
.i0g
1 1 B8 1 2 4 3 5 6 8 7 1 1 1 1 0 0 0 0 0
```
`v0`=1, `v1`=1, `v2`=`B8`, `v3..v10`=nodes (1,2,4,3,5,6,8,7), `v11..v13`=1,1,1 (`splitPar`, unsplit — 3 values for a 3D element vs. `Q4`'s 2), `v14`=1 (`mat1`), `v15`=0 (`mat2`), `v16`=0 (`mat3`), `v17`=0 (`EF`), `v18`=0 (`ULF`), `v19`=0 (`iLayer`). This layout is identical for the EAS and B-bar formulation variants — the formulation choice is not encoded in this record.

**B8 face numbering** — relevant wherever a `faceId` targets a `B8` element (`.gsl` `UNI_LOAD`, §9.3; `.ple` `POINT_LOAD`/`SURF_LOAD` with `targetType 8`, §9.4). Local nodes 1–4 form one quad face and 5–8 the opposite quad face, connected edge-to-edge (1–5, 2–6, 3–7, 4–8); the 6 face ids are assigned so that **opposite faces sum to 7**:

| Face id | Local nodes |
|---|---|
| 1 | 1-4-3-2 |
| 2 | 1-2-6-5 |
| 3 | 1-5-8-4 |
| 4 | 2-3-7-6 |
| 5 | 3-4-8-7 |
| 6 | 5-6-7-8 |

**T3 (3-node tri, 2D)**:
```
.i0g
1 1 T3 2 1 3 1 1 0 0 0 0 0
2 2 T3 1 4 3 1 1 0 0 0 0 0
```
`v0`=1, `v1`=1, `v2`=`T3`, `v3..v5`=nodes (2,1,3), then `v6..v12`=`1 1 0 0 0 0 0` (7 trailing fields vs. 8 for `Q4`/`B8` — `T3` has only 1 `splitPar` value): `v6`=1 (`splitPar`, unsplit), `v7`=1 (`mat1`), `v8`=0 (`mat2`), `v9`=0 (`mat3`), `v10`=0 (`EF`), `v11`=0 (`ULF`), `v12`=0 (`iLayer`).

A `W6` (6-node wedge/prism) tag exists, with the same layout pattern as `B8` (nodes then `1 1 1` splitPar then `mat1 mat2 mat3 EF ULF iLayer`):
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
Line 2: `<centrGloLoc> <centr1[x,y,z]> <centr2[x,y,z]>` — a cross-section **centroid-offset** mechanism for eccentric/rigid-offset beam connections: `centrGloLoc` selects global vs. local coordinates for the two offset vectors, `centr1`/`centr2` are the centroid offset at node1/node2. Zero (as here) means no eccentricity.
Line 3: `<dirDef> <dir1[x,y,z]> <dir2[x,y,z]>` — the beam's local-axis orientation vectors at each node (`dirDef` selects by-vector vs. by-point definition).
Line 4: ` 11 0 0 2 0 0` → `mat1`=11 (`v0`), `mat2`=0 (`v1`), `mat3`=0 (`v2`), `EF`=2 (`v3`), `ULF`=0 (`v4`), `iLayer`=0 (`v5`).
Line 5: end-release/hinge flag, `0` typically; if `1`, two additional (undocumented) lines follow.

### 7.3 `.itg` — Truss elements

Count-terminated: count = "number of truss elements (*.itg),(*.itm)". Each record is `TRS2` (plain truss) or `LNK2` (anchor/link) — the tag is chosen automatically: `LNK2` if either end is linked/embedded into surrounding elements, `TRS2` otherwise.

**No prestress**:
```
.itg
126 1 TRS2 158 157 0 0 0 0
 13 0 0 1 0 0
0
```
Line 1: `<idx> <number> TRS2 <node1> <node2> <SizeAt0> <SizeAt1> <AttachHexa0> <AttachHexa1>` — `SizeAt[0]`/`SizeAt[1]` are the linked-element counts at node1/node2 (`0` here, hence `TRS2` not `LNK2`); `AttachHexa[0]`/`AttachHexa[1]` are the per-end embedment type: `0`=none, `1`=embedded in continuum, `2`=embedded in structural elements.
Line 2 (material line): ` 13 0 0 1 0 0` → `mat1`=13 (`v0`), `mat2`=0 (`v1`), `mat3`=0 (`v2`), `EF`=1 (`v3`), `ULF`=0 (`v4`), `iLayer`=0 (`v5`).
Line 3: prestress record count = `0` → no prestress lines follow.

**With prestress**:
```
.itg
126 1 TRS2 158 157 0 0 0 0
 13 0 0 1 0 0
1
1.000000000000e+02 0 2  1
```
Prestress count = `1`, followed by one record `<value> <LF> <EF> <DefWay>` = `1.000000000000e+02 0 2  1` → prestress `value`=100, `LF`=0 (no load-function-driven ramp — this one genuinely is a `LOAD_FUN` reference), `EF`=2 (existence function gating the prestress), `DefWay`=1 (prestress defined by force; `0`=by stress).

### 7.4 `.iff` — Fixed Anchor Zones

Header count field: `"# number of Fixed Anchor Zones"`. Despite a non-zero header count in anchor models, the `.iff` block itself is typically completely blank (zero records). (The "Fixed anchor zone interface" text that does appear in some files is a material-catalog entry name, unrelated to this marker.) No populated `.iff` record syntax is available.

### 7.5 Shell elements: `.ilg` + `.ilt` (thickness) + `.ish` (hinges)

**`.ilg`** — count-terminated. The count that governs this block is **not** the generic header field `"number of shell elements (*.ilg),(*.ilm)"` (typically `0` even in files with real shell elements) but the separate field `"number of Shell one layer elements"`. Hand-editors adding/removing `SXQ4`/`SHQ4` records must update this second count field, not the first.

Type tags: `SXQ4` (4-node thin/one-layer shell) and `SHQ4` (8-node thick shell). Trailing block: `splitPar` (2 values), `mat1 mat2 mat3 EF ULF`, then a shell-specific `thick` field (thickness-table index, resolved against `.ilt`), then the usual trailing `iLayer`.

```
.ilg
2 1 SXQ4 1 5 6 2 1 1 1 2 2 2 3 1 1 0
```
`v0`=2 (idx), `v1`=1 (number), `v2`=`SXQ4`, `v3..v6`=nodes (1,5,6,2), `v7..v9`=1,1,1 (`splitPar`, unsplit — same pattern as `B8`'s three extra ints), `v10`=2 (`mat1`), `v11`=2 (`mat2`), `v12`=2 (`mat3`), `v13`=3 (`EF`), `v14`=1 (`ULF`), `v15`=1 (thickness-table index → `.ilt` record 1), `v16`=0 (`iLayer`). No `SHQ4` example is available.

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

**`.ish`** (hinges) is typically blank/empty; no populated example is available. It's written by the same manager as `.gsh` (§7.17) — one line of shell-hinge (`RelaxationShell`) records — but its own populated record layout isn't documented here.

### 7.6 `.ipg` — Seepage elements

Count-terminated: count = `"number of water Seepage's (*.ipg) (*.ipm)"`, typically `0`. Expected record layout, by analogy with the confirmed pattern for other element types (§7 intro):
```
<idx> <number> SPL2 <elem> <face> <node1> <node2> <mat1> <mat2> <mat3> <EF> <ULF> <iLayer>
```
each followed by one skipped line — but no real example is available to confirm this.

### 7.7 `.ivg` — Convection elements

Header count field `"# number of convection element (*.ivg)"`, typically `0`. No populated example is available.

### 7.8 `.icg` — Contacts on continuum (volumic-volumic interfaces)

Count-terminated: count = `"number of contact lines (*.icg), (*.icm)"`. Both `C_L2` (2D line contact) and `C_Q4` (3D quad-face contact) tags exist. Interface `type` codes (first field of the trailing data line): `0` = continuity with pressure, `1` = contact, `2` = continuity without pressure. An element can be excluded from ZSoil's automatic contact/interface generation entirely — see `.eie`, §13.3.

**`C_L2` (2D)**:
```
.icg
1 1 C_L2 2 1 1 3
 5 8 4 7 1 1 0
0.000000e+00 0
No name
1 2 0 0 0.000000e+00
```
Line 1: `<idx> <number> C_L2 <elem1> <face1> <OppEleNum> <OppFaceNum>` — `elem1`/`face1` is an `(element idx, local face)` pair (face numbering per §7.1); `OppEleNum`/`OppFaceNum` name the element+face on the opposite side of the contact (`0` if none defined).
Line 2: 4-node connectivity ` 5 8 4 7` (the two coincident node pairs across the interface), followed by `<genFullContinuity> <numSet> <iLayer>` = `1 1 0` — `genFullContinuity` is a continuity-override flag (setter semantics not confirmed), `numSet` is the **count of trailing type/mat/EF/ULF records** that follow (see the multi-record example below). Node order: `<elem1_faceNode_k+1> <elem1_faceNode_k> <elem2_faceNode_k> <elem2_faceNode_k+1>` — `elem1`'s face-node pair listed **reversed**, `elem2`'s **forward** (opposite winding, matching the two faces' opposite outward normals).
Line 3: `<InitialGap> <InitialGapEF>` = `0.000000e+00 0` — the interface's initial-gap distance and its existence function.
Line 4: interface name (`No name`).
Line 5 (repeated `numSet` times): `<type> <mat> <EF> <ULF> <initialGapNotUsed>` = `1 2 0 0 0.000000e+00` → type=1 (contact), mat=2, EF=0, ULF=0 (unloading function — a `LOAD_FUN` reference in the unloading role, §2.5/§7 intro), trailing=0.0 (a legacy field superseded by line 3's `InitialGap`, literally named `initialGapNotUsed` in source).

**Note**: `type`=2 ("continuity without pressure") also appears as a plain rigid tie between two independently-numbered, geometrically-coincident mesh regions that don't share node IDs (not just at staging/material boundaries) — a duplicated-position node pair with different IDs isn't necessarily a meshing error; check for a `.icg` tie before assuming one. When splitting/refining a tied element, update the tie's `elem`/`face` reference to the new sub-element, and check for *other*, unrelated ties on the same element's other edges too.

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
Line 1: `<idx> <number> C_Q4 <cnt-elem> <cnt-face> <iLayer> <nsides> <activeFlg>` → `cnt-elem`=2/`cnt-face`=1 (the contact's own defining element+face), `iLayer`=0, `nsides`=1 (single-sided; `2`=double-sided), `activeFlg` bitmask (`1`=positive side active, `2`=negative side active, `3`=both — here `2`).
Line 2: interface name. Line 3: `<paired-elem> <paired-face>` = `1 3` (shell element 1, face 3). Line 4: 8-node connectivity. Line 5–6: *(unclear)*. Line 7: `<?> <mat> <EF> <ULF> <trailing float>` = `1 3 3 0 0.000000e+00` → mat=3, EF=3, ULF=0 (unloading function — a `LOAD_FUN` reference in the unloading role).
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
Same 7-line-per-record structure as the `C_Q4` case: `<beam-elem> <side-idx> <iLayer> <nsides> <activeFlg>` header (here beam element 482, `iLayer`=0, `nsides`=1, `activeFlg`=2), name, a paired-elem/face line, 4-node connectivity, two skipped lines, and a final `<?> <mat> <EF> <ULF> <trailing>` line. `nsides`: `1` = single-sided (one `<paired-elem face>` block, as above); `2` = double-sided — a beam embedded in soil on both faces (e.g. a wall) gets **two** `<paired-elem face> / <connectivity> / <skip> / <skip> / <data>` blocks back to back, one per face, sharing the one header/name pair.

**Double-sided example** (one beam segment, contacted on both faces — two blocks back to back, sharing one header/name pair):
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
The two sides use opposite node order relative to the beam's node1→node2 direction (one reversed, one forward), matching the `.icg` reversed/forward convention (§7.8).

**Node reuse**: where two beam elements share an endpoint, ZSoil does **not** reuse one soil-side contact node across both beams' contact records — each segment gets its own, separately-numbered soil-side node at that shared point (per side). Allocate a fresh coincident node rather than reusing an adjacent segment's.

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
Header line: `<idx> <number> PILE <EF> <splitType> <SegLen> <MinSegLen> ...` = `12 1 PILE 3 8 1.000000 0.250000 20 0 1 1 126 0 1 2` → `EF`=3 (`v3`), `splitType`=8 (`v4` — despite the numeric value looking like a segment count, this is a subdivision-*method* selector: `0`=fixed segment count, `1`=by `SegLen`, `2`=automatic-equal, `3`=automatic-adaptive); `SegLen`=1.0 (`v5`), `MinSegLen`=0.25 (`v6`) — target and minimum axis-subdivision segment lengths (not diameter/perimeter); remaining fields include per-end link-attachment flags and the Qs/Qp model toggle below.
Material line: ` 11 12 0 13 0` → `PILE_MAT`=11 (`v0`), `INTERF_MAT`=12 (`v1`, skin-friction/shaft material, i.e. `qsmat`), `v2`=0, `NODE_INTERF_MAT`=13 (`v3`, tip/node-interface material, i.e. `qpmat`), `v4`=0.
Points: `nSeg+1`=9 lines tracing the pile axis, `<x> <y> <z> <nEle> <eleIdx...>` — not flags: each point's trailing data is the **count and global element numbers of the continuum (`.i0g`) elements it's found embedded in**, for soil-coupling stiffness assembly (endpoints typically embed in 1 element, interior points can span 2 where they sit on an element boundary).

**Qs/Qp material selection**: the tip-resistance-model toggle lives in the header line's `v9` field and the material line's `NODE_INTERF_MAT` (`qpmat`) field: `v9`=0 (Qstanphi model, `qpmat`=0) vs `v9`=1 (constant Qs/Qp model, `qpmat` populated).

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

**Axis-point trailing fields**: each axis-point line's trailing `<count> <id...>` is **not** node indices (despite resembling the `.pil`/`.nil` point-flag pattern) — it is `<count of .i0g continuum elements> <element idx...>` identifying which continuum element(s) the fixed-anchor-zone point falls in: `1 <id>` inside element `<id>`, `2 <id1> <id2>` on the shared edge between two elements — presumably how ZSoil distributes the bond-length load transfer into the soil mesh. Consequence: if an anchor's fixed zone is moved without remeshing, these references go stale and must be recomputed against the current `.i0g` mesh.

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
Structure per entry: a name line, a flag line whose last field is the control-point count (`1 1 1 2` → 2 points; `1 1 2 3` → 3 points), then that many `<relative-position 0..1> <offset>` point lines describing the cable profile along the host beam/truss (a straight profile for entry 1; a parabolic sag profile for entry 2). **The trailing `0 0` line's origin is uncertain** — the geometry-catalog writer itself appends nothing after its last entry, so this may belong to a different, per-instance-usage record (`CableSets`/`BeamCableDef`) rather than `.bcb` proper; treat it as present-but-unexplained rather than assuming it's part of the geometry-catalog entry shape shown above.

**`.bcl`** — typically blank; likely a duct/tendon-loss-coefficient catalog analogous to `.bcb`, but unconfirmed.

### 7.16 Heat exchangers (`.hex`, `.hef`), Groups (`.grp`)

Typically blank/`0`. `.hex`/`.hef` header: `"# number of Heat exchangers"`. No populated examples are available for any of the three.

### 7.17 `.gsh` — Shell-hinge opposite-element cache

For **every** one-layer shell element in the model (not just ones with a hinge defined). Per shell face, it precomputes which continuum (`.i0g`) elements sit on the opposite side of that face **and don't already have an explicit interface/contact** (`.icg`/`.ics`) defined — a cache the hinge/relaxation mechanism (and possibly the contact system generally) uses to know what's directly attached without re-searching the mesh.

The leading count is the number of one-layer shells (`"number of Shell one layer elements"`, the same field that drives `.ilg`, §7.5) — **not** the number of lines/records that follow, since each shell contributes one header+data block per relevant face:

```
<nShells>
<shellGlobalNumber> <nOpp>          <- per shell face
<OppEleNum> <OppFaceNum>            <- nOpp lines (0 or more), the continuum elements on the
...                                    opposite side of this face without an existing contact
...                                  <- repeated per face, per shell
```

A block is written for every shell face regardless of `nOpp` — `nOpp=0` is a valid, common case (a header line with no following data lines), which is what the large-model example below shows throughout.

---

## 8. Boundary Conditions

### 8.1 Nodal boundary conditions (`.inb`)

**`.inb` is not solely solid/displacement fixity data** it's a per-node dump of *every* kind of nodal condition the model has: solid-DOF fixity (below), plus, in the same file, water/heat/humidity nodal BCs and fluxes (§8.1's "Embedded field conditions" subsection). Count from the header ("number of boundary conditions") covers all of it combined.

**Solid/displacement fixity records**: one line per DOF-group. The record is `<idx> <nodeId> <flag>` followed by **one fixity block per translational DOF** — `[fixedFlag, prescribedValue, EF, LF, ULF]` — and ends with a **local-basis number**: `0` = the model's global Cartesian axes; any other value `N` = the `.ilb` record numbered `N` (see "Local basis definitions" below) — **this is a real index, not a binary flag**.

Within each block, **EF** (token 3) is an Existence Function reference: `0` = none (ordinary constant fixity); otherwise the fixity is only actually *enforced* while that EF's interval is active — outside it, the DOF is effectively free even though `fixedFlag` still reads `1`. **LF** (token 4) is a Load Function reference: `0` = none, `prescribedValue` stays constant; otherwise the prescribed value follows that function's time history instead. **ULF** (token 5) is an unloading-function reference — a `LOAD_FUN` entry number in the unloading role rather than the `LF` loading role (§2.5), same concept as every element/material record's trailing `ULF` field (§7 intro); always `0` in every example seen so far. Confirmed field-for-field from the writer above.

- **3D** (block width 5, 19 tokens total, flags at token indices 3, 8, 13): `<idx> <nodeId> <flag> [xFixed xVal EF LF ULF] [yFixed yVal EF LF ULF] [zFixed zVal EF LF ULF] <basisNum>`.

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

**`<flag>` selects the BC *type* (displacement/velocity/acceleration, translation/rotation), and a node can carry more than one record.** The full code set: `1`=translation lock (displacement), `2`=rotation lock (displacement), `4`=translation velocity, `5`=rotation velocity, `6`=translation acceleration, `7`=rotation acceleration. A node can have **multiple `.inb` records**, one per type, all sharing the same `<nodeId>` — and when two records both prescribe the *same DOF*, **the more dynamic type takes priority** (acceleration over velocity over displacement) for that DOF, even though the lower-priority record's `fixedFlag` still reads `1`.

This is the mechanism a base-excitation BC uses in practice: a `flag=1` record fixes a node's DOFs at a constant value (typically 0), and a second `flag=4` or `flag=6` record for the *same node* then overrides one of those DOFs with a `LOAD_FUN`-driven time history, leaving the other DOF(s) governed by the first record. Example — base node 1, `Uy` fixed at 0 (`flag=1` record), `Ux` driven by `LOAD_FUN 1`'s velocity time history scaled ×1.0 (`flag=4` record):
```
.inb
1 1 1 1 0.000000000000e+00 0 0 0 1 0.000000000000e+00 0 0 0 0 0.000000000000e+00 0 0 0 0
2 1 4 1 1.000000000000e+00 0 1 0 0 0.000000000000e+00 0 0 0 0 0.000000000000e+00 0 0 0 0
```
In the `flag=4` record's X-DOF block, the `LF` field (`1`) references `LOAD_FUN 1`; `prescribedValue` (`1.0`) is the scale multiplier applied to that function's tabulated values (same `prescribedValue`-as-`LF`-scale convention noted for `.inb` generally, §8.1 intro). The Y-DOF block in this record has `fixedFlag=0`, so it does not touch `Uy` — that stays governed by the first (`flag=1`) record.

**Local basis definitions (`.ilb`)**, referenced by the trailing local-basis number above: when nonzero, the fixity directions are expressed in the `.ilb` record with that number instead of the global axes. `.ilb` records come in 3 modes, selected by a `<id> <MODE>` header line:
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

**Embedded field conditions (types `3` and `8`)**: besides the solid-fixity `<flag>` values above (`1`/`2`/`4`/`5`/`6`/`7`), `.inb` also carries direct nodal water/heat/humidity **BC** (`<flag>=3`) and **flux** (`<flag>=8`) records, written by the same per-node pass right alongside the fixity records above — a completely different, fixed-shape record, not built from DOF blocks:

```
<idx> <nodeId> <3|8> <waterFlag> <waterVal> <waterEF> <waterLF> [<waterULF>] <heatFlag> <heatVal> <heatEF> <heatLF> [<heatULF>] <humidityFlag> <humidityVal> <humidityEF> <humidityLF> [<humidityULF>]
```

`<idx>`/`<nodeId>` as usual, then the type code, then three groups — water, heat, humidity, in that order, each independently present or absent via its own `flag`. **Confirmed against a real file** (`boxw1.inp`, `createdWithVersion` `9.05`): each group is `[flag, value, EF, LF]` — **4 fields, no `ULF`** — 15 tokens total (`4 3 3 1 -2.000000000000e+001 0 0 0 0.000000000000e+000 0 0 0 0.000000000000e+000 0 0`: node 3, water BC value `-20`, EF/LF `0`). Current source writes a 5th `ULF` field per group (18 tokens total); this sample's older format predates it, matching the version-gating pattern seen throughout this format — when hand-editing, match the width already used elsewhere in the same file. `<flag>=8` (flux) reuses the identical shape as `<flag>=3` (BC) — only the leading type code differs.

A node's *surface*-defined water/heat/humidity BCs (`.gwb`/`.gab`/`.gmb`, §8.4) get folded into this same `<flag>=3` mechanism once distributed onto individual nodes — confirmed directly in `boxw1.inp`: its `.gwb` block references nodes `3` and `6` with value `-20` each, and `.inb` records 4 and 8 are exactly `<flag>=3, node 3, val -20` / `<flag>=3, node 6, val -20`.

**`.iwb`/`.iab`/`.imb`/`.iwf`/`.ihf`/`.iuf` (§8.4) appear to be effectively unused in practice**, not genuinely duplicated data: in `boxw1.inp`/`boxw2.inp`/`boxw3.inp` (the only corpus files with any populated water BC), the header count for `.iwb` is `0` and the file itself is empty, even though `.gwb`+`.inb` carry the real, active water BC. The whole `zsoil_inp_files` corpus (~90 files) has no populated `.iwb`/`.iab`/`.imb`/`.iwf`/`.ihf`/`.iuf` anywhere, so this couldn't be fully ruled out either way — but the direct evidence points to `.inb`'s embedded mechanism being what the GUI's surface-BC tool actually populates, with the separate nodal-BC files an unused/legacy path in this corpus.

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

**Header line is `<idx> <type> <nNodes> <?> <?>`**, not just a pair count.
`<type>` is a **DOF bitmask** — the same encoding used elsewhere for
per-node DOF selection — not an opaque/fixed constant: `2`=Ux, `4`=Uy,
`8`=Uz, `16`/`32`/`64`=rotation about x/y/z, further bits for
pressure/temperature/humidity. `<nNodes>` is `2 × pairCount` (total tied
nodes, not pair count). The two trailing fields on this line are unclear
(always `0 0` in every example seen).

Then a line of numeric flags (11 zeros in the example below — purpose
unclear, but the line is present and must not be omitted), a **plane
definition** line (`<nx> <ny> <nz> <d>`, the tied plane's normal vector and
offset, satisfying `nx·x + ny·y + nz·z + d = 0`), a free-text name, then
`pairCount` lines of tied node-id pairs:

```
.pbc
<idx> <type> <nNodes> <?> <?>
<11 numeric flags, purpose unclear>
<nx> <ny> <nz> <d>
<name>
<nodeIdA1> <nodeIdB1>
...                     <- pairCount lines
```

Populated example — periodic tie linking the two side-columns of a 1D
shear-beam mesh at every row (base, top, and 4 intermediate rows), tying
all three translational DOF (`type=14=2+4+8=Ux+Uy+Uz`) across the plane
`x=0.5` (the column's mid-width):

```
.pbc
1 14 12 0 0
 0 0 0 0 0 0 0 0 0 0 0
1.000000e+00 0.000000e+00 0.000000e+00 -5.000000e-01
No name
1 2
4 3
5 6
7 8
9 10
11 12
```

The plane's `(nx, ny, nz, d)` here (`1, 0, 0, -0.5`) is numerically
identical to the corresponding `.apl` entry's own plane-definition line
(§6.5) — `.pbc` appears to carry its own copy of the plane geometry rather
than a numeric index into `.apl`, though the two are created together by
the GUI and share the same numbers; no confirmed cross-reference field
was found linking them by index.

### 8.4 Field (water/heat/humidity) boundary conditions

The following markers cover boundary conditions for seepage/thermal/humidity analyses. Each nodal-level (`i`-prefixed) marker has a **`g`-prefixed counterpart** — but, unlike `.ipg`/`.spg` etc. (§13.4), this is **not** a pre-mesh/subdomain-level vs. post-mesh distinction. Both tiers operate on the already-meshed model; the difference is *how* the BC is specified: the `i`-marker is a flat list of individually-defined per-node BC entries (same shape as `.inb`, one line per node — and, per the empirical check below, effectively never actually populated), while the `g`-marker is a **gradient/pattern BC** — up to 4 reference nodes each carrying a value, applied across a selected group of already-meshed element faces — the same general mechanism as `.gsl`'s `UNI_LOAD`/`GRAD_LOAD` (§9.3), just for boundary conditions instead of loads:

| Nodal marker (per-node list) | Header count label | Gradient/pattern counterpart |
|---|---|---|
| `.iwb` | number of water bound. cond. | `.gwb` |
| `.iph` | number of pressure Heads loads | — |
| `.iwh` | number of Surface load defined by Pressure head | — |
| `.iab` | number of Heat BC | `.gab` |
| `.imb` | number of Humidity BC | `.gmb` |
| `.iwf` | number of water flux | `.gwf` |
| `.ihf` | number of heat flux | `.ghf` |
| `.iuf` | number of humidity flux | `.guf` |

No populated example is available for the `i`-marker (nodal-list) tier. The `g`-marker (gradient/pattern) tier **is** confirmed, from `boxw1.inp`/`boxw2.inp`/`boxw3.inp`'s `.gwb`:
```
.gwb
1 VARIABLE 2 0 0 1
3 6 
No name
-2.000000000000e+001 
-2.000000000000e+001 
3 2
```
`<idx> VARIABLE <nnodes> <load_function> <exist_function> <?>` — here `nnodes`=2, `LF`=0, `EF`=0 (the trailing `1` doesn't match source's expected `unloading_function`/`siz` pair exactly — likely another instance of this file's older format predating a field added later, same pattern as `.inb`'s type-3 records above; not fully resolved). Then a line of `nnodes` reference node ids (`3 6`), a name line, `nnodes` value lines (both `-20` here — a uniform water-head value distributed via 2 equal reference points), and finally the face-selection this pattern applies to (`3 2` = element 3, face 2). See §8.1 for how this then gets folded into `.inb` as per-node type-`3` records.

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
(the local `faceId` numbering for `B8` volumic targets follows §7.1's B8 face-numbering convention; for a 2D `Q4` element, `faceId` `k` = the edge from the element's `node_k` to `node_{k+1}`, 1-indexed and wrapping — see §7.1.)

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

### 11.4 Initial nodal displacement/velocity (`.idv`)

One `NodalIniVelAcc` record per node:

```
<number> <nodeNum> <elemNum> <objType>
<lockU0> <valU0> <efU0> <lfU0>
<lockV0> <valV0> <efV0> <lfV0>
...                                <- repeated for DOFs 0-5 (6 DOFs, 12 data lines total)
<label>
```

Per DOF, two lines: `<lock?> <val?> <ef?> <lf?>` — same `[lock, value, EF, LF]` shape as `.inb`'s fixity blocks (§8.1), minus the `ULF` field. **Despite the class/marker name ("Initial Velocity/Acceleration"), the two per-DOF lines are initial *displacement* (`U`) then initial *velocity* (`V`)** — confirmed via the GUI dialog binding `valU`→`m_Dep` ("Dép", French for displacement) and `valV`→`m_Vel` (velocity); there is no separate acceleration field. Typically `0`/empty; no populated example is available to confirm field values in practice — relevant only for dynamic/seismic analyses that start from a non-zero initial displacement or velocity state.

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
<status_flag> <Orientation> <lengthType> <distL> <distR> <yposType> <yDist> <zOffsetL> <zOffsetR> <alphaStart> <alphaEnd> <diam> <nBars> <total_area> <prestress> <ReinforcementType> <material>
...                                                <- nLayers layer lines, repeated per set
<?> <nBeams> <?> <?>                              <- per reinforcement member
<reinfSetName>                                    <- name of the .brc set this member uses (matched by name, not id)
<beamId1> <beamId2> ... <beamIdN>                 <- nBeams beam element ids carrying this reinforcement
...                                                <- repeated per member
```

Per layer: `status_flag` (the real enabled/active flag — **not** the field near the end of the line, despite that being the more intuitive guess), `Orientation` (radial/circumferential, axisymmetric only), `length_type` (how the layer's extent along the beam is defined), `dist_l`/`dist_r` (left/right distances), `ypos_type`/`ydist` (position across the section), `zoffset_l`/`zoffset_r`, `alphaStart`/`alphaEnd` (circular-section angles, same as the layered-beam `GEOM->` fiber record, §4.3.2), `diam` (bar diameter), `nBars` (bar count), `total_area`, `prestress`, `ReinforcementType` (`0`=total area, `1`=density — not an enabled flag), and `material` (material id for the rebar itself, separate from the host beam's material). A reinforcement member links a named reinforcement set to a specific list of beam elements — i.e. the same rebar layout can be reused across many beams by reference name.

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

### 13.3 Elements excluded from automatic contact generation (`.eie`)

Not related to mesh tying itself — it's the list of elements flagged (via the GUI's "Exclude from contact" toggle) to be **skipped when ZSoil auto-generates interface/contact elements** (`.icg`/`.ics`, §7.8/§7.9) that would otherwise be placed on them automatically. Format:
```
<count>
<idx1> <idx2> ... <idx10>
<idx11> ... <idx20>
...                        <- global element indices (§7 intro's <idx>), 10 per line, across any
                              element type (continuum, shell, membrane, infinite, truss, beam)
```
`<count>` elements follow, wrapped at 10 per line; on read, each flags the corresponding element's `ExcludeFromContact` state. No populated example is available, but the read/write mechanism itself is fully confirmed from source.

### 13.4 Other structural-mesh markers

**`.ivd`, `.svd` — Viscous dampers.** Not initial conditions (see the correction note in §11.4) — these are dashpot/damper elements attached to element faces (same family as `.gsl`/`.ple`'s face-attached mechanisms), used for absorbing/"quiet" boundaries in dynamic analyses. `.ivd` covers already-meshed element faces (`SetOfFaces`/`SetOfEdges`), `.svd` the subdomain/macro-face equivalent before meshing (`SetOfSubDomainFaces`/`SetOfSubDomainEdges`) — the same i-prefix/s-prefix pairing as `.spg`/`.svg`/`.scg` below. No populated example available; typically empty.

**`.spg`, `.svg`, `.scg` — subdomain-level counterparts of already-documented element markers.** Confirmed via source: each is written by the *same* method as its plain counterpart, just called on `SetOfSubDomainFaces`/`SetOfSubDomainEdges` instead of `SetOfFaces`/`SetOfEdges` — i.e. these hold the macro/subdomain-level definition (before mesh generation) of the same mechanism, not separate features:
- `.spg` ↔ `.ipg` (Seepage elements, §7.6) — `writeSeePagesOn`.
- `.svg` ↔ `.ivg` (Convection elements, §7.7) — `writeConvectionsOn`.
- `.scg` ↔ `.icg` (Contacts on continuum, §7.8) — `writeSlidesOn`.

All three typically empty; record shape is expected to mirror the corresponding subdomain (`.sdm`, §6.4) face/edge referencing rather than the meshed-element one, but no populated example is available to confirm the exact fields.

**`.crc` — Contact on beam/truss elements (`ContactRC`).** A contact/interface defined directly on a beam or truss (`LinearElement`) parent, using the same `ContactParamList` structure (type/mat/EF/ULF) as `.icg`/`.ics`/`.cld` — distinct from `.ics`'s `C_L2` "paired with beam" form. **Notably write-only in the current reader**: the `.crc`-reading code in `readINP` is present but entirely commented out, so any `.crc` content is silently discarded when a file is reopened rather than round-tripped. No populated example is available.

**`.cld` — Node-to-face contact/mesh-tying (`ContactLD`).** A different mesh-tying mechanism from `.fac`/`.mrt` (§13.1): a `ContactLD` record ties a named group of "contactor" nodes (`ContactorNodes`, the tied/slave side) to a named group of "master" faces (`MasterFaces`), again via the same `ContactParamList` (type/mat/EF/ULF) pattern. The file bundles three sub-writers back to back — the contactor-node-group catalog (`SetOfContactors`), the master-face-group catalog (`SetOfMasters`), and the `ContactLD` tie records themselves (`SetOfContactLD`) — but the exact per-record field layout wasn't traced beyond this. No populated example is available.

**`.gos` — Geometrical surfaces.** **Corrects an earlier version of this doc**, which called this "write-only" — it's read as well as written, just typically empty in the sample corpus available. Populated syntax still not documented here.

**`.igl`** — see §6.5 (auxiliary points).

If a hand-edit needs to touch any of these, the most reliable path is still to reproduce the change in the ZSoil GUI on a minimal file and diff the result, rather than authoring from this reference.

---

## 14. Other Data

### 14.1 Construction/display layer names (`.ily`)

Despite sitting in the marker sequence next to the shell-related markers (§7.5), `.ily` is **not shell-specific at all** — it's the model-wide **construction/display layer name table**, i.e. the definitions that every element/material record's trailing `iLayer` field (§7 intro) indexes into:
```
<count>            <- number of named layers, excluding the implicit "Layer 0" (index 0, always
                      present in the GUI, never written here)
<layerName1>
<layerName2>
...                <- count lines, one per named layer (indices 1, 2, 3, ... — index 0 is "Layer 0")
```
`iLayer=0` on an element record means the default "Layer 0" (unnamed, always visible); `iLayer=N>0` indexes this list's `N`th entry (1-based). A `"Shell Layered"` material's core/reinforcement layers are unrelated — those are fully defined within that material's own `GEOM->` block (§4.3.2/§4.4).

---

## 15. Quick Reference

### 15.1 All 91 dot-prefixed markers, in file order

The same 91 markers, in the same order, appear in every v26 file (only their contents vary). "Doc §" is where each marker is documented; a dash means the marker is typically empty and could not be documented beyond its position in the sequence — treat these as confirmed-to-exist-but-unpopulated, not as unused/removable.

| # | Marker | Purpose | Doc § |
|---|---|---|---|
| 1 | `.icf` | Contact evolution function references | §16 (mention only) |
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
| 32 | `.ilb` | Local basis definitions (CARTESN_3D/VECTOR_3D/CYLINDR_3D), referenced by `.inb`'s trailing local-basis number | §8.1 |
| 33 | `.ihf` | Heat flux (BC) | §8.4 |
| 34 | `.iuf` | Humidity flux (BC) | §8.4 |
| 35 | `.ghf` | Heat flux, gradient/pattern form (↔ `.ihf`) | §8.4 |
| 36 | `.guf` | Humidity flux, gradient/pattern form (↔ `.iuf`) | §8.4 |
| 37 | `.gab` | Heat BC, gradient/pattern form (↔ `.iab`) | §8.4 |
| 38 | `.gmb` | Humidity BC, gradient/pattern form (↔ `.imb`) | §8.4 |
| 39 | `.gsl` | Surface loads (UNI_LOAD, GRAD_LOAD) | §9.3 |
| 40 | `.gwb` | Water BC, gradient/pattern form (↔ `.iwb`) | §8.4 |
| 41 | `.gwf` | Water flux, gradient/pattern form (↔ `.iwf`) | §8.4 |
| 42 | `.idg` | No writer found anywhere in source — see §2.2's stub-file note | §2.2 |
| 43 | `.isd` | Data super elements | §6.5 |
| 44 | `.ily` | Construction/display layer names (feeds `iLayer`, §7 intro — not shell-specific) | §14.1 |
| 45 | `.inm` | Nodal masses | §10.1 |
| 46 | `.iem` | Element masses | §10.2 |
| 47 | `.cld` | Node-to-face contact/mesh-tying (`ContactLD`) | §13.4 |
| 48 | `.igl` | Auxiliary points | §6.5 |
| 49 | `.axs` | Axes | §6.5 |
| 50 | `.pob` | Geometry points | §6.2 |
| 51 | `.gob` | Geometrical objects (GEOMLINE/ARC/PLINE) | §6.3 |
| 52 | `.sdm` | Subdomains | §6.4 |
| 53 | `.psd` | Subdomain-related | §6.4 |
| 54 | `.fac` | Face groups / mesh tying | §13.1 |
| 55 | `.mrt` | Mesh-tying related | §13.1 |
| 56 | `.idv` | Initial nodal displacement/velocity (despite the marker name, no acceleration field) | §11.4 |
| 57 | `.crc` | Contact on beam/truss elements (write-only — reader commented out) | §13.4 |
| 58 | `.spg` | Seepage elements, subdomain-level (↔ `.ipg`) | §13.4 |
| 59 | `.svg` | Convection elements, subdomain-level (↔ `.ivg`) | §13.4 |
| 60 | `.scg` | Contacts on continuum, subdomain-level (↔ `.icg`) | §13.4 |
| 61 | `.gos` | Geometrical surfaces | §13.4 |
| 62 | `.ish` | Shell hinges | §7.5 |
| 63 | `.ist` | Constant eps0 (initial strain) | §11.3 |
| 64 | `.goa` | Subdomain-related | §6.4 |
| 65 | `.sg0` | Subdomain-related | §6.4 |
| 66 | `.gnl` | Nodal links | §7.10 |
| 67 | `.pil` | Piles | §7.11 |
| 68 | `.eie` | Elements excluded from automatic contact generation | §13.3 |
| 69 | `.gbh` | Boreholes | §7.12 |
| 70 | `.ivd` | Viscous dampers (element faces) | §13.4 |
| 71 | `.svd` | Viscous dampers, subdomain-level | §13.4 |
| 72 | `.pbc` | Periodic boundary conditions | §8.3 |
| 73 | `.drz` | DRM domain, interior elements | §13.2 |
| 74 | `.dre` | DRM domain, exterior elements | §13.2 |
| 75 | `.nil` | Nails | §7.13 |
| 76 | `.anh` | Anchors | §7.14 |
| 77 | `.grp` | Groups | §7.16 |
| 78 | `.apl` | Auxiliary planes | §6.5 |
| 79 | `.gsh` | Shell-hinge opposite-element cache (not "general shell") | §7.17 |
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

### 15.2 Element type tags

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

### 15.3 Material sub-tags

`GEOM->`, `DENS->`, `FLOW->`, `CREEP->`, `NONL->`, `ELAS->`, `HEAT->`, `HUMID->`, `INIS->`, `STAB->`, `DISC->`, `DAMP->`, `MAIN->` — see §4.3 for full field-level detail per tag, and §4.2 for which formulations use which tags.

---

## 16. Worked Example

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
| 413–607 | `.ibg` through `.hex` — **81 markers, every one empty** | Every element/BC/load/geometry marker not otherwise used by this minimal model — each present as a bare marker line followed by a blank line (or `0`), per the "every marker always present" rule of §2.2. See [§15 Quick Reference](#15-quick-reference) for what each of these markers is for. |
| 609–616 | `.ple` / *(blank)* / `.pme` / *(blank)* / `.pth` / *(blank)* / `.hef` / *(blank)* | Final markers in the fixed sequence, all empty — end of file. |

**Takeaway**: even a maximally minimal model touches all 91 dot-markers — the file format doesn't omit unused sections, it just leaves them empty. When hand-editing an existing file, the safest strategy is exactly what this table demonstrates: locate the marker for the section you need by name, and edit only its block, leaving the surrounding empty markers untouched.
