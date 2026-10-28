# COAWST compile and run guide

[中文](user-guide.zh-CN.md) · [Home](../README.md) · [中文首页](../README.zh-CN.md)

This guide explains how to compile and run **COAWST** (Coupled-Ocean-Atmosphere-Wave-Sediment Transport Modeling System) v3.9 on Linux. It covers dependency installation, environment variables, application configuration, compiling and short test runs of INLET_TEST (ROMS+SWAN) and JOE_TC Coupled (WRF+ROMS+SWAN), and common failure points.

In speech, cowast / COWAST both refer to the same USGS coupled system. The official spelling is **COAWST**.

## 1. Getting started

This section states the version this guide applies to, where the source comes from, and what was verified on this machine. Paths, version numbers, and wall-clock times in later sections all come from that same measured record.

This guide matches **COAWST v3.9**. The source repository is `https://github.com/DOI-USGS/COAWST.git`. The commit that was measured is `81df2ed` (full hash `81df2edcd175fb87426b4d913f4fdaa42cfc50ad`). The host was Debian 13 (trixie), x86_64, 8 CPUs, with about 110G free under `/workspace`. The verification date was 2026-10-28 (America/Los_Angeles).

Written for readers who already know how to compile and run WRF. On this machine the dependencies and MCT were installed, and **INLET_TEST** (ROMS+SWAN Coupled) and **JOE_TC Coupled** (ROMS+SWAN+WRF) were compiled and given short coupled test runs. **Sandy and JOE_TC DiffGrid have not been built on this machine.**

Get the source with git.

```bash
cd /workspace
git clone --depth 1 https://github.com/DOI-USGS/COAWST.git
# Result: /workspace/COAWST (about 1.8G), commit 81df2ed
```

The GitHub clone succeeded on this machine; USGS GitLab was not used as a fallback. Alternate URL: `https://code.usgs.gov/coawstmodel/COAWST`.

The repository includes the user manual `/workspace/COAWST/COAWST_User_Manual.docx`, plus directories such as ROMS, WRF, WPS, SWAN, WW3, InWave, Master, Lib, Compilers, and Projects. For a coupled application, do not compile WRF by itself with `./compile em_real`. Let `build_coawst.sh` drive the build.

> **Note**
>
> The license is USGS public domain / CC0 1.0. The standard ROMS–SWAN–WRF–WW3 path does not depend on proprietary libraries.

## 2. Prerequisites

Compiling COAWST needs one Fortran / MPI stack all the way through NetCDF, MPI, MCT, and the main program. If any piece is missing, you get module mismatches or link failures.

You need a Fortran compiler (gfortran), C/C++, MPI (OpenMPI), NetCDF-C + NetCDF-Fortran (NetCDF4), HDF5, cmake, make, perl, and git. Coupling also needs **MCT** (the repository bundles `Lib/MCT`, which must be built first). Applications that include WRF also need m4, flex, bison, and `/bin/csh`.

gfortran 7 or newer is recommended. The repository `Linux-gfortran.mk` errors on gfortran 6 and older. gfortran 10 and newer needs `-fallow-argument-mismatch`, which the current mk file already has.

### 2.1. Conda toolchain (paths measured on this machine)

On this machine, `apt-get` reached the Debian mirrors through proxy `198.18.0.1` and got HTTP 500 / SSL EOF, so system packages mostly could not be installed (only the `make` 4.4.1-2 `.deb` was downloaded and installed by hand). The toolchain actually used is **Miniforge + conda-forge**, environment name `coawst`:

```bash
mamba create -y -n coawst -c conda-forge \
  gfortran gcc gxx openmpi \
  "libnetcdf=*=mpi_openmpi*" \
  "netcdf-fortran=*=mpi_openmpi*" \
  "hdf5=*=mpi_openmpi*" \
  cmake make perl zlib libcurl compilers pkg-config tcsh
```

Miniforge prefix: `/workspace/miniforge3`. After `conda activate coawst`, PATH includes `gfortran`, `gcc`, `g++`, `mpicc`, `mpif90`, `nf-config`, `nc-config`, `cmake`, `make`, and `perl`.

