# COAWST compile and run guide

Short entry point for compiling and running USGS **COAWST** v3.9 on Linux. The full notes are the user guides below. Spoken names cowast / COWAST refer to the same USGS coupled system. The official spelling is **COAWST**.

GitHub shows this file on the repository home. The Chinese landing page is [`README.zh-CN.md`](README.zh-CN.md). Full guides live under `docs/`.

| Language | Full guide |
|----------|------------|
| English | [docs/user-guide.en.md](docs/user-guide.en.md) |
| 中文 | [docs/user-guide.zh-CN.md](docs/user-guide.zh-CN.md) |

Verified on Debian 13 (trixie), x86_64, 8 CPUs, against COAWST v3.9 commit `81df2ed` (`81df2edcd175fb87426b4d913f4fdaa42cfc50ad`), on 2026-10-28 (America/Los_Angeles). This machine compiled and short-ran **INLET_TEST** (ROMS+SWAN) and **JOE_TC Coupled** (WRF+ROMS+SWAN). Sandy and JOE_TC DiffGrid were not built here.

Source: https://github.com/DOI-USGS/COAWST

---

# COAWST 编译与运行指南

[中文首页](README.zh-CN.md)

在 Linux 上编译并运行 USGS **COAWST** v3.9 的入口。全文在下面两份用户指南里。口语里的 cowast / COWAST 均指同一套 USGS 耦合系统，官方拼写为 **COAWST**。

| 语言 | 全文 |
|------|------|
| 中文 | [docs/user-guide.zh-CN.md](docs/user-guide.zh-CN.md) |
| English | [docs/user-guide.en.md](docs/user-guide.en.md) |

本机验证环境：Debian 13（trixie），x86_64，8 CPU，COAWST v3.9 提交 `81df2ed`（`81df2edcd175fb87426b4d913f4fdaa42cfc50ad`），日期 2026-10-28（America/Los_Angeles）。已完成 **INLET_TEST**（ROMS+SWAN）与 **JOE_TC Coupled**（WRF+ROMS+SWAN）的编译和短时试跑。**Sandy / JOE_TC DiffGrid 尚未在本机编过。**

源码：https://github.com/DOI-USGS/COAWST
