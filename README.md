之前试过吧里以前帖子的流量转发啊，专线+酸酸乳啊，都不太好使，前前后后花了半年冤枉钱后 前几天发现了
https://github.com/Frizz925/gbf-proxy
就体会到了凌晨的acgp的速度(没在本土玩过 所以不知道是不是本土网速)
准备工作有：一条沪日iplc专线(挑与本土ping最低的，腾讯和阿里云也可以，选弹性公网ip的时候用加速那个选项)， 一个日本vps(挑与你的iplc ping最低的)
所以从我电脑到iplc服务器延时是12ms，iplc服务器到日本vps是30ms，日本vps到gbf机房是1.几ms
效果视频：https://www.bilibili.com/video/BV1zf4y1z7CB
搭建视频：https://www.bilibili.com/video/BV1oU4y1H79t
================================================
先搞vps部分
这里用的centos7
```
wget https://github.com/Frizz925/gbf-proxy/releases/download/v0.1.0/gbf-proxy-linux-amd64 -O gbf-proxy
```
下载go程序到本地，
```
chmod +x gbf-proxy
```
加上执行权限
```
yum install tmux
tmux new -s gbf-proxy
tmux a -t gbf-proxy
```
进入窗口后
```
./gbf-proxy local --host 0.0.0.0 --port 8088
```
在8088端口运行gbf-proxy,这儿关掉防火墙
然后ctrl+b 退出窗口 就可以断开链接了
还有可以用docker 启动多个gbf-proxy 再用nginx来做负载均衡，今天先不说了

================================================
iplc nat部分
如果你的iplc只是有流量转发功能
那就是在设置那边添加
目标ip(你日本vps的ip)
目标端口(日本机子上gbf-proxy的端口，也就是8088)
本地端口(iplc映射的端口，这儿一般服务商会给出一个范围用)

如果是一个有虚拟机的iplc
```
wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubiBackup/doubi/master/iptables-pf.sh && chmod +x iptables-pf.sh && bash iptables-pf.sh
```
一键设置iptable，选1安装完后 再运行选4添加规则
和上面一样 也会有目标ip 目标断开 本地端口 这些都一样 不过这儿会有一个本地ip 就是iplc给你分配你机子的内网ip，用ifconfig查 再选tcp+udp
然后这个情况下得本地端口因为还是内网的 需要映射给公网端口，会有个地方添加nat规则
=================================================
两个都准备好后就用chrome的switchyomega 设置
按照图1里那样设置
server就是你iplc服务商的ip地址，prot就是映射后的公网端口
(你也可以在弄日本vps后就直接在这设置
server是日本服务器ip
端口是8088 看看不用专线的效果)

然后图2 设置打钩的规则
就结束了


==================================================================================================================
补上docker 部署gbf—proxy 然后利用nginx进行负载均衡
在机子上先装上docker 我这边是centos7
```
yum install docker
systemctl enable docker # 开机自动启动docker
systemctl start docker # 启动docker
mkdir ~/gbf_proxy&& cd ~/gbf_proxy && mkdir app && mkdir script
cd ~/gbf_proxy/app/
wget https://github.com/Frizz925/gbf-proxy/releases/download/v0.1.0/gbf-proxy-linux-amd64 -O gbf-proxy
```
这个就是之前下得那个gbf-proxy 把他移进~/gbf_proxy/app里也行

建dockerfile
```
cd ~/gbf_proxy/
echo 'FROM golang
MAINTAINER  MING
WORKDIR /go/src/
COPY . .
EXPOSE 8088
CMD ["/bin/bash", "/go/src/script/build.sh"]' >Dockerfile
```
dockerfile里第一行FROM golang可以换成FROM apline 更轻量 不过会有问题 到时候要自己改

然后编辑build.sh
```
cd ~/gbf_proxy/script
echo '#!/usr/bin/env bash
cd /go/src/app/ && chmod +x gbf-proxy && ./gbf-proxy local --host 0.0.0.0 --port 8088' >build.sh
```

弄好之后build docker 镜像
```
cd ~/gbf_proxy
docker build -t gbf-proxy .
```

镜像弄好后 创建容器
```
docker run -p 8099:8088 --name gbf_proxy -d gbf-proxy
```
这个的8088是docker 容器里面的，8099是本地的，后面要用
这个时候可以多运行几次 8099换成8011 8022 或者8033 --name gbf_proxy 也换成gbf_proxy2 gbf_proxy3 gbf_proxy4

```
docker ps
```

就可以看到容器已经在运行了
然后弄nginx 根据自己系统下载
下好之后把/etc/nginx/nginx.conf 改成下面的

```
load_module /usr/lib64/nginx/modules/ngx_stream_module.so;
user  nginx;
worker_processes  1;

events {
    worker_connections  1024;
}

stream {

    upstream gbf {
        server localhost:8099 weight=1;     # ip:port
        server localhost:8011 weight=2;     # ip:port
        server localhost:8022 weight=3;
        server localhost:8033 weight=4;
    }

    server {
        listen 8088;
        proxy_pass gbf;
    }
}
```
保存退出后 nginx -s reload 就可以了
要想再改进可以去找内容改nginx.conf
中转就不用改了 因为nginx在监听8088，把之前开的那个gbf-proxy关掉就行了