| Component | Version |
|-----------|---------|
| gfortran / gcc / g++ | 15.3.0 (conda-forge) |
| OpenMPI | 5.0.11 (`mpicc` / `mpif90`) |
| libnetcdf | 4.10.0 mpi_openmpi |
| netcdf-fortran | 4.6.4 mpi_openmpi |
| HDF5 | 2.2.0 mpi_openmpi |
| cmake | 4.4.5 |
| make | 4.4.1 |
| perl | 5.44 |
| tcsh | 6.24.16 |

Applications that include WRF also need:

```bash
mamba install -n coawst -c conda-forge m4 flex bison
sudo ln -sf /workspace/miniforge3/envs/coawst/bin/tcsh /bin/csh
```

> **Warning**
>
> Without `m4`, WRF configure reports `'m4' utility not found`. Without `/bin/csh`, `./compile` reports `not found` (the script shebang is `#!/bin/csh`). During the compile you can also set `NETCDF_classic=0` and `WRFIO_NCD_LARGE_FILE_SUPPORT=1`.

### 2.2. apt package list (other machines)

When apt works, the following set can be used. It was not successfully installed on this machine and is listed only as documentation:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  gfortran gcc g++ make cmake git perl \
  zlib1g-dev libcurl4-openssl-dev \
  libopenmpi-dev openmpi-bin \
  libnetcdf-dev libnetcdff-dev \
  libhdf5-dev libhdf5-openmpi-dev netcdf-bin \
  csh tcsh m4 flex bison wget unzip
```

### 2.3. Building MCT

The manual requires the bundled `Lib/MCT` (slightly modified relative to upstream, and already tested with COAWST). Use the same `mpicc` / `mpif90` as COAWST:

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST/Lib/MCT
chmod +x configure config.guess config.sub install-sh mkinstalldirs
bash ./configure --prefix=/workspace/COAWST/Lib/MCT CC=mpicc FC=mpif90 F90=mpif90
make -j4
make install
```

After clone, `configure` may not have the executable bit. Use `bash ./configure`, or `chmod +x` first.

The result on this machine was normal (Fortran/MPI type-mismatch warnings appeared because of `-fallow-argument-mismatch`; they can be ignored):

| Artifact | Path |
|----------|------|
| `libmct.a` | `/workspace/COAWST/Lib/MCT/lib/libmct.a` (925K) |
| `libmpeu.a` | `/workspace/COAWST/Lib/MCT/lib/libmpeu.a` |
| modules | `/workspace/COAWST/Lib/MCT/include/*.mod` |

Logs: `/workspace/mct-configure.log`, `/workspace/mct-make.log`, `/workspace/mct-install.log`.

SCRIP weights are needed only for different grids or nesting. v3.9 can generate the weight file at run time. The same-grid Inlet Coupled case ran on this machine without a pre-built `scrip_weights_inlet_test.nc`.

> **Warning**
>
> MCT must be built with the same `mpif90` as COAWST, and `MCT_INCDIR` / `MCT_LIBDIR` must be exported. Otherwise a common failure is “module file was not generated by this compiler”, or `m_MCTWorld` is not found. If those variables are unset, the default often falls through to `/usr/local/mct/{include,lib}`, which does not exist on this machine.

## 3. Environment variables

Before compiling or running, load the environment script so that `NETCDF`, the MCT paths, and the OpenMPI MCA parameters agree. Cases that include WRF use a separate helper. Do not put that helper's endian setting into the default script.

Before every compile or run:

```bash
source /workspace/coawst-env.sh
```

That script is equivalent to:

```bash
source /workspace/miniforge3/etc/profile.d/conda.sh
conda activate coawst
export NETCDF=$(nf-config --prefix)
# -> /workspace/miniforge3/envs/coawst
export MCT_INCDIR=/workspace/COAWST/Lib/MCT/include
export MCT_LIBDIR=/workspace/COAWST/Lib/MCT/lib
export OMPI_MCA_btl=tcp,self
export OMPI_MCA_pml=ob1
```

