# Linux 透明代理
## 什么是正向代理？
代理软件通常分为客户端（client）和服务端（server），server 运行在境外服务器（通常为 Linux 服务器），client 运行在本地主机（如 Windows、Linux、Android、iOS），client 与 server 之间通常使用 tcp 或 udp 协议进行数据通信。大多数 client 被实现为一个 http、socks5 代理服务器，一个软件如果想通过 client 进行科学上网，需要使用 http、socks5 协议与 client 进行数据交互，这是绝大多数人的使用方式。这种代理方式，我们称之为 **正向代理**。所谓正向代理就是，一个软件如果想要使用 client 的代理服务，需要经过特定的设置，否则不会经过 client 的代理。

## 什么是透明代理？
在正向代理中，一个软件如果想走 client 的代理服务，我们必须显式配置该软件，对该软件来说，有没有走代理是很明确的，大家都“心知肚明”。而透明代理则与正向代理相反，当我们设置好合适的防火墙规则（仅以 Linux 的 iptables 为例），我们将不再需要显式配置这些软件来让其经过代理或者不经过代理（直连），因为这些软件发出的流量会自动被 iptables 规则所处理，那些我们认为需要代理的流量，会被通过合适的方法发送到 client 进程，而那些我们不需要代理的流量，则直接放行（直连）。这个过程对于我们使用的软件来说是完全透明的，软件自身对其一无所知。这就叫做 **透明代理**。注意，所谓透明是对我们使用的软件透明，而非对 client、server 或目标网站透明，理解这一点非常重要。

## 透明代理如何工作？
在正向代理中，期望使用代理的软件会通过 http、socks5 协议与 client 进程进行交互，以此完成代理操作。而在透明代理中，我们的软件发出的流量是完全正常的流量，并没有像正向代理那样，使用 http、socks5 等专用协议，这些流量经过 iptables 规则的处理后，会被通过“合适的方法”发送给 client 进程（当然是指那些我们认为需要走代理的流量）。注意，此时 client 进程接收到不再是 http、socks5 协议数据，而是经过 iptables 处理的“透明代理数据”，“透明代理数据”从本质上来说与正常数据没有区别，只是多了一些“元数据”在里面，使得 client 进程可以通过 netfilter 或操作系统提供的 API 接口来获取这些元数据（元数据其实就是原始目的地址和原始目的端口）。那么这个“合适的方法”是什么？目前来说有两种：
- REDIRECT：只支持 TCP 协议的透明代理。
- TPROXY：支持 TCP 和 UDP 协议的透明代理。

因此，对于 TCP 透明代理，有两种实现方式，一种是 REDIRECT，一种是 TPROXY；而对于 UDP 透明代理，只能通过 TPROXY 方式来实现。为方便叙述，本文以 **纯 TPROXY 模式** 指代 TCP 和 UDP 都使用 TPROXY 来实现，以 **REDIRECT + TPROXY 模式** 指代 TCP 使用 REDIRECT 实现，而 UDP 使用 TPROXY 来实现，有时候简称 **REDIRECT 模式**，它们都是一个意思。

## 此脚本的作用及由来
通过上面的介绍，其实可以知道，在构建透明代理的过程中，需要的仅仅是 iptables、iproute2 以及 ss/ssr/v2ray 等支持透明代理的软件，那 ss-tproxy 脚本的作用是什么呢？如果你尝试搭建过透明代理，那么你就会体会到，这一过程其实并不容易，你需要设置许多繁琐的 iptables 规则，还要应对国内复杂的 DNS 环境，另外还要考虑 UDP 透明代理的支持，此外你通常还希望这个透明代理能实现分流操作，而不是一股脑的全走代理。于是就有了 ss-tproxy 脚本，该脚本的目的就是辅助大家快速地搭建一个透明代理环境，该透明代理支持 gfwlist、chnroute 等常见分流模式，以及一个无污染的 DNS 解析服务；除此之外，ss-tproxy 脚本不做任何其它事情；因此你仍然需要 iptables、iproute2 以及 ss/ssr/v2ray 等支持透明代理的软件，因为透明代理的底层服务是由它们共同运作的，理解这一点非常重要。

> 为什么叫做 `ss-tproxy`？因为该脚本最早的时候只支持 ss 的透明代理，当然现在它并不局限于特定的代理软件。

另外还有一点需要注意，透明代理使用的 client 与正向代理使用的 client 通常是不同的，因为正向代理的 client 是 http、socks5 服务器，而透明代理的 client 则是透明代理服务器，它们之间有本质上的区别。对于 ss，你需要使用 ss-libev 版本（ss-redir），ssr 则需要使用 ssr-libev 版本（ssr-redir），而对于 v2ray，配置好 `dokodemo-door` 入站协议即可。再次强调，透明代理只是 client 不同，并不关心你的 server 是什么版本，因此你的 vps 上，可以运行所有与之兼容的 server 版本，以 ss/ssr 为例，你可以使用 python 版的 ss、ssr，也可以使用 golang 版的 ss、ssr 等等，只要它们之间可以兼容。

