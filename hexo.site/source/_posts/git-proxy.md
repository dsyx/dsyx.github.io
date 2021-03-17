---
title: 'Git 代理'
date: 2021-03-17 20:42:39
tags: 'Git'
---

由于国内特殊的原因，我们访问 [GitHub](https://github.com/) 的速度十分缓慢。尽管如此，我们仍然需要经常地访问它，毕竟它是全球最活跃的开源代码托管站。

Git 支持使用 ssh 和 http(s) 协议进行传输。因此我们可以对其使用代理（梯子）来提高克隆 GitHub 仓库的速度。在默认情况下，国内克隆 GitHub 上的仓库的速度可能只有 15 ~ 30 KB/s，而使用代理则可以达到 1 ~ 10 MB/s（具体取决于代理的速度）。

现在你需要拥有一个代理（至于如何获得代理，请自行百度/谷歌），本文假定两个代理：

* http 代理：`127.0.0.1:1080`
* socks5 代理：`127.0.0.1:1081`

## 代理 http(s) 传输

Git 支持 http(s) 代理功能，仅仅需要简单的配置即可开启该功能：

使用 http 代理来代理 Git 的 http(s) 传输：

```bash
git config --global http.proxy 'http://127.0.0.1:1080'
git config --global https.proxy 'http://127.0.0.1:1080'
```

或使用 socks5 代理来代理 Git 的 http(s) 传输：

```bash
git config --global http.proxy 'socks5://127.0.0.1:1081'
git config --global https.proxy 'socks5://127.0.0.1:1081'
```

如果需要取消 Git 的 http(s) 传输代理，可以使用以下命令：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

## 代理 ssh 传输

利用 [OpenSSH](https://www.openssh.com/) 的 [ProxyCommand](https://man.openbsd.org/ssh_config#ProxyCommand) 特性，可以实现 ssh 传输的代理。

默认情况下， ssh 会自己建立与目标机器的连接。如果启用了 ProxyCommand 特性，则 ProxyCommand 所指定的命令负责建立连接，ssh 会直接使用命令所建立的连接。

通过 [netcat](https://en.wikipedia.org/wiki/Netcat) 可以建立连接以供 ssh 使用。

> 注意：大部分 Unix/Linux 系统都默认安装了 netcat，一般为 `nc` 命令；Windows 系统需要另行安装。

修改 OpenSSH 的 `config` 文件（Unix/Linux/Git-Bash 中的 `~/.ssh/config` ），添加如下内容之一：

使用 http 代理：

```
Host github.com
    HostName github.com
    User git
    ProxyCommand nc -v -x 127.0.0.1:1080 %h %p
```

或使用 socks5 代理：

```
Host github.com
    HostName github.com
    User git
    ProxyCommand nc -v -x 127.0.0.1:1081 %h %p
```

修改后保存即可。后续的 `git` 命令若使用 ssh 协议进行传输，则会利用此代理。
