# ssr-libev
libev port of ShadowsocksR

# 编译安装 shadowsocksr-libev

## 安装开发环境

```bash
sudo apt-get install git build-essential cmake -y
```

一般的 ubuntu 和 debian 系统都可以做到，但是我之前用的 ubuntu mate 会报错，找不到 build-essential，那么使用 aptitude 来解决依赖项

```bash
sudo apt-get install aptitude -y
sudo aptitude install build-essential -y
```

## 安装依赖项

这里列举下一般容易缺少的依赖项
libssl libsodium libpcre
这些依赖项可以去选择手动编译，也可以直接

```bash
sudo apt-get install -y libsodium-dev libpcre3 libpcre3-dev libssl-dev
```

开始编译 ssr

```bash
git clone https://github.com/shadowsocksr-backup/shadowsocksr-libev.git
cd shadowsocksr-libev
./configure --prefix=/usr/local/shadowsocksR --disable-documentation
make -j4
sudo make install
```

## 使用 ssr

```bash
sudo mkdir /usr/local/shadowsocksR/conf
sudo vi config.json
```

加入以下内容，并根据自己的服务器进行正确修改

```json
{ 
"server":"your server address here",
"server_port":1234,
"local_port":1080,
"password":"your password here", 
"timeout":600,
"method":"chacha20", 
"protocol":"auth_aes128_md5", 
"obfs":"tls1.2_ticket_auth", 
"obfsparam":"" ,
"group":"any you like",
"local_address":"0.0.0.0", 
}
```

local_address写作0.0.0.0而不是127.0.0.1，是为了方便所有局域网内的机器使用（这个问题我以前已经讨论过了），但是在switch-omega中只能写作127.0.0.1，否则会连接失败（具体原因不详），wget、curl或者bash proxy倒是可以随便写。

## 启动 ssr

```bash
/usr/local/shadowsocksR/bin/ss-local -c /usr/local/shadowsocksR/conf/config.json
```

可以配置 supervisor 守护进程

## 完善

### 利用 ss-redir 全局透明代理

```bash
cd
# 获取大陆ip地址段
curl -sL http://f.ip.cn/rt/chnroutes.txt | egrep -v '^$|^#' > chnroutes
# ipset
sudo apt-get -y install ipset
sudo ipset -N chnroutes hash:net
for i in `cat chnroutes`; do echo ipset -A chnroutes $i >> ipset.sh; done
chmod +x ipset.sh && ./ipset.sh
# 持久化 ipset 地址列表至文件
sudo ipset save chnroutes > ~/chnroutes.ipset
# 从文件恢复到 ipset
sudo ipset destroy chnroutes
sudo ipset restore < ~/chnroutes.ipset
# 新建一条链 shadowsocks
sudo iptables -t nat -N shadowsocks
# 保留地址、私有地址、回环地址 不走代理
sudo iptables -t nat -A shadowsocks -d 0.0.0.0/8 -j RETURN
sudo iptables -t nat -A shadowsocks -d 10.0.0.0/8 -j RETURN
sudo iptables -t nat -A shadowsocks -d 127.0.0.0/8 -j RETURN
sudo iptables -t nat -A shadowsocks -d 169.254.0.0/16 -j RETURN
sudo iptables -t nat -A shadowsocks -d 172.16.0.0/12 -j RETURN
sudo iptables -t nat -A shadowsocks -d 192.168.0.0/16 -j RETURN
sudo iptables -t nat -A shadowsocks -d 224.0.0.0/4 -j RETURN
sudo iptables -t nat -A shadowsocks -d 240.0.0.0/4 -j RETURN
# 1234 是 ss 代理服务器的端口，即远程 shadowsocks 服务器提供服务的端口
# 如果你有多个 ip 可用,但端口一致，就设置这个
sudo iptables -t nat -A shadowsocks -p tcp --dport 1234 -j RETURN
# your_server 是 ss 代理服务器的 ip
# 如果你只有一个 ss 服务器的 ip，却能选择不同端口,就设置此条
sudo iptables -t nat -A shadowsocks -d your_server -j RETURN
# 大陆地址不走代理
sudo iptables -t nat -A shadowsocks -m set --match-set chnroutes dst -j RETURN
# 其余的全部重定向至 ss-redir 监听端口
sudo iptables -t nat -A shadowsocks -p tcp -j REDIRECT --to-ports 1080
# OUTPUT 和 PREROUTING 链添加一条规则，重定向至 shadowsocks 链
sudo iptables -t nat -A OUTPUT -p tcp -j shadowsocks
sudo iptables -t nat -I PREROUTING -p tcp -j shadowsocks
# 持久化 iptables 规则到文件
sudo iptables-save > ~/iptables.shadowsocks
# 从文件恢复到 iptables
sudo iptables-restore < ~/iptables.shadowsocks
```



## 参考博客

精品博客：http://blog.nicesite.win/2017/08/24/ssr-install/

https://shadowsocks.be/9.html

https://0066.in/archives/category/fcn

https://www.librehat.com/about-shadowsocks-r-and-the-security-of-shadowsocks/

http://xiaocrab.net/posts/1b8b6df6/

http://neyhsu.blogspot.sg/2017/06/centosdebianubuntu-shadowsocksr.html

https://blog.xingoxu.com/2017/08/cross-compile-55-libev/

http://www.bigf.info/post-201.html

http://govfate.iteye.com/blog/2086475 端口流量监听

http://www.jb51.net/article/58768.htm

nethogs 进程流量监听
