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




- cp3
```
pcs status
pcs cluster stop node01
```

check
```
grep virtual_ip /var/log/pacemaker.log
```

re-enable
```
pcs cluster start node01
pcs resource enable virtual_ip
```


debug
```
pcs resource debug-start virtual_ip --full
```


log file position
```
/var/log/pacemaker.log
/var/log/cluster/corosync.log
/var/log/pcsd/pcsd.log
```

enable stonith->fencing mulfunctioning node
```
pcs config // check stonith
```
enable properties syntax:
```
pcs property list //for brevity
pcs property set stonith-enabled=true
```

install fencing
```
yum install fence-agents-all fence-virt
pcs stonith list
```

get info of a fencing agent:
```
pcs stonith describe fence_ilo
pcs stonith create Stonith_1 fence_ilo pcmk_host_list="node01" action=reboot --force
```
syntax:
```
pcs stonith create stonith_device_name stonith_device_type stonith_device_options
pcs stonith update stonith_device_name stonith_device_options
pcs stonith delete stonith_device_name
```

simulate a node off:
```
pcs cluster stop node01 
pcs stonith fence node01 --off
```
enable virtual_ip
```
pcs resource enable virtual_ip
```

Add the following in ` /etc/corosync/corosync.conf`
```
quorum {
provider: corosync_votequorum
two_node: 1   //at least 1 to hold the quorum
}
```











- cp4
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

- cp5
```
pcs status nodes pacemaker | corosync | both
```

```
pcs resource show virtual_ip  //or --full
```
to show options
```
pcs constraint --full
```




- cp6
```
mysqlshow dbname tablename columnname -h 192.168.0.1 -u root -p
```

list the complete set of utilities
```
ls /bin | grep mysql
```

mysqlslap (mysql pressure test)
```
--create-schema: This command specifies the database in which we will run the tests
--query: This is a string (or alternatively, a file) containing the SELECT statements used to retrieve data
--delimiter: This command allows you to specify a delimiter to separate multiple queries in the same string in --query
--concurrency: This command is the number of simultaneous connections to simulate
--iterations: This is the number of times to run the tests
--number-of-queries: This command limits each client (refer to --concurrency) to that amount of queries
```
Ex: simulate 10 concurrent connections and make 50 queries overall. This will result in clients running 5 queries each (50/10 = 5):
```
mysqlslap --create-schema=dbname --query="SELECT * FROM tablename" --concurrency=10 --iterations=2 --number-of-queries=50 -h 192.168.0.4 -u root -p
```
after execution:
```
reset query cache;
```

mimic a failover
```
pcs resource defaults resource-stickiness=value
```

<??>
Resource agents are found inside 
```
/usr/lib/ocf/resource.d
```
```
pcs resource agents standard:provider
```


list all httpd modules.
```
httpd -M
```

configure apache:
```
vi /etc/httpd/conf/httpd.conf
```
```
IncludeOptional conf.modules.d/*.conf
```
Disable the userdir module
```
#LoadModule userdir_module modules/mod_userdir.so
```
Restart the cluster resource 
```
pcs resource restart webserver
```

query cache type change:
```
SHOW VARIABLES LIKE 'query_cache_type';
SHOW VARIABLES LIKE 'query_cache_size';
```
set cache size.(Setting it too high will result in performance degradation as the system will have to allocate extra resources to manage a large cache. )  
```
SET GLOBAL query_cache_size = 102400; //100kb
```

