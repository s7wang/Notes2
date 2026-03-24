# ubuntu 环境配置

## 镜像版本 & 安装虚拟机

* 版本 ubuntu-24.04.4-live-server-amd64.iso

[下载链接](https://releases.ubuntu.com/24.04/ubuntu-24.04.4-live-server-amd64.iso)

* 虚拟机安装

[Ubuntu 24.04 Server 安装与 VMware 常见问题处理 - Yangshengzhou - 博客园](https://www.cnblogs.com/YangShengzhou/articles/19323323)

## SSH配置

* 安装并开启ssh

~~~(空)
sudo apt update
sudo apt install openssh-server -y
~~~

开启

~~~shell
# 开启
sudo systemctl start ssh
# 开机启动
sudo systemctl enable ssh
# 查看状态
sudo systemctl status ssh
~~~

如给开启了防火墙需要放行

~~~shell
sudo ufw allow ssh
sudo ufw enable
sudo ufw status
~~~

配置路径：

~~~shell
sudo vim /etc/ssh/sshd_config
~~~



## git安装

~~~shell
sudo apt update
sudo apt install git -y
~~~



## clash安装配置

参考：[GitHub - nelvko/clash-for-linux-install: 😼 优雅地使用基于 clash/mihomo 的代理环境 · GitHub](https://github.com/nelvko/clash-for-linux-install)

### 🚀 一键安装

在终端中执行以下命令即可完成安装：

```
git clone --branch master --depth 1 https://gh-proxy.org/https://github.com/nelvko/clash-for-linux-install.git \
  && cd clash-for-linux-install \
  && bash install.sh
```

- 上述命令使用了[加速前缀](https://gh-proxy.org/)，如失效可更换其他[可用链接](https://ghproxy.link/)。
- 可通过 `.env` 文件或脚本参数自定义安装选项。
- 没有订阅？[click me](https://xn--z4q834d.net/auth/register?code=oUbI)

**安装后命令无法使用参考**：

### bash: clashon: command not found

- 原因：当前 `shell` 没有加载 `clashctl.sh` 等脚本。
- 解决：查看 `~/.bashrc` 中是否具有加载 `clashctl.sh` 的命令，然后在当前终端执行下 `bash -i` 或 `. ~/.bashrc`。

### ⌨️ 命令一览



```
Usage: 
  clashctl COMMAND [OPTIONS]

Commands:
    on                    开启代理
    off                   关闭代理
    status                内核状况
    proxy                 系统代理
    ui                    Web 面板
    secret                Web 密钥
    sub                   订阅管理
    upgrade               升级内核
    tun                   Tun 模式
    mixin                 Mixin 配置

Global Options:
    -h, --help            显示帮助信息
```



💡`clashon` 同 `clashctl on`，`Tab` 补全更方便！

#### 优雅启停



```
$ clashon
😼 已开启代理环境

$ clashoff
😼 已关闭代理环境
```



- 在启停代理内核的同时，同步设置系统代理。
- 亦可通过 `clashproxy` 单独控制系统代理。

#### Web 控制台



```
$ clashui
╔═══════════════════════════════════════════════╗
║                😼 Web 控制台                  ║
║═══════════════════════════════════════════════║
║                                               ║
║     🔓 注意放行端口：9090                      ║
║     🏠 内网：http://192.168.0.1:9090/ui       ║
║     🌏 公网：http://8.8.8.8:9090/ui          ║
║     ☁️ 公共：http://board.zash.run.place      ║
║                                               ║
╚═══════════════════════════════════════════════╝

$ clashsecret mysecret
😼 密钥更新成功，已重启生效

$ clashsecret
😼 当前密钥：mysecret
```



- 可通过浏览器打开 `Web` 控制台进行可视化操作，例如切换节点、查看日志等。
- 默认使用 [zashboard](https://github.com/Zephyruso/zashboard) 作为控制台前端，如需更换可自行配置。
- 若需将控制台暴露到公网，建议定期更换访问密钥，或通过 `SSH` 端口转发方式进行安全访问。

#### `Mixin` 配置



```
$ clashmixin
😼 查看 Mixin 配置

$ clashmixin -e
😼 编辑 Mixin 配置

$ clashmixin -c
😼 查看原始订阅配置

$ clashmixin -r
😼 查看运行时配置
```



- 通过 `Mixin` 自定义的配置内容会与原始订阅进行深度合并，且 `Mixin` 具有最高优先级，最终生成内核启动时加载的运行时配置。
- `Mixin` 支持以前置、后置或覆盖的方式，对原始订阅中的规则、节点及策略组进行新增或修改。

#### 升级内核



```
$ clashupgrade
😼 请求内核升级...
{"status":"ok"}
😼 内核升级成功
```



- 升级过程由代理内核自动完成；如需查看详细的升级日志，可添加 `-v` 参数。
- 建议通过 `clashmixin` 为 `github` 配置代理规则，以避免因网络问题导致请求失败。

#### 管理订阅



```
$ clashsub -h
Usage: 
  clashsub COMMAND [OPTIONS]

Commands:
  add <url>       添加订阅
  ls              查看订阅
  del <id>        删除订阅
  use <id>        使用订阅
  update [id]     更新订阅
  log             订阅日志


Options:
  update:
    --auto        配置自动更新
    --convert     使用订阅转换
```



- 支持添加本地订阅，例如：`file:///root/clashctl/resources/config.yaml`
- 当订阅链接解析失败或包含特殊字符时，请使用引号包裹以避免被错误解析。
- 自动更新任务可通过 `crontab -e` 进行修改和管理。

#### `Tun` 模式



```
$ clashtun
😾 Tun 状态：关闭

$ clashtun on
😼 Tun 模式已开启
```



- 作用：实现本机及 `Docker` 等容器的所有流量路由到 `clash` 代理、DNS 劫持等。
- 原理：[clash-verge-rev](https://www.clashverge.dev/guide/term.html#tun)、 [clash.wiki](https://clash.wiki/premium/tun-device.html)。
- 注意事项：[#100](https://github.com/nelvko/clash-for-linux-install/issues/100#issuecomment-2782680205)

### 🗑️ 卸载



```
bash uninstall.sh
```



## git配置

### git代理配置

在 Linux 上让 **Git 走 Clash 代理**，本质就是给 Git 配置 HTTP / SOCKS 代理。你现在的 Clash 配置是：

- HTTP 代理端口：`7890`
- SOCKS5 代理端口：`7891`

下面给你分场景讲清楚（开发环境常用这几种）👇

------

**一、最推荐：Git 使用 HTTP 代理（7890）**

Git 默认最兼容 HTTP 代理（Clash 的 7890）

✅ 全局配置

```
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

------

✅ 验证是否生效

```
git config --global --get http.proxy
```

------

**二、使用 SOCKS5（7891）**

如果你更想走 SOCKS（比如 DNS 防污染）

⚠️ 注意：Git 需要 curl 支持 socks

```
git config --global http.proxy socks5h://127.0.0.1:7891
git config --global https.proxy socks5h://127.0.0.1:7891
```

为什么用 `socks5h`？

- `socks5` → 本地解析 DNS
- `socks5h` → 走代理解析（推荐，防 DNS 污染）

**三、取消代理**

~~~(空)
git config --global --unset http.proxy
git config --global --unset https.proxy
~~~

### github相关配置

**一、配置 Git 用户信息（必须）**

```
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

验证：

```
git config --global --list
```

👉 注意：

- 邮箱必须和 GitHub 注册邮箱一致（否则提交不会关联）
- `--global` 表示全局配置（推荐）

------

**二、生成 SSH Key（推荐方式）**

（避免每次 push 输入密码）

1. 生成 key

```
ssh-keygen -t ed25519 -C "你的邮箱"
```

一路回车即可，默认生成在：

```
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

------

2. 启动 ssh-agent 并添加 key

```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

------

3. 查看公钥

```
cat ~/.ssh/id_ed25519.pub
```

复制输出内容（很长一串）

------

**三、添加到 GitHub**

1. 打开 GitHub

2. 进入：

   ```
   Settings → SSH and GPG keys → New SSH key
   ```

3. 粘贴刚才的公钥

------

**四、测试连接**

```
ssh -T git@github.com
```

成功会看到类似：

```
Hi username! You've successfully authenticated...
```

------

**五、配置仓库使用 SSH（关键）**

克隆仓库时用 SSH 地址：

```
git clone git@github.com:用户名/仓库名.git
```

而不是：

```
https://github.com/xxx/xxx.git
```



------------

虚拟机信息：Ubuntu_server.zip

ubuntu-24.04.4

用户名：wangs7server

密码：000000





