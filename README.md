# 复盘：Android/TUN 客户端「一连代理就超时」，根因是订阅规则缺了节点 IP 直连

> **脱敏声明**：本文不含任何真实 IP、UUID、密钥、订阅令牌、凭据或面板口令，
> 地址一律用占位符表示。可安全提交到公开仓库或 Gist。
>
> 关键词：`mihomo` / `Clash.Meta` / `TUN` / `VpnService` / `IP-CIDR,DIRECT` / 代理自环

## TL;DR

同一份订阅、同一个节点，iPhone 正常，Android 全超时；服务器、防火墙、账号、
第二跳全部健康。根因不是被封、不是协议被识别、不是 SIM 卡，而是：

**订阅的 `rules` 里没有把节点自身 IP 设为 `DIRECT`。** Android 客户端开启 TUN 后，
「客户端 → 节点 IP」这条出站连接被 TUN 捕获，规则 `MATCH → 代理` 又把它送回代理，
形成自环，SYN 永远发不出去，表现为所有连接超时。

一行修法：

```yaml
rules:
  # 节点自身 IP 必须直连，且要放在 GEOIP / MATCH 之前
  - IP-CIDR,<INGRESS_IP>/32,DIRECT
  - IP-CIDR,<EGRESS_IP>/32,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,<PROXY_GROUP>
```

## 一、现象

- 手机 A（iOS，客户端 X）用蜂窝流量：代理正常。
- 手机 B（Android，客户端 Y）用同一运营商蜂窝流量、同一份订阅：所有连接超时。
- A 连 B 的热点：超时。B 连 A 的热点：正常。
- 换 SIM 卡、开关飞行模式、关 5G、换第三方代理，均无效。
- 服务器 SSH 正常，端口正常监听，日志无拒绝记录，账号未停用、未到期、未限额。
- 电脑连 B 的热点、跑一个**不开 TUN 的独立核心**：TCP、TLS 握手、完整代理数据流全部正常。

最后一条是最关键的对照：同一条上行链路，换个客户端/配置就通了。

## 二、走过的弯路（比根因更值得记）

| 阶段 | 当时的判断 | 为什么错 |
|---|---|---|
| 1 | 客户端半挂 / 配置坏了 | 服务器日志无任何异常，同一份配置在 iOS 上正常 |
| 2 | 服务器按来源 IP 封禁 | 服务器上根本没有 fail2ban，iptables 无 DROP，账号全启用 |
| 3 | SIM / APN / 5G SA 核心网差异 / IPv6 单栈 | 换卡无效；Android 状态页显示正常 IPv4+IPv6 双栈 |
| 4 | 运营商国际出口间歇路由黑洞（**确实存在**） | 它能解释「突然又好了」，但解释不了同一时刻不同客户端结果不同 |

**最大的方法论错误**：把不同时间点依次做的 2×2 交叉实验，当成了「同时刻」的对照。
间歇性故障窗口只有 1–8 分钟且会自愈，A 通 / B 不通很可能只是**错峰**。
凡是「换一个变量就好了」的结论，必须先确认两次测量发生在同一分钟内。

最终定案是两个因素叠加：

1. 运营商国际出口的短时路由黑洞（解释自愈、解释换卡/飞行模式无效）；
2. 客户端与配置版本差异（解释 iOS 通、Android 不通，以及同链路电脑通）。

## 三、根因机制

mihomo 系客户端在 TUN 模式下接管全部出站流量，再按 `rules` 决定走代理还是直连。
若规则里没有针对节点 IP 的例外：

```
App → 连接 <INGRESS_IP>:443
        ↓ 被 TUN 捕获
     规则匹配 MATCH → <PROXY_GROUP>
        ↓ 出站又被送回代理
     再次「连接 <INGRESS_IP>:443」→ 被 TUN 捕获 → 死循环
```

于是没有任何 SYN 到达服务器，服务端日志自然干净——这恰恰是「问题在客户端」的信号，
而不是「网络被切断」。

### 为什么 iOS 客户端不受影响

iOS 的 Network Extension 由系统在隧道之外处理代理自身的连接，
不会把「连节点」这件事再抓回隧道；Android 的 `VpnService` + TUN 则需要规则显式排除。
同理，**纯 HTTP/SOCKS 代理（不开 TUN）天然不受影响**——这也解释了为什么备用通道一直正常，
以及为什么这条线索直到用户说出「两台手机用的代理软件不一样」才浮出水面。

### 为什么「前两天还好好的」

