# Deploy-AnyConnect-on-CentOS7

## 前言
众所周知 Cisco 是一家领先的网络设备提供商，他家的 VPN 协议 AnyConnect 也是少有的可以正常使用的 VPN
在本文中，将在 CentOS 7 上部署 AnyConnect 来使用

## 系统需求
+ CentOS 7+
+ 非半虚拟化技术(OpenVZ/Virtuozzo/Xen-DomU)

### 部署 - 证书
现有的教程多半使用编译方式来安装 ocserv，实际上 EPEL 源中已经包含了这个包，而且还是最新的，因此只需要安装 EPEL 在安装 ocserv 即可
执行以下命令即可：

```
yum install epel-release -y
yum install ocserv -y
```

现有的教程还有另一个坑，都是教你使用 certtool 生成自签名证书来使用的；如果你不想被中间人攻击的话，最好别这么干
事实上有效的证书很容易获得，以前介绍过的 `Let's Encrypt` 就是很可靠的一个签发机构
如果你还不知道如何获取 `Let's Encrypt` 的证书，你可以参考此文章：[https://iclart.com/archives/391](https://iclart.com/archives/391)

获取到证书后，你需要将证书复制进 `/etc/ocserv/ssl` 这个文件夹(其他文件夹也可以，记得改配置文件就行)，没有的话用以下命令创建

```
mkdir -p /etc/ocserv/ssl
cd /etc/ocserv/ssl
```

完成后的 `/etc/ocserv/ssl` 目录应该是这样的：

[![](https://image.mxyweb.com/2019/06/26/Termius_2019-06-26_20-09-32.png)](https://image.mxyweb.com/2019/06/26/Termius_2019-06-26_20-09-32.png)

### 部署 - 配置文件
完成证书配置之后，下一步就是对配置文件进行修改
默认情况下，配置文件位于 `/etc/ocserv/ocserv.conf`，以下的配置字段需要重点关注

```
# 将验证方式设为用户名+密码
auth = "plain[passwd=/etc/ocserv/ocpasswd]"

# TCP/UDP端口，默认为443
tcp-port = 443
udp-port = 443

# 在上一步中添加的证书，请根据实际情况自行修改
server-cert = /etc/ocserv/ssl/server-cert.pem
server-key = /etc/ocserv/ssl/server-key.pem

# 最大连接数
max-clients = 20

# 每个用户的最大设备树
max-same-clients = 3

# MTU自动配置，不是True的话会奇慢无比
try-mtu-discovery = true

# 很多教程没有提及的点，默认是会启用TLS1.0的
# 还有RSA类无前向保密的算法，为了安全起见
# 此配置只启用了TLS1.2和128位以上具有前向保密的算法
tls-priorities = "SECURE128:+SECURE256:-VERS-ALL:+VERS-TLS1.2"

# 默认域名，建议设置位前一步中证书签发的域名
default-domain = ocserv.xxx.com

# 网段信息：建议修改以避免域现有网段冲突
ipv4-network = 172.31.0.0
ipv4-netmask = 255.255.255.0

# 路由信息：此配置位绕过局域网
route = 0.0.0.0/192.0.0.0
route = 64.0.0.0/192.0.0.0
route = 128.0.0.0/192.0.0.0
route = 192.0.0.0/192.0.0.0
no-route = 192.168.0.0/255.255.0.0
```

之后需要创建一个新用户，使用以下命令

```
ocpasswd -c /etc/ocserv/ocpasswd iclart
```

输入两遍密码即可，此时 `/etc/ocserv` 目录结构应该如下图：

[![](https://image.mxyweb.com/2019/06/26/Termius_2019-06-26_20-24-20.png)](https://image.mxyweb.com/2019/06/26/Termius_2019-06-26_20-24-20.png)

### 部署 - 系统配置
完成 ocserv 的配置文件编辑后，我们需要对系统进行一些配置
首先，需要开启 IPv4 转发

```
vim /etc/sysctl.conf
net.ipv4.ip_forward = 1

# 使配置生效
sysctl -p
```

其次需要对转发进行配置

```
# 启动 firewalld
systemctl start firewalld

# 放通端口，注意修改
firewall-cmd --permanent --zone=public --add-port=443/tcp
firewall-cmd --permanent --zone=public --add-port=443/udp

# 转发配置，注意将 eth0 修改位你的网卡接口
firewall-cmd --permanent --add-masquerade
firewall-cmd --permanent --direct --passthrough ipv4 -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# 使配置生效
firewall-cmd --reload
```

确定没有问题之后，就可以启动 ocserv 了

```
systemctl enable ocserv
systemctl start ocserv
```

启动后执行此命令，应该得到类似输出

```
netstat -tunlp | grep 443
```

[![](https://image.mxyweb.com/2019/06/26/Termius_2019-06-26_20-37-49.png)](https://image.mxyweb.com/2019/06/26/Termius_2019-06-26_20-37-49.png)

至此，服务端配置已全部完成，在配置客户端时，注意服务器应该填写域名

[![](https://image.mxyweb.com/2019/06/26/vpnui_2019-06-26_20-39-28.png)](https://image.mxyweb.com/2019/06/26/vpnui_2019-06-26_20-39-28.png)

## 总结
**不要使用自签名证书**
**服务器需填写域名**

**为保证良好的排版风格，请在搬运时使用 `Markdown` 版本，[本文 `Markdown` 版本](https://iclartstor.blob.core.windows.net/markdown/archive-495.md)**
