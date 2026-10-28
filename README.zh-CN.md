# COAWST 编译与运行指南

[English README](README.md) · [English user guide](docs/user-guide.en.md)

本页是中文入口。全文见 [编译与运行指南](docs/user-guide.zh-CN.md)。

本指南说明如何在 Linux 上编译并运行 **COAWST**（Coupled-Ocean-Atmosphere-Wave-Sediment Transport Modeling System）v3.9。内容覆盖依赖安装、环境变量、应用配置、INLET_TEST（ROMS+SWAN）与 JOE_TC Coupled（WRF+ROMS+SWAN）的编译与短时试跑，以及常见失败点。

口语里常见的 cowast / COWAST 均指同一套 USGS 耦合系统。官方拼写为 **COAWST**。

本指南对应 **COAWST v3.9**。源码仓库为 `https://github.com/DOI-USGS/COAWST.git`，实测提交为 `81df2ed`（完整哈希 `81df2edcd175fb87426b4d913f4fdaa42cfc50ad`）。主机环境为 Debian 13（trixie），x86_64，8 CPU，`/workspace` 下约 110G 可用。验证日期为 2026-10-28（America/Los_Angeles）。

面向已熟悉 WRF 编译与运行的用户。本机已完成依赖与 MCT 安装，并完成 **INLET_TEST**（ROMS+SWAN Coupled）与 **JOE_TC Coupled**（ROMS+SWAN+WRF）的编译与短时耦合试跑。**Sandy / JOE_TC DiffGrid 尚未在本机编过。**

## 文档

- [中文全文](docs/user-guide.zh-CN.md)
- [English user guide](docs/user-guide.en.md)
