# ESXi 安装 OpenWrt x86_64 记录

## 环境

- ESXi 8.0
- OpenWrt x86_64
- 镜像：`openwrt-25.12.5-x86-64-generic-ext4-combined.img.gz`
- 用途：OpenWrt 旁路由、Mihomo 代理

---

## 1. 下载并解压 OpenWrt 镜像

OpenWrt 25.12.5 x86/64 官方镜像目录：

<https://downloads.openwrt.org/releases/25.12.5/targets/x86/64/>

本文使用的 ext4 combined 镜像直链：

<https://downloads.openwrt.org/releases/25.12.5/targets/x86/64/openwrt-25.12.5-x86-64-generic-ext4-combined.img.gz>

下载文件：

```text
openwrt-25.12.5-x86-64-generic-ext4-combined.img.gz
```

如果虚拟机明确使用 EFI 固件，也可以下载 EFI 版本：

<https://downloads.openwrt.org/releases/25.12.5/targets/x86/64/openwrt-25.12.5-x86-64-generic-ext4-combined-efi.img.gz>

解压：

```bash
gunzip openwrt-25.12.5-x86-64-generic-ext4-combined.img.gz
```

解压后将镜像重命名为 `openwrt.img`。这是 raw 磁盘镜像，ESXi 不能直接使用，需要先转换为 VMDK。

---

## 2. 将 IMG 转换为 VMDK

以管理员身份打开 PowerShell，使用 WinGet 安装 Windows 版 QEMU：

```powershell
winget install --exact --id SoftwareFreedomConservancy.QEMU --accept-package-agreements --accept-source-agreements
```

安装完成后重新打开 PowerShell，确认 `qemu-img` 可用：

```powershell
qemu-img --version
```

如果系统提示找不到 `qemu-img`，可直接使用默认安装路径检查：

```powershell
& "C:\Program Files\qemu\qemu-img.exe" --version
```

### 转换为 ESXi 兼容的 VMDK

在 `openwrt.img` 所在目录打开 PowerShell，执行以下单行命令：

```powershell
qemu-img convert -f raw -O vmdk -o subformat=monolithicFlat openwrt.img openwrt.vmdk
```

如果 `qemu-img` 未加入 PATH，则执行：

```powershell
& "C:\Program Files\qemu\qemu-img.exe" convert -f raw -O vmdk -o subformat=monolithicFlat openwrt.img openwrt.vmdk
```

会生成：

```text
openwrt.vmdk
openwrt-flat.vmdk
```

上传到 ESXi 时，两个文件必须一起上传。

---

## 3. 创建 ESXi 虚拟机

推荐配置：

| 项目 | 配置 |
| --- | --- |
| 客户机操作系统 | Linux |
| 固件 | EFI |
| CPU | 1～2 核 |
| 内存 | 512 MB 或以上 |
| 磁盘 | 使用已有 VMDK |
| 磁盘控制器 | SATA |

OpenWrt 镜像使用 EFI 启动。

---

## 4. 扩容磁盘

OpenWrt 默认镜像较小，根分区通常只有约 100 MB。

### ESXi 扩大虚拟磁盘

关闭虚拟机，在 ESXi 中进入：

```text
编辑设置 → 硬盘 → 增加容量
```

例如将磁盘从约 120 MB 扩大到 20 GB。

### 使用 Ubuntu Live ISO 扩展分区

启动 Ubuntu Live ISO，查看磁盘：

```bash
lsblk
```

安装扩容工具：

```bash
sudo apt update
sudo apt install cloud-guest-utils
```

假设 OpenWrt 根分区是 `/dev/sda2`：

```bash
sudo growpart /dev/sda 2
sudo e2fsck -f /dev/sda2
sudo resize2fs /dev/sda2
```

其中，`e2fsck -f` 用于在扩展 ext4 文件系统前强制检查并修复文件系统。执行时应确保 `/dev/sda2` 没有挂载；如果已挂载，先执行：

```bash
sudo umount /dev/sda2
```

用以下命令检查：

```bash
lsblk
df -h
```

注意：`df -h` 只显示已挂载的文件系统；若看不到磁盘或分区，应使用 `lsblk`。

---

## 5. 配置 OpenWrt 网络

旁路由配置示例：

| 项目 | 值 |
| --- | --- |
| IP | `192.168.20.250` |
| 子网掩码 | `255.255.255.0` |
| 网关 | `192.168.20.1` |
| DNS | `223.5.5.5`、`223.6.6.6` |

使用 UCI 配置：

```bash
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.20.250'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.20.1'
uci -q delete network.lan.dns
uci add_list network.lan.dns='223.5.5.5'
uci add_list network.lan.dns='223.6.6.6'
uci commit network
/etc/init.d/network restart
```

查看配置：

```bash
uci show network.lan
```

---

## 6. 配置 Mihomo TUN

```yaml
tun:
  enable: true
  stack: system
  auto-route: true
  auto-redirect: true
```

### TUN 启动失败

错误：

```text
Start TUN listening error:
configure tun interface:
no such file or directory
```

原因通常是系统缺少 TUN 内核模块或 `/dev/net/tun` 设备。

检查：

```bash
ls -l /dev/net/tun
lsmod | grep tun
```

OpenWrt 25.12 及使用 `apk` 的版本：

```bash
apk update
apk add kmod-tun
modprobe tun
```

使用 `opkg` 的旧版本：

```bash
opkg update
opkg install kmod-tun
modprobe tun
```

再次确认：