如果没有条件使用 ss-libev、ssr-libev，或者只有 socks5 client，也可以使用 redsocks/redsocks2/ipt2socks 等工具将 iptables 透明代理数据转换为 socks5 协议数据，这样就可以与后端的 socks5 client 进行交互了。[redsocks](https://github.com/darkk/redsocks) 目前只支持 TCP 透明代理，[redsocks2](https://github.com/semigodking/redsocks) 支持 TCP、UDP 透明代理，[ipt2socks](https://github.com/zfl9/ipt2socks) 支持 TCP、UDP 透明代理。其中 ipt2socks 是我自己利用业余时间写的一个小工具，编译、配置、使用方法都比前两者简单，并且只有一个核心功能：`iptables-to-socks5`，没有附带任何不必要的功能（redsocks2 集成了很多我不想要的东西）。`iptables-to-socks5` 的具体配置方法见 FAQ。

ss-tproxy 可以运行在 Linux 软路由/网关、Linux 物理机、Linux 虚拟机等环境中，可以透明代理 ss-tproxy 主机本身以及所有网关指向 ss-tproxy 主机的其它主机的 TCP、UDP 流量。也就是说，你可以在任意一台 Linux 主机上部署 ss-tproxy 脚本，然后同一局域网内的其它主机可以随时将其网关及 DNS 指向 ss-tproxy 主机，这样它们的 TCP 和 UDP 流量就会自动走代理了。

## 脚本简介
- 添加 `global` 分流模式、`tcponly` 代理模式
- 支持 IPv4、IPv6 双栈透明代理（v4.0 优化版）
- 无需指定内网网段，利用 `addrtype` 模块进行匹配
- `v4.6.1+`版本起不再需要指定代理服务器的地址信息
- 使用 [chinadns-ng](https://github.com/zfl9/chinadns-ng) 替代原版 chinadns，修复若干问题
- 完美兼容"端口映射"，只代理"主动出站"的流量，规则更加细致化
- 支持配置要代理的黑名单端口，这样可以比较好的处理 BT/PT 流量
- 支持自定义 dnsmasq/chinadns 端口，支持加载外部 dnsmasq 配置
- ss-tproxy stop 后，支持重定向内网主机发出的 DNS 到本地直连 DNS
- 支持网络可用性检查，无需利用其它的 hook 来避免脚本自启失败问题
- 脚本逻辑优化及结构调整，尽量提高脚本的可移植性，去除非核心依赖

v4.0/v4.6 仍支持 `global`、`gfwlist`、`chnroute`、`chnlist` 4 种分流模式：
- `global` 分流模式：除保留地址外，其它所有流量都走代理出去，即全局模式。
- `gfwlist` 分流模式：`gfwlist.txt` 中的域名走代理，其余走直连，即黑名单模式。
- `chnroute` 分流模式：除了国内地址、保留地址之外，其余均走代理，即白名单模式。
- `chnlist` 分流模式：本质还是 `gfwlist` 模式，只是域名列表为国内域名，即回国模式。

> 有人可能会疑问，为什么使用 ss-tproxy 后，可以访问谷歌，但无法 ping 谷歌？<br>
> 因为 ping 走的是 ICMP 协议，几乎没有代理软件会处理 ICMP，所以 ICMP 走直连。

## 相关依赖
- `iptables`：用于配置 IPv4 透明代理规则，仅在启用 IPv4 透明代理时需要。
- `ip6tables`：用于配置 IPv6 透明代理规则，仅在启用 IPv6 透明代理时需要。
- `ipset`：用于存储 gfwlist/chnlist 的黑名单 IP、global/chnroute 的白名单 IP。
- `xt_TPROXY`：TPROXY 内核模块，在 redirect + tcponly 模式下，不需要此依赖。
- `ip`：用于配置策略路由(TPROXY)，在 redirect + tcponly 模式下，不需要此依赖。
- `dnsmasq`：DNS 服务，对于 gfwlist/chnlist 模式，该 dnsmasq 需支持 `--ipset` 选项。
- `chinadns-ng`：chnroute 模式的 DNS 服务，注意是 [chinadns-ng](https://github.com/zfl9/chinadns-ng)，不是原版 chinadns。
- `dns2tcp`：将 DNS 查询从 UDP 转为 TCP，仅在 tcponly 模式需要，注意是 [zfl9/dns2tcp](https://github.com/zfl9/dns2tcp)。

使用内置命令更新 gfwlist/chnlist/chnroute 列表时，会用到这些依赖：
- `curl`：用于更新 chnlist、gfwlist、chnroute 分流模式的相关列表。
- `base64`：用于更新 gfwlist 的域名列表，gfwlist.txt 是 `base64` 格式编码的。
- `perl`：用于更新 gfwlist 的域名列表，gfwlist.txt 是 `adblock plus` 规则，要进行转换。

如果某些模式你基本不用，那么对应的依赖就不用管。比如，你不打算使用 IPv6 透明代理，则无需关心 ip6tables，又比如你不打算使用 chnroute 模式，也无需关心 chinadns-ng。ss-tproxy 脚本在启动时会检查当前配置所需的依赖，只需要根据提示安装缺少的依赖即可。

[ss-tproxy 脚本相关依赖的安装方式参考](https://github.com/zfl9/ss-tproxy/wiki/Linux-%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86#%E5%AE%89%E8%A3%85%E4%BE%9D%E8%B5%96)

## 下载脚本
```bash
git clone https://github.com/zfl9/ss-tproxy
cd ss-tproxy
chmod +x ss-tproxy
```

## 安装脚本
> 请确保当前用户有权限读写以下目录，如没有，请先运行`sudo su`进入超级用户(root)。

- install命令可用
```bash
install ss-tproxy /usr/local/bin
install -d /etc/ss-tproxy
install -m 644 ss-tproxy.conf gfwlist* chnroute* ignlist* /etc/ss-tproxy
install -m 644 ss-tproxy.service /etc/systemd/system # 可选，安装 service 文件
```
- install命令不可用
```bash
cp -af ss-tproxy /usr/local/bin
mkdir -p /etc/ss-tproxy
cp -af ss-tproxy.conf gfwlist* chnroute* ignlist* /etc/ss-tproxy
cp -af ss-tproxy.service /etc/systemd/system # 可选，安装 service 文件
```

## 卸载脚本
```bash
ss-tproxy stop
ss-tproxy flush-postrule
ss-tproxy delete-gfwlist
rm -fr /usr/local/bin/ss-tproxy # 删除脚本
rm -fr /etc/ss-tproxy # 删除配置(做好备份)
rm -fr /etc/systemd/system/ss-tproxy.service # service文件
```

## 升级脚本
脚本目前没有自我更新能力，只能卸载后重新安装，后续也许会添加自我更新指令。

## 文件列表
- `ss-tproxy`：shell 脚本，欢迎各位大佬一起来改进这个脚本。
- `ss-tproxy.conf`：配置文件，本质是 shell 脚本，修改需重启生效。
- `ss-tproxy.service`：systemd 服务文件，用于 ss-tproxy 的开机自启。
- `chnroute.set`：存储大陆地址段的 ipset 文件（IPv4），不要手动修改。
- `chnroute6.set`：存储大陆地址段的 ipset 文件（IPv6），不要手动修改。
- `gfwlist.txt`：存储 gfwlist、chnlist 分流模式的黑名单域名，不要手动修改。
- `gfwlist.ext`：存储 gfwlist、chnlist 分流模式的扩展黑名单，可配置，重启生效。
- `ignlist.ext`：存储 global、chnroute 分流模式的扩展白名单，可配置，重启生效。

> ss-tproxy 只是一个 shell 脚本，并不是常驻后台的服务，因此所有的修改都需要 restart 来生效。

## 配置说明
> 注意，配置文件在 /etc/ss-tproxy/ 目录，不是 git clone 下来的目录！

配置项有点多，但通常只需修改 ss-tproxy.conf 前面的少数配置项（开头至`proxy`配置段）

<details><summary>注释</summary>
    
井号开头的行为注释行，配置文件本质上是一个 shell 脚本，对于同名变量或函数，后定义的会覆盖先定义的。
    
</details>

<details><summary>mode</summary>
    
分流模式，默认为 chnroute 模式，可根据需要修改为 global/gfwlist 模式。

如果需要使用 `chnlist` 回国模式，则 mode 依旧为 `gfwlist`，具体的：
- gfwlist 模式与 chnlist 模式共享 `gfwlist.txt`、`gfwlist.ext` 文件
- 首先执行 `ss-tproxy update-chnlist` 将 gfwlist.txt 替换为国内域名列表
- 手动编辑 gfwlist.ext 扩展黑名单，将其中的 Telegram IPv4/IPv6 地址段注释
- 手动修改 `dns_direct/dns_direct6` 配置项，改为本地 DNS（如 Google DNS）
- 手动修改 `dns_remote/dns_remote6` 配置项，改为大陆 DNS（如 114 DNS，走代理）

> 如果需要从 chnlist 回国模式切换为 gfwlist 国内模式，则进行相反的操作（update-gfwlist）
    
</details>

<details><summary>ipv4、ipv6</summary>
    
- 启用 IPv4/IPv6 透明代理，你需要确保本机代理进程能正确处理 IPv4/IPv6 相关数据包，脚本不检查它
- 启用 IPv6 透明代理应检查当前的 Linux 内核版本是否为 `v3.9.0+`，以及 ip6tables 的版本是否为 `v1.4.18+`
    
</details>

<details><summary>tproxy</summary>
    
true 为纯 TPROXY，false 为 REDIRECT/TPROXY 混合（具体解释前面有）：
- ss/ssr/trojan 目前是 REDIRECT/TPROXY 混合模式
- v2ray 经配置后可使用纯 TPROXY 模式（见下）
- ipt2socks 默认配置是纯 TPROXY 模式
- 其他代理软件请各位自己辨别测试

> ss-libev v3.3.5+ 已加入纯 tproxy 支持，在启动参数中增加`-T`或在json文件中添加`"tcp_tproxy": true`配置行，即可启用
    
</details>

<details><summary>tcponly</summary>

- true 表示仅代理 TCP 流量（需要依赖 dns2tcp）
- false 表示代理 TCP 和 UDP 流量（这是默认值）

</details>

<details><summary>selfonly</summary>
    
- true 表示仅代理 ss-tproxy 主机自身的流量
- false 表示代理 ss-tproxy 主机自身以及所有网关指向 ss-tproxy 主机的流量
    
</details>

<details><summary>proxy_procuser、proxy_procgroup</summary>
    
- **v4.6.1新增**：替代之前的`proxy_svraddr/proxy_svrport`配置，用来实现`本机代理进程`流量放行
- 现在只需要让`本机代理进程`以指定user/group身份运行，即可'放行'它们传出的流量（避免环路）
- `proxy_procuser`填写`本机代理进程`的user/uid，`proxy_procgroup`填写`本机代理进程`的group/gid
- 两者选其一(不建议都填)；此文档使用`proxy`用户(组)，见`proxy_startcmd/proxy_stopcmd`配置说明
    
</details>

<details><summary>proxy_svraddr4、proxy_svraddr6</summary>
    
**不建议使用此机制，请使用`proxy_procuser`/`proxy_procgroup`**：填写 VPS 服务器的外网 IPv4/IPv6 地址，IP 或域名都可以，填域名要注意，这个域名最好不要有多个 IP 地址与之对应，因为脚本内部只会获取其中某个 IP，这极有可能与本机代理进程解析出来的 IP 不一致，这可能会导致 iptables 规则死循环，应尽量避免这种情况，比如你可以将该域名与其中某个 IP 的映射关系写到 ss-tproxy 主机的 `/etc/hosts` 文件中，这样解析结果就是可预期的。允许填写多个 VPS 地址，用空格隔开，填写多个地址的目的是方便切换代理，比如我现在有两个 VPS，A、B，假设你先使用 A，因为某些因素，导致 A 的网络性能低下，那么你可能需要切换到 B，如果只填写了 A 的地址，就需要去修改 ss-tproxy.conf，将地址改为 B，修改启动与关闭命令，最后还得重启 ss-tproxy 脚本，很麻烦，更麻烦的是，如果现在 A 的网络又好了，那么你可能又想切换回 A，那么你又得重复上述步骤。但现在，你不需要这么做，你完全可以在 `proxy_svraddr` 中填写 A 和 B 的地址，假设你默认使用 A（`proxy_startcmd` 启动 A 代理进程），那么启动 ss-tproxy 后，使用的就是 A，此后如果想切换为 B，仅需停止 A 代理进程，再启动 B 代理进程（切回来的步骤则相反），该过程无需操作 ss-tproxy；这种配置下应注意 `proxy_stopcmd`，stopcmd 最好能停止 A 和 B 进程，不然切换进程后执行 ss-tproxy stop 可能不会正确停止相关的代理进程。另外，你只需填写实际会使用到的 VPS 地址，比如本机代理进程仅使用 IPv4 访问 VPS，则 `proxy_svraddr6` 可能是空的，反之，如果本机代理进程仅使用 IPv6 访问 VPS，则 `proxy_svraddr4` 可能是空的；这两个数组是否为空与 `ipv4`、`ipv6` 选项没有必然的联系，比如你可以启用 IPv4 和 IPv6 透明代理，但是本机代理进程仅使用 IPv4 访问 VPS，这是完全可以的，但不允许 `proxy_svraddr4` 与 `proxy_svraddr6` 都为空，你至少需要填写一个地址。
    
</details>

<details><summary>proxy_svrport</summary>

**不建议使用此机制，请使用`proxy_procuser`/`proxy_procgroup`**：填写 VPS 上代理服务器的外部监听端口，格式同 `ipts_proxy_dst_port`，填写不正确会导致 iptables 规则死循环。如果是 v2ray 动态端口，如端口号 1000 到 2000 都是代理监听端口，则填 `1000:2000`（含边界）。
    
</details>

<details><summary>proxy_tcpport、proxy_udpport</summary>
    
`本机代理进程`的 **透明代理** 监听端口，前者为 TCP 端口，后者为 UDP 端口，通常情况下这两端口是相同的。<br>
如果 UDP 隧道不稳定，或无法使用 UDP 代理，可使用 `tcponly` 模式，这种情况下，`proxy_udpport` 将被忽略。

> 此端口必须支持透明代理(REDIRECT/TPROXY)，请不要填写其他传入协议的端口，如`socks5`。
    
</details>

<details><summary>proxy_startcmd、proxy_stopcmd</summary>
    
前者是启动`本机代理进程`的 shell 命令，后者是关闭`本机代理进程`的 shell 命令<br>
这些命令应该能快速执行完毕，防止卡住脚本(长时间处于半启动或半关闭状态)<br>
> 具体命令例子，见 [代理软件配置](#代理软件配置)

</details>

<details><summary>dnsmasq_bind_port</summary>
    
dnsmasq 监听端口，默认 53，如果端口已被占用则修改为其它未占用的端口，如 `60053`。
    
</details>

<details><summary>dnsmasq_conf_dir、dnsmasq_conf_file</summary>
    
dnsmasq 外部配置文件/目录，被作为 dnsmasq 的 `conf-dir`、`conf-file` 选项的值。
    
</details>

<details><summary>dnsmasq_conf_string</summary>
    
shell 数组，每个元素都是一行独立的 dnsmasq 配置，多个用空白符隔开，如果元素内容包含空白符，请用引号包围。

</details>

<details><summary>chinadns_gfwlist_mode</summary>

是否启用 chinadns-ng 的 gfwlist 黑名单匹配模式。如果需要在 Android 上使用 Google Play，建议打开此选项，否则可能会遇到 `从服务器检索信息时出错，DF-DFERH-01` 错误，导致 Google Play 无法使用，究其原因是因为 `services.googleapis.cn` 这个域名的解析没有走代理导致的（连到了谷歌中国）。启用 gfwlist 匹配模式后就正常了，因为 `gfwlist.txt` 包含了 `services.googleapis.cn` 这个域名。当然也可以使用 dnsmasq 的 `--server` 选项来解决这个问题，只不过会稍微多几个步骤，这里就不详细介绍了。
    
 > 如非特殊情况，建议始终启用其模式，可以最大限度的提高 chinadns-ng 准确性，极大减少 dns 污染可能性
    
</details>

<details><summary>chinadns_privaddr4、chinadns_privaddr6</summary>

如果希望`chinadns-ng`接受**包含保留地址的解析记录**(如`192.168.1.1`)，请在此配置加入对应保留地址(段)，如`192.168.1.0/24`。前者为 IPv4 地址段数组、后者为 IPv6 地址段数组，多个用空格隔开，默认为空数组。

</details>

<details><summary>ipts_set_snat</summary>

是否设置 IPv4 的 MASQUERADE 规则（true设置，false不设置），通常 false 即可。有两种情况需要将其设置为 true：
- ss-tproxy 部署在出口路由位置且确实需要 MASQUERADE 规则（即至少两张网卡，一张连内网，一张连公网，需要源地址转换）
- 在设置为 false 的情况下，代理不正常，典型的如：白名单地址无法访问（如百度），黑名单地址正常访问，也需要将其改为 true

> 注意，MASQUERADE 规则在 ss-tproxy stop 仍然是有效的，如果你想清空这些残留规则，可以执行 `ss-tproxy flush-postrule` 命令

</details>

<details><summary>ipts_set_snat6</summary>

是否设置 IPv6 的 MASQUERADE 规则（true设置，false不设置），通常 false 即可。<br>
v4.6 版本的 IPv6 透明代理不再需要配置 ULA 私有地址，可直接使用 GUA 公网地址。
    
</details>

<details><summary>ipts_reddns_onstop</summary>

当 ss-tproxy stop 之后，是否将内网主机发往 ss-tproxy 主机的 DNS 请求重定向至本地直连 DNS（`dns_direct/dns_direct6`），为什么要这么做呢？因为其它内网主机的 DNS 是指向 ss-tproxy 主机的，但是现在我们已经关闭了 ss-tproxy（dnsmasq 关闭了），所以这些内网主机会无法解析 DNS 而无法上网；设置此选项后，这些 DNS 请求会被重定向给 114.114.114.114 等国内直连 DNS，这样它们就又可以正常上网了，在下次执行 ss-tproxy start 时，这些规则会被脚本自动删除，如果你需要手动删除这些规则，可以执行 `ss-tproxy flush-postrule` 命令。该选项的默认值为 true，如果 ss-tproxy 主机上有正常运行的 DNS 服务，那么这个选项应该设置为 false。
    
</details>

<details><summary>ipts_proxy_dst_port</summary>
    
要代理黑名单地址的哪些目的端口。所谓黑名单地址，对于 gfwlist/chnlist 模式来说，就是 gfwlist.txt/gfwlist.ext 里面的域名、IP、网段，对于 chnroute 模式来说，就是国外 IP 地址。默认值为 `1:65535`，因此只要我们访问黑名单地址，就会走代理，因为所有端口号都在其中。如果觉得端口范围太大，那么你可以修改这个选项的值，比如设置为 `1:1023,8080`，在这种配置下，只有当我们访问黑名单地址的 1 到 1023 和 8080 这些目的端口时才会走代理，访问黑名单地址的其它目的端口是不会走代理的，因此可以利用此选项来放行 BT、PT 流量，因为这些流量的目的端口通常都在 1024 以上。修改此选项需要足够小心，配置不当会导致某些常用软件无法正常走代理，因为它们使用的端口号可能不在你所指定的范围之内，因此指定为 `1:65535` 可能是最保险的一种做法。
    
</details>

<details><summary>opts_ss_netstat</summary>

告诉 ss-tproxy，使用 ss 还是 netstat 命令进行端口检测，目前检测`本机代理进程`是否正常运行的方式是直接检测其是否已监听对应的端口，虽然这种方式有时并不准确，但我现在并没有其它更好的便携方法来做这个事情。选项的默认值为 `auto`，表示自动模式，所谓自动模式就是，如果当前系统有 ss 命令则使用 ss 命令进行检测，如果没有 ss 命令但是有 netstat 命令则使用 netstat 命令进行检测，而 `ss` 选项值则是明确告诉 ss-tproxy 使用 `ss` 进行检测，同理，`netstat` 选项也是明确告诉 ss-tproxy 使用 `netstat` 进行端口检测。通常情况下保持 `auto` 即可。
    
</details>

<details><summary>opts_ping_cmd_to_use</summary>
    
告诉 ss-tproxy，使用何种 ping 命令（主要是 ping6 的问题）。默认为 `auto`，如果存在 `ping6` 且 `ping6` 并非软链接文件，则使用 `ping` 处理 ipv4 地址/域名，使用 `ping6` 来处理 ipv6 地址/域名，如果不存在 `ping6` 或 `ping6` 是 `ping` 的软链接文件，则使用 `ping -4` 处理 ipv4 地址/域名，使用 `ping -6` 处理 ipv6 地址/域名。如果选项值为 `standalone` 则明确使用 `ping/ping6` 方案，如果选项值为 `parameter` 则明确使用 `ping -4/-6` 方案。一般情况下，保持默认即可，除非遇到运行时错误等问题（如 `ping -4/-6` 选项不支持）。
    
</details>

<details><summary>opts_hostname_resolver</summary>

告诉 ss-tproxy，使用哪个工具来解析 `proxy_svraddr4/6` 中的域名(`proxy_procuser/group`模式下不需要)；默认为 `auto`，auto 模式的查找优先级为 `dig`、`getent`、`ping`，只要找到其中一个就停止搜寻；dig 需要安装 bind-utils/dnsutils 包，getent 大多数发行版都自带，ping 基本上是个系统都有；如果你的系统只有 ping 命令，且能明显感受到 1 秒左右的解析延迟，那么请安装 dig 实用工具（Debian/Ubuntu 系列基本上都有这个问题，busybox 版本的 ping 也一样）。
    
</details>

<details><summary>opts_overwrite_resolv</summary>

如果设置为 true，则表示直接使用 I/O 重定向方式修改 `/etc/resolv.conf` 文件，这个操作是不可逆的，但是可移植性好；如果设置为 false，则表示使用 `mount -o bind` 魔法来暂时性修改 `/etc/resolv.conf` 文件，当 ss-tproxy stop 之后，`/etc/resolv.conf` 会恢复为原来的文件，也就是说这个修改操作是可逆的，但是这个方式可能某些系统会不支持，默认为 `false`，如果遇到问题请修改为 `true`；此选项留空则不操作 `/etc/resolv.conf`。
    
</details>

<details><summary>opts_ip_for_check_net</summary>

指定一个允许 Ping 的 IP 地址（IPv4 或 IPv6 都行），用于检查外部网络的连通情况，如果此选项留空则表示跳过网络可用性检查（不建议）。默认为 `223.5.5.5`，注意这个 IP 地址应该为公网 IP，如果你填一个私有 IP，即使检测成功，也不能保证外网是可访问的，因为这仅代表我可以访问这个内网。根据实际网络环境进行更改，一般改为延迟较低且较稳定的一个 IP。
    
</details>

## 代理软件配置

<details><summary>ss-libev</summary>
    
`ss-redir`配置文件`/etc/ss.json`，例如
- `本地监听端口`请确保与`ss-tproxy.conf`文件中的`proxy_tcpport`、`proxy_udpport`配置项保持一致
- 若使用`proxy_svraddr/svrport`机制(不建议)，请确保`服务器地址/端口`与`proxy_svraddr/svrport`一致
- `本地监听地址`填`127.0.0.1`；对于`v4.6.1`以前的版本，填`0.0.0.0`，若只代理ss-tproxy自身，可填回环

```javascript
{
    "server": "服务器地址",
    "server_port": 服务器端口,
    "local_address": "本地监听地址",
    "local_port": 本地监听端口,
    "method": "加密方式",
    "password": "用户密码",
    "no_delay": true,
    "fast_open": true,
    "reuse_port": true
}
```

`ss-tproxy.conf`启动和停止命令，例如
```bash
#老版本(v4.6.0及以下)
#请确保proxy_svraddr/svrport与'服务器地址/端口'一致
#proxy_startcmd='(ss-redir -c /etc/ss.json -u -v </dev/null &>>/var/log/ss-redir.log &)' # -v 表示记录详细日志
proxy_startcmd='(ss-redir -c /etc/ss.json -u </dev/null &>>/var/log/ss-redir.log &)' # 这里就不记录详细日志了
proxy_stopcmd='kill -9 $(pidof ss-redir)'

#新版本(v4.6.1及以上)
#第一次运行时，请执行下面这两个操作
#1.创建proxy用户和组: useradd -Mr -d/tmp -s/bin/bash proxy
#2.授予透明代理相关权限: setcap cap_net_bind_service,cap_net_admin+ep /path/to/ss-redir
#>> 若setcap不可用，可使用suid权限位，此时需配置：proxy_procuser=''、proxy_procgroup='proxy'
#>> 将所有者(组)改为root，并授予suid权限：chown root:root /path/to/ss-redir && chmod 4755 /path/to/ss-redir
proxy_procuser='proxy'
#proxy_startcmd='su proxy -c"(ss-redir -c /etc/ss.json -u -v </dev/null &>>/tmp/ss-redir.log &)"' # -v 表示记录详细日志
proxy_startcmd='su proxy -c"(ss-redir -c /etc/ss.json -u </dev/null &>>/tmp/ss-redir.log &)"' # 这里就不记录详细日志了
proxy_stopcmd='kill -9 $(pidof ss-redir)'
```

</details>

<details><summary>ssr-libev</summary>

`ssr-redir`配置文件`/etc/ssr.json`，例如
> 基本同ss-libev，这里就不详细贴出了，随便一搜就有，注意事项也同ss-libev

`ss-tproxy.conf`启动和停止命令，例如
```bash
#老版本(v4.6.0及以下)
#请确保proxy_svraddr/svrport与'服务器地址/端口'一致
#proxy_startcmd='(ssr-redir -c /etc/ssr.json -u -v </dev/null &>>/var/log/ssr-redir.log &)'
proxy_startcmd='(ssr-redir -c /etc/ssr.json -u </dev/null &>>/var/log/ssr-redir.log &)'
proxy_stopcmd='kill -9 $(pidof ssr-redir)'

#新版本(v4.6.1及以上)
#第一次运行时，请执行下面这两个操作
#1.创建proxy用户和组: useradd -Mr -d/tmp -s/bin/bash proxy
#2.授予透明代理相关权限: setcap cap_net_bind_service,cap_net_admin+ep /path/to/ssr-redir
#>> 若setcap不可用，可使用suid权限位，此时需配置：proxy_procuser=''、proxy_procgroup='proxy'
#>> 将所有者(组)改为root，并授予suid权限：chown root:root /path/to/ssr-redir && chmod 4755 /path/to/ssr-redir
proxy_procuser='proxy'
#proxy_startcmd='su proxy -c"(ssr-redir -c /etc/ssr.json -u -v </dev/null &>>/tmp/ssr-redir.log &)"' # -v 表示记录详细日志
proxy_startcmd='su proxy -c"(ssr-redir -c /etc/ssr.json -u </dev/null &>>/tmp/ssr-redir.log &)"' # 这里就不记录详细日志了
proxy_stopcmd='kill -9 $(pidof ssr-redir)'
```

</details>

<details><summary>v2ray</summary>

v2ray 的透明代理配置比较简单，只需在原有客户端配置加上 `dokodemo-door` 入站协议，例如

> 由于 v2ray 配置复杂，在报告问题之前，请检查配置是否有问题，这里不解答任何 v2ray 配置问题<br>
> **原则上不建议在 v2ray 上配置任何分流或路由规则**，脚本会为你做这些事，否则出问题请自行解决<br>
> 据反馈，`dokodemo-door` 的 UDP 存在断流 bug，可尝试使用 `redsocks2/ipt2socks + socks5` 来缓解

```javascript
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "info" // 调试时请改为 debug
  },

  "inbounds": [
    {
      "protocol": "dokodemo-door",
      "listen": "0.0.0.0", // 如果只代理本机，可填写回环地址
      //"listen": "127.0.0.1", // v4.6.1+版本可填写回环地址
      "port": 60080, // 必须与proxy_tcpport/udpport保持一致
      "settings": {
        "network": "tcp,udp", // 注意这里是 tcp + udp
        "followRedirect": true
      },
      "streamSettings": {
        "sockopt": {
          //"tproxy": "tproxy" // tproxy + tproxy 模式 (纯tproxy)
          "tproxy": "redirect" // redirect + tproxy 模式 (redirect)
        }
      }
    }
  ],

  "outbounds": [
    {
      "protocol": "shadowsocks",
      "settings": {
        "servers": [
          {
            "address": "node.proxy.net", // 服务器地址 (如果是v4.6.1之前的版本，请与proxy_svraddr一致)
            "port": 12345,               // 服务器端口 (如果是v4.6.1之前的版本，请与proxy_svrport一致)
            "method": "aes-128-gcm",     // 加密方式
            "password": "password"       // 用户密码
          }
        ]
      }
    }
  ]
}
```

`ss-tproxy.conf`启动和停止命令，例如
```bash
#老版本(v4.6.0及以下)
#请确保proxy_svraddr/svrport与'服务器地址/端口'一致
proxy_startcmd='systemctl start v2ray'
proxy_stopcmd='systemctl stop v2ray'

#新版本(v4.6.1及以上)
#第一次运行时，请执行下面这两个操作
#1.创建proxy用户和组: useradd -Mr -d/tmp -s/bin/bash proxy
#2.授予透明代理相关权限: setcap cap_net_bind_service,cap_net_admin+ep /path/to/{v2ray,v2ctl}
#>> 若setcap不可用，可使用suid权限位，此时需配置：proxy_procuser=''、proxy_procgroup='proxy'
#>> 将所有者(组)改为root，并授予suid权限：chown root:root /path/to/{v2ray,v2ctl} && chmod 4755 /path/to/{v2ray,v2ctl}
proxy_procuser='proxy'
proxy_startcmd='su proxy -c"(v2ray -config /etc/v2ray.json </dev/null &>/dev/null &)"'
proxy_stopcmd='kill -9 $(pidof v2ray) $(pidof v2ctl)'
#当然也可以使用systemctl来封装上述startcmd/stopcmd，具体不再细说。
```

</details>

<details><summary>socks5</summary>

对于socks5代理（如：非libev版本的ss/ssr，或其他代理软件），可使用 [ipt2socks](https://github.com/zfl9/ipt2socks) 作为其前端，提供透明代理传入支持。以trojan为例（trojan支持nat传入，但只支持tcp，因此与ipt2socks配合用），配置例子来自trojan官方文档：

```javascript
{
    "run_type": "client",
    "local_addr": "127.0.0.1",
    "local_port": 1080,
    "remote_addr": "example.com", // 服务器地址 (如果是v4.6.1之前的版本，请与proxy_svraddr一致)
    "remote_port": 443, // 服务器端口 (如果是v4.6.1之前的版本，请与proxy_svrport一致)
    "password": [
        "password1" // 用户密码
    ],
    "log_level": 1,
    "ssl": {
        "verify": true,
        "verify_hostname": true,
        "cert": "",
        "cipher": "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:AES128-SHA:AES256-SHA:DES-CBC3-SHA",
        "cipher_tls13": "TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384",
        "sni": "",
        "alpn": [
            "h2",
            "http/1.1"
        ],
        "reuse_session": true,
        "session_ticket": false,
        "curves": ""
    },
    "tcp": {
        "no_delay": true,
        "keep_alive": true,
        "reuse_port": false,
        "fast_open": false,
        "fast_open_qlen": 20
    }
}
```

`ss-tproxy.conf`启动和停止命令，例如
```bash
#老版本(v4.6.0及以下)
#请确保proxy_svraddr/svrport与'服务器地址/端口'一致
tproxy='true' #ipt2socks默认为tproxy模式
proxy_startcmd='(trojan -c /etc/trojan.json </dev/null &>>/var/log/trojan.log & ipt2socks </dev/null &>>/var/log/ipt2socks.log &)'
proxy_stopcmd='kill -9 $(pidof trojan) $(pidof ipt2socks)'

#新版本(v4.6.1及以上)
#第一次运行时，请执行下面这两个操作
#1.创建proxy用户和组: useradd -Mr -d/tmp -s/bin/bash proxy
#2.授予透明代理相关权限: setcap cap_net_bind_service,cap_net_admin+ep /path/to/{trojan,ipt2socks}
#>> 若setcap不可用，可使用suid权限位，此时需配置：proxy_procuser=''、proxy_procgroup='proxy'
#>> 将所有者(组)改为root，并授予suid权限：chown root:root /path/to/{trojan,ipt2socks} && chmod 4755 /path/to/{trojan,ipt2socks}
tproxy='true' #ipt2socks默认为tproxy模式
proxy_procuser='proxy'
proxy_startcmd='su proxy -c"(trojan -c /etc/trojan.json </dev/null &>>/var/log/trojan.log & ipt2socks </dev/null &>>/var/log/ipt2socks.log &)"'
proxy_stopcmd='kill -9 $(pidof trojan) $(pidof ipt2socks)'
```

</details>

> 如果觉得配置和修改`proxy_startcmd`、`proxy_stopcmd`太麻烦（如经常切换节点），可参考：[切换代理小技巧](#切换代理小技巧)

## IPv6 透明代理的实施方式

ss-tproxy v4.0 版本需要利用 ULA 地址进行 IPv6 透明代理，而且还有许多要注意的事项，体验不是很好；但 v4.6 版本不需要任何额外的配置，如果想使用 IPv6 透明代理，直接启用 `ipv6` 选项即可，使用方法完全同 IPv4 透明代理。当然，v4.6 版本依旧可以使用 ULA 地址来进行 IPv6 透明代理（比如忍受不了 GUA 地址总是变化），使用 ULA 地址做透明代理时需要注意一点：将 ss-tproxy.conf 中的 `ipts_set_snat6` 选项设为 true，作用是防止 ULA 地址在公网上被路由。

## 非标准的 IPv4 内网地址段

标准内网地址段如：`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`，如果你将其它 IP 段作为内网使用（有人甚至将公网 IP 段作为内网使用），那么强烈建议你纠正这个错误，这不仅会导致透明代理出问题，也会隐藏其它 bug（很多软件设计者并没有考虑到你使用的是一个非标准内网地址段）。如果因为各种原因无法更改（比如公司内部），那么解决办法只有一个，编辑 ss-tproxy.conf，添加 `post_start()` 钩子函数，将当前使用的非标网段加入到 `privaddr` 这个 ipset 中。如下：

```bash
post_start() {
    if is_global_mode || is_chnroute_mode; then
        # 假设非标网段为 172.172.172.0/24
        ipset add privaddr 172.172.172.0/24
    fi
}
```

## 钩子函数

ss-tproxy 脚本支持 4 个钩子函数，分别是：
- `pre_start`：启动前执行
- `post_start`：启动后执行
- `pre_stop`：停止前执行
- `post_stop`：停止后执行

> 需要注意的是，shell 中的函数是不允许重复定义的，虽然这不会有任何报错，但是实际只有最后一个函数生效

举个例子，我需要在 ss-tproxy 启动后添加某些规则，在 ss-tproxy 停止后删除这些规则，则修改 ss-tproxy.conf，添加：

```bash
post_start() {
    iptables -A ...
    iptables -A ...
    iptables -A ...
}

post_stop() {
    iptables -D ...
    iptables -D ...
    iptables -D ...
}
```

当然，对于这种需要添加 iptables 规则的情况，可以考虑将 iptables 规则添加到 ss-tproxy 的自定义链上，这些自定义链在 ss-tproxy 停止后会自动删除，因此你只需要关心 `post_start()` 钩子函数的内容；目前有这几个自定义链：

```bash
$ipts -t mangle -N SSTP_PREROUTING
$ipts -t mangle -N SSTP_OUTPUT
$ipts -t nat    -N SSTP_PREROUTING
$ipts -t nat    -N SSTP_OUTPUT
$ipts -t nat    -N SSTP_POSTROUTING
```

它们分别挂接到去掉 `SSTP_` 前缀的同名预定义链上，如下：

```bash
$ipts -t mangle -A PREROUTING  -j SSTP_PREROUTING
$ipts -t mangle -A OUTPUT      -j SSTP_OUTPUT
$ipts -t nat    -A PREROUTING  -j SSTP_PREROUTING
$ipts -t nat    -A OUTPUT      -j SSTP_OUTPUT
$ipts -t nat    -A POSTROUTING -j SSTP_POSTROUTING
```

## 脚本开机自启

对于 `SysVinit` 发行版，直接在 `/etc/rc.d/rc.local` 开机脚本中加上 ss-tproxy 的启动命令即可：
```bash
/usr/local/bin/ss-tproxy start
```

对于 `Systemd` 发行版，将 ss-tproxy.service 文件放到 `/etc/systemd/system/ss-tproxy.service`，执行：
```bash
systemctl daemon-reload
systemctl enable ss-tproxy
```

> 不建议使用 `systemctl start|stop|restart ss-tproxy` 来操作 ss-tproxy，此服务文件应仅作开机自启用。

## 脚本命令行选项
- `ss-tproxy help`：查看帮助信息
- `ss-tproxy version`：查看版本号
- `ss-tproxy start`：启动透明代理
- `ss-tproxy stop`：关闭透明代理
- `ss-tproxy restart`：重启透明代理
- `ss-tproxy status`：查看代理状态
- `ss-tproxy show-iptables`：查看当前的 iptables 规则
- `ss-tproxy flush-postrule`：清空遗留的 iptables 规则
- `ss-tproxy flush-dnscache`：清空 dnsmasq 的查询缓存
- `ss-tproxy delete-gfwlist`：删除 gfwlist 黑名单 ipset
- `ss-tproxy update-chnlist`：更新 chnlist（restart 生效）
- `ss-tproxy update-gfwlist`：更新 gfwlist（restart 生效）
- `ss-tproxy update-chnroute`：更新 chnroute（restart 生效）
- 在任意位置指定 `-x` 选项可启用调试，如 `ss-tproxy start -x`
- 在任意位置指定 `-c cfgfile` 可使用给定路径的 ss-tproxy.conf
- 在任意位置指定 `NAME=VALUE` 可覆盖 ss-tproxy.conf 中的同名配置

**何时使用`delete-gfwlist`**

在`gfwlist/chnlist`模式下，执行了`update-gfwlist|update-chnlist`或修改了`/etc/ss-tproxy/gfwlist.ext`，则建议`start`前执行此指令，防止遗留的`gfwlist`列表导致问题。注意，执行此指令后，可能还需清空内网主机的dns缓存，并重启相关被代理的应用，如正在使用的浏览器。

**关于特殊配置项与`restart`**

如果需要修改某些特殊配置项，请先`ss-tproxy stop`，再修改，再`ss-tproxy start`生效；<br>
不要直接改配置并执行`ss-tproxy restart`，这会导致不可预估的错误，需要遵循约定的有：
- `ipv4`
- `ipv6`
- `proxy_stopcmd`
- `ipts_rt_tab`
- `ipts_rt_mark`
- `opts_overwrite_resolv`
- `file_dnsserver_pid`

> 对于其它配置项，都可以在改完配置后，执行`ss-tproxy restart`命令来生效，无需遵循上述约定

## 黑名单、白名单说明
- 对于 global 模式，白名单文件为 `ignlist.ext`，没有黑名单文件，因为默认都走代理。
- 对于 gfwlist 模式，黑名单文件为 `gfwlist.txt/ext`，没有白名单文件，因为其它都走直连。
- 对于 chnroute 模式，白名单文件为 `ignlist.ext`，没有黑名单文件，但允许开启此功能，见下。

如果想让 chnroute 模式支持黑名单扩展，请打开 chinadns-ng 的 gfwlist 模式（`chinadns_gfwlist_mode`）；开启 gfwlist 模式后，chinadns-ng 会读取 `gfwlist.txt/ext` 黑名单文件中的**域名模式**；当 chinadns-ng 收到域名解析请求时，会先检查给定域名是否在黑名单中，如果是则只向可信 DNS 发出解析请求（也就是 `dns_remote/dns_remote6`），因此解析出来的会是国外 IP（不一定，具体要看给定域名的A/AAAA记录以及其dns解析设定），然后当客户端访问该 IP 时就会走代理出去了（如果解析的地址是国外地址）。

> `chinadns_gfwlist_mode`的本意其实并不是为了支持'黑名单'，而是为了提高 chinadns-ng 的准确性，降低 dns 污染的可能性

## 内网主机tcp限速

首先限速的原理很简单，就是将超过规定速率的包给丢掉（限速一般只针对tcp，udp很少有这种需求），丢掉超过规定速率的包之后，在TCP发送方看来，就是对方(接收方)没收到我发出去的包，也就是“丢包”了，于是会触发TCP的重传机制，于是就达到了限速的目的。

一个简单的例子（这些iptables规则在ss-tproxy主机设置）：
```shell
## 连接级别
# 192.168.1.0/24网段的主机，每条tcp连接，上传限速100kb/s
iptables -t mangle -I PREROUTING -p tcp -s 192.168.1.0/24 -m hashlimit --hashlimit-name upload --hashlimit-mode srcip,srcport --hashlimit-above 100kb/s -j DROP

# 192.168.1.0/24网段的主机，每条tcp连接，下载限速100kb/s
iptables -t mangle -I POSTROUTING -p tcp -d 192.168.1.0/24 -m hashlimit --hashlimit-name download --hashlimit-mode dstip,dstport --hashlimit-above 100kb/s -j DROP

## 主机级别
# 192.168.1.0/24网段的主机，每个内网ip，上传限速100kb/s
iptables -t mangle -I PREROUTING -p tcp -s 192.168.1.0/24 -m hashlimit --hashlimit-name upload --hashlimit-mode srcip --hashlimit-above 100kb/s -j DROP

# 192.168.1.0/24网段的主机，每个内网ip，下载限速100kb/s
iptables -t mangle -I POSTROUTING -p tcp -d 192.168.1.0/24 -m hashlimit --hashlimit-name download --hashlimit-mode dstip --hashlimit-above 100kb/s -j DROP
```

如果是要限制从直连网站下载的速度，要设置一条snat规则，不然限速是不会有效的 ( 因为数据包直接由光猫/路由器发到内网客户机了，不会经过ss-tproxy主机）：
```shell
iptables -t nat -A POSTROUTING -p tcp -s 192.168.1.0/24 -j MASQUERADE
```

> 可以利用`post_start`钩子函数来设置这些规则，然后利用`post_stop`来清理这些规则（把`-I`/`-A`改为`-D`就是删除）。

## 钩子函数小技巧

1、某些系统的 TPROXY 模块可能需要手动加载，对于这种情况，可以利用 `pre_start()` 钩子来加载它：
```bash
pre_start() {
    # 加载 TPROXY 模块
    modprobe xt_TPROXY
}
```

2、不想让某些内网主机走 ss-tproxy 的透明代理，即使它们将网关设为 ss-tproxy 主机，那么可以这么做：
```bash
post_start() {
    if is_false "$selfonly"; then
        if is_true "$ipv4"; then
            # 定义要放行的 IPv4 地址
            local intranet_ignore_list=(192.168.1.100 192.168.1.200)
            for ipaddr in "${intranet_ignore_list[@]}"; do
                iptables -t mangle -I SSTP_PREROUTING -s $ipaddr -j RETURN
                iptables -t nat    -I SSTP_PREROUTING -s $ipaddr -j RETURN
            done
        fi

        if is_true "$ipv6"; then
            # 定义要放行的 IPv6 地址
            local intranet_ignore_list6=(fd00:abcd::1111 fd00:abcd::2222)
            for ipaddr in "${intranet_ignore_list6[@]}"; do
                ip6tables -t mangle -I SSTP_PREROUTING -s $ipaddr -j RETURN
                ip6tables -t nat    -I SSTP_PREROUTING -s $ipaddr -j RETURN
            done
        fi
    fi
}
```

## 切换代理小技巧

如果觉得切换代理要修改 ss-tproxy.conf 很麻烦，可以这么做：
- 将`proxy_startcmd`和`proxy_stopcmd`改为空调用，即`proxy_startcmd='true'`、`proxy_stopcmd='true'`
- 然后配好`proxy_procuser/group`或`proxy_svraddr/port`(不建议用此机制)，将所有可能会用到的值都填进去
- 最后执行`ss-tproxy start`启动，因为我们没有填写任何代理进程的启动和停止命令，所以会显示代理进程未运行，没关系，现在我们要做的就是启动对应的代理进程，假设为 ss-redir 且使用 systemd 管理，则执行 `systemctl start ss-redir`，现在你再执行 `ss-tproxy status` 就会看到对应的状态正常了，当然代理应该也是正常的，如果需要换为 v2ray，假设也是使用 systemd 管理，那么只需要先关闭 ss-redir，然后再启动 v2ray 就行了，即 `systemctl stop ss-redir`、`systemctl start v2ray`。有点类似于启动一个代理框架，后面切换代理就无需再操作 ss-tproxy，直接切换`本机代理进程`即可

## 常见问题解答
[ss-tproxy 常见问题解答](https://github.com/zfl9/ss-tproxy/wiki/Linux-%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86#%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)

如果透明代理未正常工作，请先自行按照如下顺序进行一个简单的排查：
1. 检查 ss-tproxy.conf 以及代理软件的配置是否正确，此文详细说明了许多配置细节，它们并不是废话，请务必仔细阅读此文。如果确认配置无误，那么请务必开启代理进程的详细日志（debug/verbose logging），以及 dnsmasq、chinadns-ng、dns2tcp 的详细日志（ss-tproxy.conf），日志是调试的基础。
2. 如果 ss-tproxy 在配置正确的情况下出现运行时报错，请在执行 ss-tproxy 相关命令时带上 `-x` 调试选项，以查看是哪条命令报的错。出现这种错误通常是脚本自身的问题，可直接通过 issue 报告此错误，但是你需要提供尽可能详细的信息，别一句话就应付了我，这等同于应付了你自己。
3. 如果 ss-tproxy status 显示的状态不正常，那么通常都是配置问题，`pxy/tcp` 显示 stopped 表示代理进程的 TCP 端口未监听，`pxy/udp` 显示 stopped 表示代理进程的 UDP 端口未监听，`dnsmasq`、`chinadns-ng`、`dns2tcp` 显示 stopped 时请查看它们各自的日志文件，可能是监听端口被占用了，等等。
4. 在 ss-tproxy 主机上检查 DNS 是否正常，域名解析是访问互联网的第一步，这一步如果出问题，后面的就不用测试了。这里选择 dig 作为 DNS 调试工具，因此请先安装 dig 工具。在调试 DNS 之前，先开启几个终端，分别 `tail -f` 代理进程、dnsmasq、chinadns-ng、dns2tcp 的日志；然后再开一个终端，执行 `dig www.baidu.com`、`dig www.google.com`，观察 dig 以及前面几个终端的日志输出，发现不对的地方可以先尝试自行解决，如果解决不了，请通过 issue 报告它。如果启用了 udp 透明代理，要特别注意代理的 udp relay 是否正常，udp relay 不正常会导致 `dig www.google.com` 解析失败；如果确认已开启 udp relay，那么你还要注意是否出现了 udp 丢包，某些 ISP 会对 udp 数据包进行恶意性丢弃，检查 udp 是否丢包通常需要检查本地以及 vps 上的代理进程的详细日志输出。实在不行就只能使用 tcponly 模式了。
5. 如果 ss-tproxy 主机的 DNS 工作正常，说明 UDP 透明代理应该是正常的，那么接下来应该检查 TCP 透明代理，最简单的方式就是使用 curl 工具进行检测，首先安装 curl 工具，然后执行 `curl -4vsSkL https://www.baidu.com`、`curl -4vsSkL https://www.google.com`，如果启用了 ss-tproxy 的 IPv6 透明代理支持，则还应该进行 IPv6 的网页浏览测试，即执行 `curl -6vsSkL https://ipv6.baidu.com`、`curl -6vsSkL https://ipv6.google.com`，观察它们的输出是否正常（即是否能够正常获取 HTML 源码），同时观察代理进程、dnsmasq、chinadns-ng、dns2tcp 的日志输出。
6. 如果 ss-tproxy 主机的 DNS 以及 curl 测试都没问题，那么就进行最后一步，在其它内网主机上分别测试 DNS 以及 TCP 透明代理（最简单的就是浏览器访问百度、谷歌），同时你也应该观察代理进程、dnsmasq、chinadns-ng、dns2tcp 的日志输出。对于某些系统，可能会优先使用 IPv6 网络（特别是解析 DNS 时），因此如果你没有启用 ss-tproxy 的 IPv6 透明代理，那么请通过各种手段禁用 IPv6（或者进行其它一些妥当的处理），否则会影响透明代理的正常使用。

> 在报告问题时，请务必提供详细信息，而不是单纯一句话，xxx 不能工作，这对于问题的解决没有任何帮助。
