# 3x-ui 安装、Reality 搭建及 SSL 证书申请 | V2RayN 保姆级指南

本文档将详细介绍如何在 VPS 上安装 3x-ui 面板，配置 Reality 协议，并申请 SSL 证书。

## 目录

- [前置准备](#前置准备)
- [3x-ui 介绍](#3x-ui介绍)
- [第一步：更新系统](#第一步更新系统)
- [第二步：开启 BBR 加速](#第二步开启-bbr-加速)
- [第三步：安装 3x-ui 面板](#第三步安装-3x-ui-面板)
- [第四步：搭建 Reality 协议](#第四步搭建-reality-协议配置节点)
- [第五步：V2RayN 客户端连接](#第五步v2rayn-客户端连接)
- [第六步：申请 SSL 证书（可选）](#第六步申请-ssl-证书-acmesh)
- [常见问题排查](#常见问题排查)

---

## 前置准备

1.  **一台 VPS 服务器**：[Vultr](https://www.vultr.com/)，[HostDare](https://hostdare.com/)，[搬瓦工](https://bandwagonhost.com/)，[DMIT](https://www.dmit.io/)，[CloudCone](https://cloudcone.com/)(我使用的是这个)  自己去了解，然后选择一个厂商购买vps，好好了解，有坑。
2.  **一个域名**：你需要拥有一个域名，并将其 A 记录解析到你的 VPS IP 地址。（这里我个人使用 没申请域名）
3.  **SSH 连接工具**：如  Xshell, 或 Termius。

### 3x-ui 的特色

- **系统状态监控** - 实时查看服务器状态
- **多用户多协议** - 支持 VMess、VLESS、Trojan、Shadowsocks、Dokodemo-door、Socks、HTTP、WireGuard
- **XTLS 协议支持** - 包括 RPRX-Direct、Vision、REALITY
- **流量管理** - 流量统计、流量限制、超时时间限制
- **自定义配置** - 可自定义 Xray 配置模板
- **HTTPS 面板** - 支持自建域名 + SSL 证书访问
- **一键证书** - 支持一键式 SSL 证书申请和自动续费
- **数据导入导出** - 支持从面板导出/导入数据库
- **深色/浅色主题** - 界面美观，动画流畅

---

## 3x-ui介绍

**项目地址**：[https://github.com/MHSanaei/3x-ui](https://github.com/MHSanaei/3x-ui)

3x-ui 是基于 x-ui 的后续魔改版本，持续更新内核并支持新协议和多用户管理。其亮点在于可以直接在面板中修改 Xray 核心配置，同时在界面美观度和动画流畅度上远超其他同类面板。



---

## 第一步：更新系统

在开始之前，建议先更新系统组件以确保安全和兼容性。

```bash
yum update -y                # CentOS
# apt update && apt upgrade -y   # Debian/Ubuntu
```

安装必要的工具（curl, wget 等）：

```bash
yum install curl wget tar -y
```

---

## 第二步：开启 BBR 加速

BBR（Bottleneck Bandwidth and Round-trip propagation time）是 Google 开发的 TCP 拥塞控制算法，可以显著提升网络连接稳定性和速度。

### 1. 检查内核版本

BBR 支持 Linux 内核版本 4.9 及以上。查看当前内核版本：

```bash
uname -r
```

### 2. 启用 BBR

在终端中执行以下命令来启用 BBR：

```bash
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 3. 验证 BBR 是否成功启用

执行以下命令，确认 BBR 已被启用：

```bash
sysctl net.ipv4.tcp_available_congestion_control
```

输出结果应包含 `bbr`，表示 BBR 已成功启用。可以再运行以下命令进一步确认：

```bash
lsmod | grep bbr
```

如果输出包含 `tcp_bbr` 模块，则说明 BBR 已经在系统中启用。

> **提示**：启用 BBR 后，3x-ui 项目的网络连接稳定性和速度通常会得到明显提升。

---

## 第三步：安装 3x-ui 面板

这里使用官方或广泛认可的安装脚本（以 GitHub 上流行的版本为例）。

1.  **运行安装脚本**：

    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
    ```


2.  **配置面板**：
    *   脚本运行过程中，会询问你是否安装（输入 `y`）。
    *   设置 **用户名**（默认 admin）。
    *   设置 **密码**（默认 admin）。
    *   设置 **端口**（默认 2053，建议修改为其他非常用端口，如 10000-60000 之间）。
![安装配置](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/anzhuang.png)

3.  **访问面板**：
    *   安装完成后，在浏览器输入：`http://你的IP:端口`
    *   使用刚才设置的用户名和密码登录。
 ![登录面板](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/3xuilogin.png)
---

输入用户名和密码后，你应该就能看到 3X-UI 的管理界面了。

> **安全提示**：登录后你可能会看到一个 TLS 安全警告。如果你想解决这个问题，可以在 SSH 命令行输入 `x-ui` 命令，通过菜单中的 "SSL Certificate Management" 选项申请证书（需要域名）。

---

## 第四步：搭建 Reality 协议（配置节点）

安装完 3x-ui 后，接下来就是搭建节点。我们以 VLESS + Reality 协议为例，配置一个可用的节点。

### 1. 进入入站列表

登录到 3x-ui 面板后，找到左侧菜单栏的 **入站列表**，点击进入，然后点击页面上的 **添加入站** 按钮。

![入站列表](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/3xuidashboard.png)

### 2. 配置 VLESS 节点

点击后会弹出一个配置表单，这就是搭建节点的核心配置界面。别看选项很多，其实只需要关注几个重要的地方：

![配置表单](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/peizhi.png)

**基础配置**：
- **备注 (Remark)**：随意填写，例如 `Reality-VLESS`
- **协议 (Protocol)**：选择 `vless`
- **监听 IP (Listen IP)**：留空默认
- **端口 (Port)**：输入 `443`（Reality 通常使用 443 端口效果最佳），或选择不常用端口如 `30000`
- **传输协议 (Transmission)**：选择 `tcp`

### 3. 配置 Reality 安全选项

![Reality配置](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/peizhi2.png)

往下滚动找到 **安全** 选项，选择 `reality`。这是目前比较推荐的加密方式。

- **公钥/私钥**：点击旁边的 **"Get New Cert"** 按钮，系统会自动生成一对密钥
- **目标网站 (Dest)**：填写你要伪装的网站，必须是支持 TLS 1.3 和 H2 的国外网站
  - 常见推荐：`www.yahoo.com:443`、`www.microsoft.com:443`、`gateway.icloud.com:443`
- **Server Names**：填写与 Dest 对应的域名，例如 `www.yahoo.com`
- **uTLS**：选择 `chrome` 或 `firefox`

> **提示**：Target 和 SNI 可以保持默认。如果节点连接不上，可以尝试修改这两个选项。

### 4. 配置 Flow 选项

![Flow配置](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/peizhi3.png)

回到表单上方，找到 **客户端** 这一栏并展开。

- **Flow**：从下拉菜单中选择 `xtls-rprx-vision` 或 `xtls-rprx-vision-udp443`（Vision 流控能大幅提升抗封锁能力）
- **用户 (Client)**：点击 `+` 号添加用户，确保 ID（UUID）已生成

![配置完成](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/peizhi4.png)

### 5. 保存并导出节点

填写完所有配置后，点击表单底部的 **保存** 或 **添加** 按钮。

保存成功后，在入站列表中找到刚才创建的节点，点击右侧的操作菜单（三个点图标），选择 **导出链接**。

![导出链接](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/daochu.png)

---

## 第五步：V2RayN 客户端连接

### 客户端下载地址

| 平台 | 客户端 | 下载地址 |
|------|--------|----------|
| Windows | v2rayN | [GitHub](https://github.com/2dust/v2rayN) |
| Android | v2rayNG | [GitHub](https://github.com/2dust/v2rayNG) |
| iOS | Shadowrocket | [App Store](https://apps.apple.com/app/shadowrocket/id932747118) |
| iOS | Clash | 需要外区 Apple ID |

### 1. 下载 V2RayN

访问 V2RayN 的 [GitHub Releases 页面](https://github.com/2dust/v2rayN/releases)，下载最新版本。

对于 Windows 用户，推荐下载 `v2rayN-windows-64-SelfContained.zip` 版本。

### 2. 解压安装

1. 下载完成后，找到下载的 zip 压缩包
2. 右键点击压缩包，选择 **"解压到当前文件夹"** 或 **"提取全部"**
3. 选择一个合适的位置解压文件，例如 `D:\Programs\v2rayN` 或 `C:\v2rayN`
4. 解压完成后，进入解压出来的文件夹

### 3. 创建桌面快捷方式

为了方便日后使用，建议创建桌面快捷方式：

- 右键点击 `v2rayN.exe` 文件
- 在弹出菜单中选择 **"发送到"** > **"桌面快捷方式"**

### 4. 以管理员身份运行

为了确保 V2RayN 能够正常修改系统代理设置，建议以管理员身份运行：

- 右键点击桌面上的 V2RayN 快捷方式
- 在弹出菜单中选择 **"以管理员身份运行"**

### 5. 添加订阅/导入节点

1. 打开 V2RayN 主界面
2. 在主界面顶部菜单栏中，点击 **"订阅分组"** → **"订阅分组设置"**

![订阅设置](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/lianjie1.png)

3. 在弹出的窗口中，点击 **"添加"** 按钮新增一条订阅
4. 在 **"别名"** 栏输入一个容易识别的名称
5. 在 **"可选地址(url)"** 栏粘贴从 3x-ui 面板导出的订阅链接
6. 点击 **"确定"** 保存设置

![添加订阅](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/lianjie2.png)

### 6. 更新订阅

添加订阅后，需要更新订阅以获取最新的服务器节点信息：

1. 返回 V2RayN 主界面
2. 点击顶部菜单 **"订阅分组"** → **"更新全部订阅(不通过代理)"**
3. 等待几秒钟，V2RayN 将自动获取并添加所有可用的服务器节点

### 7. 选择并连接节点

1. 在 V2RayN 主界面的节点列表中，找到您想使用的节点
2. 右键点击该节点，从弹出菜单中选择 **"设为活动服务器"**
3. 此时，系统托盘区的 V2RayN 图标会变成绿色，表示连接成功

![连接成功](https://raw.githubusercontent.com/DeepVeri/mydoc/main/images/lianjie3.png)


## 第六步：申请 SSL 证书（可选）

虽然 Reality 协议本身是伪装成目标网站（偷证书），不需要你在本地有证书文件，但如果你想给面板加 HTTPS，或者搭建其他需要证书的协议（如 TCP+TLS），则需要申请证书。

这里使用 `acme.sh` 申请 Let's Encrypt 证书。

1.  **安装 acme.sh**：

    ```bash
    curl https://get.acme.sh | sh
    ```

    安装完成后，重新加载 bash 配置文件或重新登录 SSH：

    ```bash
    source ~/.bashrc
    ```

2.  **注册账号**（替换为你的邮箱）：

    ```bash
    ~/.acme.sh/acme.sh --register-account -m my@example.com
    ```

3.  **开放 80 端口**：
    确保 VPS 的 80 端口没有被占用（如果你安装了 Nginx/Apache，请先停止它们）。

4.  **申请证书**（将 `yourdomain.com` 替换为你的域名）：

    ```bash
    ~/.acme.sh/acme.sh --issue -d yourdomain.com --standalone
    ```

5.  **安装/复制证书**：
    不要直接使用 `~/.acme.sh/` 目录下的文件，而是将其安装到指定位置。

    ```bash
    # 创建存放目录
    mkdir -p /root/cert

    # 安装证书到目录
    ~/.acme.sh/acme.sh --install-cert -d yourdomain.com \
    --key-file       /root/cert/private.key  \
    --fullchain-file /root/cert/cert.crt
    ```

    *   **证书路径**: `/root/cert/cert.crt`
    *   **密钥路径**: `/root/cert/private.key`

---

## 常见问题排查

*   **连不上？**
    *   检查 VPS 防火墙是否放行了 443 端口和面板端口。
    *   检查 `Dest` 网站是否在国内能正常访问（不要选被墙的网站）。
    *   检查客户端内核是否过旧，Reality 需要较新的 Xray 内核。
*   **端口被占用？**
    *   如果在配置 443 端口时提示被占用，请检查是否安装了 Nginx 或其他 Web 服务占用了 443。

---

> 📝 **文档生成时间**: 2025-11-30  
> 💡 **提示**: 如有问题，可在 [ deepveir blog GitHub Issues](https://github.com/DeepVeri/blog) 提问