```bash
ls -l /dev/net/tun
```

设备存在后重启 Mihomo。

---

## 7. Mihomo 配置文件与自动启动

源文件位于：

```text
E:\projects\wall\config.yaml
E:\projects\wall\openwrt\mihomo.init
```

注意：目录中实际文件名是 `config.yaml`，不是 `confirm.yaml`；启动脚本也会读取 `/etc/mihomo/config.yaml`。

### `config.yaml`

复制到 OpenWrt：

```text
/etc/mihomo/config.yaml
```

配置内容：

```yaml
mixed-port: 7890
mode: rule
#allow-lan: true

external-controller: 0.0.0.0:7788
secret: "510510510"
external-ui: /etc/mihomo/ui

tun:
  enable: true
  stack: system
  auto-route: true
  auto-redirect: true
  auto-detect-interface: true
  #dns-hijack:
  #  - any:53
  #  - tcp://any:53

dns:
  enable: true
  nameserver:
    - 223.6.6.6
    - 223.5.5.5
  #listen: 0.0.0.0:1053
  #enhanced-mode: fake-ip
  #fake-ip-range: 198.18.0.1/16

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - "BAN-US"
      - "DMIT-US"

rule-providers:
  local:
    type: http
    behavior: classical
    format: yaml
    url: "https://raw.githubusercontent.com/tongwoo/wall/main/rules/local.yaml"
    path: ./providers/local.yaml
    interval: 3600
    proxy: PROXY

  develop:
    type: http
    behavior: classical
    format: yaml
    url: "https://raw.githubusercontent.com/tongwoo/wall/main/rules/develop.yaml"
    path: ./providers/develop.yaml
    interval: 3600
    proxy: PROXY

  other:
    type: http
    behavior: classical
    format: yaml
    url: "https://raw.githubusercontent.com/tongwoo/wall/main/rules/other.yaml"
    path: ./providers/other.yaml
    interval: 3600
    proxy: PROXY

rules:
  - AND,((NETWORK,UDP),(DST-PORT,443)),REJECT
  - DOMAIN-SUFFIX,local,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - RULE-SET,local,DIRECT
  - RULE-SET,develop,PROXY
  - RULE-SET,other,PROXY
  - GEOIP,CN,DIRECT
  #- MATCH,DIRECT
  - MATCH,PROXY
```

### `mihomo.init`

复制到 OpenWrt：

```text
/etc/init.d/mihomo
```

脚本内容：

```sh
#!/bin/sh /etc/rc.common

USE_PROCD=1
START=99
STOP=10

PROG=/usr/bin/mihomo
CONF_DIR=/etc/mihomo
CONF_FILE="${CONF_DIR}/config.yaml"

start_service() {
	if [ ! -x "$PROG" ]; then
		logger -t mihomo "executable not found: $PROG"
		return 1
	fi

	if [ ! -f "$CONF_FILE" ]; then
		logger -t mihomo "configuration not found: $CONF_FILE"
		return 1
	fi

	procd_open_instance
	procd_set_param command "$PROG" -d "$CONF_DIR"
	procd_set_param file "$CONF_FILE"
	procd_set_param respawn 3600 5 5
	procd_set_param term_timeout 10
	procd_set_param reload_signal HUP
	procd_set_param stdout 1
	procd_set_param stderr 1
	procd_close_instance
}
```

### 设置自动启动

确认文件位置：

```text
/usr/bin/mihomo
/etc/mihomo/config.yaml
/etc/init.d/mihomo
```

设置权限、启用并启动服务：

```bash
chmod +x /usr/bin/mihomo
chmod +x /etc/init.d/mihomo
/etc/init.d/mihomo enable
/etc/init.d/mihomo start
```

查看运行状态和日志：

```bash
/etc/init.d/mihomo status
logread -e mihomo
```

修改配置后重启：

```bash
/etc/init.d/mihomo restart
```

---

## 8. 网络排查

```bash
ip addr
ip route
ping 192.168.20.1
ping 223.5.5.5
nslookup openwrt.org
```

| 现象 | 常见原因 |
| --- | --- |
| 无法 ping 网关 | 虚拟网卡或二层网络问题 |
| 能 ping 网关，不能 ping 公网 IP | 网关或路由问题 |
| 能 ping 公网 IP，但域名解析失败 | DNS 配置问题 |

---

## 常见问题总结

### VMDK“磁盘类型 7”不受支持

原因：`qemu-img` 默认生成了 `monolithicSparse`。

解决：使用 `-o subformat=monolithicFlat` 重新转换，并同时上传描述文件与 `-flat.vmdk` 数据文件。

### `df -h` 看不到磁盘

原因：`df` 只显示已挂载的文件系统。

解决：使用 `lsblk` 查看所有磁盘和分区。

### OpenWrt 根分区容量很小

原因：官方镜像采用最小容量分区。

解决流程：

```text
ESXi 扩大虚拟磁盘
       ↓
growpart 扩展分区
       ↓
e2fsck 检查 ext4 文件系统
       ↓
resize2fs 扩展 ext4 文件系统
```

### Mihomo TUN 无法启动

原因：缺少 `kmod-tun` 或 `/dev/net/tun`。

解决：安装 `kmod-tun`、加载 `tun` 模块，再重启 Mihomo。

---

## 最终架构

```text
ESXi
└── OpenWrt x86_64
    ├── br-lan
    ├── Mihomo TUN
    └── 旁路由
        ├── 主路由
        └── 局域网设备
```
