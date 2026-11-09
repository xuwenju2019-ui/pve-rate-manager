# CTC · CDN流控程序 3.2

CDN Traffic Controller（CTC）用于在 Proxmox VE 宿主机上管理虚拟机网卡的分时限速。3.2 的安装包版本为 **3.2.0**。

- 分时上行规则在高峰解除插件上行限速，在两个非高峰时段分别使用固定上限，下行不限速。
- 固定双向规则会写入 PVE 网卡的 `rate`。规则卡片分别显示插件上行流控和上次核验的 PVE 原生 `rate`。
- 升级后，旧自适应规则各时段的最高值成为固定上限。安装和宿主机重启后，CTC 等待手动启动。

## 下载与安装

选择 [单文件安装脚本](./install-ctc-3.2.0.sh)、[ISO 安装包](./ctc-3.2.0.iso) 或 [完整 ZIP](./ctc-3.2.0-github.zip)。[SHA-256 校验文件](./SHA256SUMS-ctc-3.2.0) 随安装包提供。

在 **PVE 宿主机**的 Shell 中以 root 执行：

```bash
curl -fL -o install-ctc-3.2.0.sh https://raw.githubusercontent.com/xuwenju2019-ui/pve-rate-manager/main/install-ctc-3.2.0.sh
curl -fL -o SHA256SUMS-ctc-3.2.0 https://raw.githubusercontent.com/xuwenju2019-ui/pve-rate-manager/main/SHA256SUMS-ctc-3.2.0
awk '$2 == "install-ctc-3.2.0.sh"' SHA256SUMS-ctc-3.2.0 | sha256sum -c - &&
bash install-ctc-3.2.0.sh --check &&
bash install-ctc-3.2.0.sh
```

也可以从电脑下载 ISO，再通过 PVE 的 **local → ISO 镜像 → 上传**传到宿主机。上传后在宿主机 Shell 执行：

```bash
(
  set -e
  CTC_MOUNT=$(mktemp -d /mnt/ctc.XXXXXX)
  trap 'if mountpoint -q "$CTC_MOUNT"; then umount "$CTC_MOUNT"; fi; rmdir "$CTC_MOUNT"' EXIT
  mount -t iso9660 -o loop,ro "$(pvesm path local:iso/ctc-3.2.0.iso)" "$CTC_MOUNT"
  bash "$CTC_MOUNT/install.sh" --check
  bash "$CTC_MOUNT/install.sh"
)
```

安装和卸载会重启 PVE 管理服务，网页连接可能短暂断开。升级会停止旧服务并保留规则与统计，完成后从 PVE 页面手动开启。

## 启动和配置

以 `root@pam` 登录 PVE，刷新页面并打开「网卡限速」。点击「开启程序」，添加规则时选择虚拟机、网卡、高峰时间和非高峰上行上限。限速单位是 MB/s。

宿主机也可以使用以下命令：

```bash
ctc start
ctc status
ctc stop
```

`ctc start` 启动本次运行，`ctc status` 显示服务及清理状态，`ctc stop` 提交停止请求。停止后可再次查看状态确认清理结果。

## 一键卸载

在 **PVE 宿主机 Shell** 以 root 执行：

```bash
ctc uninstall
```

卸载会停止服务，核验并解除规则所管理网卡的限速，移除 PVE 入口与程序文件。规则、统计和备份保留在 `/var/lib/pve-rate-manager`。

若当前安装的旧包没有内置卸载命令，可使用本页提供的新版安装脚本执行 `bash install-ctc-3.2.0.sh --uninstall`。如果提示清理失败，检查下面的报告，处理失败网卡后重新运行卸载命令：

```bash
cat /var/lib/pve-rate-manager/cleanup-status.json
```

## 验证范围

本包通过本地自动测试、浏览器检查和安装包完整性校验。PVE 实机安装、卸载和 Linux 网卡流控验收仍待完成。高峰核验可能引用旧的 PVE 配置快照，多网卡仍共享部分流控锁，抓包目录权限问题也待修复。建议先在测试节点试用。
