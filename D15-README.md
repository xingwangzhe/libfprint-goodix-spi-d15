# Huawei MateBook D15 2022 / BoF-XX 采图修复

[完整排查与修复文章](https://xingwangzhe.fun/posts/matebook-d15-goodix-gdix51c0-fingerprint/)

[English](D15-README_EN.md) · [项目首页](README.md)

本分支从 [berkekbgz/libfprint-goodix-spi](https://github.com/berkekbgz/libfprint-goodix-spi) 的 `010a665f54089a1632b1ab7be588b316ace934e2` 出发，并合入上游 main 的 `505dc7e`。硬件厂家是 **Goodix（汇顶科技）**，笔记本整机厂家是华为；指纹设备是 SPI 总线上的 `ACPI\GDIX51C0`。修改文件位于 `drivers/gdix51c0/`，可用 `git log -p -- drivers/gdix51c0/` 查看补丁。

这台 BoF-XX 在图像模式 ACK 后，传感器于约 80 ms 的 SPI 静默期内拉高中断脚。监听线程消耗该边沿，却不再检查保持高电平的 IRQ，导致等待 TLS 图像数据 `0x17` 超时。修复在静默结束后主动唤醒监听线程，并在接收 ACK 的线程内进入静默期。

**2026-10-11 本机结果：**右手食指录入完成，一次 `fprintd-verify` 返回 `verify-match`。去掉临时诊断日志后，服务能读取已录入的手指；再次验证因未触摸传感器而超时。尚未进行重启或 GDM 登录测试。

**2026-11-02 复测：**合入上游 `505dc7e`（固件 `_CID` 的 spidev 绑定修复、损坏包头拒绝与排空）后重新构建，13 个 goodix 单测通过。安装后的 udev 规则为 `acpi:GDIX51C0:*`；解绑 `spidev` 再触发 udev，设备能自动重新绑定。采图静默的 `suppress_on_ack` 布防从调用者发命令前移到 `spi_lock` 内、写完命令之后，避免残留 ACK 提前触发静默；ACK 超时重试前会清理迟到的 ACK 和图像记录。右手食指在安装后、重新录入后、重新绑定后各得到一次 `verify-match`。另外临时录入左手中指可以 `enroll-completed`，但随后三次 `verify` 均为 `verify-no-match`，未计为成功。整机重启后的 GDM 登录仍未测试。

本机安装的共享库位于 `/opt/goodix-spi-driver/libfprint/libfprint-2.so.2.0.0`；同目录下带 `.pre-ack-quiet` 后缀的备份是最初没有 ACK 静默修复的版本，重新安装会覆盖主库。systemd 配置位于 `/etc/systemd/system/fprintd.service.d/20-goodix-spi-libfprint.conf`，SPI 绑定规则位于 `/etc/udev/rules.d/70-libfprint-2.rules`。若要回退到最初版本，使用管理员权限恢复该备份并重启 `fprintd`；旧版能在本机识别设备，但无法完成录入。

本副本按 LGPL-2.1 第 3 条转换为 **GPL-3.0-only**，原作者版权与来源见 [PROVENANCE.md](PROVENANCE.md)。华为 Windows 驱动包、指纹图像、模板、密钥和校准数据均未收入 Git 仓库。
