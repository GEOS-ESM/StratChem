# StratChem architecture

**Audience:** coding agents and developers working on StratChem.
**Describes:** the MAPL2 line, branch `develop`.
**Verified against:** `StratChem` @ `cbf1c41` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `0f2e0fb`), key `StratChem`, local path `src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@StratChem`, `develop: develop`.
Other fixtures (for example GEOSadas, GEOSldas) may place this repository
elsewhere; paths inside this repository do not change.

Parent: [`../docs/ARCHITECTURE.md`](../../docs/ARCHITECTURE.md) (the Chemistry component; its sections 2.3 and 3 cover how children are added, connected and run). This document does not repeat them. See also [`../AGENTS.md`](../../AGENTS.md) and this repository's [`AGENTS.md`](../AGENTS.md), [`decisions.md`](decisions.md).

## Directory and file map
| Path | Role |
|---|---|
| `StratChem_GridCompMod.F90` | MAPL wrapper `StratChem_GridCompMod`: `SetServices`, `Initialize_`, `Run_`, `Finalize_`, `extract_` |
| `SC_GridComp/SC_GridCompMod.F90` (~3,700 lines) | Legacy module `SC_GridCompMod`: type `SC_GridComp`, `SC_GridCompInitialize/Run/Finalize`, internal table readers (`readPhotTables`, `initClimVars`, `readClimVars`, `readEMSFactor`, `mapToGrid`, `mapToSurf`), `SSGSfcFlx`, `swapSpecies`, `chemStage1/2` |
| `SC_GridComp/*.F`, `localZeroSet.F90` | Kernels, one subroutine per file; groups listed below the table |
| `SC_GridComp/SC_Registry.rc` | Import, export and internal specs (`<ImportSpec name="STRATCHEM">` etc.) for the code generator |
| `SC_GridComp/SC_Mech_Full/`, `SC_Mech_Reduced/` | `SC_Mech_Registry.rc`: species tables `SC_table::` (transported) and `XX_table::` (non-transported) |
| `SC_GridComp/SC_GridComp.rc` | Run-time switches and table file names |
| `SC_GridComp/StratChem_ExtData.rc`, `.yaml` | ExtData specs for ODS emission fluxes, OH climatology and the sulfate surface-area imports |
| `CMakeLists.txt` | Library, code generation (`-N STRATCHEM`), option `STRATCHEM_REDUCED_MECHANISM`, installs resource files into `etc` |
| `ChangeLog`, `CHANGELOG.md`, `README.md` | History (old and current) and a one-line README |

Kernel groups (*Inferred:* from file names; only the headers of `sadsts`, `sediment` and `pscs` were read):
- Rates and photolysis: `rdkrate`, `krates`, `jcalc4`, `meso_phot`, `o3colum`, `sza`, `calc_qqjk`.
- Family partitioning and solvers: `oxpandl`, `hoxpart`, `ncpandl`, `brpandl`, `chxpart`, `part`, `solverd2`, `solverest`, the `*est` variants, `sproic`.
- Heterogeneous chemistry and PSCs: `khet3d`, `gamma_shi`, `pscs`, `sadnat`, `sadsts`, `sadice`, `ternary`, `calcsulf`, `sediment`, `fallvel`.
- Bookkeeping: `getBaseChem`, `storeBaseChem`, `interp_s`, `density`, `verifyChemDt`, `localZeroSet`.

## 1. Purpose
StratChem integrates the NASA GSFC stratospheric chemistry from the tropopause upward: odd oxygen, NOx, chlorine and bromine
families, halocarbons, CH4, N2O, PSC and sulfate heterogeneous chemistry, age of air. It exports ozone and oxidants to other
chemistry children (`StratChem_GridCompMod.F90`, `SetServices`; `SC_GridComp/SC_GridCompMod.F90`, header).
It is an optional child of the Chemistry component.

