此插件为了解决pve虚拟机特别针对cdn平台带宽限速，解决流量超发的问题
后续还会增加流量自适应模块以此来适配cnd虚拟机的上传流量捕捉，以提升平台收益
下载后
复制一下shell运行
(
set -e
cd "$(mktemp -d /root/pve-rate-v2.XXXXXX)"

curl -fL --retry 3 -o pve-rate-manager-2.0.0.tar.gz \
  https://raw.githubusercontent.com/xuwenju2019-ui/pve-rate-manager/main/pve-rate-manager-2.0.0.tar.gz

echo 'bf9a860cd19551656095a249bf3842d2e1c9a090e5f1d35217ac0927a1d0294e  pve-rate-manager-2.0.0.tar.gz' \
  | sha256sum -c -

tar -xzf pve-rate-manager-2.0.0.tar.gz
cd pve-rate-manager
bash install.sh
)
