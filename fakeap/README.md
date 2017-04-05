## 利用 hostapd 建立 AP  ## 

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
 
其中auth_algs指定采用哪种认证算法，采用位域（bit fields）方式来制定，其中第一位表示开放系统认证（Open System Authentication, OSA），第二位表示共享密钥认证（Shared Key Authentication, SKA）。 

我这里设置alth_algs的值为1，表示只采用OSA；如果为3则两种认证方式都支持。如果采用认证，则设置auth_algs为1；否则把这行代码注释掉（在前面加#）。 
 
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

wpa：指定WPA类型，这是一个位域值（bit fields），第一位表示启用WPA，第二位表示启用WPA2。在我的配置中，无论设置成1、2或3，都可以正常连接；

wpa_passphrase：WPA/WPA2加密需要指定密钥，这个选项就是配置WPA/WPA2的密钥。注意wpa_passphrase要求8~63个字符。另外还可以通过配置wpa_psk来制定密钥，不过要设置一个256位的16进制密钥，不适合我们的需求； 

wpa_pairwise/rsn_pairwise：如果启用了WPA，需要指定wpa_pairwise；如果启用了WPA2，需要指定rsn_pairwise，或者采用wpa_pairwise的设定。 

经过这样一些配置，启动hostapd之后应该就可以按照自己的需求正常使用无线接入点功能了： 

    /usr/bin/hostapd  hostapd.conf 
    
        REF： 

        http://www.cnblogs.com/zhuwenger/archive/2011/03/11/1980294.html  
        http://forum.ubuntu.org.cn/viewtopic.php?t=421829  
 
## dhcp服务 ##

客户端连接上SoftAP后，要给他们分配IP地址。这里有一条最关键，就是AP的IP地址和Internet接口的IP地址不能在网一个段。比如，连接Internet网的网卡eth0，地址是10.10.10.138/24，则wlan的地址后面不能设置为同一网段，这里设置为192.168.0.1/24。

### 安装dhcp服务 ###

    apt-get install isc-dhcp-server

## 将AP的接口设置为该网段的网关 ##

    ifconfig wlan0 192.168.0.1 netmask 255.255.255.0
    route add -net 192.168.0.0 netmask 255.255.255.0 gw 192.168.1.1

### 启动DHCP服务 ###

    dhcpd -d -f -cf dhcpd.conf <INTERFACE>
    
## 使用iptables设置转发 ##

使用iptables设置转发，将用户连接SoftAP后的数据，转发到internet上，是非常关键的一步。

    iptables -t nat -A POSTROUTING --out-interface $FAKE_AP_INTERNET -j MASQUERADE    #$FAKE_AP_INTERNET是外网接口
    
注意，转发生效需要本机开启ip_forward功能。指令是：

    echo "1" > /proc/sys/net/ipv4/ip_forward
    
## 使用DNSmaq ##

### DNS服务器设置 ###

#### DNS 缓存设置 #### 

要在单台电脑上以守护进程方式启动`dnsmasq`做DNS缓存服务器，编辑`/etc/dnsmasq.conf`，添加监听地址： 

    listen-address=127.0.0.1
    
如果用此主机为局域网提供默认 DNS，请用为该主机绑定固定 IP 地址，设置： 

    listen-address=192.168.x.x
    
这种情况建议配置静态IP 

多个ip地址设置:

    listen-address=127.0.0.1,192.168.x.x 
    
#### DNS 地址文件 ####

在配置好dnsmasq后，你需要编辑`/etc/resolv.conf`让DHCP客户端首先将本地地址(localhost)加入 DNS 文件`/etc/resolv.conf`，然后再通过其他DNS服务器解析地址。配置好DHCP客户端后需要重新启动网络来使设置生效。 

一种选择是一个纯粹的`resolv.conf` 配置。要做到这一点，才使第一个域名服务器在`/etc/resolv.conf` 中指向localhost：

    /etc/resolv.conf
    nameserver 127.0.0.1
    # External nameservers
    ...
    
现在，DNS查询将首先解析dnsmasq，只检查外部的服务器如果DNSMasq无法解析查询.  不幸的是`dhcpcd`往往默认覆盖`/etc/resolv.conf`, 所以如果你使用DHCP，这是一个好主意来保护 `/etc/resolv.conf`,要做到这一点，追加 `nohook resolv.conf`到dhcpcd的配置文件：

    /etc/dhcpcd.conf
    ...
    nohook resolv.conf
 
 
### DHCP 服务器设置  ### 

dnsmasq默认关闭DHCP功能，如果该主机需要为局域网中的其他设备提供IP和路由，应该对dnsmasq配置文件`/etc/dnsmasq.conf`必要的配置如下：

    # Only listen to routers' LAN NIC.  Doing so opens up tcp/udp port 53 to
    # localhost and udp port 67 to world:
    interface=<LAN-NIC>

    # dnsmasq will open tcp/udp port 53 and udp port 67 to world to help with
    # dynamic interfaces (assigning dynamic ips). Dnsmasq will discard world
    # requests to them, but the paranoid might like to close them and let the 
    # kernel handle them:
    bind-interfaces

    # Dynamic range of IPs to make available to LAN pc
    dhcp-range=192.168.111.50,192.168.111.100,12h

    # If you’d like to have dnsmasq assign static IPs, bind the LAN computer's
    # NIC MAC address:
    dhcp-host=aa:bb:cc:dd:ee:ff,192.168.111.50
