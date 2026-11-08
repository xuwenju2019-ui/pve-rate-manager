此插件为了解决pve虚拟机特别针对cdn平台带宽限速，解决流量超发的问题
后续还会增加流量自适应模块以此来适配cnd虚拟机的上传流量捕捉，以提升平台收益

# PVE 网卡限速 v2.0.0

本版整合 PVE 页面入口、中文参数编码和界面缓存修复，入口统一为蓝底白字“网卡限速”。

支持每天、每周指定日期、仅一次、跨午夜，以及多虚拟机/多网卡独立规则。沿用 PVE 的 root@pam 登录，无独立管理密码和额外对外管理端口。上下行分别采用同一限速值，关闭时完全解除限速。

从 1.x 升级可使用同一安装器，保留规则。安装前备份、文件核验、宿主机中文回归测试、失败恢复和卸载清理已纳入安装流程。安装会短暂重启 PVE 管理服务，不重启虚拟机。

适用目标：PVE 9.2.2、QEMU 虚拟机、Linux bridge。第三方本地扩展，PVE 更新后可能需要重新安装入口。本版完整包仍需主机上的时段切换和流量验收，不能把离线测试等同全部实机通过。

附件：

- `pve-rate-manager-2.0.0.tar.gz`：宿主机安装/升级。
- `SHA256SUMS`：安装包校验值。

详细使用、回滚、卸载及发布步骤见 README。
# PVE 网卡限速 3.0.0

在 PVE 顶部点击蓝色“网卡限速”，沿用 root@pam 登录。新版本加入上行自适应和每日上行流量记录，不需要独立密码或额外 TCP 管理端口。

面向 Proxmox VE 9.2.2、Linux bridge、QEMU 虚拟机的第三方扩展。当前交付已完成离线测试，真实 PVE 的内核流控和浏览器验收仍需在部署主机执行，见 VALIDATION.md。

## 新模式的时间安排

| 宿主机本地时间 | 上行 | 下行 |
|---|---|---|
| 00:00–11:30 | 在你填写的上午最低/最高上限之间调整 | 不设本程序限速 |
| 11:30–14:00 | 完全解除本程序限速 | 不设本程序限速 |
| 14:00–17:00 | 在你填写的下午最低/最高上限之间调整 | 不设本程序限速 |
| 17:00–次日 00:00 | 完全解除本程序限速 | 不设本程序限速 |

两个非高峰范围分别填写，没有预设速度。最低值是“自动调整的速度上限”的下界，不保证平台一定产生这么多流量。最低=最高时，该时段使用固定上行上限。范围允许 0.001–1000000 MB/s、最多三位小数；硬件实际能达到的速度仍受线路、平台和虚拟机限制。内部以每 MB 1024² 字节计算，与原 PVE rate 的单位习惯一致。

自适应根据发送需求和队列积压逐步调整：初次从最低上限开始，连续有需求时提高，上限调整至少间隔 30 秒；持续低利用且无积压约 120 秒后缓慢回落。它不探测运营商剩余带宽，也不保证收益提高。高峰期不执行自适应压速。

## 从 2.0 升级

旧规则保留原来的固定双向限速、每天/每周/仅一次安排。升级不会默默把旧规则转换成单向控制。

要使用新功能：编辑旧规则，选择“高峰放开 · 非高峰上行自适应”，填写两个范围，预览并保存。这个操作会删除此网卡原有的 PVE 双向 rate，改用 TAP ingress → 独立 IFB/HTB 的上行控制；MAC、bridge、VLAN、firewall、queues 和其它网卡参数保留。新安装默认没有规则。

## 最方便的安装方式

将 install-pve-rate-manager-3.0.0.sh 上传到 PVE 宿主机 /root，然后以 root 打开宿主机 Shell：

~~~bash
cd /root
bash install-pve-rate-manager-3.0.0.sh --check
bash install-pve-rate-manager-3.0.0.sh
~~~

第一条只核验内嵌包和文件完整性，不修改 PVE。第二条自动校验、临时展开并安装，不需要你手动解压。脚本还支持 --uninstall，保留该文件便于卸载。不要在 PVE 存储的“导入/从 URL 下载”对话框里安装程序。

安装会保存安装前备份，并短暂重启 pvedaemon/pveproxy，管理网页和 Shell 可能断开；不会重启虚拟机。重新连接后用 root@pam 登录，按 Ctrl+F5，点击“网卡限速”。升级保留规则，后台按当前时段继续执行；安装期间请勿同时编辑规则。

## GitHub URL 安装

先把本次发布的单文件安装脚本和 SHA256SUMS 上传到仓库根目录并提交。下面地址在上传后才有效；本次没有替你发布到 GitHub。

