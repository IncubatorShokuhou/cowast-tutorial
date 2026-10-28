# COAWST compile and run guide

Compile and run USGS **COAWST** v3.9 on Linux. cowast / COWAST are spoken names for the same USGS coupled system. The official spelling is **COAWST**.

| Language | Guide |
|----------|-------|
| English | [docs/user-guide.en.md](docs/user-guide.en.md) |
| 中文 | [docs/user-guide.zh-CN.md](docs/user-guide.zh-CN.md) |

The Chinese landing page is [README.zh-CN.md](README.zh-CN.md).

Measured on Debian 13 (trixie), x86_64, 8 CPUs, COAWST v3.9 commit `81df2ed` (`81df2edcd175fb87426b4d913f4fdaa42cfc50ad`), 2026-10-28 (America/Los_Angeles). **INLET_TEST** (ROMS+SWAN) and **JOE_TC Coupled** (WRF+ROMS+SWAN) were compiled and short-run on this machine. Sandy and JOE_TC DiffGrid were not built.

Source: https://github.com/DOI-USGS/COAWST

---

# COAWST 编译与运行指南

[中文首页](README.zh-CN.md)

在 Linux 上编译并运行 USGS **COAWST** v3.9。cowast / COWAST 是同一套 USGS 耦合系统的口语称呼，官方拼写为 **COAWST**。

| 语言 | 指南 |
|------|------|
| 中文 | [docs/user-guide.zh-CN.md](docs/user-guide.zh-CN.md) |
| English | [docs/user-guide.en.md](docs/user-guide.en.md) |

本机实测环境是 Debian 13（trixie），x86_64，8 CPU，提交 `81df2ed`（`81df2edcd175fb87426b4d913f4fdaa42cfc50ad`），日期 2026-10-28（America/Los_Angeles）。**INLET_TEST**（ROMS+SWAN）与 **JOE_TC Coupled**（WRF+ROMS+SWAN）已编译并短时试跑。**Sandy / JOE_TC DiffGrid 尚未在本机编过。**

源码：https://github.com/DOI-USGS/COAWST
