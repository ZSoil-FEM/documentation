# Session progress log — pinning down inp-file-format.md via ZSoil source

Handoff notes for picking this work back up later. Not part of the two canonical docs
(`inp-file-format.md` = end-user reference, `verification-notes.md` = per-fact provenance
trail) — this file is a session/continuation log only, safe to delete once its content is
stale or fully absorbed elsewhere.

## What changed this session

The user made the ZSoil preprocessor's source code available, which turned most of
`inp-file-format.md`'s `(unclear)`/`(unconfirmed)` guesses into source-confirmed facts —
and caught several outright wrong guesses along the way. Source trees used:

- `C:\Users\mprei\Perforce\GEODEV_MP-LNV-NOV25_2294\v26\ZSoil\Z_Prep3D\` — preprocessor GUI/geometry/element/BC/load code.
- `C:\Users\mprei\Perforce\GEODEV_MP-LNV-NOV25_2294\v26\ZSoil\zmate\` — material/analysis-control library.
- `C:\Users\mprei\Perforce\GEODEV_MP-LNV-NOV25_2294\v26\ZSoil\H\constant.h` — shared enums.

Method throughout: grep the source for the literal marker/tag string (e.g. `".gsh"`,
`"ELAS->"`), read the writer (usually more reliable than the reader for field order), cross-check
against the doc's own existing example values for self-consistency, then update
`inp-file-format.md` with the fact and `verification-notes.md` with the file:line trail.

### Round 1 — six parallel Explore agents (materials, header/control blocks, elements/BC/geometry)
Resolved nearly every flagged field across §3.2–§3.5, §4 (materials — `BUTTONS=` full 14-flag
table, `ELAS->`/`FLOW->`/`DENS->`/`CREEP->`/`HEAT->`/`HUMID->`/`INIS->`/`STAB->`/`NONL->`
Fiber Shell, `LAYERED_BEAM_COMPONENTS`, `SIG_EPS_FUN`), §5 (`EXIST_FUNC`/`LOAD_FUN`), §7
(elements — corrected `rm1`/`rm2` → `mat1/mat2/mat3` = `InitialMaterial`/`ReplacementMaterial1`/`2`;
corrected the universal trailing `LF` → `ULF`, an unloading-function reference into the *same*
`LOAD_FUN` list, not a separate function family; B8 face table now source-exact; `.ibg`/`.itg`
"unclear" fields resolved), §8.1 (`.inb` `ULF` + velocity/acceleration BC codes), §12.1 (`.brc`).
Plan file used: `C:\Users\mprei\.claude\plans\please-create-a-documentation-jolly-dahl.md`.

### Round 2 — individual marker investigations (user-directed, one at a time)
- **`.gsh`**: not "general shell" — `HingesManager::writeAttachedElementsOn`, a per-shell-face
  cache of opposite continuum elements lacking an explicit contact.
- **`.ily`**: not shell-related at all — `Layers::WriteTo`, the construction/display layer name
  table that every element's trailing `iLayer` field indexes into. **Moved to a new chapter**,
  `## 14. Other Data` (old chapters 14/15 renumbered to 15/16 — Quick Reference, Worked Example).
- **`.eie`**: "excluded from interface element" (comment at the call site) — elements flagged to
  skip ZSoil's automatic `.icg`/`.ics` contact generation. New §13.3.
- **`.spg`/`.svg`/`.scg`**: confirmed subdomain-level counterparts of `.ipg`/`.ivg`/`.icg`
  (literally the same writer function, called on subdomain faces instead of meshed faces).
- **`.gos`**: corrected — it's read as well as written, not "write-only" as previously claimed.
- **`.crc`**: contact on beam/truss elements (`ContactRC`); its *reader* is dead code (wrapped in
  `/* */` in `readINP`), so `.crc` content is silently dropped on file reopen even though it's
  still written on save.
- **`.cld`**: node-to-face mesh-tying (`ContactLD`: contactor nodes tied to master faces),
  distinct from `.fac`/`.mrt`.
