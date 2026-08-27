# Verification notes for `inp-file-format.md`

Internal working notes: how each part of the reference was derived, so confidence levels and source material can be checked or extended later without re-deriving them from scratch. Not for end users — see `inp-file-format.md` for the actual format reference.

**Sources used throughout:**
- Sample `.inp` files, primarily the curated set in `Benchmarking\Unittests\zsoil_inp_files\` (~71 files), plus larger files under `v26\manual\` and `Benchmarking\Unittests\ZSoilPy3\tests\`.
- `zsoil_tools\zsoil_inp.py` — the Python parser (`read_inp()`), used to cross-check field order/meaning wherever it actually parses a given marker. It only reads ~25 of the 91 dot-markers; the rest are file-inspection-only.
- Interactive exploration of the ZSoil 2026 v26.03 GUI's Materials dialogs, against a live model at `C:\Mandats\M1366_Geodev\Q&A testing\Unittesting\testing\NL_beams_shells` (files: `NL_beam_traction.inp`, `NL_beam_traction_forceCtrl.inp`, `NL_shell_traction.inp`, `NL_shell_bending - Copy.inp`).
- A second interactive GUI session against a different live model, `2D_anchor_disp_s1m` at `C:\Mandats\M1366_Geodev\Q&A testing\Unittesting\testing\anchors`, covering the **Continuum** material formulation (material "Soil", `Elastic`) — see the dedicated note below.
- **ZSoil preprocessor source code** (2026-08, once the user made it available): `C:\Users\mprei\Perforce\GEODEV_MP-LNV-NOV25_2294\v26\ZSoil\Z_Prep3D\` (preprocessor GUI/geometry/element/BC code), `...\v26\ZSoil\zmate\` (material/analysis-control library), `...\v26\ZSoil\H\constant.h` (shared enums). This is the highest-confidence source used so far — actual `.inp`-writing C++, not inference from samples or GUI dialogs. See the dedicated section below for what it resolved/corrected.

## §4.3.1/§4.3.7 `HS-small strain stiffness` — ELAS-> and NONL-> fully field-mapped

User asked to refine the per-continuum-model `NONL->` layouts flagged as a gap in
`session-progress.md`, specifically HS-small-strain (not Drucker-Prager, which was in progress
when redirected). Found both writers directly in `zmate`:

- **`NONL->`**: `operator<<(fstream&, DataNonLinearHSSmallStrain&)` (`dataNonLinear.cpp:4361`).
  Real member names and GUI tooltip strings read straight from `DataNonLinearHSSmallStrain`'s
  declaration (`dataNonLinear.h:769`) and `::Default()` (`dataNonLinear.cpp:4062`) — e.g.
  `HS_E_50_ref.SetDouble(25000.0, UNIT_STRESS, "Demanded secant reference E modulus at 50% of
  qf")`, `HS_H`/`HS_M` explicitly commented `// cap` in the header, `HS_automatic_H_M_eval`
  defaulting to checked (`SetCheckBox(1, ...)`) — which is exactly why the earlier A/B test saw
  `H`/`M` change when `ELAS->`'s `m` changed: they're auto-estimated by default, not independent
  inputs. Version-gated tail (line 4's `KoSR_Setup`/`ApplyM1ForCapHardening`/
  `CutOffUndrainedShearStrength`, and the Lf/Sdata lines 5-6) transcribed directly from the
  `if (SaveAsVersionNumber >= X)` guards in the same function.
- **`ELAS->`**: `operator<<(fstream&, DataElasticHSSmallStrain&)` (`dataElastic.cpp:2182`),
  class declared `dataElastic.h:217`. Matches the doc's existing example
  (`80000 0.2 0.5 100 10 0 1 193766 0.0002 2 1 0 90 0 2 1.6`) field-for-field once mapped —
  confirms both fields the earlier session had only guessed at (`gamma_07`↔"plausibly gamma_0.7",
  `m`↔ the one A/B-confirmed field) and resolves everything else, including the ver≥23.5
  anisotropy tail (`Aniso_theta`/`Aniso_phi`/`Anisotropy`/`GhhByGvh`/`Beta` — the trailing `0 90 0
  2 1.6` in the example, self-consistent as two default orientation angles 0°/90° with anisotropy
  disabled).
- Not chased further: the exact enum values for `HS_stress_dependency`'s pre-19.02 vs. post-19.02
  encoding beyond what's quoted in the doc (both variants exist in source, only the older one was
  matched against the doc's own example since its version is `9.05`-era... actually this
  example's own file version wasn't re-checked against the ver≥12.15/19.02 gates — worth a
  follow-up spot-check against the real source `.inp` if precision on this one field matters).
## §4.3.7 `Drucker Prager` / `True Mohr-Coulomb` NONL-> — fully field-mapped

