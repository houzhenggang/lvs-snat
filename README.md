## lvs-snat
##### 版本说明
0.  lvs-snat已经移植到lvs-dpdk，见https://github.com/lvsgate/lvs-dpdk
1.  patches.lbg-lvs-v2是我厂负载均衡项目目前使用的版本，针对alibaba lvs-v2的补丁。完整的代码在 https://github.com/jlijian3/LVS/tree/lvs_v2 ，阿里的代码 https://github.com/alibaba/LVS/tree/lvs_v2
2.  我们的入口负载均衡使用lvs-v2，出口网关有两种实现方案

    a) 使用iptables的SNAT，功能完善稳定，性能较差
    
    b) 基于lvs-v2开发的SNAT网关，类似iptables SNAT功能，性能非常好，性能相对iptables提升80%以上
    
3.  patches.lbg-lvs-v2/snat-gateway-lvs-v2.patch在阿里lvs-v2的fullnat基础上实现了支持多isp的snat网关
4.  patches.lbg-lvs-v3/lvs-v3-rps.patch在lvs-v3版本的基础上增加rps_framework，模拟万兆网卡flow director，以支持千兆网卡，因为lvs-v3需要fdir来支持并行化，完整的代码 https://github.com/jlijian3/LVS。



##### lvs-snat网关特性
1. 支持源ip、目的ip、出口网卡、下一跳网关匹配，规则优先级匹配按照网络地址掩码位数由大到小
2. 支持tcp、udp、icmp
3. 增加redirect next hop功能,用于特殊选路需求和链路故障切换
4. 性能非常好，根据lvs性能接近
5. 目前不支持ipv6
6. snat ip pool选择算法支持hash(sip),hash(sip,dip),hash(sip,dip,sport,dport)
7. 兼容lvs原有功能，可以作为网关单独部署，也可以负载均衡部署在同一台机器，跟vs/nat,vs/fullnat等转发模式一起使用
8. 请注意我们使用fwmark 1作为snat的开关，并不需要iptables配合使用

## 基于lvs-v2的snat网关安装方法
### 在alibaba 的lvs基础上打补丁
	git clone git@github.com:alibaba/LVS.git
 	cd LVS
 	git checkout lvs_v2
	patch -p1 < snat-gateway-lvs-v2.patch

	#如果不想打补丁直接使用我们的完整代码
	git clone https://github.com/jlijian3/LVS.git
	cd LVS
	git checkout lvs_v2

### 编译内核
	make -j16
	make modules_install
	make install
	init 6

### 安装keepalived ipvsadm
	cd LVS/tools/keepalived
	./configure --with-kernel-dir="/lib/modules/`uname -r`/build" --prefix=/usr/ --sysconfdir=/etc/
	make
	make install
	cd ../ipvsadm
	make
	make install

### ipvsadm配置方法
	#下列是一个多isp的配置例子，网关和网卡用来区分链路，前提是路由要设置好
	#添加fwmark为1的virtual service，开启snat网关服务，注意跟iptables没有关系
    ipvsadm -A -f 1 -s snat_sched
    
	#添加snat规则，来源网段192.168.40.0/24，下一跳网关是1.1.2.1，
	#出口网卡是eth1的包，snat ip 1.1.2.100-1.1.2.110，ip选择算法random，随机选择
    ipvsadm -K -f 1 -F 192.168.40.0/24 -W 1.1.2.1 --oif eth1 -U 1.1.2.100-1.1.2.110 -O random
    
    #添加snat规则，来源网段192.168.40.0/24，下一跳网关是1.1.3.1，
    #出口网卡是eth2的包，snat ip 1.1.3.100-1.1.3.110，ip选择算法sdh，哈希源ip目的ip
    ipvsadm -K -f 1 -F 192.168.40.0/24 -W 1.1.3.1 --oif eth2 -U 1.1.3.100-1.1.3.110 -O sdh
    
    #如果还有其他isp，类似的配置，前提是你要设置好各isp的路由，确定好出口网卡，这里不赘述
    
    #网关重定向的例子，把下一跳是1.1.2.1的包，改到1.1.3.1，ip选择算法sh是源地址hash
    #当1.1.2.1的链路故障，可以这样切换，不用删路由表
    ipvsadm -K -f 1 -F 192.168.50.0/24 -W 1.1.2.1 -N 1.1.3.1 -U 1.1.3.100-1.1.3.110 -O sh
    
    #不区分isp，忽略路由表，所有包都走一条链路，ip选择算法默认sdh
    #单isp的网关这样配就可以了
    ipvsadm -K -f 1 -F 192.168.0.0/16 -N 1.1.3.1 -U 1.1.3.100-1.1.3.110
    
    #把内外机器的默认网关指向lvs的内网ip
    
### keepalived配置方法
    #fwmark 1只是开关，跟iptables一毛钱关系都没有，请关闭iptables
    virtual_server fwmark 1 {
    #192.168.40.0/24的按照多isp路由表走，多isp靠oif或者gw来区分
    snat_rule {
	    	from 192.168.40.0/24
	    	gw 1.1.3.1
	    	oif eth2
	    	snat_ip 1.1.3.71-1.1.3.73
	    	algo random
  	}   

  	snat_rule {
	    from 192.168.40.0/24
	    gw 1.1.2.1
	    oif eth1
	    snat_ip 1.1.2.71-1.1.2.73
	    algo sdh
	}
    	
    #其他网段的全部都走1.1.2.1吧，这里就不要写oif和gw了，我们只限制来源ip
    snat_rule {
	    from 192.168.0.0/16
	    new_gw 1.1.2.1
	    snat_ip 1.1.2.71-1.1.2.73
	    algo sh
	}
	
	#注意，lvs-snat本身不做链路故障检测，它只是个4层转发的内核模块
	#故障检测需要你自己写脚本完成，比如ping不通，再更改为下面这个配置
	#链路故障恢复以后，还要恢复回原来的配置
	#1.1.2.1链路故障，切到1.1.3.1吧，当然snat ip也要变
	snat_rule {
	    from 192.168.40.0/24
	    gw 1.1.2.1
	    oif eth1
	    new_gw 1.1.3.1
	    snat_ip 1.1.3.71-1.1.3.73
	    algo sdh
	}
	}

## iptables做snat网关的方法
如果不想用lvs做网关，直接使用iptables即可，不用安装lvs-v2内核，随便一个2.6.32的内核就ok
### iptables SNAT配置方法
	iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth1 -j SNAT --to-source 1.1.2.100-1.1.2.102
	iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth2 -j SNAT --to-source 1.1.3.100-1.1.3.102 --persitent
	
	#-s匹配内网网段，-o匹配出口网卡，多isp，多个上行网卡就有用了
	#to-source可以是一个ip，也可以使连续的ip断，ip选择算法不是轮询，默认是hash(sip,dip)
	# --persitent表示ip选择算法是hash(sip)，就是一个内网ip固定一个出口ip
	
	#同样，内网机器默认网关指向iptables所在机器的内网ip


