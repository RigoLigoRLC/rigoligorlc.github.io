# 如何让 Git SSH 走代理

2024/1/18 - 我发现曾经还能直连的 GitHub SSH 连接已经无法正常连接上了。

```
(base) PS D:\prj\qt\ProbeScope> git pull
ssh: connect to host github.com port 22: Connection timed out
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

立刻意识到这是 GitHub SSH 连接也已经被我使用的网络阻断了。所以得想办法把 SSH 从代理里走出去。

Windows 上从 Git 官网下载的 build 应该都会带 `connect.exe` 。在 Windows 机的 `C:\Users\用户名\.ssh\config` 中添加：

```
Host github.com
  User git
  ProxyCommand connect.exe -S 127.0.0.1:10808 %h %p
```

其中 `127.0.0.1:10808` 替换成你本地的代理软件的 SOCKS5 IP 与端口。如果使用 HTTP 代理，将 `-S` 替换为 `-H` 。添加后应该可以正常连接 GitHub 。v2rayN 中日志：

```
...
2024/01/18 00:52:12 tcp:127.0.0.1:63219 accepted tcp:github.com:22 [socks -> proxy]
...
```
