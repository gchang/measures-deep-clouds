# Changes from upstream

Upstream: https://github.com/emwilcox-dri/NASA-MEaSUREs-deep-clouds, branch `main` at commit 97a87eeacee1e7bab1c75603e5e6849c057d3ab1 (2023-11-02), which is also upstream's only tag, `v1.0.0`. Code by Eric Wilcox (Desert Research Institute) for the NASA MEaSUREs project "A Comprehensive Data Record of Marine Low-level and Deep Convective Cloud Systems Using an Object-Oriented Approach". Its output is archived at GES DISC as MOD_L2_DC and MYD_L2_DC V001 (https://doi.org/10.5067/HXRNU4HCIPA6, https://doi.org/10.5067/0Y3M8IO4XKG5).

License: the repository has no LICENSE file. The Zenodo record of this release (https://doi.org/10.5281/zenodo.10067326) lists CC-BY-4.0. All credit for the code belongs to its author.

This copy starts from upstream's file tree at 97a87ee without its git history. Upstream has no `.github/` folder, so nothing was left out. It is unchanged except for the items below.

## 2026-10-25: path buffers are big enough for long folder names

The folders given on the command line were copied into fixed-size stack buffers with `strcpy` and `sprintf`, which do not check the length: `amsrPath_root` and `mod08PathRoot` (`measures_main_v12.c:204-205`) held 50 characters, and the other folders and the folder-plus-file-name buffers built from them (`modPath`, `merraPath`, `outPath`, `maskfile`, and `fname` and `filelist` in `MOD02.c`, `MOD06.c`, `MOD08.c`, `amsr.c` and `merra2.c`) held 150. A longer path ran over into the next buffer. With a 68-character MOD08_D3 folder (the CSS layout on Discover is 67 characters plus the NUL) the overflow overwrote `merraPath`: the program looked for `184/MERRA2_400...nc4`, printed `error opening`, wrote every CAPE, shear and MERRA AOT value as fill, and still exited 0. With folders over 150 characters it crashed before writing anything. Which buffer gets hit depends on the compiler's stack layout, so a build can look fine on one machine and lose MERRA-2 on another.

Every buffer that holds a folder path, or a folder path plus a file name, is now `NCHAR_PATH` (4096) characters, defined in `MOD06.h`, `MOD08.h`, `amsr.h` and `merra2.h`. Buffers that hold only a file name are unchanged. Results do not change for paths that fit the old buffers.

## 2026-10-25: environment.yml for building outside Discover

`ccmeasures_clouds_v12` uses the HDF4 and netCDF installs on Discover. `environment.yml` lists the conda-forge packages that build and run the code: the C and Fortran compilers, HDF4 4.2 with the jpeg and zlib libraries it links against, libnetcdf (for `nc-config`), and Python 3.11 with numpy, pyhdf and nco for `measures_setup.py` and the monthly `ncrcat` step. The README has one new line on using it: set `HDF_INCLUDE` and `HDF_LIBRARY` in the compile script to `$CONDA_PREFIX/include` and `$CONDA_PREFIX/lib`. No code changes.

## Known issues left as is

These are recorded here and not changed, either because a change would alter the archived numbers or because they are part of how the code is run on Discover. Line numbers are in `measures_main_v12.c` unless named.

- **6-hour CAPE change before 06 UTC** (`:1500-1504`): for the first MERRA-2 time of the day (`it==0`) the loop reads `cape_merra[(it-1)*...]`, before the start of the array. The earlier loop at `:1378-1466` guards `it==0`, this one does not. In the archived 2016-07 Terra file, clouds from granules at 00:00 to 05:55 UTC have `change_of_CAPE` equal to CAPE/6 (median ratio exactly 1.0). Fixing it (fill, or the previous day's 18 UTC field) changes the archived values, so it is left for a new product version.
- **Shear has no fill check** (`thermo.c:432`, called at `:1374`): where 850 hPa is below the ground, the MERRA-2 winds are fill, and the shear comes out near 14,000 m/s. The archived file has -9999 for those clouds.
- **Dateline longitude index** (`:1477`): `grid_lons[i]` should be `grid_lons[j]`. It affects MERRA-2 lookups only for granules that cross 180 degrees.
- **Land fraction is fill whenever AMSR-E is missing** (`:1031`, inside `if (amsr_fileflag)`), which covers every granule after October 2011.
- **Granule time is truncated** (`:1009`), so some times are one minute early.
- **Minor:** the AMSR test at `:702` uses `||` where `&&` is probably meant (harmless at the 55-minute window); `mask.c:20` calls `atoi` on an unterminated one-character buffer; `modis_bright` (`MOD02.c:122`) uses Terra coefficients for Aqua as well.
- **Missing inputs do not give a nonzero exit code.** A missing MOD03 file, an unreadable MOD06 or MOD03 file, or no clouds stop with `exit(0)`; missing MERRA-2, MOD08_D3 or MOD021KM files give fill values and exit 0. A missing file list gives a segmentation fault (`MOD06.c:933,969`, `MOD08.c:283`, `amsr.c:156` open the lists without a NULL check). Checking the output file is the only guard.
- **The Python driver is set up for Discover and for Aqua.** `measures_slurm_call_css` has the Discover Python in its first line, SLURM settings, and hard-coded folders, years and months, and it calls `measures_clouds` while the compile script installs `measures_clouds_deep` (line 54). `measures_setup.py` uses Aqua folder names (`MYD06_L2`, `MYD021KM`, `MYD03`, `MYD08_D3`) and `write_filelists_css` writes only `myd02list` and `myd03list`, while Terra granules need `mod02list` and `mod03list`. The compile script names the Discover HDF4 folders and installs to `~/bin`. The step that renames and reformats the per-granule files into the archived monthly format is not in the repository.
- **MERRA-2 file names** are built from the stream number (`MERRA2_300` for 2001 to 2010, `MERRA2_400` after, `:1232-1238`), so reprocessed months such as `MERRA2_401` are not found.

## Check

On 2026-10-25, from a fresh local clone in a new conda environment made from `environment.yml` (gcc and gfortran 15.3.0, hdf4 4.2.15, libnetcdf 4.10.1), the compile script with the two HDF lines and the install folder set as the README says built the program in about 1 s with the same 8 old-style-definition warnings as before. Test granule: Terra MOD06_L2.A2016184.1700 (2016-07-02 17:00 UTC, eastern Pacific) with its MOD03, MOD021KM, MOD08_D3 and the day's two MERRA-2 files (1.08 GB), staged in the folder layout of `measures_setup.py`, no AMSR-E.

- Short folders (11 to 36 characters): 349 cloud objects, about 11 s and 0.65 GB peak memory, exit 0. The output file is byte-identical to the one from a build of the unchanged upstream code, and its comparison with the archived GES DISC file (`MODIS_Terra_L2_DC.V001.201607.nc`, the 349 records of this granule) is unchanged: 18 of 26 variables identical value for value, CAPE within 0.05 J/kg.
- Every folder over 150 characters (172 to 197): MERRA-2 is read, and the output is byte-identical to the short-folder run.
- The unchanged upstream code with the same long folders crashes with a segmentation fault and writes nothing. With a 68-character MOD08_D3 folder it prints `error opening 184/MERRA2_400...`, writes all 349 clouds with CAPE, shear and MERRA AOT as fill, and exits 0. The fixed build with the same 68-character folders gives the byte-identical output.