~~~bash
(
set -e
cd "$(mktemp -d /root/pve-rate-v3.XXXXXX)"
curl -fL --retry 3 -o install-pve-rate-manager-3.0.0.sh \
  https://raw.githubusercontent.com/xuwenju2019-ui/pve-rate-manager/main/install-pve-rate-manager-3.0.0.sh
curl -fL --retry 3 -o SHA256SUMS \
  https://raw.githubusercontent.com/xuwenju2019-ui/pve-rate-manager/main/SHA256SUMS
grep '  install-pve-rate-manager-3.0.0.sh$' SHA256SUMS | sha256sum -c -
bash install-pve-rate-manager-3.0.0.sh
)
~~~

校验和用于检查文件损坏，来源信任依赖 HTTPS 和你的 GitHub 仓库。建议先核对下载脚本，再执行；不使用 curl 管道直接执行。标准源码包 pve-rate-manager-3.0.0.tar.gz 也可解压后运行 bash install.sh。

## 鼠标操作

1. 添加规则，选择当前节点的虚拟机 ID 和网卡。100 是 VM ID，不是宿主机编号。
2. 选择自适应模式，填写上午和下午各自的最低、最高上限。
3. 点击“预览当前效果”；预览不会修改网卡。
4. 保存后查看规则状态、当前上限、下一次切换和执行记录。
5. 在“每日上行流量”选择日期和网卡；可以查看今天或最近 7 天并导出 CSV。

一张网卡只能有一条规则，新模式已包含上述四个时段。不同 VM、不同网卡可分别配置，单节点最多 128 条规则。关闭单条规则或暂停总调度，会尝试完全解除对应规则的限速；显示失败时不能视为已经解除。暂停调度不停止流量采集。

“程序运行中”表示能连到后台；“调度开启”表示自动执行；规则的“正在限速/未限速/失败”表示最近检查结果。PVE 退出登录不影响后台运行。后台停止后，需要在 Shell 启动，网页不能启动已经停止的后台。

## 每日流量的口径

- 采集所选 TAP 网卡 RX 字节，即虚拟机向宿主机发送的上行量。固定旧规则也可统计。
- 这是流控前的网卡发送量，包含协议、重传、所选网卡上的内网流量和可能被流控丢弃的数据；不是公网成功到达量、PCDN 平台有效流量或运营商账单。
- 以宿主机本地自然日分组，保留今天和前 6 天，正常运行时约每分钟清理一次；重新启动也会清理。删除规则后已有日记录保留至到期，不再采集该规则。
- 显示总上行、午高峰、晚高峰、非高峰，内部保存原始字节；界面 GB=10⁹ 字节，速率 MB/s=1024² 字节/秒。
- 每 5 秒采样、约每 60 秒落盘。跨午夜及高峰边界按采样间隔比例拆分，因此边界分桶是估算值。
- 初次添加不追溯历史。网卡重建、VM/宿主机重启、计数器倒退等情况重新建立基线，并标记起点或间断，不能恢复停机期间已丢失的计数。
- 若仅重启本程序且 TAP 计数连续，可从持久化基线恢复总增量；长间断会标记，时间段拆分精度下降。采样不可用或存储失败时页面显示原因，不把旧速度当成实时速度。

## 控制范围与安全行为

自适应仅添加本规则拥有的 IFB 和 ingress 重定向，不替换 TAP 的 root/egress 队列。已有其它 ingress 流控、IFB 同名但所有权不匹配时拒绝接管。解除时只删除本程序的重定向和 IFB；保留空 clsact，避免破坏其它 egress 规则。

保存规则即接管此网卡的限速。管理期间手动添加 PVE rate 会被自适应模式清除；需要手动管理时先删除规则。新模式不限制下行，但线路本身、其它外部流控或旧规则仍可能限制实际吞吐。

VM 锁定、模板、pending 配置、摘要冲突、MAC 改变、备份失败时拒绝危险写入，不强制解锁、不清 pending、不重启 VM。MAC 改变时尝试解除本程序流控，需删除旧规则再新建。VM 停止时等待 TAP 出现；报错不会视为成功。每 5 秒检查，配置/流控通常每 30 秒重核对，PVE 操作耗时和规则数量可能延长响应，不是硬实时系统。

程序正常停止会通过 ExecStopPost 清理本程序的上行 IFB。旧固定规则写入的 PVE rate 不因停服务自动消失。异常断电/清理失败时不可假设已经解除；恢复后检查状态，必要时运行下面的 release-all。数据库或规则损坏也会阻止启动，需要先处理错误。

## 多宿主机与迁移

