## 目录
- [第2章：TCP和GO：扫描器和代理](ch-2/第2章：TCP和GO：扫描器和代理)
- [第3章：HTTP客户端：扫描器和代理.md](ch-3/第3章：HTTP客户端：扫描器和代理)

### 环境搭建
本节开始之前，先下载和安装Metasploit编辑器。通过Metasploit中的`msgrpc`模块启动Metasploit控制台以及RPC侦听器。然后设置服务器地址——RPC服务监听的IP——和密码，如代码3-12：
```shell script
$ msfconsole
msf > load msgrpc Pass=s3cr3t ServerHost=10.0.1.6 [*] MSGRPC Service: 10.0.1.6:55552
[*] MSGRPC Username: msf
[*] MSGRPC Password: s3cr3t
[*] Successfully loaded plugin: msgrpc
```
代码 3-12: 启动 Metasploit 和  msgrpc 服务