旧订阅一直缺这条规则，只是之前没撞上「Android + TUN + 这份订阅」的组合。
修复发生在备份快照生成之后，于是快照把有缺陷的版本完整冻结了下来。

## 四、修复与防复发

### 1. 模板不要写死 IP

面板生成订阅时通常会渲染模板。把直连规则写成占位符，换入口地域时自动适配：

```yaml
rules:
  - IP-CIDR,{server}/32,DIRECT
  - IP-CIDR,<EGRESS_IP>/32,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,<PROXY_GROUP>
```

`{server}` 与 `proxies[].server` 共用同一个渲染变量，迁移到新中转机时无需再改规则。
**已渲染的成品订阅**则写字面 IP，与线上一致即可。

### 2. 最容易漏的一处：脚本内嵌的模板常量

很多面板把订阅模板**硬编码在脚本常量里**，磁盘上的模板文件只作为兜底。
改了 `.yaml` 却没改脚本常量，一旦模板文件缺失、脚本自动重建，旧配置立刻复活。
本次一共修了：模板文件 ×1、面板脚本常量 ×2 份副本、成品订阅 ×14 份。

可以先渲染一次做验证：

```python
SUB_TEMPLATE.format(name="test", server="203.0.113.7",
                    uuid="x", pubkey="y", sid="z")
# 应输出 IP-CIDR,203.0.113.7/32,DIRECT
```

### 3. 离线备份快照有四层，必须一起改

迁移包往往不止一个 tar：

```
外层 bundle.tar.gz
├── 已解开的目录树 tree/
├── 内层服务器快照 hk-bundle.tar.gz / us-bundle.tar.gz   ← 恢复时真正用的是这份
├── MANIFEST.sha256
└── README / 迁移文档
```

只改 `tree/` 不改内层 bundle，恢复出来仍是旧配置。
本次用 `tarfile` 精确替换成员内容，**保持成员名、顺序、数量、权限与元数据不变**，
只更新被修改成员的 mtime。

### 4. 一个 Windows 上的坑

用文本模式重写 `MANIFEST.sha256` 会被转成 CRLF，`sha256sum -c` 会把 `\r`
当成文件名的一部分，导致 142 条全部报 `No such file or directory`。
**校验清单必须用二进制方式写回 LF**。

### 5. 验收

```bash
cd <snapshot-dir>
sha256sum -c MANIFEST.sha256          # 必须 142/142 OK
tar -xzOf hk-bundle.tar.gz etc/bistro/sub-template.yaml | grep IP-CIDR
tar -xzOf us-bundle.tar.gz var/www/bistro/<name>.yaml | grep IP-CIDR
```

再从外层包全新解包到临时目录，重新跑一次 `sha256sum -c` —— 两层都要全绿。

## 五、通用检查清单

遇到「部分设备一连代理就超时」时，按顺序排除：

1. 服务器侧：防火墙 / fail2ban / 账号状态 / 端口监听 / 日志来源 IP 分布（只读）。
2. **同一分钟**做对照实验；不同时刻的结果不能推因果。
3. 记录每端**客户端名称与版本**、订阅最后更新时间、TUN 开关、当前选中的节点/代理组。
   「节点选择」被选成 `DIRECT` 也会表现为外网全超时。
4. 检查订阅 `rules` 最前面是否有节点 IP 的 `IP-CIDR,...,DIRECT`。
5. 用「不开 TUN 的纯 SOCKS/HTTP 核心」跑一次完整数据流，可一次性切开客户端与网络两类原因。
6. 修完后，把模板、脚本常量、成品订阅、内层 bundle、MANIFEST、外层包全部对齐。

## 六、绝对不能进 Git 的东西

复盘文档本身是安全的，但**工作区/迁移包不是**。以下类别必须排除或永不提交：

- SSH 私钥与公钥、任何 `*.pem` / `secrets.env` / `admin.txt` / `reality.txt`
- 客户端订阅 yaml（含 UUID、Reality 公钥、short-id、节点 IP）
- 订阅令牌、订阅 URL、面板登录口令、Basic Auth / Session key
- `*.db`（面板与流量库）、服务器完整快照 `*-bundle.tar.gz`、外层迁移包
- 任何文件名带「凭据 / 请勿分享」的文档
- 真实公网 IP、运营商出口 IP、CGNAT 地址

建议仓库里只放：脱敏复盘、通用配置模板（占位符）、通用脚本（不含常量）、
`.gitignore`（默认忽略 `backups/`、`*.db`、`*.pem`、`*credential*`、含密钥的 yaml）。

---

*本文由一次真实排障整理，所有标识符已替换为占位符。*