## 2. Place in the hierarchy
- Parent: `GEOSchem` (`GEOS_ChemGridComp.F90`). Added with `MAPL_AddChild(GC, NAME='STRATCHEM', SS=StratChem_SetServices)` when `ENABLE_STRATCHEM:` is true (default false) (`GEOS_ChemGridComp.F90`, SetServices). The parent has one phase for this child (see parent section 3).
- Children: none. `SC_GridComp` is a plain Fortran type and module called from the wrapper, not a MAPL/ESMF component (`StratChem_GridCompMod.F90`, `Initialize_`, `Run_`).
- Parent wiring involving `STRATCHEM` (`GEOS_ChemGridComp.F90`, SetServices):
  - `CHEMENV` to `STRATCHEM`: `AIRDENS`, `AIRDENS_DRYP`, `DELP`, `CN_PRCP`, `NCN_PRCP`, `LFR`.
  - `STRATCHEM` to `GOCART`: `O3CHEM` as `O3`, when both are enabled.
  - `STRATCHEM` to `ACHEM`: `O3P`, `OH` (as `OHSTRAT`), `O3CHEM` (as `O3`), `OCS_JRATE`.
  - `CARMA` to `STRATCHEM`: `CARMA_SUSAREA`, `CARMA_SUSAREAv` as `SO4SAREA`, `SO4SAREAvolc`, when the aerosol provider is `CARMA`.
  - `GOCART` to `STRATCHEM`: `SO4SAREA` (and `SO4SAREAvolc` for the volcanic sulfate species), plus `GOCART::DMS/SO2/SO4/MSA` and the `v` variants (see Observations).
  - Ozone-provider selection by `ANALYSIS_OX_PROVIDER:` (parent section 3).
  Other imports not listed (`T`, `Q`, `PLE`, `TROPP`, `LWI`, ...) are not connected in the lines read; *Inferred:* they come from the parent's own import state through the generic parent-to-child import mapping.
- Friendly tracers (`StratChem_GridCompMod.F90`, `SetServices`):
  - transported species are `FRIENDLYTO` `DYNAMICS:TURBULENCE:MOIST` plus the component name, so they enter `CHEM_TRACERS` as `STRATCHEM::<name>` (parent section 3);
  - `OX` also gets `ANALYSIS` when `ANALYSIS_OX_PROVIDER:` is `STRATCHEM`;
  - `HBR`, `HOBR`, `BRONO2` get only `DYNAMICS:TURBULENCE` (not `MOIST`).

## 3. Entry points
All in `StratChem_GridCompMod.F90`; one phase each, registered with `MAPL_GridCompSetEntryPoint`.
- `SetServices`, in order:
  - read `SC_Mech_Registry.rc` twice with `Runtime_RegistryCreate` (`SC_table::`, then `XX_table::`);
  - include the generated import spec, then add the extra imports `SO4SAREAvolc`, `SO4SAREA`;
  - add one internal spec per registry species (transported, then non-transported);
  - combine the registries into `state%scReg`;
  - add `AERO`, `AERO_DP` exports only if `AERO_PROVIDER:` is `STRATCHEM`, then include the generated export spec;
  - add timers `INITIALIZE`, `RUN`, `FINALIZE`; call `MAPL_GenericSetServices`.
- `Initialize_`, in order (`StratChem_GridCompMod.F90`, `Initialize_`):
  - `MAPL_GenericInitialize`, then `extract_`;
  - read `AREA`; create the legacy bundle `bsc` with `Species_BundleCreate(..., skipAlloc=.true.)`;
  - assert `size(InternalSpec) == scReg%nq`;
  - for each `L`, `MAPL_GetPointer(internal, NAME=short_name, ptr=bsc%qa(L)%data3d)` with `short_name` from `InternalSpec(L)`
    (so the bundle arrays are the MAPL internal fields, matched by position in the internal spec list);
  - fail if `AERO_PROVIDER:` is `STRATCHEM`; call `SC_GridCompInitialize`.
- `Run_`, in order (`StratChem_GridCompMod.F90`, `Run_`):
  - `MAPL_SunGetInsolation` for `SLR`, `ZTH`; `extract_`; read `PLE`, `DELP` (`bsc%delp = DELP`);
  - tropopause: where `TROPP /= MAPL_UNDEF` store `TROPP` in the bottom layer of `RO3OX`; where `TROPP > SC_BASE_PRESSURE`
    (default 40000 Pa) store that cap; stop with an error if any bottom-layer value is still `MAPL_UNDEF`;
  - fill `SCTROPP`, `AGCMTROPP` if requested;
  - if any `<species>_SCTEND` export is requested, save the initial transported fields;
  - call `SC_GridCompRun` `SC_SPLIT` times with `cdt = RUN_DT/SC_SPLIT`;
  - add `RUN_DT/86400` to `AOADAYS` and zero its bottom layer;
  - compute `SCTO3`, `SCTTO3` (total ozone; the second weighted by the part of each layer below the stored tropopause pressure) if requested;
  - fill tendencies `(after - before)/RUN_DT`.
