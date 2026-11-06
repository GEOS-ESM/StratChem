# AGENTS.md - StratChem

Gridded component `StratChem_GridCompMod` (MAPL2 code; child name `STRATCHEM`):
NASA GSFC stratospheric chemistry. It is a child of the GEOS Chemistry
component (`GEOS_ChemGridComp`), enabled by `ENABLE_STRATCHEM`. One outer
MAPL component wraps a legacy Fortran module (`SC_GridComp/SC_GridCompMod.F90`)
and about 34 fixed-form `.F` kernels (plus `localZeroSet.F90`).

**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `StratChem` @ `db11b02` (2026-11-06).
**Placement:** GEOSgcm fixture (`components.yaml` @ `f50622d`), key `StratChem`.

**Build status on this line:** not built, not ported. On `release/MAPL-v3`, `@GEOSchem_GridComp/CMakeLists.txt` calls `esma_add_subdirectories(GOCART)` only, with the comment "Restore subdirs as components are ported to MAPL3."; `StratChem` appears only in a commented-out list of subdirectories. The code in this repository is the MAPL2 code, kept for a later port; it differs from the MAPL2 `develop` branch only in `SC_GridComp/fallvel.F` (`USE MAPL_Constants` instead of `USE MAPL_ConstantsMod`). The Chemistry top-level library (`GEOS_ChemGridComp.F90`, which adds this component) is present but not compiled on this line: the `esma_add_library` block that builds it is commented out in `../CMakeLists.txt`. Statements about it describe the file as it is. Its build dependency `Chem_Shared` (from the Chemistry `Shared` directory) is also in the commented-out list; the other listed dependencies are `MAPL`, `MAPL_cfio_r4`, `GMAO_mpeu`, `NetCDF::NetCDF_Fortran`.

## Read first
- `docs/ARCHITECTURE.md` - placement, file map, the component in nine sections.
- `docs/decisions.md` - design patterns and their likely reasons.
- Parent chemistry docs (how `STRATCHEM` is added, connected and run):
  `../docs/ARCHITECTURE.md` and `../AGENTS.md`.
- `README.md` is one line; `ChangeLog` (old) and `CHANGELOG.md` (current) list changes.

## Map
- `StratChem_GridCompMod.F90` - MAPL wrapper: SetServices, Initialize_, Run_, Finalize_, extract_.
- `SC_GridComp/SC_GridCompMod.F90` - legacy driver (init, run, finalize, table readers).
- `SC_GridComp/*.F` - kernels (rates, photolysis, solvers, PSC surface areas, sedimentation).
- `SC_GridComp/SC_Registry.rc` - import/export/internal specs; code-generated headers.
- `SC_GridComp/SC_Mech_{Full,Reduced}/SC_Mech_Registry.rc` - species lists; one installed.
- `SC_GridComp/SC_GridComp.rc`, `StratChem_ExtData.{rc,yaml}` - run-time resources.

## Rules for working here
- Build option `STRATCHEM_REDUCED_MECHANISM` (default OFF) selects the registry
  AND defines `REDUCED` for the compiler; both change together. Do not edit one only.
- Species order in the registry fixes the order in the legacy bundle; the
  code assumes `RO3OX` is the last `XX_table` entry (registry header comment).
- `*.F` files are fixed-form; several exceed 72 columns and get `EXTENDED_SOURCE` flags in `CMakeLists.txt`.
- `SC_Registry.rc` generates `StratChem_{Import,Export}Spec___.h` and `GetPointer___.h`; edit the rc, not headers.
- Not built on this line (see build status above). Meant to build with the GEOSgcm build; this repository has no tests of its own.
- Changing exports consumed by `GOCART` or `ACHEM` (`O3CHEM`, `O3P`, `OH`, `OCS_JRATE`)
  requires a matching change in the parent's connectivity.

## Doc rules
Claims in the docs carry `(file, routine)` citations or are marked
*Inferred:* / *Unverified:*. No line numbers. Update the docs with code changes.