For small test runs on this machine you can also set, if needed:

```bash
export OMPI_MCA_rmaps_base_oversubscribe=true
```

For **JOE_TC** (which includes WRF), use this instead:

```bash
source /workspace/coawst-env-joe_tc.sh
```

That helper first sources `coawst-env.sh` above, then sets `OMPI_MCA_rmaps_base_oversubscribe=true` and the `GFORTRAN_CONVERT_UNIT=big_endian` setting that **only JOE_TC needs**.

> **Warning**
>
> Do not put `GFORTRAN_CONVERT_UNIT=big_endian` into the default `coawst-env.sh`. That would change endian behavior for cases that do not use WRF. See [5.2. JOE_TC Coupled](#52-joe_tc-coupled) for why.

What each variable does:

- `NETCDF`: required by the WRF portion and by I/O overall. With `USE_NETCDF4=on`, the ROMS side also takes paths from `nf-config` / `nc-config`.
- `MCT_INCDIR` / `MCT_LIBDIR`: point at the MCT built on this machine.
- With `USE_NETCDF4=on`, `Linux-gfortran.mk` takes NetCDF paths from `nf-config` / `nc-config`, and takes MCT from the two variables above.

You can also set `USE_MY_LIBS=yes` and edit `Compilers/my_build_paths.sh`. INLET_TEST on this machine used `USE_MY_LIBS=no`.

## 4. Building applications

`build_coawst.sh` decides which models to enable (ROMS / SWAN / WRF / WW3) from the application header, and drives the compile. When switching applications, change the application name and header paths in the script. You usually need to clean or rebuild the build products.

### 4.1. Configuring `build_coawst.sh`

The script is in the source root. Back it up before editing:

```bash
cd /workspace/COAWST
cp build_coawst.sh build_coawst.sh.orig
```

Diff of this machine relative to the distributed script:

```diff
--- build_coawst.sh.orig
+++ build_coawst.sh
@@ MY_ROOT_DIR
-export   MY_ROOT_DIR=/cygdrive/d/models/COAWST
+export   MY_ROOT_DIR=/workspace/COAWST

@@ which_MPI
- export         which_MPI=intel         # compile with mpiifort library
+ #export         which_MPI=intel         # compile with mpiifort library
-#export         which_MPI=openmpi       # compile with OpenMPI library
+export         which_MPI=openmpi       # compile with OpenMPI library
```

Leave the INLET_TEST-related settings at the distribution defaults:

| Variable | Value |
|----------|-------|
| `COAWST_APPLICATION` | `INLET_TEST` (must be all capitals; corresponds to header `inlet_test.h`) |
| `FORT` | `gfortran` |
| `USE_MPI` / `USE_MPIF90` | `on` |
| `USE_NETCDF4` | `on` |
| `USE_MY_LIBS` | `no` |
| `MY_HEADER_DIR` / `MY_ANALYTICAL_DIR` | `${MY_PROJECT_DIR}/Projects/Inlet_test/Coupled` |
| `BINDIR` | `./` |
| `SCRATCH_DIR` | `./Build_roms` |

`inlet_test.h` defines `ROMS_MODEL`, `SWAN_MODEL`, and `MCT_LIB`. It does not define WRF/WW3, so `make wrf` / `ww3` / `wrfhydro` are no-ops.

> **Note**
>
> `COAWST_APPLICATION` must be the application name in all capitals. The header file itself may be lowercase (for example `inlet_test.h`). Do not mix the older `coawst.bash` / `build_coawst.bash` names with v3.9's `build_coawst.sh`.

When switching to JOE_TC, change the following relative to the Inlet version:

```diff
-export   COAWST_APPLICATION=INLET_TEST
+export   COAWST_APPLICATION=JOE_TC
-#export  NETCDF=${NETCDF_INCDIR}/../
+export  NETCDF=$(nf-config --prefix)
-   export     MY_HEADER_DIR=${MY_PROJECT_DIR}/Projects/Inlet_test/Coupled
-   export MY_ANALYTICAL_DIR=${MY_PROJECT_DIR}/Projects/Inlet_test/Coupled
+   export     MY_HEADER_DIR=${MY_PROJECT_DIR}/Projects/JOE_TC/Coupled
+   export MY_ANALYTICAL_DIR=${MY_PROJECT_DIR}/Projects/JOE_TC/Coupled
```

Leave `FORT=gfortran`, `which_MPI=openmpi`, and `USE_NETCDF4=on` unchanged.

### 4.2. INLET_TEST (ROMS+SWAN)

INLET_TEST Coupled checks idealized-inlet ROMS–SWAN MCT coupling. It is this machine's baseline case without WRF.

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST
./build_coawst.sh -j 4
```

The script runs roughly in this order: `make mct_params` → `make wrf` → `make wrfhydro` → `make ww3` → `make swan` → the main `make` (ROMS/driver). Flags such as `-nocleanwrf` can skip cleaning.

Result on this machine: **success**, wall-clock about 58 s (SWAN cmake plus the ROMS/SWAN link). No `mpif.h` race appeared, so a serial retry was not needed.

| Item | Value |
|------|-------|
| Executable | `/workspace/COAWST/coawstM` |
| Size | 9.9M (10371976 bytes) |
| Build directory | `/workspace/COAWST/Build_roms/` |
| SWAN library | `/workspace/COAWST/SWAN/build/lib/libswan41.51.a` |
| Full log | `/workspace/coawst-build.log` |

Link line, summarized: `mpif90 … -o coawstM -lROMS libswan41.51.a -lnetcdff -lnetcdf … -lmct -lmpeu`.

The success marker is the appearance of `coawstM` (or `coawstM.exe` on Windows). A coupled build does not, and should not, produce standalone `wrf.exe` / `ocean.exe` / `swan.exe`. An occasional “undefined wrf.main” style warning is acceptable when WRF is not configured.

> **Warning**
>
> If a parallel make reports that `mpif.h` cannot be found, drop `-j` or use `-j 1` and build again.

### 4.3. JOE_TC Coupled (WRF+ROMS+SWAN)

The JOE_TC Coupled header `Projects/JOE_TC/Coupled/joe_tc.h` preselects **ExpG** (`WRF_MODEL` + `ROMS_MODEL` + `SWAN_MODEL`, `MCT_LIB`, and so on). The compile includes a full WRF, and the wall-clock time is much longer than INLET_TEST.

Before switching cases, back up the Inlet products:

```bash
cd /workspace/COAWST
cp -a coawstM coawstM.inlet_test
cp -a build_coawst.sh build_coawst.sh.inlet_test
```

In `build_coawst.sh`, `make wrfclean` interactively runs `WRF/clean -a; ./configure`. This machine answered through a pipe:

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST
printf '34\n1\n' | ./build_coawst.sh -j 4
```

Option **34** = GNU (gfortran/gcc) dmpar, nesting **1** = basic. The generated `configure.wrf` already has `DM_FC=mpif90`, `DM_CC=mpicc`, and, for gfortran 10 and newer, `FCCOMPAT=-fallow-argument-mismatch -fallow-invalid-boz`. **There is no need to patch `configure.wrf` again.** Harmless warnings: `rpc/types.h` is missing, so moving nests are unavailable (this case has `max_dom=1`, so that has no effect).

The first run of that command failed because `/bin/csh` was missing: configure succeeded, then `./compile` failed in about 61 s with exit 2, and the ROMS side reported a missing `module_wrf_top.mod`. Log: `/workspace/joe_tc-build.log`.

After `/bin/csh` was fixed, the existing `configure.wrf` and the JOE_TC `mct_params` under `Build_roms` were kept, and the build was repeated with cleaning skipped:

```bash
./build_coawst.sh -j 4 -noclean -nocleanwrf -nocleanswan
```

Result on this machine: **success**, wall-clock about **27 m 31 s** (real), exit 0. Log: `/workspace/joe_tc-build2.log`. In the log, `checkout_externals: Permission denied` (the physics submodules are already vendored) and the `wrf.exe` link missing `main` (COAWST links WRF into `coawstM` and does not produce a standalone `wrf.exe`) can be ignored. `real.exe` / `ndown.exe` / `tc.exe` were built.

| Item | Value |
|------|-------|
| Executable | `/workspace/COAWST/coawstM` |
| Size | 65 MB (68047632 bytes) |
| Backup name | `coawstM.joe_tc` |
| `nm … \| grep -ci module_wrf_top` | 34 |

> **Warning**
>
> A full clean deletes `Build_roms`. If `-nocleanwrf` is also set, the script skips `mct_params` as well. Watch this when switching applications.

### 4.4. Tips on practical use

- **Parallelism:** on this machine `./build_coawst.sh -j 4` worked for both INLET_TEST and JOE_TC. If an `mpif.h` race appears, switch to `-j 1`.
- **Skip cleaning:** after a partial failure, `-noclean -nocleanwrf -nocleanswan` keeps `configure.wrf` and existing products. When switching applications, do not rely on these flags to skip `mct_params`.
- **WRF configure prompts:** GNU dmpar + basic nesting corresponds to `printf '34\n1\n'`.
- **gfortran 15:** `-fallow-argument-mismatch` is already in `Linux-gfortran.mk` and in the MCT configure. MPI type-mismatch warnings can be ignored.
- **mk options:** do not leave Intel-only options (such as `-assume byterecl`) in the gfortran mk. On older trees that makes SWAN/gfortran fail.
- **Do not compile WRF by itself for a coupled application.** `build_coawst.sh` handles it when WRF is enabled.
- **Switching cases requires a rebuild:** after changing the header or paths, clean or recreate `Build_roms` and the WRF products.
- **Success marker:** `coawstM` appears in the root directory. INLET_TEST is about 9.9M; JOE_TC is about 65 MB.

## 5. Running applications

A coupled run starts `coawstM` from the repository root. It reads `coupling_*.in` and assigns MPI ranks to each model from the `Nnodes*` values. `mpirun -np` must equal the sum of the `Nnodes*` values.

### 5.1. INLET_TEST Coupled

In `Projects/Inlet_test/Coupled/coupling_inlet_test.in`, `NnodesWAV=1` and `NnodesOCN=1`, so the run needs **`mpirun -np 2`**.

Short test run (measured on this machine):

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST
timeout 240s mpirun -np 2 ./coawstM Projects/Inlet_test/Coupled/coupling_inlet_test.in \
  > /workspace/inlet_test_run.out 2>&1
```

Result: startup was normal, and ROMS↔SWAN began exchanging fields. After 240 s the process exited because of `timeout` (exit 124); the run itself was healthy. The full case uses `NTIMES=720` and `DT=60 s` (about 12 h of model time). The short run reached about timestep **254** (about 4.2 h of model time).

stdout contained the following:

- `Ocean Model MPI nodes: 00000 - 00000` / `Waves Model MPI nodes: 00001 - 00001`
- `ROMS version 4.3`, compiler `mpif90` (gfortran), `MCT_LIB`, `SWAN_COUPLING`
- `SWAN grid 1 sent wave data to ROMS` / `ROMS … sent data to SWAN` repeated, and the HSIGN / DEPTH / VEL exchanges were finite values
- Files written: `ocean_his_coupled.nc`, `ocean_avg_coupled.nc`, `ocean_sta.nc`, `PRINT01`

Full run:

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST
mpirun -np 2 ./coawstM Projects/Inlet_test/Coupled/coupling_inlet_test.in
```

The tree does not contain `Projects/Inlet_test/Coupled/scrip_weights_inlet_test.nc`. Same-grid Coupled uses the MCT same-grid path, and this machine ran without a pre-generated weight file. For multiple grids or different grids, v3.9 can generate SCRIP weights at run time (`SCRIP_COAWST_NAME` names the file; an existing file is reused).

This case checks idealized-inlet ROMS–SWAN MCT coupling: the south side is closed, the north side has a sinusoidal water level, and SWAN sends waves of about 1 m height from north to south. Grids and inputs are already in `Projects/Inlet_test/Coupled`.

On PBS-style machines, follow the repository's `run_coawst` example or the `run_nemo` example in the manual, and rewrite the `mpirun` and `coupling_*.in` paths in the batch-system form. On SLURM, use `sbatch` / `squeue`.

### 5.2. JOE_TC Coupled

`coupling_joe_tc.in`: `NnodesATM=1`, `NnodesWAV=1`, `NnodesOCN=1`, so the total is **`mpirun -np 3`**. WRF inputs are in `Projects/JOE_TC/` (not in `Coupled/`): `namelist.input`, `wrfinput_d01`, `wrfbdy_d01` (`run_hours=120`, `time_step=25`, `nproc_x=nproc_y=1`). ROMS: `NtileI=NtileJ=1`, `NTIMES=17280`, `DT=25 s`. Date 2026-10-28 (America/Los_Angeles, about 07:22–08:03).

#### 5.2.1. Before you run (repository root)

```bash
cd /workspace/COAWST
cp Projects/JOE_TC/{namelist.input,wrfinput_d01,wrfbdy_d01} .
for f in WRF/run/*; do
  b=$(basename "$f")
  case $b in namelist*|*.exe|README*|CMake*|Makefile) ;; *)
    [ -e "$b" ] || ln -s "$f" "$b"
  ;; esac
done
```

Symlink `RRTM_DATA`, the `*.TBL` files, `CAMtr*`, and similar files into the root directory. The manual says to copy namelist / wrfinput / wrfbdy into `root_dir`.

#### 5.2.2. big_endian is required

Without the endian setting, the WRF process dies immediately at:

`At line 6925 of file module_ra_rrtm.f90 (unit = 10, file = 'RRTM_DATA') Fortran runtime error: End of file`

WRF object files were compiled with `-fconvert=big-endian`, but gfortran applies that conversion only when the **main program** is compiled. The coupled main program is ROMS `master.f90`, which was compiled without that flag, so at run time it reads the big-endian WRF table files as little-endian and hits EOF. Export this before the run:

```bash
export GFORTRAN_CONVERT_UNIT=big_endian
```

A permanent fix is to add `-fconvert=big-endian -frecord-marker=4` to `FFLAGS` in the `USE_WRF` branch of `Compilers/Linux-gfortran.mk` and rebuild. **This machine did not modify the mk file.**

You can also load the helper directly:

```bash
source /workspace/coawst-env-joe_tc.sh
```

#### 5.2.3. Short test run (measured on this machine)

```bash
source /workspace/coawst-env-joe_tc.sh
cd /workspace/COAWST
timeout --kill-after=30s 600s mpirun -np 3 ./coawstM Projects/JOE_TC/Coupled/coupling_joe_tc.in \
  > /workspace/joe_tc_run.out 2>&1
```

Result: startup was normal. Ocean / Waves / Atmos were assigned ranks 0 / 1 / 2. After 600 s the process exited because of `timeout` (exit 124), with no error in the run. WRF advanced to about **2005-09-01_03:05:25** (about 3 h of 120 h), at about 1.25 s wall-clock per 25 s model step. Three-way exchange completed **19** coupling cycles (all 12 message classes appeared, including `WRF … sent data to ROMS`, `SWAN … sent data to WRF`, and `ROMS … sent data to SWAN`).

Files written: `wrfout_d01_2005-09-01_00:00:00` (about 100 MB), `ocean_avg.nc`, `ocean_dia.nc`, `ocean_rst.nc` (also `ocean_joe_tcs_his.nc`, whose mtime looked anomalous; its contents were not checked on this machine).

At this rate, the full **120 h** on 3 ranks would take about **2 days**. Increasing `NnodesATM` (and keeping `nproc_x * nproc_y` equal to it) and `NtileI * NtileJ` shortens the wall-clock time.

Full run (drop timeout):

```bash
source /workspace/coawst-env-joe_tc.sh
cd /workspace/COAWST
mpirun -np 3 ./coawstM Projects/JOE_TC/Coupled/coupling_joe_tc.in
```

You can also switch to ExpA1 (WRF+SWAN only) or ExpA (WRF+ROMS only). For WRF–ROMS coupling, set `sst_update = 1` and configure `io_form_auxinput4`.

### 5.3. Processor layout

Coupled rank layout is fixed by three layers of numbers, and they must agree:

| Layer | Source | Meaning |
|-------|--------|---------|
| `NnodesATM` / `NnodesWAV` / `NnodesOCN` | `coupling_*.in` | MPI ranks given to atmosphere / waves / ocean; their sum equals `mpirun -np` |
| `nproc_x` / `nproc_y` | WRF `namelist.input` | Decomposition inside the WRF domain; `nproc_x * nproc_y` should equal `NnodesATM` |
| `NtileI` / `NtileJ` | ROMS ocean `.in` | ROMS tile decomposition; `NtileI * NtileJ` should equal `NnodesOCN` |

Short-run layout on this machine:

| Case | `mpirun -np` | Nnodes* | WRF | ROMS |
|------|--------------|---------|-----|------|
| INLET_TEST Coupled | 2 | WAV=1, OCN=1 | (none) | (set by NnodesOCN) |
| JOE_TC Coupled | 3 | ATM=1, WAV=1, OCN=1 | `nproc_x=nproc_y=1` | `NtileI=NtileJ=1` |

> **Note**
>
> When increasing parallelism, change `coupling_*.in`, the WRF namelist, and the ROMS `.in` together, then set `mpirun -np` to the new sum of `Nnodes*`.

## 6. Common pitfalls and recommended practices

This section collects failure points that were verified while deploying and test-running on this machine, and what to do about them. The numbering is not a priority order.

1. **apt on this machine is unusable** (proxy `198.18.0.1` → 500/SSL). What to do: use the conda-forge environment `coawst`, plus the `make` `.deb` installed with curl.
2. **MCT `./configure` is not executable.** What to do: `bash ./configure` or `chmod +x`.
3. **MCT must use the same `mpif90` as COAWST**, and `MCT_INCDIR` / `MCT_LIBDIR` must be exported. Otherwise a common failure is “module file was not generated by this compiler”, or `m_MCTWorld` is not found.
4. **`COAWST_APPLICATION` must be all capitals** (`INLET_TEST`), matching `inlet_test.h`. A lowercase match fails.
5. **The distributed `MY_ROOT_DIR` is a Cygwin path**, `/cygdrive/d/models/COAWST`. On Linux it must be changed to the real path (on this machine, `/workspace/COAWST`).
6. **The distributed setting is `which_MPI=intel`.** This stack uses `openmpi`.
7. **gfortran 15:** `-fallow-argument-mismatch` is already in `Linux-gfortran.mk` and in the MCT configure. MPI type-mismatch warnings can be ignored.
8. **`mpirun -np` equals the sum of the `Nnodes*` values** (2 for Inlet Coupled; 3 for JOE_TC Coupled).
9. **`./build_coawst.sh -j 4` worked on this machine.** If an `mpif.h` race appears, rebuild without parallelism or with `-j 1`.
10. **Do not compile WRF by itself for a coupled application.** `build_coawst.sh` handles it when WRF is enabled.
11. **Switching cases requires a rebuild:** after changing the header or paths, clean or recreate `Build_roms` and the WRF products.
12. **Do not leave Intel-only options in the gfortran mk** (such as `-assume byterecl`). On older trees that makes SWAN/gfortran fail.
13. **WRF configure needs m4 / flex / bison** (conda-forge). `./compile` needs `/bin/csh` (you can `ln -sf …/tcsh /bin/csh`).
14. **WRF configure prompts:** `printf '34\n1\n' | ./build_coawst.sh -j 4` (GNU dmpar + basic nesting).
15. **A JOE_TC run must set `GFORTRAN_CONVERT_UNIT=big_endian`**, or `RRTM_DATA` hits EOF. Use `/workspace/coawst-env-joe_tc.sh`. Do not put that setting into the default `coawst-env.sh`.
16. **JOE_TC namelist / wrfinput / wrfbdy live in `Projects/JOE_TC/`**, not in `Coupled/`. Copy them to the root before the run and symlink the `WRF/run` table files.
17. **Partial rebuild flags:** `-noclean -nocleanwrf -nocleanswan` can keep `configure.wrf`. A full clean deletes `Build_roms`, and with `-nocleanwrf` the script also skips `mct_params`.

Recommended practice:

- First use INLET_TEST to check the ROMS+SWAN+MCT toolchain, then build JOE_TC.
- Keep the default environment separate from the WRF environment: `coawst-env.sh` and `coawst-env-joe_tc.sh`.
- Before switching applications, back up `coawstM` and `build_coawst.sh` (this machine has `coawstM.inlet_test` / `coawstM.joe_tc`).
- Cut short runs off with `timeout`. After the coupling exchanges and output files look right, drop `timeout` and integrate the full length.

## 7. Further applications

The following cases exist in the repository and **have not been compiled or test-run on this machine**. Paths and model combinations come from the repository layout and excerpts of the manual.

| Case | Path | Models |
|------|------|--------|
| JOE_TC DiffGrid | `Projects/JOE_TC/DiffGrid` | ROMS+SWAN and WRF on different grids |
| Sandy | `Projects/Sandy/` | Full storm; `coupling_sandy.in`; often includes a PBS `run_coawst` example |

DiffGrid: ROMS/SWAN share one grid and WRF uses another. v3.9 can automatically generate weights such as `scrip_joe_tc_diffgrid.nc`, or you can build `Lib/SCRIP_COAWST` first and run `scrip_coawst`. The header must turn on `MCT_INTERP_WV2AT` and `MCT_INTERP_OC2AT` (DiffGrid case H already sets them).

Sandy: a full storm case, often used as the PBS `run_coawst` example. Rank counts and inputs follow `coupling_sandy.in` and `sandy.h` in that directory. Set `mpirun -np` to `NnodesATM + NnodesWAV + NnodesOCN`. When switching applications, rebuild from a clean state and change `COAWST_APPLICATION` and `MY_*_DIR` as above.

## 8. References

| Resource | URL / path |
|----------|------------|
| USGS overview | https://www.usgs.gov/centers/whcmsc/science/coawst-a-coupled-ocean-atmosphere-wave-sediment-transport-modeling-system |
| USGS components | https://www.usgs.gov/centers/whcmsc/science/coawst-system-components |
| GitHub | https://github.com/DOI-USGS/COAWST |
| USGS GitLab | https://code.usgs.gov/coawstmodel/COAWST |
| Wiki / training | https://github.com/DOI-USGS/COAWST/wiki · https://github.com/DOI-USGS/COAWST/wiki/COAWST-Trainings |
| Issues | https://github.com/DOI-USGS/COAWST/issues |
| User manual (in the repository) | `/workspace/COAWST/COAWST_User_Manual.docx` |
| Environment script | `/workspace/coawst-env.sh` |
| JOE_TC environment (includes big_endian) | `/workspace/coawst-env-joe_tc.sh` |
| Deployment log from this machine | `/workspace/cowast-deploy-log.md` |
| JOE_TC measured-run log | `/workspace/cowast-joe_tc-log.md` |
| Research notes | `/workspace/cowast-research.md` |
| Manual excerpts | `/workspace/cowast-manual-excerpts.md` |
| Inlet build log | `/workspace/coawst-build.log` |
| Inlet short-run output | `/workspace/inlet_test_run.out` |
| JOE_TC build logs | `/workspace/joe_tc-build.log`, `/workspace/joe_tc-build2.log` |
| JOE_TC short-run output | `/workspace/joe_tc_run.out` |
| Unofficial conda notes | https://github.com/wingtorres/coawst-conda |
| Warner et al. (2010) | *Development of a Coupled Ocean–Atmosphere–Wave–Sediment Transport (COAWST) Modeling System* |
| scikit-learn User Guide (style reference) | https://scikit-learn.org/stable/modules/linear_model.html |
