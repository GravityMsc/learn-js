> 前一阵子被一个蜜汁bug困扰：Node.js代码能在服务器上跑起来，但从浏览器却无法访问服务器80端口。于是在本地玩了一周。今天突然想起来可能是防火墙的配置问题。

之前用的是iptables来管理的防火墙，后来发现CentOS7.0中已经用firewalld取代iptables了，于是与时俱进，停用了iptables。

```bash
systemctl stop iptables.service
```

然后来启动firewalld吧。

```bash
systemctl start firewalld.service
```

结果给我报了这个错误。

> Failed to start firewalld.service: Unit firewalld.service is masked. 

查了很久没找到解决办法，于是试着输入了下面这行命令，解决了。

```bash
systemctl unmask firewalld.service
```

启动firewalld.service

```bash
systemctl start firewalld.service
```

把80端口添加到防火墙开放端口中

```bash
firewall-cmd --premanent --zone=public --add-port=80/tcp
```

重启一遍firewalld服务使其生效

```bash
systemctl restart firewall.service
```

检查更改是否生效

```bash
firewall-cmd --zone=public --query-port=80/tcp
```

