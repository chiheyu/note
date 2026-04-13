## 一、文档说明
### 1.1 配置背景
- 问题：Hyper-V环境下 GNS3 VM 默认使用自动分配 IP，重启或断网后 IP 漂移，导致 GNS3 GUI 需频繁修改服务器配置。
- 目标：为 GNS3 VM 配置永久固定 IP，彻底解决 IP 漂移问题，同时为后续公网连接预留完整基础。

### 1.2 环境信息
| 项目 | 详情 |
|------|------|
| 虚拟化平台 | Hyper-V（Windows 10/11 专业版/企业版） |
| GNS3 VM版本 | 3.0.6（基于 Ubuntu 24.04 Noble） |
| 虚拟交换机类型 | 内部虚拟交换机（名称：`GNS3-Static`） |
| 主机虚拟网卡IP | `172.30.184.1/24`（网关/外网留空） |
| GNS3 VM固定IP | `172.30.184.10/24`（网关指向主机虚拟网卡） |
| 网卡名称（GNS3 VM） | `eth0` |
| 认证信息 | SSH：`gns3/gns3`；GNS3 Web UI：`admin/admin` |

---

## 二、完整配置步骤
### 2.1 Hyper-V虚拟交换机配置（创建固定网段内网）
1. 打开 **Hyper-V管理器** → 右侧“虚拟交换机管理器”。
2. 选择“内部” → “创建虚拟交换机”。
3. 名称填写：`GNS3-Static`，点击“确定”完成创建。

> 注意：必须选择“内部虚拟交换机”，专用交换机仅虚拟机互通、主机无法访问；外部交换机用于桥接物理网络（公网方案见后文）。

### 2.2 主机虚拟网卡静态IP配置（固定网段）
1. 打开 Windows“设置 → 网络和 Internet → 高级网络设置”。
2. 找到 `vEthernet (GNS3-Static)` 虚拟网卡，点击“编辑 IP 地址”。
3. 选择“手动”，开启 IPv4，按以下配置填写：

| 配置项 | 填写内容 | 注意事项 |
|--------|----------|----------|
| IP地址 | `172.30.184.1` | 作为 GNS3 VM 的网关 IP |
| 子网前缀长度 | `24` | 对应子网掩码 `255.255.255.0` |
| 默认网关 | **留空** | 禁止填写，否则导致主机上网异常 |
| 首选DNS服务器 | `8.8.8.8`（可选） | 仅用于 VM 上网时域名解析，内网互通可留空 |
| 备用DNS服务器 | `114.114.114.114`（可选） | 同上 |

4. 点击“保存”完成配置，此时虚拟交换机网段永久固定为 `172.30.184.x/24`。

### 2.3 GNS3 VM网络适配器绑定
1. Hyper-V 管理器中右键 `GNS3 VM` → “设置”。
2. 选择“网络适配器”，虚拟交换机选择 `GNS3-Static`。
3. 勾选“启用此网络适配器”，点击“确定”，重启 GNS3 VM 生效。

### 2.4 GNS3 VM静态IP配置（Netplan）
#### 2.4.1 登录GNS3 VM控制台
- 用户名：`gns3`，密码：`gns3`
- 切换 root 权限：`sudo -i`（密码仍为 `gns3`）

#### 2.4.2 编辑Netplan配置文件
1. 查看 Netplan 配置文件路径：`ls /etc/netplan/`（默认文件为 `00-installer-config.yaml`）。
2. 编辑配置文件：`nano /etc/netplan/00-installer-config.yaml`
3. 本次配置场景为文件为空，直接写入以下配置，严格遵循 2 空格缩进，禁止使用 Tab：

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      dhcp6: no
      addresses:
        - 172.30.184.10/24
      routes:
        - to: default
          via: 172.30.184.1
      nameservers:
        addresses:
          - 8.8.8.8
