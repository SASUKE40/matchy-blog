---
title: 尝试一下 JetBrains Projector
tags:
  - ide
excerpt: 告别 CLion 的脑瘫 full remote 模式，体验像 VSCode remote 一样的 headless server 开发模式。
categories:
  - misc
date: 2021-10-31 22:48:00
updated: 2021-11-08 01:01:00
---


## 为什么要用？

生活所迫，我需要写 C++，还是个颇为庞大的 C++ 项目。每次打开项目 indexing 都要花费好长时间，编译速度也比较捉急。而且我主要为高性能科学计算项目贡献代码，一次测试需要大量资源和特定硬件，几乎无法在本机进行 integration test。只能先把本机代码 push 到远端再在服务器上 pull + 编译进行测试，非常繁琐。综上所述，我对 remote development 的需求非常迫切。

说到 remote development，最好的模式当然是像 VSCode remote 那样，代码完全在服务器上，远端起一个 headless server，IntelliSense、编译、测试一条龙。但 VSCode 的 C++ IntelliSense 实在是太弱了，不能达到 CLion 的体验。而 CLion 的 "full remote" 模式其实并不 "full"， IntelliSense 还在本地，不能避免 indexing 花费大量时间的问题，还要不断地把本地代码文件上传到服务器，属于是脑瘫体验了。为此咱还在推特上发表过如下感慨：

<center>
<blockquote class="twitter-tweet"><p lang="zh" dir="ltr">CLion 的 remote 模式为什么不能像 VSCode 那样直接在远端起一个 headless server 本地只是个壳呢</p>&mdash; 迷你麦吃 (@233matchy) <a href="https://twitter.com/233matchy/status/1428604015659474951?ref_src=twsrc%5Etfw">August 20, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</center>

