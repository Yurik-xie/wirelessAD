要搭建钓鱼Wifi很简单，必备的6个东西：

1、无线网卡，这里我用的是拓石N87网卡

2、KaliLinux操作系统，这里就不用说了，必备的

3、isc-dhcp-server服务器。安装好KaliLinux后只需要apt-get update 然后apt-get install isc-dhcp-server即可

4、Aircrack-ng套件 #用来发送数据

5、sslstrip 用来突破SSL加密

6、ettercap 用来嗅探劫持

后面三个软件KaliLinux都自带有，不用安装即可。

首先强调下，后面的bash脚本适用于使用isc-dhcp-server这个bash脚本，建立钓鱼热点。

安装dhcp服务

apt-get install isc-dhcp-server

配置文件分别在/etc/default/isc-dhcp-server和/etc/dhcp/dhcpd.conf，前者可以配置监听端口，这里以wlan0为例

配置dhcp文件后，断开wlan0的网络，分配一个ip

ifconfig wlan0 192.168.1.2/24

启动dhcp服务

/etc/init.d/isc-dhcp-server start 或者

service isc-dhcp-server start

建立热点：

将下文写好的airssl.sh添加执行权限

bash airssl.sh

然后分别是AP建立，DHCP建立，sslstrip开启，ettercap开启。
