# Goodix GDIX51C0 Linux 指纹驱动 · Huawei MateBook D15 2022

[完整排查与修复文章](https://xingwangzhe.fun/posts/matebook-d15-goodix-gdix51c0-fingerprint/)

[English](README_EN.md)

## 硬件厂家与适用机型

**指纹传感器厂家是深圳市汇顶科技股份有限公司（Goodix）；华为是笔记本整机厂家。** 本仓库适配的指纹硬件是 Goodix `GDIX51C0`，通过 ACPI 枚举、SPI 通信，并使用 GPIO 中断与复位线。实测设备的芯片 ID 为 `0x2504`，配置为 ChicagoHS、80 × 64 像素。华为提供的 Windows 驱动包 `gfspi.inf` 同样匹配 `ACPI\GDIX51C0`，但其 DLL、固件和安装程序**没有**收入本仓库。

| 主机 | 验证范围 |
| --- | --- |
| Huawei MateBook D15 2022，BoF-XX / M1010 | 本分支完成右手食指录入，`fprintd-verify` 返回 `verify-match`；尚未验证重启后的 GDM 登录 |
| Huawei MateBook 16s | [上游项目](https://github.com/berkekbgz/libfprint-goodix-spi)声明完成硬件验证；本分支未在该机型复测 |

## 修复内容

上游驱动在图像采集时让 SPI 总线静默约 80 ms。BoF-XX 的传感器在静默期把中断脚拉高；监听线程消耗中断边沿后，直到下一个边沿才读取数据，因此图像响应 `0x17` 超时。本分支在解除静默时唤醒监听线程并重新检查中断电平，同时在收到采图 ACK 的线程内立即进入静默期。2026-11-02 又合入上游 `505dc7e`：固件带 `_CID` 时 udev 规则放宽为 `acpi:GDIX51C0:*`，修复重启后 spidev 不绑定；校验失败的包头会被拒绝并排空，避免吞包。采图静默的布防移到 SPI 总线锁内，防止迟到 ACK 提前静默。详细日志、修改范围和回退方法见 [D15-README.md](D15-README.md)。

## 安装和使用

本项目是 [libfprint-goodix-spi](https://github.com/berkekbgz/libfprint-goodix-spi) 的派生版本，安装脚本会获取指定版本的 libfprint，构建影子库并配置 `fprintd`。默认构建还会安装带学习持久化功能的 fprintd；若要保留发行版 fprintd，可使用 `--without-fprintd`。

```bash
./install.sh --without-fprintd
fprintd-enroll -f right-index-finger
fprintd-verify
```

传感器首次使用需要 TLS PSK。默认安装流程可能生成并写入新密钥；双系统切换时 Windows 和 Linux 的密钥可能相互替换。请先阅读上游 [首次运行说明](https://github.com/berkekbgz/libfprint-goodix-spi#first-run)。指纹图像和模板在主机处理与保存；这是一项便利解锁功能，不应作为唯一的身份验证方式。

## 协议和来源

**本派生版本使用 GNU GPL v3.0（`GPL-3.0-only`）。** 上游原本使用 `LGPL-2.1-or-later`；该协议第 3 条允许对给定副本改用普通 GPL，并指定版本 3。本仓库已同步调整源码许可证声明，保留原作者版权信息，在 [PROVENANCE.md](PROVENANCE.md) 记录来源和转换。仓库内针对 fprintd 的补丁原本来自 `GPL-2.0-or-later` 代码，在本副本按 GPLv3 分发。许可证全文见 [LICENSE](LICENSE)。

本仓库不包含 Goodix 或华为的 Windows 驱动二进制、固件、指纹图像、模板或设备密钥。公开源码与本机安装状态是两回事；提交或下载本仓库不会上传本机 `/var/lib/fprint` 中的数据。