```

保存退出：`Ctrl+O` → 回车 → `Ctrl+X`

#### 2.4.3 应用配置与权限修复
应用网络配置：

```bash
netplan apply
```

若出现 `Permissions for /etc/netplan/xxx.yaml are too open` 安全警告，执行权限修复命令：

```bash
chmod 600 /etc/netplan/00-installer-config.yaml
```

再次执行 `netplan apply`，警告消失，配置永久生效。

### 2.5 GNS3 GUI 永久绑定固定 IP
1. 打开 GNS3 GUI → 顶部菜单 `Edit → Preferences`。
2. 左侧选择 `Server`，取消勾选 `Enable Local Server`。
3. 填写远程服务器信息：

| 配置项 | 填写内容 |
|--------|----------|
| Host | `172.30.184.10` |
| Port | `80` |
| Username | `admin` |
| Password | `admin` |

4. 点击 `Test`，提示 `Connection successful` 后，点击 `Apply → OK` 保存。

配置完成后，无论重启主机、虚拟机或断网，GNS3 VM IP 都会保持固定，无需重复修改。

## 三、配置验证与问题排查
### 3.1 配置验证步骤
GNS3 VM 端验证：

```bash
# 验证 IP 生效
ip a show eth0
# 验证与主机互通
ping 172.30.184.1 -c 4
```

主机端验证：

```cmd
# 验证与 VM 互通
ping 172.30.184.10
# SSH 连接验证
ssh gns3@172.30.184.10
```

GNS3 GUI 验证：
- 打开 GNS3 项目，确认拓扑可正常加载，设备可正常启动。

### 3.2 常见问题排查
| 问题现象 | 排查与解决 |
|----------|------------|
| GNS3 VM 无 IP 分配（仅 IPv6 地址） | 1. 确认 Netplan 配置缩进正确；2. 确认虚拟交换机为内部类型；3. 重新执行 `netplan apply` |
| SSH 无法连接 GNS3 VM | 1. 确认主机与 VM 同网段；2. 关闭 VM 防火墙：`sudo ufw disable`；3. 检查 SSH 服务状态：`systemctl status ssh` |
| 主机虚拟网卡显示“无网络访问权限” | 正常现象，内部虚拟交换机为私有内网，无外网出口，不影响内网互通 |
| Netplan 权限警告 | 执行 `chmod 600 /etc/netplan/00-installer-config.yaml` 修复权限 |

## 四、公网连接方案与建议
### 4.1 方案一：NAT 转发（推荐，家用 / 实验场景）
#### 4.1.1 方案原理
通过 Windows 主机的 NAT 转发，让 GNS3 VM 通过主机物理网卡访问公网，无需修改物理网络，安全性高，完全兼容本次固定 IP 配置。

#### 4.1.2 配置步骤
1. 以管理员身份打开 PowerShell。
2. 查看主机物理上网网卡名称（如“以太网”“WLAN”）：

```powershell
Get-NetAdapter | Where-Object {$_.Status -eq "Up"}
```

3. 创建 NAT 转发规则：

```powershell
New-NetNat -Name GNS3NAT -InternalIPInterfaceAddressPrefix 172.30.184.0/24
```

4. 验证 NAT 创建：`Get-NetNat`
5. 在 GNS3 VM 内执行 `ping 8.8.8.8`，可通即说明上网正常。

#### 4.1.3 优缺点
| 优点 | 缺点 |
|------|------|
| 无需修改物理网络，安全性高 | 依赖主机网络，主机断网则 VM 断网 |
| 配置简单，一次配置永久生效 | 仅适合家用 / 实验场景，不适合生产环境 |
| 完全兼容本次固定 IP 配置，无需修改 VM 网络 | 无法为 VM 分配独立公网 IP |

### 4.2 方案二：桥接外部虚拟交换机（直接连物理网络）
#### 4.2.1 方案原理
将 GNS3 VM 桥接到主机物理网卡，直接获取物理网络 IP，可直接访问公网，适合需要 VM 独立公网 IP 的场景。

#### 4.2.2 配置步骤
1. 在 Hyper-V 中创建“外部虚拟交换机”，绑定主机物理上网网卡。
2. 将 GNS3 VM 网络适配器绑定到该外部交换机。
3. 为 GNS3 VM 配置静态 IP（与物理网络同网段，网关为物理路由器 IP）。
4. 验证上网：`ping 8.8.8.8`

#### 4.2.3 优缺点
| 优点 | 缺点 |
|------|------|
| VM 直接访问公网，可分配独立公网 IP | 需修改物理网络，存在安全风险 |
| 不依赖主机网络 | 需物理路由器支持，IP 可能漂移（需 DHCP 保留） |
| 适合生产 / 远程访问场景 | 需重新配置 VM 网络，与本次内部交换机配置不兼容 |

### 4.3 公网连接安全建议
- 防火墙配置：GNS3 VM 内执行 `sudo ufw enable`，仅开放必要端口（如 `22/80`），禁止全端口开放。
- 端口映射：若需远程访问 GNS3，仅映射必要端口，禁止全端口映射。
- 弱密码整改：修改默认密码（`gns3/admin`），使用强密码。
- 定期更新：定期更新 GNS3 VM 系统与 GNS3 服务，修复安全漏洞。
- 内网隔离：保持 GNS3 实验网段与物理内网隔离，避免实验流量影响物理网络。

## 五、关键避坑清单
- 缩进要求：Netplan YAML 配置必须使用 2 空格缩进，禁止使用 Tab，否则配置失效。
- 网关要求：主机虚拟网卡禁止填写默认网关，否则导致主机上网异常。
- 交换机类型：必须使用“内部虚拟交换机”，专用交换机主机无法访问 VM。
- 权限要求：Netplan 配置文件权限必须为 `600`，否则出现安全警告。
- IP 冲突：确保 GNS3 VM IP 与主机虚拟网卡 IP 同网段且无冲突。
- 服务状态：GNS3 服务异常时，执行 `sudo systemctl restart gns3` 重启服务。

## 六、附录：常用命令速查
### 6.1 GNS3 VM 端常用命令
```bash
# 切换 root 权限
sudo -i
# 查看网卡信息
ip a
# 查看默认路由
ip route show default
# 编辑 Netplan 配置
nano /etc/netplan/00-installer-config.yaml
# 应用网络配置
netplan apply
# 修复 Netplan 权限
chmod 600 /etc/netplan/00-installer-config.yaml
# 关闭防火墙
sudo ufw disable
# 重启 GNS3 服务
sudo systemctl restart gns3
```

### 6.2 主机端常用命令
```powershell
# 查看虚拟网卡信息
Get-NetAdapter
# 创建 NAT 转发
New-NetNat -Name GNS3NAT -InternalIPInterfaceAddressPrefix 172.30.184.0/24
# 查看 NAT 状态
Get-NetNat
# 测试连通性
ping 172.30.184.10
# SSH 连接
ssh gns3@172.30.184.10
```
