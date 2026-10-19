# AWS Lightsail 搭建 Hysteria 2 节点教程

从创建 Debian 服务器、设置 UDP 端口，到配置证书和客户端，本文带你完成一条可以逐步检查的搭建流程。示例使用 Amazon Lightsail，区域为美国俄勒冈州 `us-west-2`，操作系统为 Debian 12，节点端口为 `26818/UDP`。

本文按新建的 Debian 12 实例编写。如果服务器上已有其他业务，操作前先备份配置，并妥善保管密码、私钥及节点分享链接。

## 一、开始前准备什么

你需要一个能够正常使用 Lightsail 的 AWS 账号、一台电脑，以及支持 Hysteria 2 的客户端。没有域名也可以跟着本文操作：我们使用自签名证书，并通过证书指纹确认服务器身份。

先分清三个容易混淆的概念：

- **VPS** 是运行服务的云服务器，本文选择 AWS Lightsail。
- **Hysteria 2** 是部署在服务器和客户端上的代理程序，本文使用普通 QUIC 模式，节点连接需要 UDP。
- **本机代理端口** 是电脑上的入口。本文的 `127.0.0.1:5678` 只在本机使用，不需要在 AWS 防火墙开放。

Lightsail 采用套餐计费。截图中的方案是每月 7 美元，包含 1 GB 内存、2 vCPU、40 GB SSD 和 2 TB 套餐流量；价格、地区及流量计费条件以创建页面和 [AWS 官方价表](https://aws.amazon.com/lightsail/pricing/) 为准。新账号能否使用额度、额度期限和覆盖范围，需要查看自己的账号页面，不能据此承诺永久免费。

## 二、创建 Lightsail 实例

### 1. 进入 Lightsail 并选择区域

登录 [AWS 管理控制台](https://console.aws.amazon.com/)，搜索并进入 **Lightsail**，选择 **Create instance**。本文以俄勒冈州为例，你也可以根据自己的访问线路和资源情况选择其他区域；区域本身不保证某一种访问速度。

创建页面的区域、镜像、套餐和实例名称需在付款创建前核对清楚。首次使用可参照 [AWS Lightsail 入门文档](https://docs.aws.amazon.com/lightsail/latest/userguide/getting-started-with-amazon-lightsail.html)。

### 2. 选择 Linux 和 Debian 12

在 **Select a platform** 中选择 **Linux operating system**，在 **Select a blueprint** 中选择 **Debian 12**。截图显示的是 Debian 12.15，后续维护版本号可能变化，本文命令按 Debian 12 编写。

配图说明：文中五张配图依据实际操作截图，经 AI 整理为示意图，关键选项和数值已与原素材核对；配图不作为实时控制台状态或节点连通性的证明。

![图1 选择 Linux operating system 和 Debian 12](assets/01-debian12.png)

*图1：Linux 平台与 Debian 12 选择示意，依据提供的操作截图整理。*

### 3. 选择通用套餐和双栈网络

在 **Select a plan type** 中选择 **General purpose**，在网络类型中选择 **Dual-stack**。这样可获得公网 IPv4 和 IPv6；本文先用 IPv4 完成连接，减少客户端网络差异带来的干扰。

![图2 选择 General purpose 和 Dual-stack](assets/02-dual-stack.png)

*图2：通用型套餐与双栈网络示意。页面中的免费额度提示需结合自己的账号资格阅读。*

截图选择的是每月 7 美元的 1 GB 内存套餐，适合作为本文的配置示例。不要因为教程截图选中了某档套餐，就忽略自己账号上显示的实际费用。

![图3 每月7美元套餐示例](assets/03-plan.png)

*图3：每月 7 美元套餐选择示意，创建时请再次核对价格。*

### 4. 填写名称并创建

实例名称可以填写 `test2`，也可以自定义容易识别的名字。自动快照属于可选项目，启用前了解存储费用。核对区域、系统、套餐、数量及收费信息后，再由你决定是否点击 **Create instance**。

![图4 填写实例名称并核对创建选项](assets/04-instance-name.png)

*图4：实例名称示例。创建操作会产生云资源及相应费用。*

等待实例进入 **Running**。接着在 Lightsail 的网络功能中创建静态公网 IPv4，并绑定到同区域的这台实例，客户端后续使用这个地址。普通动态公网 IPv4 在实例停止再启动后可能改变，具体操作见 [创建并绑定静态 IP](https://docs.aws.amazon.com/lightsail/latest/userguide/lightsail-create-static-ip.html)。

## 三、先配置防火墙，再连接服务器

打开实例详情，进入 **Networking**。本文只使用一个节点端口，不设置端口跳跃。

在 **IPv4 Firewall** 中保留 SSH 所需的 TCP 22，并添加一条自定义规则：协议选 **UDP**，端口填 `26818`。如果所有客户端都来自固定公网 IP，可以将节点规则的来源限制到这些地址；如果需要在不同网络使用，再根据实际需要设置来源范围。

SSH 使用本机终端时，应尽量限制到自己的公网 IP；使用 Lightsail 的浏览器 SSH 时，需要保留其支持的浏览器连接来源设置，按 [AWS 防火墙说明](https://docs.aws.amazon.com/lightsail/latest/userguide/understanding-firewall-and-port-mappings-in-amazon-lightsail.html) 操作，避免把自己锁在服务器外。

**只开放 TCP 26818 无法代替 UDP 26818。** IPv6 的防火墙与 IPv4 独立；计划用 IPv6 连接时，还需要在 **IPv6 Firewall** 配置对应 UDP 规则。本文不要求开放 All TCP、All UDP，也不要求开放 TCP 80 或 TCP 443。

完成后进入 **Connect → Connect using SSH**，打开浏览器终端。先确认当前系统：

```bash
cat /etc/os-release
whoami
```

本文后续命令在服务器 SSH 终端执行，输入以下命令进入 root 会话：

```bash
sudo -i
```

## 四、使用官方脚本安装 Hysteria 2

先安装所需工具，再下载官方安装脚本：

```bash
apt-get update
apt-get install -y curl ca-certificates openssl nano less
curl -fsSL https://get.hy2.sh/ -o /root/hysteria-install.sh
```

可以先用 `less /root/hysteria-install.sh` 查看脚本，按 `q` 退出。确认下载成功且来源正确后，执行：

```bash
bash /root/hysteria-install.sh
hysteria version
```

官方安装器会安装程序、systemd 服务和示例配置，**仍需手动完成服务端配置**，操作范围见 [官方安装脚本说明](https://v2.hysteria.network/docs/getting-started/Server-Installation-Script/)。请不要在下载失败后继续执行旧文件，也不要为了下载成功而关闭 HTTPS 证书验证。

![图5 安装器输出成功提示](assets/05-installer-output.png)

*图5：安装阶段的输出示意。下方采用官方手动配置流程；图中提示不代表服务已经启动或客户端已经连通。*

## 五、配置证书、密码和监听端口

### 1. 生成自签名证书和独立密码

在服务器 root 终端执行：

```bash
hysteria cert --host aws-hy2.local \
  --cert /etc/hysteria/server.crt \
  --key /etc/hysteria/server.key

chown root:hysteria /etc/hysteria/server.crt /etc/hysteria/server.key
chmod 644 /etc/hysteria/server.crt
chmod 640 /etc/hysteria/server.key
openssl rand -hex 24
```

`hysteria cert` 的参数可在 [Hysteria 官方证书命令源码](https://github.com/apernet/hysteria/blob/master/app/cmd/cert.go) 中核对。它会生成证书和私钥；最后一条 OpenSSL 命令生成随机密码，将这串密码安全保存，下一步与客户端都要使用同一串值。

`aws-hy2.local` 是本文设置的证书名称，稍后作为客户端 SNI；它不是 AWS 域名，也不需要将它解析到公网。私钥 `server.key` 留在服务器上，不要上传或发给他人。上述文件权限适用于官方安装器默认的 `hysteria` 服务用户。

### 2. 写入服务端配置

打开配置文件：

```bash
nano /etc/hysteria/config.yaml
```

将文件中的示例内容替换为以下配置，并把密码占位文字换成刚才生成的随机密码。YAML 使用空格缩进，不要使用 Tab：

```yaml
listen: :26818

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key

auth:
  type: password
  password: "替换为刚才生成的随机密码"
```

在 nano 中按 `Ctrl+O` 保存，回车确认文件名，再按 `Ctrl+X` 退出。此处端口必须和 Lightsail 防火墙规则一致；认证与 TLS 字段说明见 [服务端完整配置](https://v2.hysteria.network/docs/advanced/Full-Server-Config/)。

收紧配置文件权限后启动服务：

```bash
chown root:hysteria /etc/hysteria/config.yaml
chmod 640 /etc/hysteria/config.yaml
systemctl enable --now hysteria-server.service
```

这适用于首次启动；以后修改配置后，运行 `systemctl restart hysteria-server.service` 使新配置生效。

### 3. 检查服务与证书指纹

```bash
systemctl status hysteria-server.service --no-pager
ss -lunp | grep ':26818'
journalctl -u hysteria-server.service -n 50 --no-pager
openssl x509 -noout -fingerprint -sha256 -in /etc/hysteria/server.crt
```

依次检查：服务状态应为 `active (running)`；UDP 列表能找到 `26818`；近期日志没有证书、配置或认证相关启动错误。最后一条命令输出证书 SHA-256 指纹，复制等号后面的指纹值，供客户端固定验证服务器身份。

这些检查通过，表示服务器端准备就绪；最终能否使用，还要完成下一节的客户端握手和实际访问测试。

## 六、配置客户端

### 1. 先保存四项连接信息

准备自己的**静态公网 IPv4、端口 26818、随机密码、证书 SHA-256 指纹**。下方 `203.0.113.10` 是文档示例地址，必须替换；所有密码与指纹占位文字也必须替换。

为了避免不同图形客户端对证书参数的处理差异，先给出 [Hysteria 官方客户端](https://v2.hysteria.network/docs/getting-started/Installation/) 可直接识别的配置。按操作系统与 CPU 架构下载对应程序，将配置保存为本机的 `client.yaml`：

```yaml
server: "203.0.113.10:26818"
auth: "替换为服务端的随机密码"

tls:
  sni: aws-hy2.local
  insecure: true
  pinSHA256: "替换为服务器证书的SHA-256指纹"

socks5:
  listen: 127.0.0.1:5678
```

这里的 `insecure: true` 与正确的 `pinSHA256` **配套使用**，不能只保留关闭验证的选项。也可以改为显式信任自己的证书，具体方式见 [客户端 TLS 说明](https://v2.hysteria.network/docs/getting-started/Client/)。证书更换后，需要同步更新客户端指纹。

将下载的程序放在配置文件所在目录，按实际文件名运行。Linux 示例：

```bash
chmod +x ./hysteria-linux-amd64
./hysteria-linux-amd64 client -c client.yaml
```

Windows 在 PowerShell 中的示例：

```powershell
.\hysteria-windows-amd64.exe client -c client.yaml
```

上述文件名仅对应 amd64 下载包；其他平台使用对应的程序名。看到 `connected to server` 后，再做下文的访问测试。

### 2. 使用 v2rayN 导入分享链接

也可以将连接信息整理为分享链接，导入 v2rayN。模板如下，所有占位项都要替换成自己的值：

```text
hysteria2://YOUR_PASSWORD@203.0.113.10:26818/?sni=aws-hy2.local&insecure=1&pinSHA256=YOUR_CERT_SHA256#AWS-Hysteria2
```

官方 URI 中密码若包含特殊字符，需要做 URL 编码；本文由 `openssl rand -hex 24` 生成的十六进制密码没有这类特殊字符。链接字段见 [Hysteria 2 URI 规范](https://v2.hysteria.network/docs/developers/URI-Scheme/)。

从 [v2rayN 官方仓库](https://github.com/2dust/v2rayN) 获取客户端后，复制填好的链接，通过“从剪贴板导入批量 URL”或当前版本的对应入口导入。打开节点编辑页，逐项核对服务器地址、端口、密码、SNI 和证书验证信息，再设为活动节点。

**不要假定每个版本都会正确保留分享链接中的指纹。** v2rayN 的内核和证书设置曾有调整，维护者说明见 [证书验证参数说明](https://github.com/2dust/v2rayN/discussions/9460)。如果当前版本要求完整证书，可以通过可信 SSH 连接读取服务器的公有证书 `server.crt`，按该版本说明导入；不要复制私钥。仍无法配置时，先使用上面的官方客户端验证，不要靠关闭全部证书验证解决问题。

## 七、怎样确认节点真正可用

先确认客户端已经和服务器完成握手，再通过本机 SOCKS5 端口发起请求。使用前面官方客户端的 `5678` 配置时，在本机执行：

```bash
curl --fail --show-error --socks5-hostname 127.0.0.1:5678 https://example.com/
```

Windows PowerShell 请使用 `curl.exe`。如果使用 v2rayN，则将端口改成其设置里实际显示的 SOCKS 端口，不能直接假设也是 `5678`。

应能收到 Example Domain 页面。接着用同一代理访问自己日常使用的网站，检查客户端日志、实际请求及服务器流量情况。需要确认出口时，可访问自己信任的公网 IP 查询服务，核对出口是否为这台 AWS 实例的预期地址。

浏览器普通访问是否走代理，取决于系统代理、浏览器代理或客户端 TUN 设置。仅选中一个节点，或看到“已连接”的按钮，不代表所有应用已经经过它；上面的显式 SOCKS5 请求可以单独核验这条路径。

最终检查应同时满足：服务器服务在运行、UDP 端口监听正常、客户端握手成功、实际请求经过预期节点。普通网页的 TCP `curl -I` 或 TCP 端口检测不能代替 Hysteria 2 的 UDP 握手检查。

## 八、连接失败时按什么顺序排查

- **SSH 打不开：** 先确认实例为 Running、公网地址正确、TCP 22 与浏览器 SSH 来源设置正确，再检查自己使用的连接方式。
- **服务启动失败：** 查看 `journalctl`，检查 YAML 缩进、密码引号、证书路径以及 `hysteria` 用户能否读取配置和私钥。
- **客户端一直超时：** 核对静态公网 IP、实际 UDP 端口、Lightsail 对应 IP 协议族的防火墙，以及系统防火墙是否放行同一端口。本地网络若限制 UDP，也会影响连接。
- **证书验证失败：** 从可信 SSH 终端重新读取指纹，核对客户端参数；证书重新生成后旧指纹失效。
- **握手成功但网页打不开：** 检查本机代理端口、应用是否使用该代理、DNS 及客户端路由设置。
- **能用但不稳定：** 先看服务器 CPU、流量、日志和本地网络，不要先填入远高于实际线路的带宽数值。没有设置带宽参数也可以先测试基本连接。

若系统使用 UFW，先运行 `ufw status verbose` 查看现状；已经启用时，按既有策略添加 `26818/udp` 规则并保留 SSH。新装系统无需为了本教程先启用一套新防火墙，也不要清空现有 iptables/nftables 规则。安装报错应先看错误原因，不必直接重装系统或执行 DD。

## 九、后续维护和费用检查

保存自己的配置记录，定期检查服务日志和系统更新。自签名证书需要在到期前更新；重新生成证书后同步更换客户端指纹。如果以后改成域名与受信任证书，可以按 [官方服务端指南](https://v2.hysteria.network/docs/getting-started/Server/) 配置 ACME，届时再根据证书验证方式开放所需端口。

普通 Lightsail 实例停止后仍会计费；确定不再使用时，按控制台流程删除不需要的资源，并单独检查快照、磁盘及静态 IP。静态 IPv4 绑定实例时与空置时的计费不同，具体以 [AWS 计费 FAQ](https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-frequently-asked-questions-faq-billing-and-account-management.html) 为准。

搭建的终点是“服务运行、客户端握手、实际访问”三项都通过。把端口、证书和客户端配置对应起来，后续排查会清楚得多。
