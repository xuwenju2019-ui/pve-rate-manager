# CDN 调度控制台 3.1

用于 Proxmox VE 9.2.2 / Linux bridge / QEMU 虚拟机的本地扩展。沿用 PVE root@pam 登录，顶部原生蓝色“网卡限速”入口，不新增公网管理端口。版本文件使用 3.1.0。

## 先看 Web 界面

打开交付目录的“CDN调度控制台-3.1-Web预览.html”，无需安装、联网或登录 PVE。页面中的主机、流量与一键抓包结果为明确标记的演示数据，操作仅改变当前网页内存。可切换总览、网卡规则、带宽计量、上行抓包、执行记录和设置；真实上传的 PCAP 在浏览器 Worker 中本地解析。预览不连接你的主机。

## 3.1 功能

- 苹果风格浅色界面、侧栏导航、柔和圆角、清晰层级；主标题“CDN 调度控制台”，移除顶部写死时段的说明。
- 明确区分“生效规则”与“正在限速”，高峰上行放开仍算规则正常生效。
- 每张网卡独立设置午/晚高峰，默认策略 11:30–14:00、17:00–次日 00:00；自定义支持跨午夜，两个高峰不能重叠或首尾相接，必须留下两个非高峰区间。
- 高峰解除本程序上行限制；晚高峰结束至午高峰开始使用第一组自适应范围，午高峰结束至晚高峰开始使用第二组；下行不设本程序上限。最低上限不是实际流量保证。
- 带宽计量显示上传/下载累计与实时速度、按日期/网卡筛选、15 分钟历史均值曲线、双向 CSV。保留今天与前 6 个自然日，每分钟清理。没有有效采样时显示不可用。
- 一键选择 VM/网卡、省份、运营商，抓 30/60/120 秒上行，分析出省、跨运营商、地域/运营商分布、TOP 对端和上行曲线。tcpdump 不可用时给出安装提示，不影响基础调度。
- PCAP/PCAPNG 文件可在浏览器本地导入，需填写服务器 IP 确定上行方向，支持多个 IPv4/IPv6。多接口 PCAPNG 拒绝直接相加，提示导出单接口。

## 地域和运营商分析的前提

**安装包没有附带商业或未经授权的 IP 归属数据库，也不会访问第三方抓包网站。** 设置 → 导入归属数据库 CSV。字段为 network,country,region,isp，network 是 IPv4/IPv6 CIDR，country 使用 CN 等国家代码；支持最长前缀匹配。文件最多 8 MiB / 150000 条网段。可下载格式示例，但示例不是可用于生产的数据库。需自行取得有权使用且覆盖目标地区的数据，并保持更新。

没有归属库时仍能采集、查看对端和上行曲线，出省/跨运营商显示“不可计算”。无法识别、境外、内网与保留地址单列，云服务商不会被强行归为电信/联通/移动。

出省比例=境内已识别省份的外省上行字节/境内已识别省份的上行字节。跨运营商比例使用境内已识别运营商的字节作分母，分别显示识别覆盖率。抓包样本不代表全天，IP 归属不证明实际传输路径，不等于平台有效流量/账单/收益。

## 安装 / 从 2.0、3.0 升级

不需要先卸载。把 install-pve-rate-manager-3.1.0.sh 传到宿主机 /root，在宿主机 root Shell 执行：

~~~bash
bash /root/install-pve-rate-manager-3.1.0.sh --check
bash /root/install-pve-rate-manager-3.1.0.sh
~~~

--check 仅校验内嵌包，不修改 PVE；正常执行自动临时展开，不需手动解压。安装保存程序/PVE 集成快照，校验文件、运行离线与 PVE Perl 中文测试，失败尝试还原。短暂重启 PVE 管理服务，网页可能断开，不重启虚拟机。完成后 Ctrl+F5，使用 root@pam 点击蓝色“网卡限速”。安装期间不要编辑规则。

旧固定双向规则仍保持原方向及每天/每周/仅一次策略。已有 3.0 自适应规则自动采用默认高峰；新安装不创建任何默认规则。首次启动 3.1 以 SQLite 一致性备份复制 3.0 统计到独立 traffic-3.1.sqlite3，再迁移结构；原 traffic.sqlite3 保留，便于安装失败恢复原程序。旧下载数据不存在，显示“暂无历史”，不会补为真实零流量。原库可能额外占用约 32 MiB。

修改高峰时段前按旧策略结算可读取的增量，修改后按新策略采集；历史分项不重算。网卡停机时允许编辑，重新采样会建立新起点，间断不追溯。

### 通过 PVE 网页传 ISO

可以使用 pve-rate-manager-3.1.0.iso（真正的 ISO9660 数据光盘，包含同一个安装器）。在 local → ISO 镜像 → 上传；或上传 GitHub 后使用文件直链“从 URL 下载”。下载只存储文件，不会自动安装。完成后在宿主机 Shell：

~~~bash
(
set -e
PRM_MOUNT=$(mktemp -d /mnt/pve-rate.XXXXXX)
trap 'if mountpoint -q "$PRM_MOUNT"; then umount "$PRM_MOUNT"; fi; rmdir "$PRM_MOUNT"' EXIT
mount -t iso9660 -o loop,ro "$(pvesm path local:iso/pve-rate-manager-3.1.0.iso)" "$PRM_MOUNT"
bash "$PRM_MOUNT/install.sh" --check
bash "$PRM_MOUNT/install.sh"
)
~~~