- `Finalize_`: `extract_`; `SC_GridCompFinalize`; destroys the bundle and registry; `MAPL_GenericFinalize`.
- `extract_`: retrieves the private `StratChem_state` (registry, legacy state, bundle), reads `RUN_DT:` and `SC_SPLIT:` (default 1), converts the clock to `nymd`, `nhms`.
- `SC_GridCompRun` (`SC_GridComp/SC_GridCompMod.F90`), order as read in the routine:
  - read imports; `readEMSFactor`, `readClimVars`, `mapToGrid` / `mapToSurf` for emissions and climatologies;
  - loop over columns (`i`, `j`): `sza`, then loop over levels:
    - `krates`; `swapSpecies(FROM_BUNDLE)`; `SSGSfcFlx` at level 1 (under `doFlux`, *Inferred:* from the surrounding conditions);
    - `meso_phot`; `storeBaseChem` / `getBaseChem` (call order verified; the conditions under which they run are *Unverified:*);
    - `chemStage1` (calls `pscs`, `khet3d`, `part`, `oxpandl`, `solverest`) and `chemStage2` (calls `partest`, `oxpandlest`, `solverd2`);
    - `swapSpecies(TO_BUNDLE)`; `sediment` when `doPSCs` and `doSediment`;
  - after the column loop: `WetRemovalGOCART` and `convection` for the bromine species (`WetRemovalMod`, `ConvectionMod`).
  Chemistry details are not documented here.

## 4. State
- Imports: generated from `SC_GridComp/SC_Registry.rc` (`StratChem_ImportSpec___.h`); `SO4SAREA`, `SO4SAREAvolc` are added in `SetServices`.
  - Grid and thermodynamics: `DELP`, `PLE`, `T`, `Q`, `QCTOT`, `AIRDENS`, `AIRDENS_DRYP`, `TROPP`, `AREA`, `LWI`, `ZPBL`, `ZLE`, `FCLD`.
  - Surface fractions: `FRLAKE`, `FROCEAN`, `FRACI`.
  - Moisture and convection: `LFR`, `CN_PRCP`, `NCN_PRCP`, `CNV_QC`, `CNV_MFC`, `CNV_MFD`, `PFL_LSAN`, `PFI_LSAN`, `DQDT`.
  - ODS fluxes (no-restart): `F11FLUX`, `F12FLUX`, `F113FLUX`, `MCFFLUX`, `CCL4FLUX`, `HCFC22FLUX`, `CHBR3FLUX`, `CH2BR2FLUX`; also `OH3DClim`.
- Exports: generated (`StratChem_ExportSpec___.h`).
  - Diagnostics: `SZARAD`, `SCBASEK`, `NATSAD`, `ICESAD`, `SCTROPP`, `AGCMTROPP`, `SCTO3`, `SCTTO3`; surface fluxes `*FLX`.
  - Ozone: `O3` (kg/kg) and `O3PPMV` per the comment in `SetServices`.
  - Tendencies: `OX_TEND`, `H2O_TEND`, one `<species>_SCTEND` per transported species.
  - Rates and others: `QQJ001...`, `QQK001...`, `OCS_JRATE`, `rDryAir`, ocean/land loss diagnostics.
  - Optional `AERO`, `AERO_DP` (see traps).
- Internal state (`SetServices`): one 3-D field per `SC_table::` species (transported, friendly) and per `XX_table::` species
  (`FRIENDLYTO` = component name, `ADD2EXPORT`, so exported under the same name).
  - Non-transported species include `O3CHEM`, `O3P`, `O1D`, `OH`, `HO2`, radicals, `CH2O`, `RO3OX` (`SC_Mech_Registry.rc`).
  - Restart: generic MAPL internal restart; the parent template keeps the `STRATCHEM` restart keys commented out (parent section 3).
- Private Fortran state: `StratChem_state` (wrapped, stored with `ESMF_UserCompSetInternalState`) holds the combined registry, the `SC_GridComp` legacy state and the bundle `bsc` (`StratChem_GridCompMod.F90`, `SetServices`, `extract_`).
- Species order: the legacy bundle index equals the order of `SC_table::` then `XX_table::` (`Runtime_RegistryCombine`).

## 5. Configuration that changes structure
- `STRATCHEM_REDUCED_MECHANISM` (CMake, default OFF): defines `REDUCED` and selects `SC_Mech_Reduced` (`CMakeLists.txt`).
  - In the reduced registry, 19 species move from `SC_table::` to `XX_table::` (`AOADAYS` stays in `SC_table::`):
    `CFC114`, `CFC115`, `HCFC141B`, `HCFC142B`, `H1202`, `H2402`, `CHBR3`, `CH2BR2`, `CH2BRCL`, `CHBRCL2`, `CHBR2CL`,
    `HFC23`, `HFC32`, `HFC125`, `HFC134A`, `HFC143A`, `HFC152A`, `CO2`, `SF6` (`SC_Mech_*/SC_Mech_Registry.rc`).
  - Code in `#ifndef REDUCED` blocks is excluded. Import/export specs and ExtData do not change, so related exports
    exist but are not valid (`SC_GridCompMod.F90`, header comment).
