# COAWST compile and run guide

[中文](user-guide.zh-CN.md) · [Home](../README.md) · [中文首页](../README.zh-CN.md)

Compile and run **COAWST** (Coupled-Ocean-Atmosphere-Wave-Sediment Transport Modeling System) v3.9 on Linux. The guide covers dependencies, environment variables, and application configuration, then the builds, short runs, and failures for INLET_TEST (ROMS+SWAN) and JOE_TC Coupled (WRF+ROMS+SWAN).

cowast / COWAST are spoken names for the same USGS coupled system. The official spelling is **COAWST**.

## 1. Getting started

The measured tree is **COAWST v3.9** from `https://github.com/DOI-USGS/COAWST.git`, commit `81df2ed` (full hash `81df2edcd175fb87426b4d913f4fdaa42cfc50ad`). Paths, version numbers, and wall-clock times below are from that record: Debian 13 (trixie), x86_64, 8 CPUs, about 110G free under `/workspace`, dated 2026-10-28 (America/Los_Angeles).

Standalone WRF build and run steps are not repeated here. Dependencies and MCT were installed on this machine. **INLET_TEST** (ROMS+SWAN Coupled) and **JOE_TC Coupled** (ROMS+SWAN+WRF) were compiled and short-run. **Sandy and JOE_TC DiffGrid were not built on this machine.**

The source is cloned with git.

```bash
cd /workspace
git clone --depth 1 https://github.com/DOI-USGS/COAWST.git
# Result: /workspace/COAWST (about 1.8G), commit 81df2ed
```

The clone on this machine came from GitHub. USGS GitLab was not used. Alternate URL: `https://code.usgs.gov/coawstmodel/COAWST`.

The tree includes the user manual `/workspace/COAWST/COAWST_User_Manual.docx` and the directories ROMS, WRF, WPS, SWAN, WW3, InWave, Master, Lib, Compilers, and Projects. Coupled cases are driven by `build_coawst.sh`. Do not compile WRF alone with `./compile em_real`.

> **Note**
>
> The license is USGS public domain / CC0 1.0. The standard ROMS–SWAN–WRF–WW3 path does not depend on proprietary libraries.

## 2. Prerequisites

NetCDF, MPI, MCT, and the main program must share one Fortran / MPI stack. A missing piece produces module mismatches or link failures.

Required components are a Fortran compiler (gfortran), C/C++, MPI (OpenMPI), NetCDF-C + NetCDF-Fortran (NetCDF4), HDF5, cmake, make, perl, and git. Coupling also needs **MCT**. The repository bundles `Lib/MCT`, which must be built first. Applications that include WRF also need m4, flex, bison, and `/bin/csh`.

gfortran 7 or newer is recommended. `Linux-gfortran.mk` errors on gfortran 6 and older. gfortran 10 and newer needs `-fallow-argument-mismatch`, which the current mk already contains.

### 2.1. Conda toolchain (paths measured on this machine)

On this machine, `apt-get` reached the Debian mirrors through proxy `198.18.0.1` and got HTTP 500 / SSL EOF, so system packages mostly could not be installed. The only package installed by hand was the `make` 4.4.1-2 `.deb`. The toolchain actually used is **Miniforge + conda-forge**, environment name `coawst`.

```bash
mamba create -y -n coawst -c conda-forge \
  gfortran gcc gxx openmpi \
  "libnetcdf=*=mpi_openmpi*" \
  "netcdf-fortran=*=mpi_openmpi*" \
  "hdf5=*=mpi_openmpi*" \
  cmake make perl zlib libcurl compilers pkg-config tcsh
```

The Miniforge prefix is `/workspace/miniforge3`. After `conda activate coawst`, PATH includes `gfortran`, `gcc`, `g++`, `mpicc`, `mpif90`, `nf-config`, `nc-config`, `cmake`, `make`, and `perl`.

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

Applications that include WRF also need m4, flex, bison, and `/bin/csh`.

```bash
mamba install -n coawst -c conda-forge m4 flex bison
sudo ln -sf /workspace/miniforge3/envs/coawst/bin/tcsh /bin/csh
```

> **Warning**
>
> Without `m4`, WRF configure reports `'m4' utility not found`. Without `/bin/csh`, `./compile` reports `not found` (the script shebang is `#!/bin/csh`). The compile can also be given `NETCDF_classic=0` and `WRFIO_NCD_LARGE_FILE_SUPPORT=1`.

### 2.2. apt package list (other machines)