Follow-up to the HSS work above, per the user's explicit "continue with Drucker-Prager/True-MC".
Both share one data struct, `DataNonLinearDP_MC` (`dataNonLinear.h:522`, comment "// for Drucker
Prager and Mohr-Coulomb" right above it) — own fields `DPAdjust`/`DPxsi`/`NonLocal`, plus the
common `GlobPhi`/`GlobPsi`/`GlobCoh`/`tensile_cut_off`/`CutOff_Value` inherited from `DataMaterial`
(same base every other `NONL->`-adjacent struct in this codebase inherits from — `HS-small-strain`
's `DataMaterial&` cast is the same pattern).

- **`DataNonLinearDP_MC::operator<<`** (`dataNonLinear.cpp:2303`) — the shared `.inp` writer,
  used unmodified by `NonLinearGroupDruckerPrager::WriteToFile` (`dataNonLinear.cpp:2533`, a
  one-line delegate: `file << (DataNonLinearDP_MC&)(*Data)`).
- **`DPAdjust` enum** (`eexternal=0, einternal=1, eplaneStrain=2, eelastic=3, eintermediate=4`)
  found at `dataMaterial.h:133`; the writer remaps `eintermediate`(4) to literal `-1` on disk
  (`tempDPAdj = (DPAdjust.IntValue==4) ? -1 : DPAdjust.IntValue`) — the only case where the
  extra trailing `DPxsi` field is written. Combo-box label strings confirm the physical meaning:
  `IDS_ADJ_EXT_EDGES`/`IDS_ADJ_INT_EDGES`/`IDS_ADJ_PS`/`IDS_ADJ_ELAS`/`IDS_ADJ_INTER`.
- **`NonLinearGroupTrueMC::WriteToFile`** (`dataNonLinear.cpp:2731`) does **not** call the shared
  `operator<<` — it inlines the same first few fields manually, then appends
  `DilatancyCutOff`/`eMax`/`NonLocal` (+ a `DataNonlocalContinuum` block if `NonLocal` is set)
  before the same `Lf`/`Sdata` tail (no `EvolFun` tail observed for True-MC in what was read,
  unlike Drucker-Prager's, though `WriteDataEvolFun` exists as a separate method — not chased
  further whether/where it's actually invoked).
- Confirms `DataNonLinearDP_MC::GetAllFields(..., bool dpMode)`'s branch (`dpMode` ? push
  `DPAdjust`/`DPxsi` : push `DilatancyCutOff`/`eMax`) — i.e. the GUI only lets you edit one set
  depending on formulation, but the file always has room for both.
- **Not resolved**: `NonLinearGroupTrueMC::GiveGroupNlinesCode()`'s actual return value (would
  confirm total line count precisely) wasn't read; `WriteDataEvolFun`'s call site for True-MC
  wasn't traced.

## §4.3.7 `Multilaminate` / `Multilam. Menetrey` NONL-> — fully field-mapped

User asked to check the multilaminate model's `NONL->` group next. Two classes:
`DataMultiLaminate` (`dataNonLinear.h:221`, one lamination plane) and `DataNonLinearMLam`
(`dataNonLinear.h:244`, holds `ML1, ML2, ML3` — three `DataMultiLaminate` planes — plus
`AdvancedMLModel`, commented `// DP, MC, HB etc`).

- **Per-plane writer**: `operator<<(fstream&, DataMultiLaminate&)` (`dataNonLinear.cpp:536`) —
  `alpha, beta, phi, psi, coh, DefFlags, Ft`, 7 fields, confirmed against `Default()`
  (`dataNonLinear.cpp:448`) which sets the `ValueName`/unit-type strings for each (`alpha`/`beta`
  are `ANGLE_DEG`, matching "orientation angles"). `DefFlags` bit meanings from the `#define
  ML_CODE_CUTOFF_FT 1` / `ML_CODE_PLANE_ACTIVE 2` macros right above the class (`dataNonLinear.h:
  218-219`); confirmed only plane 1 defaults active (`ML1.DefFlags = ML_CODE_PLANE_ACTIVE` in
  `DataNonLinearMLam::Default`, `dataNonLinear.cpp:606`, planes 2/3 left at the all-zero default).
- **Three-plane writer**: `operator<<(fstream&, DataNonLinearMLam&)` (`dataNonLinear.cpp:647`) —
  `file << dat.ML1 SP; file << dat.ML2 SP; file << dat.ML3 SP;`, no `"\n"` anywhere in either this
  or the per-plane operator — confirmed `SP` is literally `#define SP << "  "`
  (`dataNonLinear.cpp:61`, two spaces), not a newline macro as it might read at a glance. So all
  21 fields (3 planes × 7) are one continuous space-separated run, structurally speaking (exact
  line-wrapping in a real file not checked against a sample).
- **`AdvancedMLModel` isn't a field in either operator<<** — found where it actually shows up:
  `NonLinearGroupMLam::WriteToFile` (`dataNonLinear.cpp:795`) writes the `DataNonLinearMLam` block
  unconditionally, then — `if (Data->AdvancedMLModel) file << (DataNonLinearMenetrey&)(*Data);` —
  conditionally appends a *full Menetrey block* right after. So the "combined" mode isn't a flag
  in the record at all; it shows up as extra trailing data, and (per `MATERIAL.CPP:438`, `char*
  name[] = { "Multilaminate","Multilam. Menetrey" };`) is signaled to the reader via the
  material's own `<type>` string on the `MATERIAL` record itself (§4.1), not anything inside
  `NONL->`.
- **Reuses the still-open Menetrey block**: the appended block is the same
  `DataNonLinearMenetrey` structure used by `Mohr Coulomb`/`Hoek Brown`/`Rankine`/`Huber Mises`
  (`NonLinearGroupMenetrey::WriteToFile`, `dataNonLinear.cpp:5620`, `file << (DataNonLinearMenetrey&)
  (*Data)`) — this was the investigation in progress before being redirected to HSS, then
  Drucker-Prager/True-MC. Its own field-by-field layout is still not mapped; `NonLinearGroupMenetrey
  ::WriteData` (`dataNonLinear.cpp:5483`, a *different*, `.dat`-oriented function, not the `.inp`
  writer — same distinction as elsewhere in this codebase) gives a partial field list by name
  (`GlobCoh`/`GlobPhi`/`Ft`/`Fc`/`E`/`PlasticFlow`/`GlobPsi`/`Sizeadj`/`Cutoffft(i1)`/`Softwr`/
  `Softa`/`Softb`) but the *actual* `.inp`-writing `operator<<(fstream&, DataNonLinearMenetrey&)`
  hasn't been located/read yet — that's the concrete next step if this family is picked up again.
- Not resolved: exact line-wrapping of the 21-field plane sequence in a real file (no populated
  multilaminate sample found in the `zsoil_inp_files` corpus to check against).

## §4.3.7 Menetrey-Willam family (Mohr-Coulomb/Hoek-Brown/Rankine/Huber-Mises) and Concrete Plastic Damage — fully field-mapped

Direct continuation, per explicit user request ("Mohr-coulomb, concrete plastic damage and
hoek-brown need the same treatment") of the Menetrey thread left open in the Multilaminate round.

- **Found the actual class** — it's not in `dataNonLinear.h`/`.cpp` at all, it's in a dedicated
  file: `MenetreyParam.h`/`.cpp`. `DataNonLinearMenetrey : public MenetreyBaseData`
  (`MenetreyParam.h:51`) adds no new data members of its own — all 35 fields live in
  `MenetreyBaseData` (`MenetreyParam.h:15`). The actual `.inp` writer is
  `operator<<(fstream&, DataNonLinearMenetrey&)` (`MenetreyParam.cpp:893`) — confirmed via the
  read-side `operator>>` (`MenetreyParam.cpp:918`) matching field-for-field, including the
  version-gated `fb_by_fc`-vs-`E` swap at the 3rd field (`if (p.GiveVersion() < 9.03f) ... dat.E =
  Tmp ... else dat.fb_by_fc = Tmp`).
- **`Mentype`/`TypeMaterial` enum**: `enum { MenHuberMises = 0, MenDruckerPrager, MenRankine,
  MenMohrCoulomb, MenHoekBrown, MENETREY_MAX }` (`dataMaterial.h:135`). **`PlasticFlow`** enum:
  `enum { pfDeviatoric = 0, pfDruckerPrager, pfRankine, pfTensileMeridian, pfHoekBrown }`
  (`dataMaterial.h:134`).
- **Confirmed the outer `<type>` string is genuinely shared** (not just an earlier guess): all
  three of `Mohr_Coulomb`/`Hoek_Brown`/`Rankine`'s `MatProperty` entries (`MATERIAL.CPP:417-436`)
  pass the literal `_T("Menetrey")` as the type string and `_T("PLAS_ME_V")` as the DAT code —
  the friendly names ("Mohr-Coulomb (M-W)" etc.) are a separate trailing constructor argument
  used only for the GUI catalog, confirmed never written to `.inp`. So the sub-model is entirely
  determined by `NONL->`'s own `Mentype` field, not by anything in the `MATERIAL` record's own
  header lines (§4.1).
- **Not resolved**: whether `Mentype` and `TypeMaterial` (both present, both apparently using the
  same 5-value enum) always agree, or serve genuinely different purposes (e.g. one is the
  originally-configured type, the other the currently-effective one after some remapping) —
  presented in the doc as two separate fields without asserting they're redundant.
- **Concrete Plastic Damage**: `DataNonLinearConcreteDamagePlastic` (`dataNonLinear.h:1242`,
  comment `// for Concrete elastic plastic damage (Lee-Fenves)` / `// CDPM_1_V` right above it) —
  a flat `UNIT_field CDPM_par[CDPM_NONL_MAX]` array (24 entries) with a fully-named index enum
  right there in the header. Writer: `operator<<(fstream&, DataNonLinearConcreteDamagePlastic&)`
  (`dataNonLinear.cpp:7216`) — genuinely wraps at 10 fields/line (`if ((i+1)%10==0) file <<
  "\n"`), the only NONL-> writer seen in this whole investigation that does real line-wrapping
  rather than one continuous space-separated run. Field descriptions quoted verbatim from
  `Default()`'s `SetDouble(..., "description")`/`SetCheckBox(..., "description")` calls
  (`dataNonLinear.cpp:7027`). Two `MatProperty` entries exist (`MATERIAL.CPP:311`, `:493`) — one
  `GROUP_ADDITIONAL` labeled "for shell" (likely a 2nd-`NUM_MATERIALS=`-pass layer-material
  option alongside `Fiber Shell`/`Orthotropic shell`, §4.4) and one `GROUP_CONTINUUM` — both share
  the same `NonLinearGroupConcreteDamage`/DAT code `CDPM_1_V`. The shell variant's relationship to
  the existing `Fiber Shell`/`Orthotropic shell` §4.4 mechanism wasn't chased further — worth a
  follow-up if that specific formulation matters.

## §4.3.7 Duncan-Chang / Cap Model / Cam-Clay / Hujeux — fully field-mapped

Closes out the remaining `NONL->` gap list, per explicit user request. All four classes are
declared in `dataNonLinear.h`, writers in `dataNonLinear.cpp`:

- **Duncan-Chang**: `DataNonLinearDuncanChang` (`dataNonLinear.h`, near line 500),
  `operator<<` at `dataNonLinear.cpp:2031`. No `SetDouble(..., "description")` tooltip strings
  found in `Default()` (`dataNonLinear.cpp:1979`) for `G`/`F`/`d` specifically — their physical
  meaning (`ν = G − F·log₁₀(σ₃/Pa)`) is standard Kulhawy-Duncan literature knowledge applied to
  the field names, not confirmed via an in-source description the way most other fields in this
  investigation have been. Flagged accordingly in the doc rather than stated as source-certain.
- **Cap Model**: `DataNonLinearCapModel` (`dataNonLinear.h:584`), `operator<<` at
  `dataNonLinear.cpp:3089`. `Default()` (`dataNonLinear.cpp:~2880`) has rich `SetDouble`/
  `SetComboBox` tooltip strings for every field that *is* written. Confirmed `SigVM`/`KoNC`/
  `DefDirectly` (all declared on the class) are absent from `operator<<` entirely — cross-checked
  by re-reading the full write statement, not just skimming — so they're GUI-only, feeding into
  `NLPco`'s value via some derivation not itself persisted.
- **Cam-Clay**: `DataNonLinearCamClay` (`dataNonLinear.h:640`), `operator<<` at
  `dataNonLinear.cpp:3500`. Simplest of the four — no version gates on the writer side.
- **Hujeux**: `DataNonLinearHujeux` (`dataNonLinear.h:698`), `operator<<` at
  `dataNonLinear.cpp:3798`. Confirmed the `Lf`/`Sdata` lines are dead code — the exact same
  fields are still declared as a `friend operator<<` and referenced in a `//`-commented-out block
  right above the real write statements (`dataNonLinear.cpp:3801-3802`), and the real function
  body never reaches them. The 10 internal mechanism coefficients (`am/ac/b/cm/d/cc/rkel/rkhys/
  rkmbl/r4el`) have real default values in `Default()` but no descriptive strings — same honesty
  flag as Duncan-Chang's `G/F/d`, but here there wasn't even a standard-literature mapping
  attempted since Hujeux's own notation already matches the source field names directly.
- Not resolved for any of the four: whether `Pa` in NONL-> and any independently-set `Pa`
  elsewhere in the material record are actually the same underlying reference pressure or happen
  to share a name — not chased, low priority.

## Corpus spot-check (`zsoil_inp_files`) — resolved the `.inb`/`.iwb` open question, plus sanity checks

User asked to check the real sample corpus at
`C:\Users\mprei\Perforce\GEODEV_MP-LNV-NOV25_2294\Benchmarking\Unittests\zsoil_inp_files\`
(~90 files) directly, on the heels of the `.inb` rescan below. Findings:

- **Scanned all ~90 files for populated `.iwb`/`.iab`/`.imb`/`.iwf`/`.ihf`/`.iuf`/`.gwb`/`.gab`/
  `.gmb`**: only `.gwb` is ever populated, in exactly `boxw1.inp`/`boxw2.inp`/`boxw3.inp`. Every
  `.iwb`-family file is blank in all ~90 files, including those same three.
- **`boxw1.inp` settles the `.inb`/`.iwb` open question empirically** (as much as this corpus
  can): its `.gwb` block (`1 VARIABLE 2 0 0 1` / `3 6` / `No name` / `-20` / `-20` / `3 2`) —
  reference nodes 3 and 6, value −20 each, applied to element 3 face 2 — has its values show up
  verbatim in `.inb` as two type-`3` records (`4 3 3 1 -2.0e+01 0 0 0 ...` and `8 6 3 1 -2.0e+01 0
  0 0 ...`, nodes 3 and 6, value −20). Meanwhile `.iwb`'s own header count is `0` and the file is
  empty. This is real evidence (not proof for every possible authoring path) that `.inb`'s
  embedded type-3 mechanism is what the GUI's surface-BC tool actually populates, and `.iwb`
  itself is unused in this corpus — added to §8.1/§8.4 accordingly, phrased as "appears to be
  effectively unused" rather than a categorical claim, since only one authoring path was observed.
- **Field-count correction from the same record**: source's current `writeNodalBCOn`/
  `SurfaceBC::writeGeomOn` write a 5-field `[flag,value,EF,LF,ULF]` group per quantity (water/
  heat/humidity) and a 6-field `.gwb` header line — but `boxw1.inp` (`createdWithVersion 9.05`)
  shows only 4 fields per group (no `ULF`) and 4 header fields (not 5) on `.gwb`'s first line.
  Consistent with the version-gating pattern seen everywhere else in this format; documented both
  the current-source shape and the observed older-file shape, flagged as version-dependent rather
  than picking one as "the" answer.
- **`.gwb`'s exact middle two header fields** (position after `nnodes`, expected
  `load_function exist_function` per source) aren't fully pinned down against the observed `0 0
  1` — plausibly `LF=0, EF=0`, then either `unloading_function` or `siz` for the trailing `1`, but
  which one wasn't resolved. Flagged as unresolved rather than guessed in the doc.
- **Bonus sanity check**: `.idg` confirmed blank in `boxw1.inp`/`boxd1.inp`/`pile.inp` — consistent
  with the "no writer, blanket-stub-created" finding from the earlier `.idg` investigation.

## §8.1 `.inb` full rescan (user asked to re-verify — section had grown large)

Traced the entire write chain from `NodesManager::writeBoundaryConditionsOn`
(`nodemngr.cpp:494`) → `GeomPoint::writeNthNodalCondition` (`GeomPoint.cpp:2426`), which calls,
in order, into the *same open `.inb` file*: `writeSolidBoundaryConditionOn` →
`BoundaryCondition::writeBCOn` (`boundarycondition.cpp:369`, the fixity records already
documented), then `writeNodalBCOn` (`GeomPoint.cpp:2283`, type `3` — nodal water/heat/humidity
BC), then `writeNodalFluxOn` (`GeomPoint.cpp:2353`, type `8` — nodal water/heat/humidity flux),
then a final loop converting `SurfToNodBCList` entries (i.e. `.gwb`/`.gab`/`.gmb` surface BCs
distributed onto individual nodes, §8.4) into more type-`3` records. This was previously entirely
undocumented — the doc only described the fixity (type `1`/`2`/`4`/`5`/`6`/`7`) records.

- **`writeBCOn` re-read carefully**: confirms the `[lock, valU, efU, lfU, ulfU]` 5-field-per-DOF
  shape exactly as already documented, *and* resolves the trailing field precisely —
  `LocBaseNum = IsLocalDefined ? locBas.number : 0`. This is a real `.ilb` record number, not a
  binary 0/1 flag as the doc previously claimed (a leftover from before the field's exact
  semantics were traced this carefully). Corrected throughout §8.1.
- **Type `3`/`8` records**: field-exact from `GeomPoint::writeNodalBCOn`/`writeNodalFluxOn`'s
  literal `fprintf` format strings — `"%d %d 3 %d %16.12le %d %d %d %d %16.12le %d %d %d %d
  %16.12le %d %d %d\n"` (and the same shape with `8` for flux). 18 tokens, three
  `[flag,value,EF,LF,ULF]` groups (water/heat/humidity) after `<idx> <nodeId> <type>`.
- **Open, not resolved**: `.iwb`/`.iab`/`.imb`/`.iwf`/`.ihf`/`.iuf` (§8.4) are written via a
  *different* code path (`NodalObjManager::writeOn`, `NodalObjManager.CPP:242`, the generic
  `NodalObj`/`writeElementCmpOn` pattern shared with `.idv`) from what looks like the same
  underlying `NodalBC` objects (`GeomPoint::nodalWaterBc` etc.) that `writeNodalBCOn` also writes
  into `.inb` as type-3 records. Confirmed **both** writers are live (both called unconditionally
  from `geometricalmodel.cpp`'s `writeOn`) and **both** readers are live
  (`GetNodalSolidBCManager()->readSoildBCFrom(..., "inb", ...)` at line 596 and
  `GetNodalFluidBCManager()->readFrom(..., "iwb", ...)` at line 603, both unconditional). Did not
  trace far enough to determine whether this is genuine on-disk duplication of the same BC, two
  BCs that happen to look similar, or one path silently wins over the other at read time — a real
  open question, not a gap in searching. If picked up again: compare a real file's `.inb` type-3
  entries against its `.iwb` entries for the same node to settle it empirically, since source
  alone left it ambiguous.

## §2.2/§15.1 `.idg` — genuinely no writer; found the assembly mechanism instead

Same targeted-grep approach as every marker in this batch, but this one came back **empty** —
zero hits for `"idg"` (case-insensitive, substring, whole tree) in either `Z_Prep3D` or `zmate`.
Sanity-checked against a real sample file (`Hotline\20230202 #47 GwangWook\Hwanghak
Tunnel5err.inp:44317`) to confirm `.idg` really exists at the documented position (it does,
empty, between `.gwf` and `.isd`) — so this wasn't a mis-transcribed marker name.

Explanation found by chasing the assembly step instead of a per-marker writer: recall from the
very first exploration round that `GeometricalModel::writeOn` (`Z_Prep3D`) writes most markers to
**separate per-extension fragment files** (`<basename>.xxx`), then calls `ZMATE_SaveINP` to
assemble the final `.inp` — at the time, `ZMATE_SaveINP` itself wasn't found in any available
source. It's now visible in `zmate\zmate.cpp:1462`, and it calls `ZMATE_CompressFiles`
(`zmate.cpp:998`), which does:
```cpp
for (int i = 0; i < numberOfFiles; i++)
    CompressFile(name, PrePro_Ext[i], Inpfile);
```
`PrePro_Ext[]`/`numberOfFiles` are populated once, at startup, by `InitPreProFiles`
(`zmate.cpp:394`) reading `<PathCFG>\CFG\prepro.fil` — **not present in this Perforce sync**
(searched the whole workspace, no hits), so the literal ordered list with its 91 entries and any
comments couldn't be read directly. But `ZMATE_CreateNoExistingPreproFile` (`zmate.cpp:930`) — run
before every save — settles the question regardless: it loops the same `PrePro_Ext[]` list and
`fstream file1(fileName, ios::out)`-creates (i.e. **truncates to empty**) any fragment file that
doesn't already exist. Since no code anywhere writes a `.idg` fragment, this stub-creation step is
the *only* thing that ever touches it — confirming `.idg` is a genuine always-empty placeholder
slot in the current codebase, not a gap in this search. Used the same finding to explain the
doc's pre-existing "every marker present, even empty" claim in §2.2 — now stated as a confirmed
mechanism (fixed external extension list + blanket stub-creation) rather than just an observed fact.

**Follow-up, done immediately**: the `.ghf`/`.guf`/`.gwf`/`.gmb`/`.gwb`/`.gab` cluster near §8.4 was
in fact fully resolved as a side effect of reading the `.idg` neighborhood
(`geometricalmodel.cpp:1256-1291`) — all six have confirmed writers: each pairs with an
already-documented nodal-level (`i`-prefixed) marker, e.g. `GetSurfHeatBCManager()->writeOn(name,
"gab")` sits right next to `GetNodalHeatBCManager()->writeOn(name, "iab")`.

**Correction (user caught it — "did you mean subdomain-level instead of surface-level?")**: the
first write-up of this pairing called it the same `g`/`i` = macro/subdomain-level-vs-meshed
pairing as `.ipg`/`.spg` (§13.4). That's wrong — checked `SurfBCManager.h`: `createVariableBC`/
`createGradientBC` take `std::vector<ElemPart*>& faces` (already-*meshed* faces, not pre-mesh
subdomain geometry) plus `GeomPoint** _nds` (up to 4 reference nodes, matching `SurfaceBC::values[4]`
in `SurfaceBC.h`). `SurfBCManager::writeOn` (`SurfBCManager.cpp:297`) casts its objects to
`SurfLoad*` and calls `writeGeomOn` — the *same* writer used for `.gsl`'s surface loads. So the
real distinction is: `i`-marker = flat per-node BC list (like `.inb`), `g`-marker = a
gradient/pattern BC (reference-node values distributed across a face selection), mechanically
the BC-equivalent of `.gsl`'s `UNI_LOAD`/`GRAD_LOAD` (§9.3) — not a mesh-timing tier at all. Both
operate on the same already-meshed model. Rewrote §8.4 and the six Quick Reference rows to say
this instead. `.ipg`/`.spg` etc. (§13.4) are unaffected by this correction — those genuinely are
`SetOfFaces` vs. `SetOfSubDomainFaces`, a real pre-mesh/post-mesh distinction, directly confirmed
and distinct from this `SurfBCManager` mechanism.

## §13.4 remaining markers (`.crc`,`.idv`,`.spg`,`.svg`,`.scg`,`.gos`,`.cld`) — found the writers, resolved by request

Same targeted-grep approach, applied to every marker still in the old "unconfirmed" table. All write call sites are in `geometricalmodel.cpp`'s `writeOn` (writer, ~line 1230-1381) and `readINP` (reader, ~line 480-620, 755-777).

- **`.spg`/`.svg`/`.scg`**: confirmed by direct code comparison — `.spg` calls the exact same `writeSeePagesOn` as `.ipg` (line 1297-1307) but on `GetSetOfSubDomainFaces()`/`GetSetOfSubDomainEdges()` instead of `GetSetOfFaces()`/`GetSetOfEdges()`; same pattern for `.svg`↔`.ivg` (`writeConvectionsOn`, 1313-1324) and `.scg`↔`.icg` (`writeSlidesOn`, 1349-1360). High confidence — literally the same function pointer, different receiver object.
- **`.gos`**: `GetGeomSurfManager()->writeOn(fname, "gos", SaveIcfVersionNumber)` (line 1235) AND `->readFrom(fname, "gos")` (line 481, inside `CreateObjTab()`'s read path) — both present, so the doc's prior "write-only" claim was simply wrong (an artifact of not having found the reader in the earlier pass, not a real asymmetry).
- **`.crc`**: writer at line 1375 (`GetContactRCManager()->writeOn(file)`, logged `"Contact RC"`), reader at line 761-768 but **the entire block is wrapped in `/* ... */`** — genuinely dead/disabled code, not just unused. `ContactRC : public T3DElement` (`ContactRC.h:17`) has `ContactParamList params` and `LinearElement *parent` — so it's a contact record hung directly off a beam/truss element, same param shape as `.icg`/`.ics`/`.cld`. Didn't chase what "RC" stands for beyond the class name itself (no comment found explaining the acronym) — described the mechanism, not the acronym, in the main doc.
- **`.cld`**: three sub-writers share the file — `SetOfContactors`, `SetOfMasters`, `SetOfContactLD` (lines 769-777 reader, matching writer not individually re-read but same triple pattern expected). `ContactLD : public GraphicalObject` (`ContactLD.h:18`) has `ContactParamList params`, `ContactorNodes *cnt`, `MasterFaces *mst` — a node-group-to-face-group tie, distinct mechanism from `.fac`/`.mrt`'s mesh tying (§13.1). Did not read `ContactorNodes.cpp`/`MasterFaces.cpp`/the three managers' actual `writeOn` bodies, so no field-level layout — mechanism-level only.
- **`.idv`**: `GetNodalSolidIniCondManager()->writeOn(fname, "idv")` (line 1246, right after `.inc`'s `NodalSolidBCManager::writeOn`) / `readFrom(fname, "idv", DeliveredVersion)` (line 612, logged `"Solid IC"`). Confirmed this manager holds `NodalIniVelAcc` objects specifically by finding `GeomPoint.h:60`'s `NodalIniVelAcc* nodalIniVelAcc` member and the dedicated `NodalIniVelAcc.h`/`.cpp`/`_Dlg.cpp` files.
  - **Follow-up (field-level, found on request)**: the record-level write/read functions aren't named `writeOn`/`readFrom` (those are inherited/no-ops here) but `writeElementCmpOn`/`readElementCmpFrom` (`NodalIniVelAcc.cpp:250,270`) — missed on the first pass because the grep for this only tried the generic names. Record: `<number> <ndNum> <elNum> <objType>` then, per DOF (6 total): `<lockU> <valU> <efU> <lfU>` / `<lockV> <valV> <efV> <lfV>`, then a trailing label line. **Which field is which physical quantity was the open question — resolved via `NodalIniVelAcc_Dlg.cpp:449-465`**: `dlg.m_Vel[i] = bc->valV[i]` and `dlg.m_Dep[i] = bc->valU[i]` — `V`=velocity, `U`=`m_Dep`="Dép(lacement)" (French, matching this codebase's other French-origin naming, e.g. `FCTCHARG.CPP`="fonction de chargement"). So despite the class/marker being named "IniVelAcc", the two stored quantities are **initial displacement and initial velocity**, not velocity and acceleration — there is no acceleration field anywhere in this class. This is a real, slightly misleading legacy name, not a documentation error on our part once traced to source.

**Correction discovered as a side effect, not directly asked about but too significant to leave wrong**: `.ivd`/`.svd` were documented in the (pre-this-session) doc as "Initial velocities/accelerations", filed under §11.4. That's wrong — `SetOfParts::writeViscDampOn` (`SetOfParts.cpp:3669`) writes `ElemPart`-attached `ViscDamp` objects (`ElemPart.h:317`, `BOOL hasViscDamp()`) — genuine viscous-damper elements on faces, an `ElementOnFace` family member (`ElemPart::writeNthElementOnFaceOn`, same generic dispatch used elsewhere for face-attached mechanisms), nothing to do with initial conditions. `.ivd` uses `SetOfFaces`/`SetOfEdges` (meshed elements), `.svd` uses `SetOfSubDomainFaces`/`SetOfSubDomainEdges` (subdomain/macro, same i/s-prefix pairing as `.spg`/`.svg`/`.scg`) — confirmed at `geometricalmodel.cpp:1329-1339`. Moved this content to §13.4 and rewrote §11.4 around `.idv` instead. Did not find `ViscDamp`'s own field-level write function, so §13.4's `.ivd`/`.svd` entry is mechanism-level only, same limitation as the others in this batch.

## §13.3 `.eie` — found the writer, resolved by request

Same targeted-grep approach as `.gsh`/`.ily`, for `.eie`/`"eie"`. One hit: `Z_Prep3D\geometricalmodel.cpp:1544/904`, both write and read call sites, each preceded by the comment `// excluded from interface element` and logged as `tc.SetLog("Excluded faces")`. Implementation at `geometricalmodel.cpp:2437` (`GeometricalModel::WriteExludedElemFromContact`) → `GetExludedElemFromContact(v)` (line 2422) collects flagged elements across every element-manager type (`QuadsManager`, `HexasManager`, `ShellsManager`, `Shells1LManager`, `MembranesManager`, `InfiniteElemManager`, `TrussesManager`, `BeamsManager`) into one list, then `writeElemList` (line 2463) writes `<count>` followed by global element numbers 10-per-line. `ReadExludedElemFromContact` (line 2447) reads the same shape and calls `element->SetExcludeFromContact(1)` per entry — confirming this is a GUI "Exclude from contact" toggle's persisted state, used to suppress automatic `.icg`/`.ics` interface generation on specific elements. Simple enough that no further reading was needed. Not investigated: the corresponding `ExludeSelectedElemFromContact`/`SetExcludeFromContact` setter side (how a user actually toggles this in the GUI), and whether a real populated `.eie` sample exists anywhere in the corpus to spot-check the write format against.

## §7.5/§7 intro `.ily` — found the writer, resolved by request

Same targeted-grep approach as `.gsh` below, this time for `.ily`/`"ily"`. Two hits: `Z_Prep3D\geometricalmodel.cpp:1577` (`layers->WriteTo(file)`, the writer) and `zmate\MakeDat.cpp:6463` (the `.dat`-compiler's reader, unrelated to `.inp` semantics). `layers` is a `Layers*` member of `GeometricalModel` — its class (`Layers.h`/`Layers.cpp`) is the GUI's construction/display-layer manager (`Add`/`Modify`/`Delete`/`Merge`, a `CurLayer` concept, visibility flags), **not** anything shell/fiber-related — the marker name is misleading, same class of trap as `.gsh`. `Layers::WriteTo` (`Layers.cpp:134`): `fprintf(file, "%d\n", siz-1)` then one line per layer name for `i=1..siz-1` — index 0 ("Layer 0") is pushed in the constructor and always present but deliberately never written. This is exactly the layer-name table that every element/material record's trailing `iLayer` field (found during the earlier six-agent pass, §7 intro) indexes into — confirms `iLayer` wasn't just a plausible guess, it's a real cross-reference into a real table. `ReadFrom` (`Layers.cpp:147`) confirms the same shape symmetrically. Not investigated: whether `VisFlg` (per-layer visibility) or anything beyond the name is ever written to `.ily` itself (`WriteTo` only writes the name — visibility isn't persisted here, consistent with `ReadFrom` defaulting every loaded layer's visibility to `1`).

## §7.17 `.gsh` — found the writer, resolved by request

The user asked directly whether a parser/writer for `.gsh` had been found — it hadn't been searched for specifically in the earlier six-agent pass. A targeted grep for `.gsh`/`"gsh"` across `Z_Prep3D` found exactly one hit: `geometricalmodel.cpp:1343`, `GetHingesManager()->writeAttachedElementsOn(name, "gsh")`, called immediately after `GetHingesManager()->writeOn(name, "ish")` (`.ish`, shell hinges) — i.e. `.gsh` is hinge-adjacent infrastructure, not literally "General shell" as the doc's old section title guessed from the marker name alone. Full implementation read at `ElemOnPartManager.cpp:776` (`HingesManager::writeAttachedElementsOn`): iterates every one-layer shell (`Shells1LManager::GetT3DElements`, not filtered to shells with an actual hinge object — the `attachedShells`/`M_SelBin` computed earlier in the function is dead code, never used), and for each shell face, finds opposite continuum elements via `HexaedronFace::GetOppositeContinuumElemFaces` and writes only those without an existing `hasSlide()` (i.e. no `.icg`/`.ics` contact already defined). This resolves the doc's old "inconsistent line count" confusion completely — the leading count is the shell count (confirmed, matches the doc's prior observation), but each shell contributes a variable number of lines (`1 + nOpp`) depending on how many uncontacted opposite elements its face(s) have, not a fixed 1 line/shell.

Not otherwise investigated: whether a one-layer shell can contribute more than one header+data block (if `GetHexaedronFaces()` returns more than one non-null face) — the large-model example's exact 1:1 shell-count-to-line-count match is consistent with 1 relevant face per shell in that particular model, but isn't proof the mechanism never emits more than one block per shell in general. `.ish`'s own record format (the shell-hinge/`RelaxationShell` objects themselves) also wasn't read in this pass — only its write call site was located.

## Source-code pass (Z_Prep3D / zmate / constant.h) — corrections and resolutions

Six parallel Explore agents read the real `.inp` writer/reader code (`GeometricalModel::writeOn`/`readINP` in `geometricalmodel.cpp`; the per-tag `operator<<` functions in `zmate`), then a direct read of `constant.h` resolved one more item. Full agent transcripts aren't retained, but every finding below was cross-checked against the doc's existing examples before being applied — field values in the doc's own quoted examples were re-derived under the new field mapping and confirmed self-consistent (notably `FLOW->`'s default example: `v18`=isotropic-flow=`1` now correctly explains why `v0=v1=v2` in that example, and `v24`=`Ks`=`1e+38` now makes physical sense as an inert/incompressible default, neither of which held together under the old field mapping).

**Corrections (the doc had these actively wrong, not just unconfirmed):**
- Every element/beam/truss/shell material-line's `rm1`/`rm2` fields don't exist — they're `mat2`/`mat3`, confirmed via `SingleT3DElement::setDefaultParameters()` (`singlet3delement.cpp:105`) and `constant.h`'s `eParType` enum (`T3D_PARAM_MAT1,MAT2,MAT3,EF,UNLFCT`). Per the user directly: `mat1`/`mat2`/`mat3` are `InitialMaterial`/`ReplacementMaterial1`/`ReplacementMaterial2` — a material-replacement mechanism, not generic "secondary/tertiary material slots" as first described. Exact replacement-trigger mechanism not otherwise confirmed in this pass.
- That same enum shows the field every such record calls **`LF`** is actually **`UNLFCT`** (unloading function) — still a `LOAD_FUN` entry number, but used in the unloading role rather than the loading role (per the user's correction: ULF and LF reference the *same* `LOAD_FUN` list, just different usages of an entry, not two separate function catalogs). Same concept as `.inb`'s already-identified `ULF` field. Applied throughout §7 and cross-referenced from §2.5/§8.1.
- B8 face-node table: source-exact from `Hexaedron8::createHexaedronFaces()` (`hexa8.cpp:425`) — face 1 is `1,4,3,2`, not `1,2,3,4` as previously written from visual/pattern inference; faces 3/5 no longer need an "inferred" hedge.
- `FLOW->` continuum: `v0/v1/v2` are `Kx/Ky/Kz` (not `kx/kz` with an unexplained duplicate); `v4-v9` are two 3-component orientation vectors `m`/`v`, not flags; **`v14`/`v27` were swapped** (v14 is really `SkipGravityTerm`, v27 is `krFunction` Irmay/Mualem); the `v18/v21/v22` Bishop trio is resolved and `v18` turns out to be `isotropicFlow`, unrelated to Bishop at all (`dataFlow.cpp:322`).
- `ELAS->`'s "3 unclear secondary lines" are load-function/spatial-data/evolution-function refs for E/nu, not anisotropy or damping data (`dataElastic.cpp:117,793`).
- `Fiber Shell`'s leading `GEOM->` field is `Area_M` ("used only for membranes"), not a type code (`dataGeometry.cpp:1298`, confirmed via the actual `.inp` writer `GenericGroup::WriteDataToFile`/`dataGeneric.cpp:152` — a separate `WriteData()` method on the same classes writes a different field order but is unused for `.inp`, a trap worth remembering if this file is revisited).
- `Shell Layered` `GEOM->`'s `v[1]`/`v[2]`: `v[1]` **is** the shear correction factor itself (previously an open question — thought to be missing from the record entirely); `v[2]` is `nlayer`, the integration-layer count, not a mysterious constant (`dataGeometry.cpp:895`).
- `.brc` per-layer record: the enabled/active flag is field 1 (`status_flag`), not field 16; field 16 is `ReinforcementType` (`Reinforcement_Sets.cpp:161`).
- `.pil` header `v5,v6`: `SplitDef::SegLen`/`MinSegLen` (target/min axis-segment lengths), not diameter/perimeter (`EmbeddedLinearElement.cpp:658`, `SplitDef.h:24`).
- `.itg` prestress record and the 4 "unclear" trailing fields on line 1: fully resolved via `trussload.cpp:109` and `truss.cpp:497-531` (`AttachHexa[]`/`SizeAt[]`, explains the `LNK2`/`TRS2` tag choice).
- `.ibg`'s "six always-zero floats" line: a real centroid-offset mechanism (`BarDirection::writeOn`, `BarDirection.cpp:1091`), not opaque.
- `.ics`'s "`<?> <?>`" pair before `nsides`/`side`: one field, `iLayer`, not two (`SlideStr.cpp:1041`).

**Gaps filled** (previously unconfirmed, now resolved from source — see `inp-file-format.md` itself for the resulting field tables, not repeated here): `BUTTONS=`'s full 14-flag mapping (`dataMaterial.cpp:62-75`); `CONTROL`'s fields 9-11 and second-line fields 5-10 (`DATAJOB.CPP:89`); `DYN_CONTROL`/`PSH_CONTROL` full field layouts (`DynCtrl.cpp`, `PshNode.cpp`); `NONL_GEOM`/`CONSTRUCTION`/`THM_SETTINGS`/`BISHOP_FLAG`/`PROJECT_PRESELECTION` (`DATAJOB.CPP:598`, `MultIniState.cpp:117`, `TwoPhaseSettings.cpp:50`, `PrjPreSel.cpp`); `DENS->`'s 4-line structure (`dataDens.cpp:146`); `CREEP->`'s 14+ fields (`dataCreep.cpp:551`, matches the doc's own long-standing example exactly); `HEAT->`/`HUMID->` (`dataHeat.cpp:592`, `dataHumidity.cpp:256`); `INIS->` continuum fields 4-12 (`dataInitialState.cpp:167` — beam `INIS->` still not located, real beam materials mostly don't attach this group); `STAB->` (`dataStab.cpp:141`); `NONL->` Fiber Shell's remaining fields (`dataNonLinear.cpp:197`); `LAYERED_BEAM_COMPONENTS`'s `v[2]` = `g` unit weight (`CompositeMat.cpp:52` — genuinely just left at its unedited default of 25 in both the concrete and steel example lines, not vestigial); `SIG_EPS_FUN`'s two version-gated formats (`dataFun.cpp:273,534`); `LOAD_FUN`'s flags-line bit (`FC_FLAG_SKIP_TIME_STEPS`) and the line-3 `FlagBitCode` field (`dataFcCh.cpp:330,474`); the header-count table's entries 33/49/88 as confirmed upstream copy-paste label bugs (`GenInf.cpp:44-145`); `.pil`'s interior-point trailing data (coordinates + embedding-`.i0g`-element list, not flags — same pattern as the already-documented `.anh`, `EmbeddedLinearElement.cpp:802`); `.icg`'s `OppEleNum`/`OppFaceNum`, `genFullContinuity`/`InitialGap`/`initialGapNotUsed` (`slide.cpp:103,566`, `ContactParam.cpp:95`); `.ics`'s `iLayer`/`nsides`/`activeFlg` header (`SlideStr.cpp:1041`); `.inb`'s previously-"unclear" token 5 as `ULF`, and BC type codes 5/7 (rotation velocity/acceleration) alongside the already-known 1/2/4/6 (`boundarycondition.cpp:369`).

**Explicitly not resolved, left flagged in the doc**: `T3D_PARAM_*`'s literal enum values beyond ordering (values live where `constant.h` doesn't declare them, only the enum names/order do); per-continuum-model `NONL->` field layouts for Mohr-Coulomb/Hoek-Brown/Drucker-Prager/Cam-Clay/Hujeux/HS-small-strain/Duncan-Chang/Cap model (classes located — `Mohr_Coulomb.cpp`, `Hoek_Brown.cpp`, `NonLinearDruckerPrager.cpp`, etc., all `GenericNonLinearGroup` subclasses in `dataNonLinear.h` — but not individually field-mapped); `.pil`'s `code` field and `.icg`'s `genFullContinuity` setter semantics (declared/written, no setter/enum found in this checkout); beam `INIS->`; `.bcb`'s trailing `0 0` line (the geometry-catalog writer itself appends nothing after its last entry — this line may belong to a different class, `CableSets.cpp`/`BeamCableDef`, not confirmed either way).

## §3 File Header

- §3.1 version line: three-field breakdown (majorVersion/createdWithVersion/lastOpenedWithVersion) is an inference from comparing version lines across samples with different `.inb` record widths (see §8.1 note) — not GUI-confirmed.
- §3.2 header count table: transcribed directly from real files (`boxd1.inp` et al.), consistent across every sample checked.
- §3.4 CONTROL fields: field-by-field meaning came from the user directly (domain knowledge), not derived by us. Example values from `boxd1.inp`'s "Default" set.
- §3.5 DRIVERS keyword table: names are self-explanatory; values not decoded from any source.

## §4 Materials — GUI-verified material

All of §4.2–4.4's GUI-attributed content came from one interactive session against the live ZSoil GUI (v26.03), material "Shells"/"Shell Layered" in `NL_beams_shells`, cross-checked against the underlying `.inp` files listed above. Specific confirmations:

- **DENS->**: "Unit weight/mass" dialog. γ (weight/vol) and ρ (mass/vol) fields confirmed directly; "Data mode" dropdown (Standard/Load function/Super element data) observed but its effect on the trailing fields not verified byte-for-byte.
- **FLOW-> (shell variant)**: "Shell flow" dialog. Field *set* confirmed (fully-permeable flag, anisotropic flag, kx'/ky'/kz', orientation vector, van Genuchten Sr/alpha) but exact token positions in the raw 11-value line were not re-derived by hand after seeing the dialog — treat the field order as approximate.
- **NONL-> (Fiber Shell)**: "Non Linear Truss" dialog showed fc above ft visually, but cross-checking the *raw file* for a second sample (`NL_shell_bending - Copy.inp`, concrete: file has `10002 100001`, GUI showed ft=10002/fc=100001) confirmed the actual stored order is ft-then-fc — the GUI's visual field order does NOT match file storage order. This is a good example of why raw-file cross-checks matter more than trusting a dialog's layout.
- **FIRE_DATABASE**: "Extension to fire problems" dropdown, options `Off`/`EC2:2008`. Only `Off` (flag=0) observed in any sample; the "on" encoding is a guess.
- **ELAS-> (Fiber Shell)**: "Material" dialog shows only a "Young modulus" field — no nu field in the GUI, even though the file stores 2 values. Not fully explained; assumed nu is vestigial/unused for this material.
- **ELAS-> (Orthotropic shell)**: "Material" dialog showed E1, E2, ν12 (labeled "v1" in the GUI), Go — matched exactly against raw file `2.7e+07 2.7e+07 0.2 1.1e+07`.
- **GEOM-> (Fiber Shell vs Orthotropic shell)**: "Geometry" dialog, "Direction vector" X/Y/Z. Fiber Shell's raw GEOM-> has a leading `type` field before the vector; Orthotropic shell's doesn't — confirmed by direct comparison of the two materials' raw lines in the same file.
- **"Material definition for layered shell" dialog**: this is the actual editor for a `Shell Layered` material's `GEOM->`. The example model has 4 layers: 1 Core (concrete, Fiber Shell) + 3 Reinforcement (1× steel/Fiber Shell, 2× "ortho shell"/Orthotropic Shell) — richer than the 2-fiber example quoted in the doc from `NL_shell_traction.inp`. The Distance dropdown's 3 options ("from top"/"from bottom"/"Relative (-1,1)") were read directly from the dropdown, confirming the `distanceFrom` 0/1/2 codes inferred earlier from raw-file inspection alone.
- **"Shear correction factor"** field exists in this dialog (value `1` in the example) but its position in the raw `GEOM->` record was never located — this is a genuine open question, not just an unconfirmed-guess placeholder.
- **HEAT->/HUMID->/DAMP-> first field(s)**: "Heat"/"Humidity"/"Damping" dialogs on the parent Shell material — heat/hygral dilatancy and Rayleigh α₀/β₀ respectively. Only field 1 (or 1–2 for damping) checked; remaining fields not explored via GUI.
- Note: none of §4's "beam" or "continuum" or "truss" material rows were GUI-checked — only the shell-family rows (`Shell`, `Shell Layered`, `Fiber Shell`, `Orthotropic shell`) went through the interactive GUI pass. Beam/continuum/truss content is file-inspection-only, same confidence level as before that session.

## §4 Materials — GUI-verified Continuum material (2nd session)

Model `2D_anchor_disp_s1m`, material #1 ("Soil", `Continuum`/`Elastic`). Confirmed the GUI's exact label is **"Continuum"** (not "Volumics" — that was our own descriptive term, now corrected in the §4.2 table to match the GUI). This material's checkbox set: `Elastic`, `Unit weights`, `Flow`, `Creep` (present but unchecked), `Initial Ko State`, `Heat`, `Humidity`, `Damping` — no `Non linear` (plain `Elastic` has no plasticity model, consistent with `NONL->` always empty for this formulation).

- **ELAS-> (continuum)**: "Material" (Elastic) dialog showed Young modulus E=25000, Poisson ratio ν=0.3, Data mode=Standard — exact match to the raw file's `ELAS-> 25000  0.3`, confirming the 2-field layout with full confidence (previously inferred from file inspection only).
- **INIS-> (continuum) — corrected a wrong guess.** The doc previously guessed the first two `INIS->` fields might be "initial void ratio or saturation." The GUI's "Initial State Ko (2D)" dialog showed **Ko(x′)=0.385, Ko(z)=0.385, inclination angle ⟨x′,x⟩=0** — matching the raw file's `INIS-> 0.385  0.385  0  ...` exactly. So fields 1–3 are the assumed Ko (at-rest lateral earth pressure coefficient) values and inclination angle, not void ratio. This is a meaningful correction, not just an elaboration — worth double-checking anywhere else in the doc that might have repeated the old guess (checked: nowhere else did).
- **FLOW-> (continuum)**: the "2D Flow" dialog is rich — Darcy's law (isotropic flag, k, inclination angle, skip-gravity flag), van Genuchten retention curve (Sr, α, n, permeability function Irmay/Mualem), undrained-behavior settings (penalty factor K^F/K, suction cutoff), fluid compressibility (Kf, air-compressibility flag, Ka), Bishop's effective stress (global-setting flag, pore-pressure weighting term), and Biot coefficient (enable flag, α, Ks). Cross-checked against the raw ~28-value FLOW-> line: several *values* matched unambiguously by distinctiveness (2000000→Kf, 20→α, 1000000→K^F/K, 100→Ka or suction cutoff, 1e+38→Ks, 2→n), but the exact token-by-token order was not fully re-derived — several boolean flags could not be pinned to a specific index with confidence. Treat the §4.3.5 continuum FLOW-> field list as "confirmed to exist, approximate order" rather than fully indexed.
- Did not get to check `Creep` (unchecked in this material) or the beam-family (`nail`), truss-family (`anchor`), or interface-family (`Scell` ×2) materials also present in this same model — worth a follow-up pass if beam/truss ELAS->/GEOM-> confirmation is ever needed.

**Gotcha hit during this session**: calling `open_application` on an already-open ZSoil instance spawned a *new*, separate blank "Untitled"/"Untitled2" project window rather than focusing the existing one. Two blank instances had to be closed (harmless — no data existed in either) before returning to the original model's Materials dialog. If ZSoil is already frontmost, prefer clicking directly rather than calling `open_application` again.

## §4.3.5 `FLOW->` continuum — pinned by edit-and-diff (3rd session)

Method: opened the disposable scratch copy `2D_anchor_disp_s1m_test.inp` (confirmed byte-identical to the real `2D_anchor_disp_s1m.inp` before editing — see below), gave every editable "2D Flow" dialog field on the `Soil`/`Continuum`/`Elastic` material a distinct value (101, 102, 103, ... plus dropdown changes: permeability function → Mualem, Bishop weighting term → `S`, Bishop "use global setting" → off, Biot "Enable" → on, "Skip gravity term" → on), saved, then diffed the resulting `FLOW->` line against the *original* file's `FLOW->` line (which — conveniently — was already the exact example quoted in the doc, so no extra baseline read was needed). Every value that changed was attributable to a specific field by direction and magnitude; this is a much stronger method than reading the GUI alone, since it doesn't depend on trusting the dialog's visual layout (see the `NONL->` ft/fc note above for why that trust would have been misplaced).

Gotchas hit during this pass, worth remembering for next time:
- **`triple_click` and `ctrl+a` do not reliably select-all in these ZSoil numeric edit fields.** Both left the existing text in place and inserted the typed value at the start, producing concatenated garbage (e.g. typing `105` into a field showing `20` gave `20105`, not `105`). The reliable method that worked every time: click the field, press `Home`, then `shift+End`, then type — full manual select-to-end-of-line rather than any select-all shortcut.
- **`open_application` on an already-open ZSoil instance is unreliable** — sometimes correctly focuses the existing window, sometimes spawns a new blank "Untitled"/"UntitledN" instance instead (happened twice more this session, on top of the one logged above). If a field edit or click starts erroring with "the desktop shell is frontmost" even though the screenshot shows ZSoil, that's the tell — call `open_application` once, screenshot to confirm, and if a blank instance appears instead of the expected project, close *that* window (its title bar will say `Untitled`/`UntitledN`) rather than assuming data was lost.
- Two numeric fields have live validation that silently recomputes the value rather than accepting arbitrary input: **Biot α** must be in `[0,1]` (typing `111` triggered "Enter a number between 0 and 1"), and **Biot Ks** has a lower bound tied to porosity (typing `112` triggered "greater than porosity", and a subsequent large value was auto-recalculated to `33333.3333333333` rather than kept literally as typed — don't assume the raw file will contain exactly what was typed for fields like this without checking).

Result — the 28-field continuum `FLOW->` record, fully positionally confirmed except 9 flag positions and 3 positions shared between 2 Bishop settings (see the table now in §4.3.5 of the main doc). Not yet done: isolating `v18`/`v21`/`v22` by toggling the two Bishop settings *separately* (one save each) rather than together; identifying the 9 remaining always-0-or-1 flag positions (`v4`–`v9`, `v13`, `v15`, `v19`) would need the same one-flag-at-a-time save/diff treatment, which wasn't attempted here given time already spent working around the UI issues above. `2D_anchor_disp_s1m_test.inp` is now left in the edited state (values 101–112 etc., not reverted) — it's a disposable scratch copy by design, so this should be fine, but flag it if the user expects it back to a pristine baseline for other purposes.

## §7 Elements — B8 face numbering

The face-id table in §7.1 came from cross-referencing `.i0g` connectivity against `.ing` coordinates and `.ple` `SURF_LOAD` targets in a file `3D_with_loads.inp` (a cube meshed into 216 `B8` elements, surface load on the `x=0.5` boundary via 36 explicit `eleId faceId` pairs). Faces 1, 2, 4, 6 were each checked against a distinct element (elements 215, 214, 181/216, 213 respectively). Faces 3 and 5 were never observed directly — inferred by elimination + the "opposite faces sum to 7" pattern. If this matters for a real analysis, verify faces 3/5 against a fresh GUI-built example before relying on them.

## §8 Boundary Conditions — `.inb` width discrepancy

Two 2D samples with different `@..._version:` sub-version numbers (`boxd1.inp`, created with `9.05`; `el_simple_load_2D_tri.inp`/`2D_anchor_disp_s1m.inp`, created with `24.06`) showed different `.inb` record widths (16 vs 19 tokens) despite both being 2D. The "format evolution, not a 2D/3D distinction" conclusion follows from this comparison and from `zsoil_inp.py`'s own commented-out alternate-indices branch (`[3,7,11]` for `analysis_type==2`), which suggests the parser used to handle a narrower layout explicitly. This is inference from two data points, not a confirmed changelog — if more sample files across sub-versions become available, worth re-checking with a larger sample.

## §8.3 `.pbc` / §6.5 `.apl` — periodic BC now has a populated example

Source: the user's own domain knowledge (not derived from the corpus or GUI this time) plus a real working file built collaboratively this session, `C:\Mandats\M100\papers\NUMGE2027\calc\zsoil\shear_column\Model_N5.inp` — a periodic tie linking the two side-columns of a 1D shear-beam mesh at every row. The user stated directly that `.pbc`'s header `<type>` field is a **DOF bitmask** (2=Ux, 4=Uy, 8=Uz, 16/32/64=rotations, then pressure/temperature/humidity bits), not an opaque constant — `type=14` in the example decodes as `2+4+8` = Ux+Uy+Uz tied. This also resolved the previously-unexplained numerical match between a `.pbc` block's own plane-definition line and a corresponding `.apl` entry's plane line (§6.5): they're the same plane, `.apl` being the plane geometry `.pbc` ties nodes across — so `.apl` is not purely a GUI sketch aid as the doc previously implied, it's functionally load-bearing whenever a periodic BC exists. Still unconfirmed: the 11-numeric-flags line in `.pbc` (purpose unknown), the `.pbc`/`.apl` cross-reference mechanism (numerically identical plane data in both, but no confirmed index field linking them), and the `.apl` plane-type field (`3` in the only example seen).

## §5.2 `LOAD_FUN` — flags-line `#N` is the interpolation selector, not the numeric line's 3rd field

Corrected mistake, same source (user's domain knowledge, same session as the `.pbc` entry above). The doc previously described the numeric line after the flags line as `[t0, scale, type]`, guessing `type` (3rd field) was the interpolation-mode selector. Wrong — the flags line itself carries that selector, as `#1` (solver interpolates the table at its own time increments) or `#0` (solver is forced to step through the LTF's own listed time points); the numeric line's 3rd field is a genuinely separate, still-unidentified field (always `0` in examples seen), and its 1st/2nd fields are a time **shift** and a **scale**, not "t0"/"scale" in the sense implied before. `No flags` (rather than `#N`) also appears in at least one real example (the doc's own trivial §5.2 example) — how it relates to `#0`/`#1` is unconfirmed.

## §3.4 `DYN_CONTROL` field layout / §3.5 `DRIVERS` record structure

Same session/source as the `.pbc`/`.apl`/`LOAD_FUN`/`.inb`-flag entries above — `Model_N5.inp`, this time reading the pre-existing `DYN_CONTROL` and `DRIVERS` blocks the user had already set up (not authored by us) and checking them against the doc.

- **`DYN_CONTROL`**: previously an undecoded stub ("field-by-field layout not decoded here"). Positionally confirmed the `alpha`/`beta`/`gamma` fields on the main numeric line via the standard HHT-Newmark identity `beta=(1-alpha)^2/4, gamma=0.5-alpha` — the example's `alpha=-0.3` gives exactly `beta=0.4225, gamma=0.8`, matching the file. This is a strong confirmation (the math has to work out, not just plausible field-guessing) but only from one example (one alpha value); most other fields on all 5 lines remain unclear. Also noted a second `alpha` value on line 5, numerically identical to the main line's in this example — genuinely unconfirmed whether it's a redundant copy or a real secondary control that could disagree in some other file.
- **`DRIVERS`**: the doc previously claimed this followed §3.4's generic 2-line (name+numeric) named-set pattern. It doesn't — this was already known more precisely in the `zsoil` Claude Code skill's own gotcha notes (an `.inb`-adjacent finding from an earlier part of this same overall effort, not re-derived from scratch this session) but had never been carried into this canonical doc. Corrected here: 3 lines per driver record (name / numeric / solver-settings-name, the last of which must match a `CONTROL` name), a mandatory `init` driver not counted in `<n>`, and the numeric line's `<type>` being a 0-indexed `DriversType` enum (`DYNAMICS`=4). Also flagged 4 trailing lines after all driver records that neither the skill notes nor this doc explain — present unchanged in the example, left as an open gap rather than guessed at.

## §6.1 `.ing` — trailing flag must be an integer literal

Found by reproducing a real failure: a batch coordinate-update script reused the line's float formatter for every field, including the trailing flag, writing `0.000000000000e+00` instead of `0`. ZSoil silently refused to open the resulting file. Reverting just that field to a bare `0` fixed it. Doc now states the rule directly (§6.1) without the story.

## §7 Elements — `<idx>` vs `<number>` counters

Observed in a model with 3943 `.i0g` continuum elements followed by 32 `.ibg` beams: the first beam's `<idx>` was `3944` (continuing the file-wide sequence), not `1`, while its `<number>` restarted at `1`. Confirmed `<idx>` (not `<number>`) is what `.ics`/`.icg` paired-element fields and `.anh` truss references use, by cross-checking those references against the referenced element's actual `<idx>`. Gaps in the `<idx>` sequence appear tolerated when hand-inserting elements — ordering, not contiguity, is what other records seem to rely on — but this wasn't stress-tested beyond one or two inserted elements.

## §7.1 `Q4` node order / face numbering

Confirmed by cross-checking multiple real `.ics`/`.icg` records' "paired-elem face" fields against the coordinates of the nodes on the named face in `.ing` — the counter-clockwise / `face k = node_k → node_{k+1}` rule held in every case checked.

## §7.8 `.icg` — connectivity node order and real-world usage

**Node order**: confirmed against a concrete example — a tie where `elem1` face 3 ran `n3=1806 → n4=1802` and `elem2` face 1 ran `n1=1801 → n2=1805` produced the connectivity line ` 1802 1806 1801 1805` (elem1's face-node pair reversed, elem2's forward). This opposite winding is consistent with the two faces having opposite outward normals — doc now states the rule without this specific instance.

**`type=2` as a generic rigid tie**: encountered in a real model at a plain soil-soil seam ~30 m from any structural or staging feature — two nodes at identical coordinates but different IDs, no shared node, no `.ikg` kinematic constraint, tied only by a `.icg type=2` record. This is why the doc now warns not to assume a duplicated-position, differently-numbered node pair is a meshing error without checking `.icg` first.

## §7.9 `.ics` — double-sided beam contact example

Full example that the doc's condensed version was distilled from (a wall's bottom beam segment, contacted on both faces): beam `3944` connects nodes `14`(lower) → `1804`(upper). West side: paired-elem `1480` face `2`; connectivity ` 1806 14 1804 14` = `<soil@upper> <soil@lower> <beam@upper> <beam@lower>`. East side: paired-elem `1833` face `2`; connectivity ` 14 1807 14 1804` = `<soil@lower> <soil@upper> <beam@lower> <beam@upper>` — west reversed relative to the beam's own node1→node2 direction, east forward, matching the `.icg` §7.8 convention.

## §7.14 `.anh` — axis-point trailing fields

Confirmed by checking that the referenced `.i0g` element(s)' node coordinates actually bound the axis point's own `(x,y)` — for several axis points across at least one anchor, both the single-element (`1 <id>`) and shared-edge (`2 <id1> <id2>`) cases checked out.

## §8.1 continuum-node `.inb` flag values (1/4/6) and multi-record priority

Same source/session as the `.pbc`/`.apl` entries above, from building and debugging a private project file's base boundary condition (originally acceleration-driven, then changed to velocity-driven mid-session — both went through this same mechanism, just with `flag=6` vs `flag=4`). Established across this and an earlier (compacted) part of the same overall effort: for continuum/solid nodes `<flag>` selects BC type (`1`=displacement, `4`=velocity, `6`=acceleration), a node can carry multiple `.inb` records (one per type), and when two records prescribe the same DOF the more dynamic type wins (acceleration > velocity > displacement) even though the lower-priority record's `fixedFlag` still reads `1`. Not derived from the corpus or GUI — confirmed empirically by observing the intended behavior (a base node with a `flag=1` record fixing it at 0 plus a `flag=4`/`flag=6` record referencing a `LOAD_FUN` on the same DOF) actually driving the node as intended rather than staying fixed. This is a different meaning of `<flag>` than the 3D-beam translation/rotation split documented just above it in §8.1 — don't conflate the two.

**Now independently corroborated by the shared corpus**: `col1D_el_fixedBase.inp` (a fixed-base 1D column dynamic test, `LOAD_FUN 1` = "horizontal base velocity") carries the *exact same* two-record pattern on its own base node — `1 1 1 1 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0` (`flag=1`, `Uy` fixed at 0) immediately followed by `2 1 4 1 1 0 1 0 0 0 0 0 0 0 0 0 0 0 0` (`flag=4`, `Ux` driven by `LOAD_FUN 1` ×1.0) — byte-for-byte the same shape as the doc's own worked example in §8.1 (which was written from the private file before this corpus file existed). The doc's example is left as-is since it already matches; noting this here so the claim is no longer resting solely on an unavailable file.

## §4.2/§4.3.1/§4.3.7/§4.3.10 `HS-small strain stiffness` continuum material

Source: not the corpus, not the GUI — an A/B diff test between two `.inp` files from the NUMGE2027
project (`C:\Mandats\M100\papers\NUMGE2027\calc\zsoil\shear_column_hss\Model_HSS_N40.inp` vs.
`Model_HSSm0_N40.inp`), both hand-built by the user via the ZSoil GUI, differing only in one GUI
toggle ("pressure-independent" stiffness). `diff`-ing the two files isolated exactly which raw
fields that toggle controls: `ELAS->` field 3 (`0.5`→`0`) and two values in the following `NONL->`
block's first numeric line (auto-recomputed, not independently settable). This is a strong,
mechanical confirmation for field 3's identity (`m`, the stress-exponent) specifically — it is not
a GUI walkthrough, so no other `ELAS->` field position was independently isolated the same way;
the doc's guesses for fields 1/2/4/9 (`G0_ref`/`nu_ur`/`pref`/`gamma_0.7`) are value/position
analogies to the standard Hardening-Soil-small parameter set, not confirmed the way field 3 is —
flag this distinction if extending the entry further. The `INIS->` cross-reference (same Ko-state
field shape as the already-GUI-confirmed plain continuum `Elastic` case, §4 "GUI-verified Continuum
material" above) is a structural-match inference, not a fresh GUI confirmation for this specific
formulation.

## Corpus coverage caveats (applies throughout §7–§13)

Most "typically empty" / "no populated example available" statements in the doc reflect the ~71-file `zsoil_inp_files` corpus plus spot checks in the larger `v26\manual` and `ZSoilPy3\tests` trees — not an exhaustive search. Absence of an example there is evidence of rarity, not proof the marker is unused in general. Markers flagged this way (`.iff`, `.ipg`, `.ivg`, `.ikg`, `.gnl`, `.gbh`, `.brc`, most of §11.3, most of §13.3, etc.) are good candidates for follow-up if a real project ever populates them — regenerate a minimal GUI example and diff. (`.pbc` was on this list too; no longer — see the dedicated §8.3/§6.5 entry above.)

## §6.5/§7.6/§7.10/§12.1 — corpus sweep for populated examples ("add minimal examples" pass)

Ran a full-corpus awk scan (98 `.inp` files across `zsoil_inp_files/` + `ZSoilPy3/tests/*`) checking every marker the doc flagged as lacking a populated example, for non-blank/non-`0` content immediately following the marker line. Findings:

- **`.ipg` (§7.6)**: genuinely populated in `ZSoilPy3\tests\diaphragm-wall\HS-Brick-Exc-Berlin-Sand-2phase.inp` (2-phase-flow excavation model). Real records only carry 4 trailing integer fields (`mat1 mat2 EF ULF`), not the 6-field `mat1/mat2/mat3/EF/ULF/iLayer` pattern the doc had extrapolated by analogy from §7's intro — that guess is now replaced with the confirmed shorter shape. Each record is followed by a skipped name line (`"No name"` in this file), same convention as `.icg`'s per-set name line (§7.8).
- **`.ijg` (§7.10)**: genuinely populated in `ZSoilPy3\tests\2D-case-1.inp` (2 joint elements, tag `M_L2`). Added the raw record as an example; did not assert a specific split of the 9 trailing integer fields — no independent source/GUI cross-check was done for `.ijg` this round, so the field breakdown stays a stated analogy, not a confirmed mapping.
- **`.glk` (§6.5)**: genuinely populated in only one file across both corpora (`ZSoilPy3\tests\diaphragm-wall\HS-Brick-Exc-Berlin-Sand-2phase.inp`), a single local-basis record `1 1904 3971 3 0 1`. No writer/reader for `.glk` was found via literal-string grep in `Z_Prep3D` (consistent with this session's earlier finding that many extensions are assembled via the `PrePro_Ext[]`/external-config fragment mechanism rather than hardcoded per-tag code) — field meaning stays unconfirmed, only the raw shape was added.
- **`.axs` (§6.5)**: populated in many `zsoil_inp_files` files (e.g. `2D_anchor_disp_s1m.inp`, `2D_anchor_disp_s1m_nl.inp`), blank (`0`) in others (e.g. `boxd2.inp`). Example: `1` / `Default` / `X 5` / `0.0`. No writer found via grep either; doc now shows the observed 4-line-per-entry shape (`name` / `label n` / `value`) but keeps "meaning unclear" rather than guessing what an axis entry is for or what references it.
- **`.brc` (§12.1)**: the earlier awk scan flagged `.brc` as "populated in nearly every file" — this was a **false positive**: the line right after `.brc` is always the header pair `0 0` (zero sets, zero members), which is non-blank/non-`"0"` as a literal string match but is in fact the empty-count line. Rechecked explicitly across the full corpus (`grep` + per-file first-line dump): `.brc` is `0 0` everywhere, no exceptions. Doc's existing field-by-field layout (built from the source `.inp` writer, not a sample) is left as-is, with a note added that no populated real-world example exists in this corpus.
- **`.ilt` type=`1` variant (§7.5)**: checked all 12 corpus files with non-default `.ilt` content (shell-bending/traction, winkler, plate tests, bridge-loads). All show `type=0` (the already-documented variant); none show `type=1`. No doc change — the type=1 variant remains an unconfirmed gap, now double-checked against the corpus rather than left purely theoretical.
- Markers reconfirmed as genuinely blank everywhere in both corpora (no doc change, existing "no populated example" language stands, now double-checked): `.ish`, `.ivg`, `.scs`, `.ims`, `.ikg`, `.gnl`, `.hex`, `.hef`, `.grp`, `.iem`, `.pme`, `.iag`, `.iav`, `.img`, `.imv`, `.ieg`, `.ist`, `.eie`, `.spg`, `.svg`, `.scg`, `.crc`, `.cld`, `.gos`, `.idv`, `.fac`, `.mrt`, `.drz`, `.dre`, `.pth`, `.bcl`, `.isd`.
- `.izg`/`.iig` also showed up as "populated" in the scan but the doc already carries real examples for both (§11.1/§11.2) — no action needed.

## Known gaps in `inp-file-format.md` — consolidated list (as of 2026-08-27)

The main doc flags each of these inline with a short marker (`(unclear)`, `unconfirmed`, "no populated example", etc.), per its own convention — this section is just a single-place index of all of them, grouped by doc section, so a future session doesn't have to grep the whole file to find what's still open. Regenerate by grepping `inp-file-format.md` for `unclear|not confirmed|unconfirmed|no populated example|not documented|not resolved|unresolved|not identified|not decoded|not established` if this list goes stale.

**§3 Header/Control**
- ~~`DYN_CONTROL` line 2 fields 1–4 and line 3~~ — **resolved**, see dedicated entry below.
- Header count-block row 146 (`4 0` flag pair immediately before `CONTROL`) — meaning unclear.

**§4 Materials**
- Plain `Shell` formulation's own `ELAS->`/`GEOM->` content — not documented at all.
- Duncan-Chang `G`/`F`/`d` (variable-ν triple) — formula is standard literature knowledge, not independently confirmed field-by-field via source/GUI (no GUI tooltip text found for these three).
- Continuum `GEOM-> <value>` — single value, meaning unclear.
- Shell `FLOW->` — field *set* and meaning known, but exact token positions not independently verified.
- Beam `INIS->` — same 12-field shape as continuum assumed, but whether the first 3 (Ko-related) fields are even applicable to a beam is unconfirmed, and the remaining fields' meaning is unclear.
- `DAMP->` — only fields 1–2 (α₀/β₀, Rayleigh damping) are known; remaining 5 fields unclear.

**§6 Geometry**
- `.ing` trailing `<flag>` — always observed as `0`; meaning unclear.
- `.glk` — how bases are referenced from elements is unconfirmed; the one populated example's field-by-field meaning isn't confirmed either (no writer/reader located via source grep).
- ~~`.axs`~~ — **resolved as a construction/snap-grid definition**, see dedicated entry below (structural interpretation only, no writer/reader source-confirmed).
- `.apl` `<type>` field — always `3` in every example; meaning unclear.

**§7 Elements**
- `.ish` (shell hinges) — populated record layout not documented (typically blank).
- `.icg` `genFullContinuity` flag — setter semantics not confirmed (declared/written, no setter/enum found in this checkout).
- `.ics` lines 5–6 — unclear.
- `.scs`/`.ims` — no populated example for either.
- `.ijg` — 9 trailing integer fields' exact split not confirmed (analogy to other element records only).
- `.gbh` per-borehole record layout — inferred structurally, not confirmed against real populated data.
- `.bcl` — likely a duct/tendon-loss-coefficient catalog analogous to `.bcb`, but unconfirmed.

**§8 Boundary Conditions**
- `.ilb` field layout — not independently verified against a populated example.
- `.pbc` — two trailing fields on one line, and a separate 11-numeric-flag line, purpose unclear on both; no confirmed cross-reference field ties a `.pbc` record to its `.apl` plane by index (they just share numeric plane-equation values).
- `.gwb` header's trailing field — doesn't match source's expected `unloading_function`/`siz` pair; not fully resolved which it is.

**§9 Loads**
- `GRAD_LOAD` — record fields beyond reference-node/direction/data unconfirmed (no populated example).
- `.ple` `<LF2>` — exact meaning unconfirmed.
- `.pth` (movement paths) — record syntax unconfirmed (always empty in samples seen).

**§10 Masses**
- `.iem` line 2 — purpose unclear.
- `.pme` — record shape unconfirmed; assumed to mirror `.ple`'s `LINE_LOAD`/`SURFACE_LOAD` shape but for mass, not verified.

**§11 Initial Conditions**
- §11.3 (heat/humidity/strain initial conditions) — shape is an extrapolation from the `.izg`/`.iig` pattern, not confirmed.
- `.idv` — field layout confirmed (displacement+velocity, no acceleration, per GUI dialog binding), but no populated example exists to confirm real field values in practice.

**§13 Mesh Tying & Domain Reduction**
- `.mrt` — purpose unconfirmed beyond "sits next to `.fac`, likely a related mesh-tying setting".
- `.fac`/`.mrt`/`.drz`/`.dre`-family markers — record shape expected to mirror `.sdm`'s face/edge referencing, but no populated example confirms the exact fields.

**§14 Other Data**
- `.gos` — populated syntax not documented (confirmed read+write this session, but no populated example).

**Explicitly out of scope, not chased this session** (per the original plan, still true):
- `T3D_PARAM_*` literal enum values (ordering confirmed via `constant.h`, exact values not — not needed since ordering is what determines field position).
- `.pil`'s `code` field.
- `ContactRC`/`ContactLD`'s own `writeOn` bodies — `.crc`/`.cld`'s exact per-record field layout wasn't traced beyond the manager/mechanism level.
- `.ivd`/`.svd`'s own field-level record layout (the `ViscDamp` object's write function itself) — only the mechanism/manager level was traced.

## §3.4 `DYN_CONTROL` fully field-mapped; `PSH_CONTROL` cross-checked

Source: a new corpus file, a 1D column dynamic/seismic test (`col1D_el_fixedBase.inp` — fixed-base
elastic column, `LOAD_FUN 1` = "horizontal base velocity", `DirichletBC`-driven). Full field mapping
found directly in `DynCtrl.cpp:116-135` (`operator<<`) and `DynCtrl.h` (member declarations),
cross-referenced against `DynCtrl_Dlg.cpp`'s `DDX_*` bindings (`TypeMass`→`R_MASS_LUMPED`/
`R_MASS_CONSIS` radio, `RotInertia`→`B_MASS_ROT` check, `MassFilter`→`B_MASS_FILT` check + gates
the `Dir[]` checkboxes' enabled state, `AlgorType`→the 5 `R_ALGOR_*` radios matching `eAlgorTpe`
exactly in declaration order, `alphaDamp`/`betaDamp`→`E_DAMP_A`/`E_DAMP_B` with a `B_DAMP_EVAL`
button, `alphaAlgorFluid`/`thetaAlgor`→`E_ALGOR_ALPHA_F`/`E_ALGOR_THETA`, shown/hidden together with
the main `alpha`/`beta`/`gamma` fields depending on `AlgorType` — confirming these are genuinely
separate fields, not a redundant copy as the doc previously guessed). `DefWay`/`w_f_T_1`/`w_f_T_2`/
`xsi1`/`xsi2` traced through `DynCtrl_Dlg::OnDampEvaluate()` (`DynCtrl_Dlg.cpp:269-305`) into a
`NewmarkPar_Dlg` sub-dialog — this is the standard two-target-frequency (`w1`/`w2`) two-damping-ratio
(`xsi1`/`xsi2`) Rayleigh-damping evaluator, an alternative way to arrive at `alphaDamp`/`betaDamp`
rather than entering them directly (`DefWay` selects which). `eAlgorTpe` enum from `DynCtrl.h:13-20`.
In the sample file: line 2 = `0 0 0 0 3 -0.3 0.4225 0.8` (`TypeMass`=0/lumped, `RotInertia`=0,
`alphaDamp`=`betaDamp`=0, `AlgorType`=3/HHT-α-displacement, `alpha`=-0.3); line 3 = `0 9.80655 1 1 1`
(`MassFilter`=0, `Acc`=9.80655, `Dir`=[1,1,1] — note `MassFilter`=0 here despite `Dir` being
non-default, so the direction fields aren't gated by `MassFilter`'s value in the file itself, only
in the dialog's enabled/disabled rendering); line 4 = `0` (`localBaseFlg`=0, no extra line); line 5 =
`-0.3 0.5 1` (`alphaAlgorFluid`=-0.3 — same as `alpha`, consistent with "coincidentally equal in a
simple/uncoupled example" rather than a hard redundancy; `thetaAlgor`=0.5; `includeDarcyLaw`=1);
line 6 = `0 0 0 0 0` (`DefWay`=0, direct-entry mode, so `w_f_T_1`/`w_f_T_2`/`xsi1`/`xsi2` stay at 0).
§3.4 rewritten with the full field table.

Same file's `PSH_CONTROL 1` / `Default` / `0 0 0 0 9.80655 1 1 1` matches the doc's already-fully-
mapped `<TypeMass> <RotInertia> <MassFilter> <ForcePattern> <Acc> <DirPsh_x> <DirPsh_y> <DirPsh_z>`
field-for-field with no discrepancy — first real populated cross-check for that struct, no doc
change needed.

## §6.5 `.axs` — read as a construction/snap-grid definition; §8.3 `.pbc` cross-check

Source: the new `col1D_el_fixedBase.inp` corpus file, plus a re-check of an earlier sample
(`2D_anchor_disp_s1m.inp`) that turns out to carry the exact same block. Both show the *full*
`.axs` entry (earlier scans only captured the first few lines): `<count>` / `<name>` / then 3
sub-blocks, one per axis (`X`/`Y`/`Z`): `<label> <n>` followed by `<n>` coordinate values. Both
files show byte-identical content (`Default`, 5 gridlines at `0,1,2,3,4` on all 3 axes) despite
having unrelated model geometry — this rules out the entry being derived from the model itself and
points to a GUI-level default (a construction/snap-grid preset) instead. No writer/reader was
located via literal-string grep in `Z_Prep3D` (same non-finding as `.glk` — consistent with the
extension-fragment-file mechanism documented in §2.2), so this is a structural read, not a
source-confirmed one; §6.5 updated with the fuller example and this interpretation, hedged
accordingly.

Same file's `.pbc` block (`1 14 82 0 0`, 41 tied node pairs, same `type=14`/`Ux+Uy+Uz` and the same
plane values `1,0,0,-0.5` as the doc's existing example) is a second, larger-scale confirmation of
the existing `.pbc` field mapping — `<nNodes>` = `2×pairCount` holds again (`82` = `2×41`). No new
fields resolved (the two trailing header fields and the 11-flag line are still `0`/all-zero here
too); no doc change needed beyond noting the corroboration.

## Corrected mistake (kept here as a flag, not in the main doc)

An earlier version of this documentation effort (surfaced via the `zsoil` Claude Code skill's gotcha notes) implied there was ambiguity about which of `LAYERED_BEAM_COMPONENTS` (beams) vs. the `Fiber Shell`/`Orthotropic shell` sub-materials (shells) supplies a layered element's operative capacity. There isn't — they're simply different mechanisms for different element types, not competing candidates for the same element. The skill's gotcha text was corrected accordingly (see `Unittests\.claude\skills\zsoil\SKILL.md` and `references\file-formats.md`).