不要把 GitHub /blob/main/ 网页地址当文件直链；HTML 文件不能挂载成 ISO。镜像供宿主机读取安装器，不需要挂到虚拟机，也不是启动盘。

### GitHub URL

先将 GitHub 上传包里的目录内容提交到仓库根目录，以下地址才会存在。此交付未替你发布。

~~~bash
(
set -e
cd "$(mktemp -d /root/pve-rate31.XXXXXX)"
curl -fL --connect-timeout 10 --max-time 90 -o install-pve-rate-manager-3.1.0.sh \
  https://raw.githubusercontent.com/xuwenju2019-ui/pve-rate-manager/main/install-pve-rate-manager-3.1.0.sh
bash install-pve-rate-manager-3.1.0.sh --check
bash install-pve-rate-manager-3.1.0.sh
)
~~~

宿主机访问 GitHub 的网络问题不会因换格式而消失，可在电脑下载 ISO 后用 PVE 网页上传。来源信任依赖你的仓库和 HTTPS，校验和仅检查完整性。

## 一键上行抓包

选择已运行 QEMU VM 的实际网卡，程序从其 TAP 接口采集，使用 Linux 入站方向（对应虚拟机发出方向），不会采集整个 vmbr 或其它 VM。转发多个 MAC 的虚拟机也按接口方向采集。只分析 IPv4/IPv6 报文，支持 Ethernet/VLAN、RAW、Linux SLL/SLL2；不解密 HTTPS 内容。采集保存 128 字节头部片段，统计使用原始报文长度；不能保证片段完全不包含应用数据。

宿主机需有 tcpdump 和 nobody 用户，若未安装，可在确认软件源可用后用 apt install tcpdump。安装器不自动联网安装此可选依赖；缺少它时其它功能正常运行。捕获过程打开接口后由 tcpdump 降权，解析在独立 nobody 进程中进行，限制时间/内存/文件大小，不阻塞规则调度。

每节点一个采集/分析任务，单次最多 100 MiB、总原始包预算 500 MiB，开始前检查磁盘余量。原始包 24 小时清理，报告最多保留 7 天/50 份。主动取消不生成报告。达到大小上限时提前结束，尾部不完整报文不参与统计，报告说明截断；采集丢包情况单列。不常驻全量抓包。

界面显示 TOP 6 分布及 TOP 10 对端，CSV 对端最多 100 条；内部最多跟踪 10000 个对端，超限部分明确计入未知，不能假装已识别。外部导入限 100 MiB、5 分钟样本；浏览器解析超时会停止。原始报文长度包含协议和重传，抓包样本不会再次累计入带宽计量。

## 空间、路径与节点范围

| 内容 | 路径 / 限额 |
|---|---|
| 规则 | /var/lib/pve-rate-manager/rules.json |
| 新版统计与曲线 | /var/lib/pve-rate-manager/traffic-3.1.sqlite3，通常约 32 MiB 页上限 |
| 原统计迁移备份 | /var/lib/pve-rate-manager/traffic.sqlite3，不由 3.1 自动删除 |
| 程序日志 | /var/log/pve-rate-manager/manager.log，5 MiB × 5 |
| 网卡写前备份 | 最多 100 个 / 约 10 MiB |
| 集成备份 | 最多 20 个 / 约 64 MiB |
| 安装快照 | 成功安装后最多 3 份 / 约 128 MiB |
| 抓包、报告和归属库 | /var/cache/pve-rate-manager-capture，原始包最多 500 MiB，归属库最多 8 MiB，报告各有上限 |

限额不等于整块宿主机磁盘总容量保证，临时文件、SQLite journal、系统 journal 和手工备份另计。后台运行时每分钟清理过期数据；程序停止期间不执行清理，恢复后继续。浏览器本地导入的抓包报告仅存在于当前页面，不写入宿主机历史。

每个节点分别安装，登录哪个节点地址就管理哪个节点；不自动跟随迁移，不在节点间同步配置或流量。迁移前先在原节点解除并删除规则，迁移后重新添加。保留 MAC、bridge、其它 net 参数与其它网卡；发现 pending/锁/摘要冲突时拒绝写入。

## 检查与卸载

~~~bash
systemctl status pve-rate-manager --no-pager
pvesh get /rate-manager/status --output-format json
python3 /opt/pve-rate-manager/integration.py check
~~~

先暂停调度确认解除，或临时停服后执行：

~~~bash
systemctl stop pve-rate-manager
python3 /opt/pve-rate-manager/manager.py release-all
~~~

卸载使用 bash /root/install-pve-rate-manager-3.1.0.sh --uninstall。先停服并解除规则，再撤销自己的 PVE 标记和文件；解除失败时中止，不继续删除。规则、统计、抓包数据和备份保留，重新安装可继续使用。不要用旧完整 PVE 文件覆盖更新后的文件。

## 验证边界

见 VALIDATION.md。本地 Python 测试、原生入口契约和独立浏览器测试不代表真实 PVE 的 tc/IFB、TAP 抓包、PVE 登录/API/CSRF、服务升级与回退已实测。请先在测试 VM/网卡上验收；新安装没有规则，不会自动接管 VM 100。

MIT License。
