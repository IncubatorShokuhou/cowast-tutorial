# COAWST 编译与运行指南

[English README](README.md) · [English user guide](docs/user-guide.en.md)

**COAWST**（Coupled-Ocean-Atmosphere-Wave-Sediment Transport Modeling System）通过 MCT 耦合 ROMS、SWAN/WW3 与可选的 WRF。本仓库文档说明在 Linux 上用 v3.9 编译并短时试跑 **INLET_TEST**（ROMS+SWAN）与 **JOE_TC Coupled**（WRF+ROMS+SWAN）。

全文：[编译与运行指南](docs/user-guide.zh-CN.md) · [English](docs/user-guide.en.md)

> **Note**
>
> 官方拼写为 **COAWST**（口语里的 cowast / COWAST 指同一系统）。验证主机：Debian 13（trixie），x86_64，8 CPU，提交 `81df2ed`，日期 2026-10-28（America/Los_Angeles）。Sandy / JOE_TC DiffGrid 未验证。
