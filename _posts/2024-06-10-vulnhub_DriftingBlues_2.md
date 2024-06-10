---
layout:     post
title:      "vulnhub_DriftingBlues_2"
subtitle:   信息搜集/php反弹shell/nmap提权
date:       2024-6-10
author:     Robin
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Note
    - Linux
    - Information security
    - vulnhub
---
# 靶机 DriftingBlues：2

## 1. 获取靶机

[DriftingBlues: 2 ~ VulnHub](https://www.vulnhub.com/entry/driftingblues-2,634/)

![]({{ "assets/2024-06-10-18-44-55-image.png" | absolute_url }})

导入virtualbox，默认是桥接网络，会在子网获取ip，通过arp-scan进行探测找到靶机的mac对应ip

![]({{ "assets/2024-06-10-18-48-21-image.png" | absolute_url }})

## 2. 信息搜集

### 2.1 扫描存活端口

nmap -A -sV -T4 -p- 10.1.1.35

-A 启用高级扫描，操作系统检测、版本、路由

-sV 尝试访问端口获取返回信息

-T4 设置扫描速度为4

-p- 全端口扫描

![]({{ "assets/2024-06-10-18-50-39-image.png" | absolute_url }})

### 2.2 探测端口服务

#### 2.2.1 探测ftp端口21

根据扫描结果，ftp允许匿名登录，登录上去查看就一个图片文件，无其他有效信息

![]({{ "assets/2024-06-10-18-53-51-image.png" | absolute_url }})

#### 2.2.2 访问80端口web

访问web就一个静态页面，源文件也无有效信息

![]({{ "assets/2024-06-10-18-55-57-image.png" | absolute_url }})

#### 2.2.3 目录爆破80端口

用dirsearch进行目录爆破发现隐藏路径，以及特殊的重定向

![]({{ "assets/2024-06-10-18-57-52-image.png" | absolute_url }})

将重定向域名写入hosts，指向靶机ip

![]({{ "assets/2024-06-10-19-02-45-image.png" | absolute_url }})

重新通过域名进行dirsearch扫描，找到基于wordpress框架的管理界面可访问登录入口

![]({{ "assets/2024-06-10-19-07-23-image.png" | absolute_url }})

## 3. 渗透测试

### 3.1 用wpscan工具对wordpress框架进行测试

对博客路径进行扫描，成功枚举出一个用户名albert

```wpscan
wpscan --url http://driftingblues.box/blog/ -e
```

![]({{ "assets/2024-06-10-20-39-43-image.png" | absolute_url }})

![]({{ "assets/2024-06-10-20-40-18-image.png" | absolute_url }})

### 3.2 爆破获取的用户名密码，登录后台

再次用wpscan工具对用户枚举结果进行密码爆破

```
wpscan --url http://driftingblues.box/blog/ -e u --passwords /usr/share/wordlists/rockyou.txt
```

成功获取到密码：albert/scotland1

![]({{ "assets/2024-06-10-20-42-20-image.png" | absolute_url }})

成功登录到后台：

![]({{ "assets/2024-06-10-20-43-06-image.png" | absolute_url }})

### 3.3 将反弹shell马写入页面，get shell

找到当前激活的主题模板，尝试修改模板文件，以获得执行恶意代码的触发

![]({{ "assets/2024-06-10-20-47-36-image.png" | absolute_url }})

尝试修改404页面的php脚本

![]({{ "assets/2024-06-10-20-48-26-image.png" | absolute_url }})

在页面的php脚本部分放入kali自带的php resvershell脚本(/usr/share/webshells)

![]({{ "assets/2024-06-10-20-58-38-image.png" | absolute_url }})

修改脚本中的反弹配置

![]({{ "assets/2024-06-10-20-59-14-image.png" | absolute_url }})

访问一个博客解析过程中不存在的连接地址，成功获取反弹shell，用户为www-data

![]({{ "assets/2024-06-10-21-00-58-image.png" | absolute_url }})

### 3.4 通过shell继续搜集信息

在`/home`目录下找到一个用户的目录`freddie` ，在此用户目录的`.ssh`目录中找到了`id_rsa`登录密钥文件

![]({{ "assets/2024-06-10-21-11-54-image.png" | absolute_url }})

### 3.5 通过凭证登录获得flag1

将id_rsa复制到本机，用ssh登录指定密钥尝试成功

```
 ssh freddie@10.1.1.35 -i ./id_rrsa
```

![]({{ "assets/2024-06-10-21-14-43-image.png" | absolute_url }})

### 3.6 寻找提权路径,发现sudo可执行nmap

执行`sudo -l` 发现nmap被授权在无密码下可执行

![]({{ "assets/2024-06-10-21-19-38-image.png" | absolute_url }})

### 3.7 查询nmap提权方式，提权成功

通过`https://gtfobins.github.io/` 查询nmap提权的方式

 ![]({{ "assets/2024-06-10-21-20-37-image.png" | absolute_url }})

复制到本机执行后，成功提权

![]({{ "assets/2024-06-10-21-21-20-image.png" | absolute_url }})

在`root` 目录中找到`root.txt` 获得flag2，完成渗透

![]({{ "assets/2024-06-10-21-22-32-image.png" | absolute_url }})
