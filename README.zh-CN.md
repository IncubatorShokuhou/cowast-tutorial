# COAWST 编译与运行指南

[English README](README.md) · [English user guide](docs/user-guide.en.md)

在 Linux 上编译并运行 **COAWST**（Coupled-Ocean-Atmosphere-Wave-Sediment Transport Modeling System）v3.9。全文在 [编译与运行指南](docs/user-guide.zh-CN.md)。

cowast / COWAST 指同一套 USGS 耦合系统。官方拼写为 **COAWST**。

实测提交为 `81df2ed`（`81df2edcd175fb87426b4d913f4fdaa42cfc50ad`）。主机是 Debian 13（trixie），x86_64，8 CPU，`/workspace` 下约 110G 可用，日期 2026-10-28（America/Los_Angeles）。本机已编译并短时试跑 **INLET_TEST**（ROMS+SWAN Coupled）与 **JOE_TC Coupled**（ROMS+SWAN+WRF）。**Sandy / JOE_TC DiffGrid 尚未在本机编过。**

## 文档

- [中文全文](docs/user-guide.zh-CN.md)
- [English user guide](docs/user-guide.en.md)
