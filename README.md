# COAWST compile and run guide

[中文首页](README.zh-CN.md) · [中文全文](docs/user-guide.zh-CN.md)

**COAWST** (Coupled-Ocean-Atmosphere-Wave-Sediment Transport Modeling System) couples ROMS, SWAN/WW3, and optionally WRF through MCT. These docs cover compiling and short-running **INLET_TEST** (ROMS+SWAN) and **JOE_TC Coupled** (WRF+ROMS+SWAN) on Linux with v3.9.

| Language | Full guide |
|----------|------------|
| English | [docs/user-guide.en.md](docs/user-guide.en.md) |
| 中文 | [docs/user-guide.zh-CN.md](docs/user-guide.zh-CN.md) |

> **Note**
>
> Official spelling **COAWST** (spoken cowast / COWAST is the same system). Verified on Debian 13 (trixie), x86_64, 8 CPUs, commit `81df2ed`, 2026-10-28 (America/Los_Angeles). Sandy and JOE_TC DiffGrid were not verified.

Source: https://github.com/DOI-USGS/COAWST

---

# COAWST 编译与运行指南

[中文首页](README.zh-CN.md)

**COAWST** 通过 MCT 耦合 ROMS、SWAN/WW3 与可选的 WRF。本仓库文档说明在 Linux 上用 v3.9 编译并短时试跑 **INLET_TEST** 与 **JOE_TC Coupled**。

| 语言 | 全文 |
|------|------|
| 中文 | [docs/user-guide.zh-CN.md](docs/user-guide.zh-CN.md) |
| English | [docs/user-guide.en.md](docs/user-guide.en.md) |

> **Note**
>
> 官方拼写 **COAWST**。验证主机：Debian 13（trixie），x86_64，8 CPU，提交 `81df2ed`，2026-10-28（America/Los_Angeles）。Sandy / JOE_TC DiffGrid 未验证。

源码：https://github.com/DOI-USGS/COAWST