- **`.idv`**: turned out to be the *real* match for header count field 82 ("number of initial
  velocities/accelerations") — full record layout traced to `NodalIniVelAcc::writeElementCmpOn`.
  Per DOF (6 total): `[lockU,valU,efU,lfU]` then `[lockV,valV,efV,lfV]`. **Naming surprise**:
  despite the class/marker name, the GUI dialog binds `valU`→`m_Dep` (French "Déplacement") and
  `valV`→`m_Vel` — so it's initial **displacement + velocity**, no acceleration field at all.
- **Correction found as a side effect**: `.ivd`/`.svd` — previously documented *as* "initial
  velocities/accelerations" — are actually **viscous dampers** (`ViscDamp` elements on element
  faces), unrelated to initial conditions. Moved their content from §11.4 to §13.4; §11.4 rewritten
  around `.idv`.

All of the above are logged with exact file:line citations in `verification-notes.md`.

## Current document structure (post-renumbering)

`inp-file-format.md` chapters: 1 Introduction · 2 General Conventions · 3 File Header · 4 Materials
· 5 Existence & Load Functions · 6 Geometry · 7 Elements · 8 Boundary Conditions · 9 Loads ·
10 Masses · 11 Initial Conditions · 12 Reinforcement · 13 Mesh Tying & Domain Reduction
(13.1 `.fac`/`.mrt`, 13.2 `.drz`/`.dre`, 13.3 `.eie`, 13.4 `.ivd`/`.svd`/`.spg`/`.svg`/`.scg`/`.crc`/`.cld`/`.gos`/`.igl`)
· **14 Other Data** (14.1 `.ily`, new this session) · 15 Quick Reference · 16 Worked Example.

## Round 4 — `.inb` full rescan (user asked to re-verify; section had grown large)

Traced the entire write chain (`NodesManager::writeBoundaryConditionsOn` →
`GeomPoint::writeNthNodalCondition`) and found `.inb` is not solely solid/displacement fixity
data — the same file also embeds nodal water/heat/humidity **BC** (type `3`) and **flux** (type
`8`) records in a completely different 18-token shape, previously entirely undocumented. Also
corrected the trailing field on every fixity record: it's a real `.ilb` record **number**, not a
binary 0/1 flag as previously claimed. Both now in §8.1, with the type-3/8 addition as a new
subsection. Left one open, flagged question: `.iwb`/`.iab`/`.imb`/`.iwf`/`.ihf`/`.iuf` (§8.4) look
like they may duplicate the same data via a separate writer — both code paths are confirmed live,
but not resolved which (if either) is authoritative. See `verification-notes.md` for the exact
trace and a concrete next step (compare a real file's `.inb` type-3 entries against its `.iwb`
entries for the same node).

## Round 5 — corpus spot-check (`zsoil_inp_files`)