幸运的是，在发布这条推特后，热心推友向我介绍了 JetBrains Projector 这个东西。根据[官网介绍](https://lp.jetbrains.com/projector/)，这基本就和 VSCode remote 是一个模式，完全符合我的需求！于是便有了这篇试水博客。

## 安装过程

我直接使用了 projector installer 版本，照着 [JetBrains Projector GitHub repo 的 README](https://github.com/JetBrains/projector-installer#Installation) 的指示做就行了。也有 Docker 版本和 IDE 插件版本。不过比较麻烦的是我的 lab cluster 内服务器没有公网 IP，所以还要多配置一步。

### 1. Server 端安装

我的 host server 安装的是 Debian 10，下面的操作都是基于这一操作系统的。其他系统应该大同小异。

首先安装必要的依赖库：`less`, `libext6`, `libxi6`, `libxrender1`, `libxtst6` 和`libfreetype6`。

```shell
sudo apt install less libxext6 libxrender1 libxtst6 libfreetype6 libxi6 -y
```

然后安装 `python3`、`python3-pip3` 和 `python3-cryptography`，并更新 `pip`。

```shell
sudo apt install python3 python3-pip -y
sudo apt install python3-cryptography -y
python3 -m pip install -U pip
```

最后使用 `pip3` 为当前用户安装 `projector-installer`。

```shell
pip3 install projector-installer --user
```

`projector-installer` 会被安装在 `$HOME/.local/bin` 中，将该路径添加到 `$PATH` 即可在命令行使用 `projector` 命令。我的 `$HOME/.profile` 中已经写有相应的命令了，所以直接 `source $HOME/.profile` 就可以了。

```bash
$ cat ~/.profile
...
# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/.local/bin" ] ; then
    PATH="$HOME/.local/bin:$PATH"
fi
...
$ source ~/.profile
```

检测 `projector` 正常安装：

```shell
$ projector --version
projector, version 1.4.0
```

接下来只需执行 `projector install`，选择想要使用的 IDE 并按照提示安装。我当然是选择 CLion，且为了稳定选择了 Projector-tested IDE only。

安装完成后会生成一个 config 文件。可以使用 `projector config show <config-name>` 查看（如果只有一个 config file 不加 `<config-name>` 也能查看），默认是监听任何地址向 9999 端口的请求。如果安装多个 IDE，则会依次监听 `10000`、`10001`…等端口。想要修改可以 `projector config edit <config-name>`，我就不做修改了。

```shell
$ projector config show CLion
Configuration: CLion
Projector port: 9999
Projector listening address: *
IDE path: /path/to/your/home/.projector/apps/clion-2020.3.4
Product name: CLion, version=2020.3.4, build=203.8084.11
Use separate config: no
Update channel: tested
Projector uses secure config (https/wss)= no
```

此后便可使用 `projector run CLion` 运行所安装的 IDE，

```shell
$ projector run CLion
To access IDE, open in browser
        http://localhost:9999/
        http://127.0.0.1:9999/
        http://<your-lan-ip>:9999/

To see Projector logs in realtime run
        tail -f "/path/to/your/home/.projector/configs/CLion/projector.log"

Exit IDE or press Ctrl+C to stop Projector.
```

此时如果 server 有公网 IP，已经可以通过访问 `http://<your-wan-ip>:9999` 来使用 IDE 了。

### 2. 配置 server NAT rules

如前所述，我安装的服务器并没有公网 IP，只能通过 bastion server 访问。Bastion server 本身还充当了整个 cluster 的软路由。因此还需要配置端口转发，把针对 bastion server 9999 端口的请求都转发到真正的 host server 上去。

我的 bastion server 安装的是 Ubuntu 18.04.6 LTS，下面的操作都是基于这一操作系统的。其他系统应该大同小异。

首先的首先，是开启 bastion server 的 IP forwarding 功能，这随便谷歌一下就有[教程](https://linuxconfig.org/how-to-turn-on-off-ip-forwarding-in-linux)。可以通过修改 `/etc/sysctl.conf` 中 `net.ipv4.ip_forward` 的值来永久开启。

然后是开放 bastion server 的 `9999` 端口。

```shell
sudo iptables -A INPUT -p tcp --dport 9999 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
```

向 nat table 中添加两条 routing 规则，把请求转发到合理的端口。

```bash
sudo iptables -t nat -A PREROUTING -p tcp -d $BASTION_WAN_IP --dport 9999 -j DNAT --to-destination $HOST_LAN_IP:9999
```

```bash
sudo iptables -t nat -A POSTROUTING -p tcp -s $HOST_LAN_IP --sport 9999 -j SNAT --to-source $BASTION_WAN_IP:9999
```

我的 bastion server 并没有默认 `ACCEPT` 所有的 packets，所以还要特别允许导向 host server 的 traffic。

```bash
sudo iptables -A FORWARD -p tcp -d $HOST_LAN_IP --dport 9999 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

现在就可以在浏览器中打开 `http://<bastion-wan-ip>:9999` 来访问 IDE 啦。

![成功在浏览器中打开 IDE](https://i.loli.net/2021/10/31/bc3sX2PaMfiQFVk.png)

### 3. Client 端安装

根据 [JetBrains Projector 的文档](https://jetbrains.github.io/projector-client/mkdocs/latest/ij_user_guide/accessing/#known-issues)，在浏览器中打开 IDE 还有一些已知的 bug，官方建议是使用 client launcher。在 [JetBrains/projector-launcher 的 release page](https://github.com/JetBrains/projector-client/releases) 找到包含 "launcher" 字样的 release, 下载 Assets 中适合自己平台的 zip 压缩包即可。

![JetBrains/projector-launcher 的 release page](https://i.loli.net/2021/10/31/wXBIpV9NlJsxDC5.png)

解压后运行 `projector.exe` 或 `projector`，在弹出的界面输入框内按格式输入 `http://<your-wan-ip>:9999` 即可使用 IDE。

![Projector launcher 界面](https://i.loli.net/2021/10/31/TES7LgfiorBx3kz.png)

## 后续折腾 (placeholder)

1. 1202 年了怎么还能用 `http` 呢？过于不安全。有时间把 `https` 搞起来。
2. 把 `projector` 添加为 `systemd` 的一个 service，开机自启。
3. `iptables` 的命令分好几行太丑，写在一行又太长。有时间拆开介绍一下。
