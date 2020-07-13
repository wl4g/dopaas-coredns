## 二次开发coredns-redisc

#### 1，下载项目
首先git clone https://github.com/coredns/coredns （运行外部插件需先下载coredns主库项目）

#### 2，配置插件
修改配置文件coredns/plugin.cfg，如，在forward插件上一行添加我们的插件(注：因coredns使用的caddy会按照插件配置顺序决定执行顺序)，这里放到forward插件前的好处是，可以控制dns向上递归解析查询。

```
vim coredns/plugin.cfg
...
#开发环境建议直接使用本地目录名coredns-redisc即可，无需使用github.com/wl4g/coredns-redisc地址。
coredns-redisc:coredns-redisc
#coredns-redisc:github.com/wl4g/coredns-redisc
forward:forward
...
```

#### 3，编译（合并插件）
在执行make之前，可以修改Makefile来修改配置实现交叉编译，如：

```
在SYSTEM:=后面追加"GOOS=linux GOARCH=amd64",则生成的是linux系统的二进制文件:
SYSTEM:=GOOS=linux GOARCH=amd64
SYSTEM:=GOOS=windows GOARCH=amd64
SYSTEM:=GOOS=darwin GOARCH=amd64
```

#### 4，配置文件Corefile

更多配置项可参考coredns官网查看，如 我们给出常规示例：

```
.:53 {
    # Load zones records from local /etc/hosts.
    hosts {
        fallthrough
    }
    # Load zones records from redis-cluster.
    coredns-redisc {
        address localhost:6379,localhost:6380,localhost:6381,localhost:7379,localhost:7380,localhost:7381
        password "123456"
        connect_timeout 30000
        read_timeout 30000
        ttl 360
        prefix _dns:
    }
    # Up recursive DNS query server list.
    # e.g. Google dns servers: 8.8.8.8，china telecom dns servers: 114.114.114.114,202.96.134.133,202.96.212.68
    forward . 8.8.8.8 114.114.114.114
    log
}
```

#### 5，启动运行

如果一切正常，编译后会在coredns/目录下生成coredns执行文件，启动运行：

```
./coredns -conf Corefile
```

#### 6，测试运行

添加测试数据：
```
redis-cli> hset example.net. me "{\"a\":[{\"ttl\":300, \"ip\":\"10.0.0.166\"}]}"
```

dns客户端查询测试：
```
dig me.example.net


; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> me.example.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2609
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;me.example.net.                      IN      A

;; ANSWER SECTION:
me.example.net.               600     IN      A       10.0.0.166

;; Query time: 2664 msec
;; SERVER: 100.100.2.138#53(100.100.2.138)
;; WHEN: Mon Jul 13 12:58:47 CST 2020
;; MSG SIZE  rcvd: 53
```