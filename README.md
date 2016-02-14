#### pkcenthiperf
- cp1
config 2-machine cluster. using virtualbox  
1:Bridged Adapter  
2:Internal Adapter  

with static IP addresses and host names:  
node01 [192.168.0.2] and node02 [192.168.0.3]  
gateway router call gateway [192.168.0.1].

Go to
```
/etc/sysconfig/network-scripts
```
identify card names
ifcfg-enp0sX

find the address
```
HWADDR="XX:XX:...."
TYPE="Ethernet"
BOOTPROTO="static"
NAME="enp0s3"
ONBOOT="yes"
IPADDR="192.168.0.2"
NETMASK="255.255.255.0"
GATEWAY="192.168.0.1"
PEERDNS="yes"
DNS1="8.8.8.8"
DNS2="8.8.4.4"
```

restart
```
systemctl restart network.service
```

check whether is updated
```
ip addr | grep 'inet' 
```



Edit /etc/hosts with the following content:
```
192.168.0.2	node01
192.168.0.3	node02
192.168.0.1	gateway
```
verify
```
ip link show enp0s8
```



ping nodes:
```
ping node01/node02
```

==

install imperative software:
```
yum install pacemaker corosync pcs
```


using iptables:
```
yum install iptables-services
systemctl stop firewalld.service
systemctl disable firewalld.service
systemctl enable iptables.service
systemctl start iptables.service
```


sethome from node01->node02
```
hostnamectl set-hostname node02
systemctl reboot
```
connect two machines using ssh
```
ssh-keygen -t rsa
```

connect from node01 to node02
```
cat .ssh/id_rsa.pub | ssh root@node02 'cat >> .ssh/authorized_keys'
```

connect from node01 to node01
```
cat .ssh/id_rsa.pub | ssh root@node01 'cat >> .ssh/authorized_keys'
```

connect:
```
ssh node02
ssh node01
```

(if not successful, enable sshd)
```
systemctl start sshd
```



- cp2
create a corosync file:
```
cp /etc/corosync/corosync.conf.example /etc/corosync/corosync.conf
```
```
systemctl enable corosync
systemctl enable pacemaker
systemctl start pacemaker
```
Perform the following operations on both nodes:
```
systemctl enable pacemaker
ln -s '/usr/lib/systemd/system/pacemaker.service' '/etc/systemd/system/multi-user.target.wants/pacemaker.service'
systemctl enable corosync
ln -s '/usr/lib/systemd/system/corosync.service' '/etc/systemd/system/multi-user.target.wants/corosync.service'
systemctl start pcsd
systemctl enable pcsd
```
Now set the password for the hacluster Linux account, which was created automatically when PCS was installed.
```
passwd hacluster
```
troubleshooting:
```
journalctl -xn
```

ckeck exists of corosync

```
yum –y install net-tools && netstat -npltu | grep -i corosync
```





- cp5
elrepo
```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
```
verify
```
yum repo list | grep elrepo
```
install modules:
```
yum update && yum install drbd84-utils kmod-drbd84
lsmod | grep -i drbd
```

If it is not loaded automatically, you can load the module to the kernel on both nodes.
```
modprobe drbd
```
load during boot
```
echo drbd >/etc/modules-load.d/drbd.conf
```
