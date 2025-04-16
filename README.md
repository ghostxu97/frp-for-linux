# **Linux搭建frp服务，实现内网穿透服务，实现外网到内网的在线访问**

## **一：frp简介**

### **frp 是什么？**

frp 采用 Golang 编写，支持跨平台，仅需下载对应平台的二进制文件即可执行，没有额外依赖。

frp 是一个专注于内网穿透的高性能的反向代理应用，支持 TCP、UDP、HTTP、HTTPS 等多种协议。可以将内网服务以安全、便捷的方式通过具有公网 IP 节点的中转暴露到公网。

市面上一些主流的内网穿透工具有：Ngrok，Natapp，花生壳，Ssh、autossh，Frp，Lanproxy，Spike。

这里介绍使用frp工具。

Ngrok可参考我另一篇文章：搭建ngrok服务器，实现内网穿透服务

## **二：安装frp**

### **1：准备一台公网服务器（配置无要求网络稳定就行），作为服务器端，如公网IP：123.32.12.32。**

内网客户端（准备要穿透出去的设备），客户端，如内网IP：192.168.152.103。

### **2：下载解压安装包**

github地址：[https://github.com/fatedier/frp](https://github.com/fatedier/frp),可以在 Release 页面中下载到最新版本的客户端和服务端二进制文件，所有文件被打包在一个压缩包中。

找到自己Linux合适的版本，下载，主流Linux版本基本上都是amd64。

```
#下载
wget https://github.com/fatedier/frp/releases/download/v0.44.0/frp_0.44.0_linux_amd64.tar.gz
#解压
tar -zxvf frp_0.44.0_linux_amd64.tar.gz
#进入目录
cd frp_0.44.0_linux_amd64/
```

进入文件夹，有两个名称文件frpc（c结尾代表client）和frps（s结尾代表server），分别是服务端程序和服务端配置程序。

需要将frpc拷贝至客户端，即内网服务器，或者在客户端直接下载也可以，客户端只需要使用frpc文件即可。

### **3: 赋予执行命令权限**

```
sudo chmod 755 ./frps
```

## **三：配置服务器端和客户端，及启动**

### **1：配置服务器端frps.ini文件**

这里是为服务端配置frp 只关注frps和frps.ini即可，原始最简单配置为。

```
cat frps.ini
[common]
#隧道通道，服务器和客户端通过此端口通讯
bind_port = 7000
```

最简单也可以直接使用，先不配置其他测试使用先。

### **2：配置客户端frpc.ini文件**

只关注frpc和frpc.ini即可，修改frpc.ini。

```
vim frpc.ini

[common]
#bind_port：表示用于客户端和服务端连接的端口，这个端口号我们之后在配置客户端的时候要用到
bind_port = 7000

#dashboard_port：是服务端仪表板的端口，若使用7500端口，在配置完成服务启动后可以通过浏览器访问 x.x.x.x:7500 （其中x.x.x.x为公网服务器的IP）查看frp服务运行信息
dashboard_port = 7500
#token是：用于客户端和服务端连接的口令，请自行设置并记录，稍后会用到
token = 12345678
#dashboard_user、dashboard_pwd：表示打开仪表板页面登录的用户名和密码，自行设置即可
dashboard_user = admin
dashboard_pwd = 123456

#开启ssh连接配置
[ssh]
type = tcp
#本机IP
local_ip = 127.0.0.1
#本机需要映射的端口22
local_port = 22
#远程服务器映射的端口为6000
remote_port = 6000
```

## **3：分别启动服务器端和客户端**

**注：服务器，如有防火墙，请开启7000端口和有需要的端口。**

服务器运行启动：

```
./frps -c frps.ini
```

客户端运行启动：

```
./frpc -c frpc.ini
```

可以看到提示，都已经启动成功

测试ssh连接，这里用第三方工具xshell测试。
连接IP 为公网IP地址，端口为6000端口。

连接登录，即可登录到内网的192.168.152.103机器。

最简单的ssh端口映射就完成了。

## **四：后台启动**

服务端：

```
nohup  ./frps -c frps.ini > /var/log/frp.log 2>&1  &
```

客户端:

```
nohup ./frpc -c frpc.ini > /opt/frp/frp.log 2>&1 &
```

到此，frp服务器搭建完成。

## **五、配置后台启动服务**

## **linux**

### **1.创建service文件**

`touch /etc/systemd/system/frp.service`

### **2.编辑service文件**

打开service文件

💡 `vi /etc/systemd/system/frp.service`

填入以下内容**(注意修改frp文件夹路径)**

```
[Unit]
Description=frp daemon

[Service]
Type=simple
User=root
ExecStart=/opt/frp/frp_0.44.0_linux_amd64/frpc -c /opt/frp/frp_0.44.0_linux_amd64/frpc.ini
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

保存并退出

### **3.重新加载systemctl daemon**

💡 `systemctl daemon-reload`

### **4.启动frp**

💡 `systemctl start frp.service`

### **5.设置frp开机自启动**

💡 `systemctl enable frp.service`

### **以下为frp相关的管理命令**

### **启动frp**

💡`systemctl start frp.service`

### **重启frp**

💡`systemctl restart frp.service`

## **查看frp运行状态**

💡`systemctl status frp.service`

## **查询日志**

### **查看指定服务日志**

`journalctl -u 服务名`

查看指定日期日志

`journalctl --since="2021-10-10 10:10:00" --until="2021-10-11 10:10:00" -u 服务名`
类似tail -f  `journalctl -f -n 20 -u 服务名-n 查看尾部多少行-f 滚动形式`
查看日志占用的磁盘空间 `journalctl --disk-usage`

---

## **windows**

因为要设置成开机自启动，所以可以需要将frp客户端设置后台运行
这里需要借助另外一个软件nssm，附上nssm的下载地址:https://nssm.cc/release/nssm-2.24.zip
下载完后用管理员身份打开Powershell窗口,然后cd进nssm的目录后运行如下命令：

```
./nssm.exe install frp
```

运行后如下：

![](https://img-blog.csdnimg.cn/0b68069c3b06444cb035ec983f60195a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA56CB5byf55qE5Y2a5a6i,size_17,color_FFFFFF,t_70,g_se,x_16)

Path选择frpc.exe,然后Arguments选择-c frpc.ini
完成后，可以在windows中搜索服务:

![](https://img-blog.csdnimg.cn/0add30c51a1e4ee49008d81aeae38030.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA56CB5byf55qE5Y2a5a6i,size_20,color_FFFFFF,t_70,g_se,x_16)

如果frp没有启动进行启动，然后检查是不是自动

启动后可以通过检查端口监听状态看是否正常，可以使用如下命令:

```
netstat -an | findstr frp服务器的端口
```

到这一步已经可以正常使用，进行远程连接。

# **window配置远程桌面连接**

详见 https://blog.csdn.net/qq_37207266/article/details/122717327