Where apt works, install the packages below. They were not installed on this machine; the list is only a record.

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

The manual requires the bundled `Lib/MCT`. It is slightly modified relative to upstream and has already been tested with COAWST. The compilers must match COAWST: `mpicc` / `mpif90`.

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST/Lib/MCT
chmod +x configure config.guess config.sub install-sh mkinstalldirs
bash ./configure --prefix=/workspace/COAWST/Lib/MCT CC=mpicc FC=mpif90 F90=mpif90
make -j4
make install
```

After clone, `configure` may lack the executable bit. Use `bash ./configure`, or `chmod +x` first.

The install on this machine finished. Fortran/MPI type-mismatch warnings come from `-fallow-argument-mismatch` and can be ignored.

| Artifact | Path |
|----------|------|
| `libmct.a` | `/workspace/COAWST/Lib/MCT/lib/libmct.a` (925K) |
| `libmpeu.a` | `/workspace/COAWST/Lib/MCT/lib/libmpeu.a` |
| modules | `/workspace/COAWST/Lib/MCT/include/*.mod` |

Logs: `/workspace/mct-configure.log`, `/workspace/mct-make.log`, `/workspace/mct-install.log`.

SCRIP weights are required only for different grids or nesting. v3.9 can generate the weight file at run time. Same-grid Inlet Coupled had no pre-built `scrip_weights_inlet_test.nc` and still ran on this machine.

> **Warning**
>
> Build MCT with the same `mpif90` as COAWST, and export `MCT_INCDIR` / `MCT_LIBDIR`. Otherwise a common failure is “module file was not generated by this compiler”, or `m_MCTWorld` is not found. If the variables are unset, the default often falls through to `/usr/local/mct/{include,lib}`, which does not exist on this machine.

## 3. Environment variables

Load the environment script before compiling or running, so that `NETCDF`, the MCT paths, and the OpenMPI MCA parameters agree. Cases that include WRF use a separate helper. Keep `GFORTRAN_CONVERT_UNIT` in that helper only.

Load the default script before every compile or run.

```bash
source /workspace/coawst-env.sh
```

The script is equivalent to the settings below.

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

Small test runs on this machine may also enable oversubscription.

```bash
export OMPI_MCA_rmaps_base_oversubscribe=true
```

**JOE_TC** (includes WRF) loads the helper instead.

```bash
source /workspace/coawst-env-joe_tc.sh
```

The helper first sources `coawst-env.sh` above, then sets `OMPI_MCA_rmaps_base_oversubscribe=true` and the `GFORTRAN_CONVERT_UNIT=big_endian` value that **only JOE_TC needs**.

> **Warning**
>
> Do not put `GFORTRAN_CONVERT_UNIT=big_endian` into the default `coawst-env.sh`. Endian behavior of cases without WRF would change. See [5.2. JOE_TC Coupled](#52-joe_tc-coupled).

- `NETCDF`: required by the WRF portion and by I/O overall. With `USE_NETCDF4=on`, the ROMS side also takes paths from `nf-config` / `nc-config`.
- `MCT_INCDIR` / `MCT_LIBDIR`: point at the MCT built on this machine.
- With `USE_NETCDF4=on`, `Linux-gfortran.mk` takes NetCDF paths from `nf-config`/`nc-config` and takes MCT from the two variables above.

`USE_MY_LIBS=yes` is an alternative, together with edits to `Compilers/my_build_paths.sh`. INLET_TEST on this machine used `USE_MY_LIBS=no`.

## 4. Building applications

`build_coawst.sh` enables ROMS / SWAN / WRF / WW3 from the application header and drives the compile. Switching applications means changing the application name and header paths in the script, and usually cleaning or rebuilding the products.

### 4.1. Configuring `build_coawst.sh`

The script is in the source root. Back it up before editing.

```bash
cd /workspace/COAWST
cp build_coawst.sh build_coawst.sh.orig
```

On this machine the distributed Cygwin `MY_ROOT_DIR` was changed to `/workspace/COAWST`, and `which_MPI` was changed from `intel` to `openmpi`.

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

INLET_TEST keeps the distribution defaults.

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

`inlet_test.h` defines `ROMS_MODEL`, `SWAN_MODEL`, and `MCT_LIB`, and does not define WRF/WW3, so `make wrf` / `ww3` / `wrfhydro` are no-ops.

> **Note**
>
> `COAWST_APPLICATION` must be the application name in all capitals. The header file itself may be lowercase (for example `inlet_test.h`). Do not mix the older `coawst.bash` / `build_coawst.bash` names with v3.9's `build_coawst.sh`.

For JOE_TC, change the application name, `NETCDF`, and the header directories on top of the Inlet settings.

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

INLET_TEST Coupled checks ROMS–SWAN MCT coupling on an idealized inlet. It is the baseline on this machine without WRF.

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST
./build_coawst.sh -j 4
```

The script order is `make mct_params` → `make wrf` → `make wrfhydro` → `make ww3` → `make swan` → the main `make` (ROMS/driver). Flags such as `-nocleanwrf` skip cleaning.

The build on this machine succeeded. Wall-clock time was about 58 s, covering SWAN cmake and the ROMS/SWAN link. No `mpif.h` race appeared, so a serial retry was not required.

| Item | Value |
|------|-------|
| Executable | `/workspace/COAWST/coawstM` |
| Size | 9.9M (10371976 bytes) |
| Build directory | `/workspace/COAWST/Build_roms/` |
| SWAN library | `/workspace/COAWST/SWAN/build/lib/libswan41.51.a` |
| Full log | `/workspace/coawst-build.log` |

The link line is `mpif90 … -o coawstM -lROMS libswan41.51.a -lnetcdff -lnetcdf … -lmct -lmpeu`.

Success puts `coawstM` in the root directory (`coawstM.exe` on Windows). A coupled build does not produce standalone `wrf.exe` / `ocean.exe` / `swan.exe`. With WRF disabled, an occasional “undefined wrf.main” warning is acceptable.

> **Warning**
>
> If a parallel make reports that `mpif.h` cannot be found, drop `-j` or use `-j 1` and build again.

### 4.3. JOE_TC Coupled (WRF+ROMS+SWAN)

The header `Projects/JOE_TC/Coupled/joe_tc.h` preselects **ExpG** (`WRF_MODEL` + `ROMS_MODEL` + `SWAN_MODEL`, plus `MCT_LIB` and related macros). This configuration compiles a full WRF. Wall-clock time is longer than the INLET_TEST figure of about 58 s.

Before switching cases, back up the Inlet binary and script.

```bash
cd /workspace/COAWST
cp -a coawstM coawstM.inlet_test
cp -a build_coawst.sh build_coawst.sh.inlet_test
```

`make wrfclean` inside `build_coawst.sh` runs `WRF/clean -a; ./configure` interactively. This machine answered through a pipe.

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST
printf '34\n1\n' | ./build_coawst.sh -j 4
```

Option **34** is GNU (gfortran/gcc) dmpar. Nesting option **1** is basic. The generated `configure.wrf` already has `DM_FC=mpif90`, `DM_CC=mpicc`, and, for gfortran 10 and newer, `FCCOMPAT=-fallow-argument-mismatch -fallow-invalid-boz`. **Do not patch `configure.wrf` again.** Missing `rpc/types.h` disables moving nests. This case has `max_dom=1`, so that warning has no effect.

`/bin/csh` was missing at the time. configure succeeded, then `./compile` failed after about 61 s with exit 2, and the ROMS side reported a missing `module_wrf_top.mod`. The log is `/workspace/joe_tc-build.log`.

After `/bin/csh` was added, the existing `configure.wrf` and the JOE_TC `mct_params` under `Build_roms` were kept, and the build was repeated with cleaning skipped.

```bash
./build_coawst.sh -j 4 -noclean -nocleanwrf -nocleanswan
```

The build on this machine succeeded. Wall-clock time was about **27 m 31 s** (real), exit 0. The log is `/workspace/joe_tc-build2.log`. `checkout_externals: Permission denied` comes from physics submodules that are already vendored. The `wrf.exe` link lacks `main` because COAWST links WRF into `coawstM` and does not emit a separate `wrf.exe`. Both messages can be ignored. `real.exe` / `ndown.exe` / `tc.exe` were built.

| Item | Value |
|------|-------|
| Executable | `/workspace/COAWST/coawstM` |
| Size | 65 MB (68047632 bytes) |
| Backup name | `coawstM.joe_tc` |
| `nm … \| grep -ci module_wrf_top` | 34 |

> **Warning**
>
> A full clean deletes `Build_roms`. If `-nocleanwrf` is also set, the script skips `mct_params`. Watch this when switching applications.

### 4.4. Tips on practical use

- **Parallelism:** `./build_coawst.sh -j 4` worked on this machine for both INLET_TEST and JOE_TC. If an `mpif.h` race appears, switch to `-j 1`.
- **Skip cleaning:** after a partial failure, `-noclean -nocleanwrf -nocleanswan` keeps `configure.wrf` and existing products. When switching applications, do not rely on these flags to skip `mct_params`.
- **WRF configure:** GNU dmpar plus basic nesting corresponds to `printf '34\n1\n'`.
- **gfortran 15:** `-fallow-argument-mismatch` is already in `Linux-gfortran.mk` and in the MCT configure. MPI type-mismatch warnings can be ignored.
- **mk options:** do not leave Intel-only options (such as `-assume byterecl`) in the gfortran mk. On older trees that makes SWAN/gfortran fail.
- **WRF:** when WRF is enabled, `build_coawst.sh` compiles it. Do not build WRF alone for a coupled application.
- **Switching cases:** after changing the header or paths, clean or recreate `Build_roms` and the WRF products.
- **Success marker:** `coawstM` appears in the root directory. INLET_TEST is about 9.9M; JOE_TC is about 65 MB.

## 5. Running applications

`coawstM` in the repository root reads `coupling_*.in` and assigns MPI ranks from the `Nnodes*` values. `mpirun -np` must equal the sum of those values.

### 5.1. INLET_TEST Coupled

`Projects/Inlet_test/Coupled/coupling_inlet_test.in` sets `NnodesWAV=1` and `NnodesOCN=1`, so the run uses **`mpirun -np 2`**.

The short run on this machine was cut off with a 240 s `timeout`.

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST
timeout 240s mpirun -np 2 ./coawstM Projects/Inlet_test/Coupled/coupling_inlet_test.in \
  > /workspace/inlet_test_run.out 2>&1
```

Startup was normal, and ROMS and SWAN began exchanging fields. After 240 s the process exited because of `timeout` (exit 124). The run itself reported no error. The full case is `NTIMES=720`, `DT=60 s` (about 12 h of model time). The short run reached about timestep **254** (about 4.2 h of model time).

stdout records the rank split and the ROMS version, plus the field exchange and the output files.

- `Ocean Model MPI nodes: 00000 - 00000` / `Waves Model MPI nodes: 00001 - 00001`
- `ROMS version 4.3`, compiler `mpif90` (gfortran), `MCT_LIB`, `SWAN_COUPLING`
- `SWAN grid 1 sent wave data to ROMS` / `ROMS … sent data to SWAN` repeated, and the HSIGN / DEPTH / VEL exchanges were finite
- Files written: `ocean_his_coupled.nc`, `ocean_avg_coupled.nc`, `ocean_sta.nc`, `PRINT01`

The full integration drops `timeout`.

```bash
source /workspace/coawst-env.sh
cd /workspace/COAWST
mpirun -np 2 ./coawstM Projects/Inlet_test/Coupled/coupling_inlet_test.in
```

The tree has no `Projects/Inlet_test/Coupled/scrip_weights_inlet_test.nc`. Same-grid Coupled uses the MCT same-grid path, and this machine ran without a pre-generated weight file. For multiple grids or different grids, v3.9 can generate SCRIP weights at run time. `SCRIP_COAWST_NAME` sets the file name; an existing file is reused.

The south boundary is closed, the north boundary is a sinusoidal water level, and SWAN sends waves of about 1 m height from north to south. Grids and inputs are already in `Projects/Inlet_test/Coupled`.

On PBS, follow `run_coawst` in the repository or `run_nemo` in the manual, and write `mpirun` and `coupling_*.in` into the job script. On SLURM, use `sbatch` / `squeue`.

### 5.2. JOE_TC Coupled

`coupling_joe_tc.in` sets `NnodesATM=1`, `NnodesWAV=1`, and `NnodesOCN=1`, so the run uses **`mpirun -np 3`**. WRF inputs are in `Projects/JOE_TC/`, not in `Coupled/`: `namelist.input`, `wrfinput_d01`, `wrfbdy_d01` (`run_hours=120`, `time_step=25`, `nproc_x=nproc_y=1`). ROMS uses `NtileI=NtileJ=1`, `NTIMES=17280`, `DT=25 s`. The measured window was 2026-10-28 (America/Los_Angeles, about 07:22–08:03).

#### 5.2.1. Before you run (repository root)

Place the WRF inputs and run-time tables in the repository root.

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

Symlink `RRTM_DATA`, the `*.TBL` files, and `CAMtr*` into the root. The manual requires namelist / wrfinput / wrfbdy to be copied to `root_dir`.

#### 5.2.2. big_endian is required

Without the endian setting, the WRF process fails immediately with `At line 6925 of file module_ra_rrtm.f90 (unit = 10, file = 'RRTM_DATA') Fortran runtime error: End of file`.

WRF objects were compiled with `-fconvert=big-endian`. gfortran applies that conversion only when the main program is compiled. The coupled main program is ROMS `master.f90`, which was compiled without the flag, so at run time it reads the big-endian WRF tables as little-endian and hits EOF. Export the variable before the run.

```bash
export GFORTRAN_CONVERT_UNIT=big_endian
```

A permanent fix adds `-fconvert=big-endian -frecord-marker=4` to `FFLAGS` in the `USE_WRF` branch of `Compilers/Linux-gfortran.mk`, then rebuilds. **This machine did not modify the mk file.**

The helper can be loaded directly.

```bash
source /workspace/coawst-env-joe_tc.sh
```

#### 5.2.3. Short test run (measured on this machine)

The short run on this machine was cut off with a 600 s `timeout`, and `SIGKILL` was sent 30 s later.

```bash
source /workspace/coawst-env-joe_tc.sh
cd /workspace/COAWST
timeout --kill-after=30s 600s mpirun -np 3 ./coawstM Projects/JOE_TC/Coupled/coupling_joe_tc.in \
  > /workspace/joe_tc_run.out 2>&1
```

Startup was normal. Ocean / Waves / Atmos were on ranks 0 / 1 / 2. After 600 s the process exited because of `timeout` (exit 124), with no error in the run. WRF advanced to about **2005-09-01_03:05:25** (about 3 h of 120 h), at about 1.25 s of wall-clock time per 25 s model step. Three-way exchange completed **19** coupling cycles. All 12 message classes appeared, including `WRF … sent data to ROMS`, `SWAN … sent data to WRF`, and `ROMS … sent data to SWAN`.

The run wrote `wrfout_d01_2005-09-01_00:00:00` (about 100 MB), `ocean_avg.nc`, `ocean_dia.nc`, and `ocean_rst.nc`. `ocean_joe_tcs_his.nc` was also present; its mtime looked anomalous, and its contents were not checked on this machine.

At this rate the full **120 h** on 3 ranks would take about **2 days**. Increasing `NnodesATM` (with `nproc_x * nproc_y` kept equal to it) and increasing `NtileI * NtileJ` shortens the wall-clock time.

The full integration drops `timeout`.

```bash
source /workspace/coawst-env-joe_tc.sh
cd /workspace/COAWST
mpirun -np 3 ./coawstM Projects/JOE_TC/Coupled/coupling_joe_tc.in
```

ExpA1 (WRF+SWAN only) and ExpA (WRF+ROMS only) are alternatives. For WRF–ROMS coupling, set `sst_update = 1` and configure `io_form_auxinput4`.

### 5.3. Processor layout

Coupled rank counts are fixed by three layers of numbers. The three layers must agree.

| Layer | Source | Meaning |
|-------|--------|---------|
| `NnodesATM` / `NnodesWAV` / `NnodesOCN` | `coupling_*.in` | MPI ranks for atmosphere / waves / ocean; their sum equals `mpirun -np` |
| `nproc_x` / `nproc_y` | WRF `namelist.input` | Decomposition inside the WRF domain; `nproc_x * nproc_y` should equal `NnodesATM` |
| `NtileI` / `NtileJ` | ROMS ocean `.in` | ROMS tile decomposition; `NtileI * NtileJ` should equal `NnodesOCN` |

The two short runs on this machine used `mpirun -np` 2 and 3.

| Case | `mpirun -np` | Nnodes* | WRF | ROMS |
|------|--------------|---------|-----|------|
| INLET_TEST Coupled | 2 | WAV=1, OCN=1 | (none) | (set by NnodesOCN) |
| JOE_TC Coupled | 3 | ATM=1, WAV=1, OCN=1 | `nproc_x=nproc_y=1` | `NtileI=NtileJ=1` |

> **Note**
>
> When increasing parallelism, change `coupling_*.in`, the WRF namelist, and the ROMS `.in` together, then set `mpirun -np` to the new sum of `Nnodes*`.

## 6. Common pitfalls and recommended practices

The failures below were checked during deployment and the short runs on this machine, with the corresponding fix. The numbers are not a priority order.

1. **apt on this machine is unusable** (proxy `198.18.0.1` → 500/SSL). Fix: the conda-forge environment `coawst`, plus the `make` `.deb` installed with curl.
2. **MCT `./configure` is not executable.** Fix: `bash ./configure` or `chmod +x`.
3. **MCT must use the same `mpif90` as COAWST**, and `MCT_INCDIR` / `MCT_LIBDIR` must be exported. Otherwise a common failure is “module file was not generated by this compiler”, or `m_MCTWorld` is not found.
4. **`COAWST_APPLICATION` must be all capitals** (`INLET_TEST`), matching `inlet_test.h`. A lowercase match fails.
5. **The distributed `MY_ROOT_DIR` is the Cygwin path** `/cygdrive/d/models/COAWST`. On Linux it must be the real path (on this machine, `/workspace/COAWST`).
6. **The distributed setting is `which_MPI=intel`.** This stack uses `openmpi`.
7. **gfortran 15:** `-fallow-argument-mismatch` is already in `Linux-gfortran.mk` and in the MCT configure. MPI type-mismatch warnings can be ignored.
8. **`mpirun -np` equals the sum of the `Nnodes*` values** (2 for Inlet Coupled; 3 for JOE_TC Coupled).
9. **`./build_coawst.sh -j 4` worked on this machine.** If an `mpif.h` race appears, rebuild without parallelism or with `-j 1`.
10. **Do not compile WRF alone for a coupled application.** `build_coawst.sh` handles it when WRF is enabled.
11. **Switching cases requires a rebuild.** After changing the header or paths, clean or recreate `Build_roms` and the WRF products.
12. **Do not leave Intel-only options in the gfortran mk** (such as `-assume byterecl`). On older trees that makes SWAN/gfortran fail.
13. **WRF configure needs m4 / flex / bison** (conda-forge). `./compile` needs `/bin/csh` (`ln -sf …/tcsh /bin/csh` is enough).
14. **WRF configure prompts:** `printf '34\n1\n' | ./build_coawst.sh -j 4` (GNU dmpar + basic nesting).
15. **A JOE_TC run must set `GFORTRAN_CONVERT_UNIT=big_endian`**, or `RRTM_DATA` hits EOF. Use `/workspace/coawst-env-joe_tc.sh`. Do not put that setting into the default `coawst-env.sh`.
16. **JOE_TC namelist / wrfinput / wrfbdy live in `Projects/JOE_TC/`**, not in `Coupled/`. Copy them to the root before the run and symlink the `WRF/run` table files.
17. **Partial rebuild flags:** `-noclean -nocleanwrf -nocleanswan` can keep `configure.wrf`. A full clean deletes `Build_roms`, and with `-nocleanwrf` the script also skips `mct_params`.

- Check the ROMS+SWAN+MCT toolchain with INLET_TEST before building JOE_TC.
- Keep the default environment separate from the WRF environment: `coawst-env.sh` and `coawst-env-joe_tc.sh`.
- Before switching applications, back up `coawstM` and `build_coawst.sh` (this machine has `coawstM.inlet_test` / `coawstM.joe_tc`).
- Cut short runs off with `timeout`. After the coupling exchange and the output files check out, drop `timeout` and integrate the full length.

## 7. Further applications

These cases are in the repository and **were not compiled or test-run on this machine**. Paths and model combinations come from the repository layout and from excerpts of the manual.

| Case | Path | Models |
|------|------|--------|
| JOE_TC DiffGrid | `Projects/JOE_TC/DiffGrid` | ROMS+SWAN and WRF on different grids |
| Sandy | `Projects/Sandy/` | Full storm; `coupling_sandy.in`; often includes a PBS `run_coawst` example |

In DiffGrid, ROMS/SWAN share one grid and WRF uses another. v3.9 can generate weights such as `scrip_joe_tc_diffgrid.nc`, or `Lib/SCRIP_COAWST` can be built first and `scrip_coawst` run. The header must turn on `MCT_INTERP_WV2AT` and `MCT_INTERP_OC2AT`. DiffGrid case H already sets both.

Sandy is a full storm case and is often the PBS `run_coawst` example. Rank counts and inputs follow `coupling_sandy.in` and `sandy.h` in that directory. Set `mpirun -np` to `NnodesATM + NnodesWAV + NnodesOCN`. When switching applications, rebuild from a clean state and change `COAWST_APPLICATION` and `MY_*_DIR` as above.

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