每个 PVE 节点分别安装、分别登录管理：登录哪个节点地址，就管理那个节点。左侧树点击另一个节点不会切换本程序后台。没有跨节点同步配置、共享流量汇总或自动迁移跟随。

迁移前在原节点删除规则并确认解除，迁移后在目标节点新建规则。其它主机可使用相同程序、不同 VM/NIC/速度范围；不要让两个节点同时接管同一 VM。

## 检查、日志与空间

~~~bash
systemctl status pve-rate-manager --no-pager
pvesh get /rate-manager/status --output-format json
python3 /opt/pve-rate-manager/integration.py check
tail -n 80 /var/log/pve-rate-manager/manager.log
timedatectl status
~~~

| 内容 | 路径 | 清理策略 |
|---|---|---|
| 规则 | /var/lib/pve-rate-manager/rules.json | 原子保存，不循环增大 |
| 每日流量 | /var/lib/pve-rate-manager/traffic.sqlite3 | 7 个自然日；数据库页数限制通常约 32 MiB |
| 程序日志 | /var/log/pve-rate-manager/manager.log | 5 MiB × 5，约 25 MiB |
| 网卡写前备份 | /var/lib/pve-rate-manager/backups | 最多 100 个且约 10 MiB |
| PVE 集成备份 | /var/lib/pve-rate-manager/integration-backups | 自己命名的文件最多 20 个且约 64 MiB |
| 成功安装后的快照 | /var/lib/pve-rate-manager/install-backups | 最多 3 份且约 128 MiB |
| 私有接口 | /run/pve-rate-manager/api.sock | root 专用，不监听额外 TCP 端口 |

限额是程序管理的数据量，不是整块宿主机磁盘用量保证；SQLite 日志、安装过程临时文件、系统 journal 另计。页面近期事件仅保留 100 条内存记录，显示最近 40 条，完整轮转文件用于排查。重复同类错误 15 分钟内去重，轮询不写访问日志；systemd 输出限频但不修改系统 journal 总容量配置。旧版本手动建的 entry-fix 备份目录不会擅自删除。失败安装留下的快照保留供恢复，下次成功安装会清理数字命名快照。

## 停用与卸载

暂时停止程序并解除所有管理规则的限速：

~~~bash
systemctl stop pve-rate-manager
python3 /opt/pve-rate-manager/manager.py release-all
~~~

重新启用：

~~~bash
systemctl enable --now pve-rate-manager
~~~

使用单文件安装器卸载，不需要手动解压：

~~~bash
bash /root/install-pve-rate-manager-3.0.0.sh --uninstall
~~~

或在源码目录执行 bash uninstall.sh。卸载先停服务，解除所有规则的限速，撤销自己的入口后删除程序；失败会停止，不继续删除程序。规则、日流量、日志和备份保留，重新安装可继续使用，过期日记录会自动清理。

仅撤销界面入口、保持后台调度：

~~~bash
python3 /opt/pve-rate-manager/integration.py remove
systemctl restart pvedaemon pveproxy
~~~

PVE 软件更新可能覆盖入口，更新后检查并用兼容安装包重装。不要把旧完整 PVE 文件覆盖到新版本上。安装失败会尝试恢复当次安装前的程序文件和服务状态，不覆盖当前规则、日流量。恢复快照必须用对应版本 deployment.py，且不能跨后续 PVE 包升级盲目还原。降级 2.0 前须先删除/转换自适应规则并解除 IFB，否则旧程序无法处理新规则。

## 上传 GitHub

解压 pve-rate-manager-3.0.0-github.zip，将里面 pve-rate-manager 文件夹的内容上传到仓库根目录。包含源码、测试、说明、GitHub Actions、单文件安装器、标准 tar 包和 SHA256SUMS；不要再套一层目录，也不要上传宿主机真实状态/备份文件。

可创建 v3.0.0 Release 并附安装器、tar 包和 SHA256SUMS；发布说明见 RELEASE_NOTES.md。GitHub Actions 的离线与隔离网络任务需你上传后实际运行，不能把配置了任务当成已通过。

## 技术参考

- [Linux IFB ingress redirect](https://man7.org/linux/man-pages/man8/tc-mirred.8.html)
- [HTB 流控](https://man7.org/linux/man-pages/man8/tc-htb.8.html)
- [Linux 网卡统计口径](https://docs.kernel.org/networking/statistics.html)
- [iproute2 filter JSON 输出](https://github.com/iproute2/iproute2/blob/main/tc/tc_filter.c)
- [iproute2 class JSON 输出](https://github.com/iproute2/iproute2/blob/main/tc/tc_class.c)
- [PVE 认证与 CSRF](https://github.com/proxmox/pve-manager/blob/master/PVE/HTTPServer.pm)

MIT License。
