---
tags:
  - blog
  - 网络配置
categories:
  - 生活
title: MT-Photos
date: 2024-03-31T02:00:33.349Z
lastmod: 2024-09-18T02:38:15.000Z
---
#### FPR 内网穿透

需要内网的机器的强大的CPU 给AI识别提供助力\
借用了腾讯云内网穿透来联通

1. 腾讯云服务器运行的为**服务端**，
2. 局域网内机子运行的为**客户端**
3. NAS机器称为**访问端**

##### 一 下载 frp包

按推荐linux下载了 0.37或者0.38 试验下来使用稳定

```shell
wget https://github.com/fatedier/frp/releases/download/v0.38.0/frp_0.38.0_linux_amd64.tar.gz
```

##### 二  服务端安装

1. 执行`tar -zxvf frp_0.38.0_linux_amd64.tar.gz`解压
2. 执行 `cd frp_0.38.0_linux_amd64/`进入文件夹
3. 修改配置文件[#三 服务端配置](#%E4%B8%89%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E9%85%8D%E7%BD%AE)
4. sudo chmod 755 ./frps 赋予
5. 启动运行  在安装好的目录内  执行`./frps -c ./frps.ini`前台启动命令

*后期可以ctrl+c 终止程序，再执行`nohup ./frps -c ./frps.ini &` 后台保持启动*\
\_后期还可以设置自动运行\
\#TODO：

##### 三 服务端配置

```python
[common]
bind_addr=0.0.0.0
bind_port = 7000
token=12345678

dashboard_port = 7500
dashboard_user = admin
dashboard_pwd = admin123
```

> \[!NOTE] 配置说明：\
> "bind\_addr"是服务器本地IP，不改。\
> "bind\_port"是frp监听端口。\
> "token"是验证token建议设置上。\
> "dashboard\_port"是frp面板端口。\
> "dashboard\_user""dashboard\_pwd"是面板的账户密码。

token 建议复杂一些，屏蔽一些扫描访问

##### 四 访问frp控制面板

面板仅供参考，可用可不用。访问 **http://服务器ip:7500**

上面配置的7500端口，使用上面配置的用户名和密码 admin/admin123

##### 五 客户端配置修改

客户端与服务端下载同样的文件 frp\_0.38.0\_linux\_amd64.tar.gz\
创建文件夹，执行`tar -zxvf frp_0.38.0_linux_amd64.tar.gz`解压 然后进入，\
编辑客户端配置"frpc.ini"文件

```
[common]
server_addr = 公网IP 腾讯服务器的地址
server_port = 7000  上面服务端配置的端口
token=1235678    相当于密码

[RDP]
type = tcp
local_ip = 127.0.0.1   本地回环地址
local_port = 8060         局域网客户端的端口
remote_port = 6000        服务端的端口，用于给访问端使用
```

> \[!NOTE] 说明：\
> 客户端连接腾讯云的地址  公网IP  端口7000  token\
> 访问端连接腾讯云的公网IP 6000  ，会frps转到  客户端的8060端口

##### 六 客户端连接服务端

执行`./frpc -c ./frpc.ini`前台启动命令 连接服务端

*后期可以ctrl+c 终止程序，再执行`nohup ./frpc -c ./frpc.ini &` 后台保持启动*

##### 七 MT Photo AI

跑 ai的机器使用的ubuntu系统

1. 安装docker
2. 下载镜像

```shell
sudo docker pull mtphotos/mt-photos-ai
```

3. 运行脚本

```shell
#API_AUTH_KEY自行设置

sudo docker run -i -p 8060:8000 -e API_AUTH_KEY=mt_photos_ai_extra_secret --name mt-photos-ai2 mtphotos/mt-photos-ai:latest
```

会监听 8060端口\
可以使用下面命令来看看通不通

```shell
curl --location --request POST 'http://127.0.0.1:8060/check' \
--header 'api-key: mt_photos_ai_extra_secret'
```

##### 八 跑通整个链路

MT Photo 调用 公网IP:6000 端口 api-key\
FRPS 会去调用 客户端的 8060端口

##### 九  是否需要开机启动

目前使用的功能是一个不常用的功能，FRPS连接没有识别来源所以计划随用隋开，不自动启动。

###### 9.1 设置云服务器端开机自启动

1. 执行sudo vim /etc/systemd/system/frps.service创建服务，编辑如下

```shell
[Unit]
Description=frps daemon
After=syslog.target  network.target
Wants=network.target

[Service]
Type=simple
ExecStart=/etc/frps/frp_0.38.0_linux_amd64/frps -c /etc/frps/frp_0.38.0_linux_amd64/frps.ini		# 编辑的时候一定要删除注释 这里更改为自己安装frps的绝对路径
Restart= always
RestartSec=1min
[Install]
WantedBy=multi-user.target
```

2. 开启自启动\
   执行如下指令

```bash
#启动frps
systemctl daemon-reload
systemctl start frps
#设置为开机启动
systemctl enable frps
```

9.2 设置局域网机子frpc开机自启动

1. 执行sudo vim /etc/systemd/system/frpc.service创建服务，编辑如下

```shell
[Unit]
Description=frpc daemon
After=syslog.target  network.target
Wants=network.target

[Service]
Type=simple
ExecStart=/home/wentao/frps/frp_0.37.0_linux_amd64/frpc -c /home/wentao/frps/frp_0.37.0_linux_amd64/frpc.ini		
# 编辑的时候一定要删除注释 这里更改为自己安装frpc的绝对路径
Restart= always
RestartSec=1min
[Install]
WantedBy=multi-user.target
```

2. 开启自启动\
   执行如下指令

```shell
#启动frpc
sudo systemctl daemon-reload
sudo systemctl start frpc
#设置为开机启动
sudo systemctl enable frpc
```

##### 十、复杂配置

**以下配置中密码和公网IP都使用XX进行了代替，复制使用时请注意修改**

1. 远程服务端frps.ini

```python
[common]
# frp监听的端口，默认是7000
bind_port = 7000
# 授权码,请自定义更改
token = XXXXX
# 这个token之后在客户端会用到

# frp管理后台端口，请按自己需求更改
dashboard_port = 7500
# frp管理后台用户名和密码，请改成自己的
dashboard_user = admin
dashboard_pwd = XXXXX
enable_prometheus = true

# frp日志存放位置、日志等级及日志存储最大天数
log_file = /var/log/frps.log
log_level = info
log_max_days = 7

```

2. 客户端的frpc.ini文件

```python
[common]
# 远程服务器ip，远程frp服务端口以及远程登录密码
server_addr = 119.XX.XX.119
server_port = 7000
token = XXXXX

# 日志存放位置、日志等级及日志存储最大天数
log_file = /var/log/frpc.log
log_level = info
log_max_days = 7

# 将本地机的22端口映射至远程20022端口 中括号内[ssh]只作为标识，可自定义
[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 8060
remote_port = 6000

# 将本机的5901端口映射至远程25901端口
[vnc]
type = tcp
local_ip = 127.0.0.1
local_port = 8000
remote_port = 6001

# 将本机所在局域网内的192.168.0.129机子的5901端口映射至远程55901端口
[tsj-vnc]
type = tcp
local_ip = 192.168.0.129
local_port = 5901
remote_port = 55901
# 代理本机所在的内网，即若用户设置代理为公网IP+端口即可访问本机所在内网，使用的是socks5代理协议，plugin_user及后面的可设置可不设，但不设的风险较大
[socks5_proxy] 
type = tcp
remote_port = 7890   
plugin = socks5
plugin = socks5
plugin_user = name
plugin_passwd = password
use_encryption = true
use_compression = true


```

3. 重启frp服务

> 服务端\
> sudo systemctl restart frps.service\
> 客户端\
> sudo systemctl restart frpc.service

#### Docker 配置

```yaml
version: "3"

services:
  mtphotos:
    image: mtphotos/mt-photos:1.30.2
    container_name: mt-photos
    restart: always
    ports:
      - 8063:8063
    volumes:
      - /share/Container/mt_photos:/config
      - /share/Public/mt_photos_upload:/upload
      - /share/HomePhoto/Tao:/photos/Tao
      - /share/HomePhoto/Zhu:/photos/Zhu
      - /share/HomePhoto/Chao:/photos/Chao
      - /share/HomePhoto/Juan:/photos/Juan
      - /share/Media:/Multimedia
      - /share/Media_Old_Family:/Multimedia_Old
      - /share/Media_Small_Family:/Multimedia_Small
    environment:
      - TZ=Asia/Shanghai
      - PUID=1000
      - PGID=100
      - SCAN_INTERVAL=60
    depends_on:
      - mtphotos_ai
      - mt-photos-deepface
  mtphotos_ai:
    image: mtphotos/mt-photos-ai:onnx-latest
    container_name: mtphotos_ai
    restart: always
    ports:
      - 8060:8000
    environment:
      - API_AUTH_KEY=mt_photos_ai_extra_secret
  mt-photos-deepface:
    image: mtphotos/mt-photos-deepface:noavx-latest
    container_name: mt-photos-deepface
    restart: always
    ports:
      - 8066:8066
    environment:
      - API_AUTH_KEY=mt_photos_ai_extra
```