- Parent keys: `ENABLE_STRATCHEM`, `ANALYSIS_OX_PROVIDER` (`OX` friendlies, `O3` export priority), `AERO_PROVIDER` (STRATCHEM is refused by `Initialize_`).
- Component keys (`SetServices`, `extract_`, `Run_`): `RUN_DT`, `SC_SPLIT`, `SC_BASE_PRESSURE`.
- `SC_GridComp.rc` (read with `I90_label` in `SC_GridCompInitialize`):
  - switches: `doPSCs`, `doSediment`, `doFlux`, `useSolCyc`, `spinup`, `importSulfateSA`, `addSAclim`;
  - year adjustments: `SO4saYr`, `GHGYrAdj`, `ODSYrAdj`; files: `KRateTables`, `photolysisFile`, `climatologiesFile`, `solarCycleFile`; numeric cutoffs.
  - `importSulfateSA` on takes the SO4 surface area from the `SO4SAREA`/`SO4SAREAvolc` imports instead of the climatology,
    or added to it when `addSAclim` is set (`SC_GridCompRun`).

## 6. Dependencies
- Libraries (`CMakeLists.txt`): `Chem_Shared`, `MAPL`, `MAPL_cfio_r4`, `GMAO_mpeu`, `NetCDF::NetCDF_Fortran`.
- Modules used: `Runtime_RegistryMod`, `Species_BundleMod`, `Chem_Mod`, `Chem_StateMod`, `Chem_ConstMod`, `Chem_UtilMod`, `m_inpak90`, `WetRemovalMod`, `ConvectionMod` (`SC_GridCompMod.F90`), from the Chemistry shared code, not from this repository.
- Data files read on the root and broadcast (`readPhotTables`, `initClimVars`, `readClimVars`, `readEMSFactor`): `KRateTables`, `photolysisFile`, `climatologiesFile` (all under `ExtData/g5chem/...`), via NetCDF with optional parallel I/O (`H5_HAVE_PARALLEL`).
- Build-time code generation: `SC_Registry.rc` to `StratChem_{ExportSpec,ImportSpec,GetPointer}___.h` and `StratChem_History___.rc` (`CMakeLists.txt`).
- Consumers of exports: `GOCART` (`O3CHEM`), `ACHEM` (`O3P`, `OH`, `O3CHEM`, `OCS_JRATE`) (parent wiring above); transported species also reach dynamics and turbulence as friendly tracers.

## 7. Known traps
- The registry and the compile definition are separate: the registry choice and `REDUCED` come from one CMake option, but an experiment that copies a different `SC_Mech_Registry.rc` would desynchronize them (`CMakeLists.txt`). *Inferred.*
- `INTERNAL` count must equal the registry species count (assert in `Initialize_`).
- `RO3OX` last in `XX_table::` and its bottom layer stores the working tropopause pressure (`Run_`; registry header).
- Vertical index: the wrapper stores fields top-down in the bundle and `SC_GridCompRun` reverses them (`km:1:-1` indexing). *Inferred:* from the reversed indexing.
- Kernels are fixed-form with some lines above 72 columns; compiler flags are set per file in `CMakeLists.txt`.
- `AERO`, `AERO_DP` exports exist only when `AERO_PROVIDER` is `STRATCHEM`, which `Initialize_` rejects.

## 8. Decisions
See [`decisions.md`](decisions.md).

## 9. Observations (noted, not fixed)
- `SetServices` declares `AERO`/`AERO_DP` exports for `AERO_PROVIDER: STRATCHEM`, while `Initialize_` stops with an error for the same setting. Question for maintainers: is the `AERO` branch kept intentionally?
- The `SC_GridCompMod.F90` header says `-DREDUCED` is added to a `GNUmakefile` and that `stratchem_setup` rewrites `Chem_Registry.rc`; `CMakeLists.txt` now selects the registry and the definition. The text predates the CMake option (*Inferred:*).
- The parent connects `GOCART::DMS`, `GOCART::SO2`, `GOCART::SO4`, `GOCART::MSA` (and `v` variants) to `STRATCHEM`; `SC_Registry.rc` and `SetServices` declare no imports with these names (grep of this repository). *Unverified:* how MAPL treats connectivity to a missing destination name.
- `SC_GridCompRun` has commented-out `SO4SAimp` code and `gcrProfile` is zero in both branches of `useSolCyc` (comment says it awaits implementation).
- `SC_Registry.rc` import long names for `F12FLUX` and `F113FLUX` are `flux_of_cfc11`.
- `Run_`: `PRINT *` then `STATUS = 1` for invalid tropopause pressure; `ChangeLog` is not updated past 2019 (a separate `CHANGELOG.md` exists).
- Parent document: its section 3 lists `STRATCHEM` in the friendly-tracer targets; the code confirms this. No contradiction found.
