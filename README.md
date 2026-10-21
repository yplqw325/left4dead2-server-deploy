# 腾讯云服务器搭建求生之路：1核2G 就够用，从选配置、装服务端到好友联机的全过程

想在腾讯云上开一台《求生之路2》服务器，配置门槛比想象中低：1核2G、硬盘 40G 以上的机器就能跑，流程是「装 SteamCMD → 下载 222860 服务端 → 写 server.cfg → 放行 27015 端口 → screen 挂后台」。真正会卡住人的是两处细节：2024 年 11 月之后匿名账号已经下不动服务端，以及防火墙只放行 TCP 会连不上。下面按操作顺序走一遍。

> **省流结论**：2～6 人开黑，1核2G、40G 以上硬盘的轻量应用服务器或 CVM 就够，系统选 Debian 12，带宽 1Mbps 起步。腾讯云的套餐和活动价经常调整，具体以下单页为准：👉 [打开腾讯云活动页看当前可选配置](https://bit.ly/yunhuodong)

## 配置怎么选：先看你几个人玩

私有服不吃 CPU，吃的是人数、插件和第三方地图。2023 年有篇搭建教程里，作者用 1核2G、1Mbps 的腾讯云学生机（当时 10 元/月）带几个朋友玩，结论是「绰绰有余」。按现在的玩法大致分三档：

| 场景 | CPU / 内存 | 硬盘 | 带宽 |
| --- | --- | --- | --- |
| 2～4 人，纯净服或少量插件 | 1核2G | 40G | 1Mbps |
| 4～8 人，带 SourceMod 和第三方图 | 2核2G | 60G 以上 | 3Mbps 起 |
| 10 人以上对抗、插件较多 | 2核4G 起 | 80G 以上 | 5Mbps 起 |

硬盘这一列值得单独说：服务端本体 9～10GB，每次 `validate` 更新还要临时占空间，买 20G 硬盘的机器大概率会在更新时报错。40G 是相对稳妥的底线。

腾讯云这边有两个产品线可选，搭 L4D2 都能用：

| 产品线 | 特点 | 适合谁 | 入口 |
| --- | --- | --- | --- |
| 轻量应用服务器 | 配置打包成固定套餐，带宽、硬盘一起给 | 配置需求固定、不想研究计费 | [查看当前套餐](https://bit.ly/yunhuodong) |
| 云服务器 CVM | 配置档位多，支持升降配 | 想灵活调整，或蹲学生机/活动价 | [查看当前活动](https://bit.ly/yunhuodong) |

地域上选离大家近的节点就行，延迟能低一点。这一步不用纠结太久，几个人玩的私有服对区域不敏感。

## 系统镜像：选 Debian 12 或 Ubuntu，别装 CentOS 8

新机器直接选 Debian 12 或 Ubuntu 22.04/24.04。CentOS 8 在 2021 年底停止维护，CentOS 7 也在 2024 年 6 月走完了生命周期；网上 2023 年前后的教程清一色用 CentOS，是当时的习惯，现在照着装会遇到软件源失效、依赖装不上的麻烦。

下面的命令以 Debian/Ubuntu 为准，CentOS 系的差异只有依赖包名，会单独标出来。

## 装依赖和 SteamCMD

SteamCMD 是 32 位程序，64 位系统要先补 32 位运行库，否则运行时会直接报错。

bash
sudo apt update
sudo apt install lib32gcc-s1 wget screen -y


CentOS 系对应的是 `yum install glibc.i686 libstdc++.i686 -y`。

接下来建议建一个普通用户跑服务端，不要全程 root：

bash
adduser l4d2
su l4d2
mkdir ~/steam && cd ~/steam
wget https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz
tar xvf steamcmd_linux.tar.gz
./steamcmd.sh


第一次运行会自更新，等出现 `Steam>` 提示符就装好了。

## 下载 222860：匿名登录已经不好使了

L4D2 服务端的 App ID 是 222860，现在下载它必须用一个已购买游戏的个人 Steam 账号登录，匿名账号大约从 2024 年 11 月起就下不动了。早年教程里的流程是：


login anonymous
app_update 222860 validate


现在这么执行，大概率会看到：


ERROR! Failed to install app '222860' (Invalid platform)


这是 Steam 收紧了匿名下载，跟你的操作无关。换成个人账号：


login 你的Steam账号 你的密码


绑了手机令牌的话会提示 `Two-factor code:`，输入动态验证码即可。登录成功后指定安装目录并下载：


force_install_dir /home/l4d2/steam/l4d2-ds
app_update 222860 validate


下载量 9～10GB。这里有个很多人没算过的账：1Mbps 带宽的理论下载速度大约 128KB/s，下完 9GB 要按几十个小时算。CVM 的带宽可以选按固定带宽或按使用流量计费，临时把带宽开高、装完再调回去是个省时间的办法；轻量应用服务器的带宽包含在套餐里不能单独调，买之前心里要有数。

另外，有玩家反馈用个人账号下载时可能把已订阅的创意工坊内容一起拉下来，如果目录明显超过 10GB，可以检查一下是不是这个原因。下完 `quit` 退出。

如果机器还没买，硬盘和带宽直接决定这一步的体验：👉 [先开一台 40G 以上硬盘的腾讯云服务器](https://bit.ly/yunhuodong)

## 启动、server.cfg 与后台运行

启动就是一行命令，配置写进 `server.cfg`，后台交给 screen 挂着。

bash
cd ~/steam/l4d2-ds
screen -S l4d2
./srcds_run -game left4dead2 -insecure +port 27015 +map c2m1_highway +mp_gamemode coop +exec server.cfg


几个参数的意思：`-insecure` 关闭 VAC 反作弊，私有服一般这么用；`+port 27015` 是默认游戏端口；`+map c2m1_highway` 指定首张地图（死亡中心第一关）；`+mp_gamemode coop` 是战役模式。启动后按 `Ctrl+A` 再按 `D` 分离会话，SSH 断开服务也不停；下次回来用 `screen -r l4d2`。

`server.cfg` 放在 `left4dead2/cfg/` 目录下，最小可用配置大概是这样：

cfg
hostname "和朋友开黑"
rcon_password "换成你自己的密码"
mp_gamemode "coop"
z_difficulty "normal"
sv_region 4
sv_allow_lobby_connect_only 0
sv_tags "hidden"


难度可选 easy、normal、hard、impossible。`sv_tags "hidden"` 是有教程作者特意加的：把服从公开列表里藏起来，减少被扫描捣乱的概率。改了端口的话，好友连接地址和下面的防火墙规则都要一起改。

想要服务器重启后自动拉起，可以写一个 systemd 服务把 `srcds_run` 设为 `ExecStart`、`Restart=always`，再 `systemctl enable` 开机自启；偶尔开黑的话，screen 够用了。

## 防火墙放行 27015，好友才连得进来

腾讯云的机器默认不放行游戏端口，这是搭完「连不上」的头号原因。轻量应用服务器在控制台的「防火墙」里加规则，CVM 在「安全组」里加，注意 TCP 和 UDP 都要放行 27015——游戏两种协议都用，只开 TCP 的表现就是一直卡在连接中。

好友进服的方式：游戏里打开 选项 → 键盘/鼠标 → 允许使用开发者控制台，按 `~` 调出控制台，输入：


connect 服务器公网IP:27015


如果设了 `sv_tags hidden`，在服务器浏览器里搜不到是正常的，直接 connect 或加收藏即可。

## 插件：先把纯净服跑通再加

超过 4 人、管理菜单、100 tick 这些功能都要靠插件，但纯净服能进之前别急着装。基础是 SourceMod + MetaMOD，把解压出的 `addons` 和 `cfg` 两个文件夹上传到服务端的 `left4dead2/` 目录合并覆盖。想突破战役 4 人上限要装 L4DToolz 这类插件——顺带纠正一个常见误解：启动命令里写 `+maxplayers 8` 并不会真的变成 8 人，人数上限是插件在管。Tickrate Enabler 用来解除默认 30 tick 的限制，不需要可以不管。

安装顺序建议：纯净服能进 → MetaMOD → SourceMod → 功能插件。反过来先堆插件，出问题时分不清是服务端、插件还是配置的锅。

管理员权限在 `addons/sourcemod/configs/admins_simple.ini` 末尾添加一行，引号里换成自己的 SteamID：

ini
"STEAM_1:1:12345678" "99:z"


## 常见报错对照

搭建过程中会遇到的问题基本就这几类，对着表处理：

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `Failed to install app '222860' (Invalid platform)` | 匿名下载被限制 | 换已购游戏的个人账号登录 |
| `state is 0x202 after update job` | 磁盘空间不足 | 扩硬盘或清理后重试 |
| 错误代码 0x2 | 网络波动 | 重试，或换个时间段 |
| 好友 connect 一直没响应 | 端口没放行 | 安全组/防火墙放行 TCP + UDP 27015 |
| 服务器列表搜不到自己的服 | 私有服默认不公开 | 直接 connect IP，或加收藏 |

## 能带几个人，什么时候该升配

4 人左右纯净战役，1核2G + 1Mbps 就够；8 人以上带插件和第三方图，再考虑 2核4G 和更高带宽。判断依据很简单：CPU 平时基本闲着，内存被插件和地图吃，带宽被人数吃。出现卡顿先看上行有没有跑满、内存有没有吃紧，再决定升不升。《求生之路2》是 2009 年的游戏，服务端逻辑本身很轻。

如果只是同一局域网或者同城几个人玩，本地开服或内网穿透也能解决，云服务器的价值在于成员分散、网络环境复杂（比如校园网 NAT）、想随时在线这几种情况。

已经确定要 8 人以上带插件，就别在 1核2G 上省了：👉 [在腾讯云活动页筛选 2核4G 档位](https://bit.ly/yunhuodong)

## 最后：按你的情况选

- 2～4 人、纯净或轻插件：1核2G + 40G 硬盘，最便宜的档位就够
- 6～8 人、插件和第三方图都要：2核2G 或 2核4G + 60G 以上
- 只是偶尔同城开黑：先试本地开服，不一定要买云服务器

下单前确认两件事：硬盘不低于 40G（服务端本体 9～10GB，更新还要临时空间），系统镜像选 Debian 12 或 Ubuntu。买好之后照着上面的顺序操作，顺利的话半小时内能进服。

👉 [去腾讯云挑一台能跑求生之路的服务器](https://bit.ly/yunhuodong)