Scanned the real ~90-file sample corpus directly, mainly to chase the `.inb`/`.iwb` open
question from Round 4. Found it: `boxw1.inp`/`boxw2.inp`/`boxw3.inp` are the only corpus files
with a populated water BC, and in all three, `.gwb`'s gradient BC shows up verbatim as `.inb`
type-3 records while `.iwb` stays empty (header count `0`) — real evidence that `.inb`'s embedded
mechanism is what's actually used, not `.iwb`. Also empirically confirmed a version-gating detail
(older files write 4 fields per water/heat/humidity quantity, not 5 — no `ULF`) and did a quick
`.idg`-stays-blank sanity check across 3 files. §8.1 and §8.4 updated with the real example and
citations; `verification-notes.md` has the full trace including what's still not fully pinned
(the exact meaning of `.gwb`'s 3rd/4th header fields).

## Known remaining gaps (not yet chased)

From the plan's "explicitly not resolved" list, still open:
- `T3D_PARAM_*` literal enum values (ordering confirmed via `constant.h`, values themselves not
  needed since ordering is what matters for field position).
- Per-continuum-model `NONL->` field layouts (Mohr-Coulomb, Hoek-Brown, Drucker-Prager, Cam-Clay,
  Hujeux, HS-small-strain, Duncan-Chang, Cap model) — classes located
  (`Mohr_Coulomb.cpp`, `Hoek_Brown.cpp`, etc., all `GenericNonLinearGroup` subclasses in
  `dataNonLinear.h`) but not individually field-mapped.
- `.pil`'s `code` field, `.icg`'s `genFullContinuity` *setter* semantics (declared/written, no
  setter/enum found in this checkout).
- Beam-material `INIS->` (continuum's is fully resolved; beam's isn't located).
- `.bcb`'s trailing `0 0` line — the geometry-catalog writer itself appends nothing after its
  last entry, so this may belong to a different class (`CableSets.cpp`/`BeamCableDef`) — not
  confirmed either way.

**Round 3 (this update)**: `.idg` and the `.ghf`/`.guf`/`.gab`/`.gmb`/`.gwb`/`.gwf` cluster near
§8.4 are now resolved too.
- `.ghf`/`.guf`/`.gab`/`.gmb`/`.gwb`/`.gwf`: all confirmed writers — each pairs with an
  already-documented nodal-level (`i`-prefixed) marker (`GetSurfHeatBCManager`/
  `GetNodalHeatBCManager` etc., `geometricalmodel.cpp:1256-1291`). **Corrected after user
  feedback**: this is not a pre-mesh/subdomain-level vs. meshed distinction (unlike `.ipg`/`.spg`,
  §13.4) — both tiers act on the already-meshed model. The `g`-marker is a gradient/pattern BC
  (`SurfBCManager`, up to 4 reference-node values distributed across a face selection, same
  writer as `.gsl`'s surface loads), the `i`-marker a flat per-node list like `.inb`. §8.4 and the
  six Quick Reference rows updated accordingly.
- `.idg`: genuinely has **no writer anywhere** in the available source — confirmed by finding the
  actual assembly mechanism instead. `GeometricalModel::writeOn` writes most markers to separate
  per-extension fragment files; `ZMATE_SaveINP`/`ZMATE_CompressFiles` (now visible in
  `zmate\zmate.cpp`, previously thought to be compiled-only) concatenate them in an order loaded
  at startup from an external config file, `CFG\prepro.fil` (not present in this Perforce sync).
  A blanket stub-creation step (`ZMATE_CreateNoExistingPreproFile`) empty-creates any fragment file
  in that list that nothing else has written — which is what `.idg` (and presumably a few other
  perpetually-empty markers) actually is. Documented as a general mechanism in §2.2, not just a
  one-off fact about `.idg`.
`.ivd`/`.svd`'s own field-level record layout (the `ViscDamp` object's write function itself)
wasn't traced — only the mechanism/manager level.
`.crc`/`.cld`'s exact per-record field layout wasn't traced beyond the mechanism level either
(`ContactRC`/`ContactLD`'s own `writeOn` bodies not read).

## State / next steps

Both `inp-file-format.md` and `verification-notes.md` are modified but **uncommitted** (consistent
with this project's established practice — ask before committing). Natural next steps, roughly
in order of likely value:
1. Sweep the remaining `.ghf`/`.guf`/`.gab`/`.gmb`/`.gwb`/`.gwf`/`.idg` cluster the same way.
2. Field-map one or two of the per-continuum-model `NONL->` layouts (Mohr-Coulomb is probably the
   highest-value one, being the most commonly used nonlinear soil model).
3. Trace `ContactRC`/`ContactLD`'s own write functions for full `.crc`/`.cld` field layouts.
4. Spot-check a few of this session's biggest reinterpretations (`mat1/mat2/mat3`, the `LF`→`ULF`
   correction, `.idv`'s displacement/velocity split) against a real populated sample file, if one
   turns up — none of the corrections above have a populated real-file example to verify against yet.
