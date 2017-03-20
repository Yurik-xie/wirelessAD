## 0x00 信息收集： 复制网站 ##
使用的工具是 httrack 网站复制机
### 安装 ### 

    apt-get install webhttrack

### 使用 ###

    httrack

## 0x01 攻击AP并且伪造AP ##
## 0x02 劫持DNS ## 
使用的工具是 ettercap 
 
ettercap是LINUX下一个强大的 欺骗工具，当然WINDOWS也能用，你能够用飞一般的速度创建和发送伪造的包.让你发送从网络适配器到应用软件各种级别的包.绑定监听数据到一个本地端口:从一个客户端连接到这个端口并且能够为不知道的协议解码或者把数据插进去(只有在arp为基础模式里才能用) 
 
ettercap -G 打开我们的ettercap工具 
指定网卡 Sniff -> Unified sniffing -> Ok 
然后通过 Hosts->Scan for hosts 扫描一下局域网的机器,通过 Hosts->Hosts list 显示出来 
 
我们选中192.168.2.105 Add to Target 1 
选中网关 192.168.2.1 Add to Target 2 
 
选中 Mitm -> Arp poisoning -> 勾选Sniff remote connections 点ok 
选中 Plugins -> Manage the Plugins -> 双击 dns_spoof  
 
这样我们就完成了受害机器的dns劫持过程,我们修改一下,使其无论打开什么网址,其实打开的都是 192.168.1.104 

    vim /etc/ettercap/etter.dns 
 

 

