# StratChem decisions

**Audience:** coding agents and developers working on StratChem.
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `StratChem` @ `db11b02` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `f50622d`), key `StratChem`.

See [`ARCHITECTURE.md`](ARCHITECTURE.md). Reasons are inferred from the code only; no history was mined.

### Outer MAPL wrapper around a legacy module
- Pattern: `StratChem_GridCompMod` owns specs and MAPL calls; `SC_GridCompMod` is plain Fortran called from `Initialize_`, `Run_`, `Finalize_` with the import/export states and a species bundle (`StratChem_GridCompMod.F90`; `SC_GridComp/SC_GridCompMod.F90`). [Verified]
- Likely reason: chemistry code predates MAPL and was kept. [Inferred]

### Species bundle shares MAPL internal arrays
- Pattern: `Initialize_` creates the legacy bundle without allocating and points each entry at the matching MAPL internal field (`StratChem_GridCompMod.F90`, `Initialize_`). [Verified]
- Likely reason: legacy code reads and writes species in place; transport and restart see the same memory. [Inferred]

### Species lists live in a registry file chosen at build time
- Pattern: `SC_Mech_Registry.rc` supplies `SC_table::` and `XX_table::`; CMake installs the Full or Reduced file and defines `REDUCED` (`CMakeLists.txt`). [Verified]
- Likely reason: the reduced mechanism lowers wall-clock for data assimilation (`SC_GridCompMod.F90`, header comment). [Verified, comment]

### Non-transported species exported through the internal state
- Pattern: `XX_table::` species are internal specs with `ADD2EXPORT` and friendly only to the component (`SetServices`). [Verified]
- Likely reason: other components read radicals and ozone as exports without transporting them. [Inferred]

### Chemistry applied down to a capped tropopause pressure
- Pattern: `Run_` stores `TROPP` in the bottom layer of `RO3OX` where `TROPP` is not `MAPL_UNDEF`, and stores `SC_BASE_PRESSURE` where `TROPP` exceeds it; where `TROPP` is `MAPL_UNDEF` the layer keeps the value it already had (initial or restart), and `Run_` stops with an error if any value is still `MAPL_UNDEF` (`Run_`). [Verified]
- Likely reason: handle occasional undefined or anomalously high imported `TROPP` (comments in `Run_`). [Verified, comment]
- Question for maintainers: is storing the working value in a species field intended to stay?

### Sub-stepping inside Run
- Pattern: `SC_SPLIT` repeats `SC_GridCompRun` with `RUN_DT/SC_SPLIT` (`Run_`, `extract_`). [Verified]
- Likely reason: none recorded.
