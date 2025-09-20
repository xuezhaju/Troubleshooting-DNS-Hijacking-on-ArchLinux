# 解决 Arch Linux 上因 ISP DNS 污染导致的 GitHub 访问失败问题

## 问题描述

在 Arch Linux 系统上，Firefox 浏览器无法访问 `github.com`，错误信息为：
> **Unable to connect**  
> Firefox can't establish a connection to the server at github.com.

通过终端测试，发现 `curl` 和 `dig` 命令同样失败，并且显示 `github.com` 被异常解析到了 `127.0.0.1`（本地回环地址）。

## 环境信息

- **操作系统**: Arch Linux
- **网络接口**: `enp3s0` (有线连接)
- **网络管理器**: NetworkManager
- **DNS 解析器**: systemd-resolved (初始状态)

## 诊断过程与发现

### 1. 初步排查

首先排除了浏览器扩展、系统代理和防火墙问题：
- Firefox 安全模式下问题依旧。
- 系统防火墙 (`firewalld`) 未启用。
- 无全局代理环境变量 (`env | grep -i proxy` 无输出)。

### 2. 关键发现：DNS 解析异常

使用 `dig` 命令进行深度诊断，发现了问题的核心：

```bash
dig github.com

; <<>> DiG 9.20.12 <<>> github.com
;; QUESTION SECTION:
;github.com.                    IN      A

;; ANSWER SECTION:
github.com.             600     IN      A       127.0.0.1  # <-- 异常解析！

;; Query time: 6 msec
;; SERVER: 211.140.13.188#53(211.140.13.188) (UDP) # <-- 问题源头！


诊断结论：系统使用的 DNS 服务器 211.140.13.188（中国移动 ISP 服务器）对 github.com 的查询返回了伪造的 127.0.0.1，这是一种典型的 DNS 污染/劫持。
3. 尝试的解决方案（失败）

在定位根本原因前，按常见解决方案尝试均告失败：

    ✅ 检查 /etc/hosts：文件干净，无异常条目。

    ❌ 配置 systemd-resolved：修改 /etc/systemd/resolved.conf 无效。

    ❌ 使用 resolvectl 命令：虽显示设置成功 (Link 2 (enp3s0): 1.1.1.1 8.8.8.8)，但实际查询仍被劫持。

    ❌ 通过 nmtui/nmcli 配置 NetworkManager：DNS 设置被更高优先级的机制覆盖。

根本原因

ISP DNS 污染：网络运营商（本例中为中国移动）的 DNS 服务器 (211.140.13.188) 对特定域名（如 github.com）进行了投毒，返回错误的 127.0.0.1 作为解析结果，导致任何连接尝试都指向本机而失败。

systemd-resolved 的顽固性：即使使用 resolvectl 正确配置了备用 DNS，systemd-resolved 服务仍优先使用从 DHCP 获取的 ISP DNS 服务器，导致自定义设置不生效。
最终解决方案

通过完全禁用系统默认的 DNS 解析器，并手动指定可靠的公共 DNS 服务器来解决。
步骤摘要

    禁用并屏蔽 systemd-resolved：
    bash

sudo systemctl stop systemd-resolved.service
sudo systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved.service # 防止被其他服务重新启动

创建静态 DNS 配置：
bash

sudo rm -f /etc/resolv.conf # 删除旧的符号链接或文件
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf # 使用 Cloudflare DNS
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf # 添加 Google DNS 作为备用

锁定配置文件防止覆盖：
bash

sudo chattr +i /etc/resolv.conf # 设置文件为不可变

重启网络服务：
bash

sudo systemctl restart NetworkManager

验证结果

解决方案生效后的 dig 命令输出：
bash

; <<>> DiG 9.20.12 <<>> github.com
;; ANSWER SECTION:
github.com.             58      IN      A       20.26.156.215 # <- 正确解析到真实IP

;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP) # <- 正在使用干净的公共DNS

随后，curl 和 Firefox 均可正常访问 GitHub。
结论与建议

    核心教训：在某些网络环境下，ISP 提供的 DNS 服务可能不可靠。始终建议将公共 DNS（如 1.1.1.1, 8.8.8.8）作为首选。

    解决方案优先级：当遇到类似问题时，应首先使用 dig 或 nslookup 命令检查 DNS 解析结果，这是定位 DNS 污染问题最快的方法。

    本方案的普适性：此方法不仅适用于解决 GitHub 访问问题，也适用于任何因 DNS 污染导致的网站无法访问的情况。

    注意事项：使用 chattr +i 锁定文件后，若需再次修改 DNS，需先执行 sudo chattr -i /etc/resolv.conf 解除锁定。

关联资料

    Cloudflare Public DNS

    Google Public DNS

    Arch Wiki: Domain name resolution