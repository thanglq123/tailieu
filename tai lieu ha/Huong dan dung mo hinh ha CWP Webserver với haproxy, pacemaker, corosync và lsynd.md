# Hướng dẫn dựng mô hình ha CWP Webserver với haproxy, pacemaker, corosync và lsynd

## Mục lục

[1. Mô hình](#1)

[2. Hướng dẫn cấu hình](#2)

- [2.1 Cấu hình môi trường](#2.1)

- [2.2 Cài đặt CWP](#2.2)

- [2.3 Cấu hình Mysql cluster](#2.3)

- [2.4 Cấu hình CWP trên 2 node còn lại](#2.4)

- [2.5 Cấu hình haproxy + pacemaker + corosync](#2.5)

- [2.6 Cài đặt và cấu hình lsyncd & rsync](#2.6)

[3. Một số lỗi thường gặp và cách khắc phục](#3)


-----------------

<a name="1"></a>
## 1. Mô hình

<img src="https://i.imgur.com/6g7ZGi5.png">

Môi trường lab: KVM

Phiên bản OS sử dụng: CentOS 7

**IP Planning**

<img src="https://i.imgur.com/oQTcxoG.png">

<a name="2"></a>
## 2. Hướng dẫn cấu hình

<a name="2.1"></a>
### 2.1 Cấu hình môi trường

- Cài đặt ip theo ip plan

- Set hostname trên cả 3 máy

```
hostnamectl set-hostname web1
hostnamectl set-hostname web2
hostnamectl set-hostname web3
```

- Tắt selinux và firewalld trên cả 3 node

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
systemctl disable firewalld
systemctl stop firewalld
```

- Khai báo host trên cả 3 node

```
echo "192.168.22.21 web1" >> /etc/hosts
echo "192.168.22.22 web2" >> /etc/hosts
echo "192.168.22.23 web3" >> /etc/hosts
```

- Cài đặt ntp server cho cả 3 node (bỏ qua nếu có)

```
yum -y install chrony
sed -i 's/server 0.centos.pool.ntp.org iburst/ \
server 1.vn.pool.ntp.org iburst \
server 0.asia.pool.ntp.org iburst \
server 3.asia.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 1.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf
sed -i 's/server 2.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf
sed -i 's/server 3.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf
sed -i 's/#allow 192.168.0.0\/16/allow 192.168.100.0\/24/g' /etc/chrony.conf
systemctl enable chronyd.service
systemctl start chronyd.service
systemctl restart chronyd.service
chronyc sources
```


<a name="2.2"></a>
### 2.2 Cài đặt CWP

- Cài đặt CWP trên cả 3 node

```
cd /usr/local/src
wget http://centos-webpanel.com/cwp-el7-latest
sh cwp-el7-latest
```

- Tạo New Accounts & Share IP chọn Private IP: 192.168.22.21 (Web1)

<img src="https://i.imgur.com/ZEGKRbe.png">

- Copy cấu hình CWP sang 2 node còn lại

`rsync -ar /usr/local/cwpsrv/htdocs/ root@web2:/usr/local/cwpsrv/htdocs/`
`rsync -ar /usr/local/cwpsrv/htdocs/ root@web3:/usr/local/cwpsrv/htdocs/`

<a name="2.3"></a>
### 2.3 Cấu hình Mysql cluster

- Trong gói cài đặt CWP đã có cài đặt sẵn MariaDB Galera Cluster

  - Sao lưu file cấu hình của mariadb

  `cp /etc/my.cnf.d/server.cnf  /etc/my.cnf.d/server.cnf.orig`

  - Khởi động mysql trên node web1

  `systemctl start mariadb`
  
  - Tắt mysql trên node web2 & web3

  `systemctl stop mariadb`
  
  - Xem password root Mysql

 `cat /root/.my.cnf`

  - Đặt password cho MariaDB

  ```
  password_galera_root=bwV2q312aN8CA
  cat << EOF | mysql -uroot
  GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '$password_galera_root';FLUSH PRIVILEGES;
  GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY '$password_galera_root';FLUSH PRIVILEGES;
  GRANT ALL PRIVILEGES ON *.* TO 'root'@'ctl1' IDENTIFIED BY '$password_galera_root';FLUSH PRIVILEGES;
  GRANT ALL PRIVILEGES ON *.* TO 'root'@'127.0.0.1' IDENTIFIED BY '$password_galera_root';FLUSH PRIVILEGES;
  EOF
  ```

  - Sửa file cấu hình trên web1

  ```
  cat << EOF >> /etc/my.cnf.d/server.cnf
  [galera] 
  wsrep_on=ON
  wsrep_provider=/usr/lib64/galera/libgalera_smm.so
  binlog_format=row
  default_storage_engine=InnoDB
  innodb_autoinc_lock_mode=2
  #innodb_log_file_size=100M
  #innodb_file_per_table
  #innodb_flush_log_at_trx_commit=2
  # Allow server to accept connections on all interfaces.
  bind-address=192.168.22.21
  #add your node ips here
  wsrep_cluster_address="gcomm://192.168.22.21,192.168.22.22,192.168.22.23"
  #Cluster name
  wsrep_cluster_name="clustername"
  # this server ip, change for each server
  wsrep_node_address="192.168.22.21"
  # this server name, change for each server
  wsrep_node_name="web1"
  wsrep_sst_method=rsync
  #max_connections = 10240
  #max_allowed_packet = 16M
  #key_buffer = 16M
  #thread_stack = 192K
  #thread_cache_size = 8
  #innodb_buffer_pool_size = 64M
  EOF
  ```

  - Sửa file cấu hình trên web2

  ```
  cat << EOF >> /etc/my.cnf.d/server.cnf
  [galera]
  wsrep_on=ON
  wsrep_provider=/usr/lib64/galera/libgalera_smm.so
  binlog_format=row
  default_storage_engine=InnoDB
  innodb_autoinc_lock_mode=2
  #innodb_log_file_size=100M
  #innodb_file_per_table
  #innodb_flush_log_at_trx_commit=2
  # Allow server to accept connections on all interfaces.
  bind-address=192.168.22.22
  #add your node ips here
  wsrep_cluster_address="gcomm://192.168.22.21,192.168.22.22,192.168.22.23"
  #Cluster name
  wsrep_cluster_name="clustername"
  # this server ip, change for each server
  wsrep_node_address="192.168.22.22"
  # this server name, change for each server
  wsrep_node_name="web2"
  wsrep_sst_method=rsync
  #max_connections = 10240
  #max_allowed_packet = 16M
  #key_buffer = 16M
  #thread_stack = 192K
  #thread_cache_size = 8
  #innodb_buffer_pool_size = 64M
  EOF
  ```

  - Sửa file cấu hình trên web3

  ```
  cat << EOF >> /etc/my.cnf.d/server.cnf
  [galera]
  wsrep_on=ON
  wsrep_provider=/usr/lib64/galera/libgalera_smm.so
  binlog_format=row
  default_storage_engine=InnoDB
  innodb_autoinc_lock_mode=2
  #innodb_log_file_size=100M
  #innodb_file_per_table
  #innodb_flush_log_at_trx_commit=2
  # Allow server to accept connections on all interfaces.
  bind-address=192.168.22.23
  #add your node ips here
  wsrep_cluster_address="gcomm://192.168.22.21,192.168.22.22,192.168.22.23"
  #Cluster name
  wsrep_cluster_name="clustername"
  # this server ip, change for each server
  wsrep_node_address="192.168.22.23"
  # this server name, change for each server
  wsrep_node_name="web3"
  wsrep_sst_method=rsync
  #max_connections = 10240
  #max_allowed_packet = 16M
  #key_buffer = 16M
  #thread_stack = 192K
  #thread_cache_size = 8
  #innodb_buffer_pool_size = 64M
  EOF
  ```

  - Tắt mariadb trên node web1 đã bật

  `systemctl stop mariadb`

  - Tạo cluster, thực hiện trên web1

  `galera_new_cluster`

  - Bật mysql trên 2 node còn lại

  `systemctl start mariadb`
  

- Cấu hình để haproxy check dịch vụ mysql trên cả 3 node

	- Cài đặt xinetd
	
	`yum install -y xinetd`

  	- Tạo file cấu hình /usr/local/bin/clustercheck

```
#!/bin/bash
#
# Script to make a proxy (ie HAProxy) capable of monitoring Percona XtraDB Cluster nodes properly
#
# Author: Olaf van Zandwijk <olaf.vanzandwijk@nedap.com>
# Author: Raghavendra Prabhu <raghavendra.prabhu@percona.com>
#
# Documentation and download: https://github.com/olafz/percona-clustercheck
#
# Based on the original script from Unai Rodriguez
#

if [[ $1 == '-h' || $1 == '--help' ]];then
    echo "Usage: $0 <user> <pass> <available_when_donor=0|1> <log_file> <available_when_readonly=0|1> <defaults_extra_file>"
    exit
fi

# if the disabled file is present, return 503. This allows
# admins to manually remove a node from a cluster easily.
if [ -e "/var/tmp/clustercheck.disabled" ]; then
    # Shell return-code is 1
    echo -en "HTTP/1.1 503 Service Unavailable\r\n"
    echo -en "Content-Type: text/plain\r\n"
    echo -en "Connection: close\r\n"
    echo -en "Content-Length: 51\r\n"
    echo -en "\r\n"
    echo -en "Percona XtraDB Cluster Node is manually disabled.\r\n"
    sleep 0.1
    exit 1
fi

MYSQL_USERNAME="${1-clustercheckuser}"
MYSQL_PASSWORD="${2-clustercheckpassword!}"
AVAILABLE_WHEN_DONOR=${3:-0}
ERR_FILE="${4:-/dev/null}"
AVAILABLE_WHEN_READONLY=${5:-1}
DEFAULTS_EXTRA_FILE=${6:-/etc/my.cnf}

#Timeout exists for instances where mysqld may be hung
TIMEOUT=10

EXTRA_ARGS=""
if [[ -n "$MYSQL_USERNAME" ]]; then
    EXTRA_ARGS="$EXTRA_ARGS --user=${MYSQL_USERNAME}"
fi
if [[ -n "$MYSQL_PASSWORD" ]]; then
    EXTRA_ARGS="$EXTRA_ARGS --password=${MYSQL_PASSWORD}"
fi
if [[ -r $DEFAULTS_EXTRA_FILE ]];then
    MYSQL_CMDLINE="mysql --defaults-extra-file=$DEFAULTS_EXTRA_FILE -nNE --connect-timeout=$TIMEOUT \
                    ${EXTRA_ARGS}"
else
    MYSQL_CMDLINE="mysql -nNE --connect-timeout=$TIMEOUT ${EXTRA_ARGS}"
fi
#
# Perform the query to check the wsrep_local_state
#
WSREP_STATUS=$($MYSQL_CMDLINE -e "SHOW STATUS LIKE 'wsrep_local_state';" \
    2>${ERR_FILE} | tail -1 2>>${ERR_FILE})

if [[ "${WSREP_STATUS}" == "4" ]] || [[ "${WSREP_STATUS}" == "2" && ${AVAILABLE_WHEN_DONOR} == 1 ]]
then
    # Check only when set to 0 to avoid latency in response.
    if [[ $AVAILABLE_WHEN_READONLY -eq 0 ]];then
        READ_ONLY=$($MYSQL_CMDLINE -e "SHOW GLOBAL VARIABLES LIKE 'read_only';" \
                    2>${ERR_FILE} | tail -1 2>>${ERR_FILE})

        if [[ "${READ_ONLY}" == "ON" ]];then
            # Percona XtraDB Cluster node local state is 'Synced', but it is in
            # read-only mode. The variable AVAILABLE_WHEN_READONLY is set to 0.
            # => return HTTP 503
            # Shell return-code is 1
            echo -en "HTTP/1.1 503 Service Unavailable\r\n"
            echo -en "Content-Type: text/plain\r\n"
            echo -en "Connection: close\r\n"
            echo -en "Content-Length: 43\r\n"
            echo -en "\r\n"
            echo -en "Percona XtraDB Cluster Node is read-only.\r\n"
            sleep 0.1
            exit 1
        fi
    fi
    # Percona XtraDB Cluster node local state is 'Synced' => return HTTP 200
    # Shell return-code is 0
    echo -en "HTTP/1.1 200 OK\r\n"
    echo -en "Content-Type: text/plain\r\n"
    echo -en "Connection: close\r\n"
    echo -en "Content-Length: 40\r\n"
    echo -en "\r\n"
    echo -en "Percona XtraDB Cluster Node is synced.\r\n"
    sleep 0.1
    exit 0
else
    # Percona XtraDB Cluster node local state is not 'Synced' => return HTTP 503
    # Shell return-code is 1
    echo -en "HTTP/1.1 503 Service Unavailable\r\n"
    echo -en "Content-Type: text/plain\r\n"
    echo -en "Connection: close\r\n"
    echo -en "Content-Length: 44\r\n"
    echo -en "\r\n"
    echo -en "Percona XtraDB Cluster Node is not synced.\r\n"
    sleep 0.1
    exit 1
fi
```
  - Tạo file `/etc/xinetd.d/mysqlchk`

  ```
  service mysqlchk
  {
        disable = no
        flags = REUSE
        socket_type = stream
        port = 9200            	# This port used by xinetd for clustercheck
        wait = no
        user = nobody
        server = /usr/local/bin/clustercheck
        log_on_failure += USERID
        only_from = 0.0.0.0/0 	# Thay bằng Bind Address - 192.168.22.21/24
        per_source = UNLIMITED
  }
  ```

  - Thêm service

  `echo 'mysqlchk      9200/tcp    # MySQL check' >> /etc/services`

  - Tạo user cho mysql và phân quyền, thực hiện trên node web1

  ```
  mysql> GRANT PROCESS ON *.* TO 'clustercheckuser'@'localhost' IDENTIFIED BY 'clustercheckpassword!';
  mysql> FLUSH PRIVILEGES;
  ```

  - Check lại script

  ```
  $ /usr/local/bin/clustercheck > /dev/null
  $ echo $?
  0
  ```

  Nếu ok, kết quả trả về sẽ là `0`

  - Bật xinetd

  ```
  systemctl start xinetd
  systemctl enable xinetd
  ```

<a name="2.4"></a>
### 2.4 Cấu hình CWP trên 2 node còn lại

- Đăng nhập -> Apache Settings -> Select WebServers -> Chọn Apache & Nginx Reverse Proxy -> Save & Rebuild Configuration

- User Accounts -> List Accounts -> Edit Account
	- IP Address sửa thành Private IP của node đó (Ex: 192.168.22.22) 

- Apache Settings -> Rebuild Virtual Hosts



<a name="2.5"></a>
### 2.5 Cấu hình haproxy + pacemaker + corosync

- Thao tác trên cả 3 node
	- Tải packages

	`yum install pacemaker corosync haproxy pcs fence-agents-all resource-agents psmisc policycoreutils-python -y`

	- Cho phép VIP

	```
	echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
	```

	- Đặt password cho hacluster (đặt pass giống nhau trên cả 3 node)

	`passwd hacluster`

	- Enable và start dịch vụ 
	
	```
	systemctl enable pcsd.service pacemaker.service corosync.service haproxy.service
	systemctl start pcsd.service
	```

- Thực hiện xác thực trên 1 node bất kì (web1)

`pcs cluster auth web1 web2 web3`

- Set up cluster

```
pcs cluster setup --start --name web_cluster web1 web2 web3
```

- Enable cluster

`pcs cluster enable --all`

- Kiểm tra

`pcs status`

- Set properties

```
pcs property set no-quorum-policy=ignore
pcs property set stonith-enabled=false
pcs property set default-resource-stickiness="INFINITY"
```

- Tạo resource VIP và haproxy

```
pcs resource create VIP_Web ocf:heartbeat:IPaddr2 ip=103.54.250.xxx cidr_netmask=27 op monitor interval=30s
pcs resource create HAproxy systemd:haproxy op monitor interval=5s
```

- Thêm rule

```
pcs constraint colocation add HAproxy Virtual_IP INFINITY
pcs constraint order Virtual_IP then HAproxy
```

- Cấu hình cho file `/etc/haproxy/haproxy.cnf` trên cả 3 node

```
global
    daemon
    group  haproxy
    log  127.0.0.1 local0 # Change to 127.0.0.1 use rsyslog
    log /dev/log    local1 notice
    maxconn  16000
    pidfile  /var/run/haproxy.pid
    stats  socket /var/lib/haproxy/stats
    tune.bufsize  32768
    tune.maxrewrite  1024
    user  haproxy


defaults
    log  global
    maxconn  8000
    mode  http
    option  redispatch
    option  http-server-close
    option  splice-auto
    retries  3
    timeout  http-request 20s
    timeout  queue 1m
    timeout  connect 10s
    timeout  client 1m
    timeout  server 1m
    timeout  check 10s

listen stats
    bind 103.54.250.xxx:8080
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    
listen mysqld
    bind 103.54.250.xxx:3306
    balance  leastconn
    mode  tcp
    option  httpchk
    option  tcplog
    option  clitcpka
    option  srvtcpka
    timeout client  28801s
    timeout server  28801s
    server web1 192.168.22.21:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3
    server web2 192.168.22.22:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3 backup
    server web3 192.168.22.23:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3 backup

listen webnginx
    bind 103.54.250.xxx:80
    balance source
    mode http
    option  forwardfor
    option  httpchk
    option  httpclose
    option  httplog
    stick  on src
    stick-table  type ip size 200k expire 30m
    timeout  client 3h
    timeout  server 3h
    server web1 192.168.22.21:80  weight 1 check
    server web2 192.168.22.22:80  weight 1 check
    server web3 192.168.22.23:80  weight 1 check
```

- Khởi động lại lần lượt các server để nhận cấu hình. Lưu ý reset từng con một. Kiểm tra xem đã nhận ip vip chưa. Kiểm tra trang dashboard ở địa chỉ `http://103.54.250.xxx:8080/stats`

<a name="2.6"></a>
### 2.6 Cài đặt và cấu hình lsyncd & rsync (Đồng bộ file)

Lsyncd đồng bộ file qua giao thức rsync nên yêu cầu cần ssh qua lại được giữa các node

- Tạo ssh-keygen trên 3 node

	`ssh-keygen`

- Chép Public key từ node web1 sang web2 & web3 (Node web2 & web3 thao tác giống vậy)

	`ssh-copy-id root@web2`
	
	`ssh-copy-id root@web3`

- Cài đặt lsyncd

	`yum install -y lsyncd rsync`

- Edit file /etc/lsyncd.conf trên 3 node

```
settings{
        logfile = "/var/log/lsyncd.log",
        statusFile = "/var/log/lsyncd-status.log",
        maxDelays = 10
}
servers = { '192.168.22.22', '192.168.22.23' }
for _, server in ipairs( servers ) do
  sync {
    default.rsync,
    source = "/home/ha",
    target = server .. ":/home/ha",
    delay = 5,
    rsync    = {
        archive  = true,
        compress = true,
        perms    = true,
        owner    = true,
        update   = true,
        whole_file = true,
        checksum = true
     }
  }
end
```

- Start dịch vụ

`systemctl start lsyncd`


<a name="3"></a>
3. Một số lỗi thường gặp và cách khắc phục
-

****innodb lock table***

```
#!/bin/bash

cd /var/lib/mysql
mkdir bak
mv aria_log* bak/
cp -a bak/aria_log* .
mv ibdata1 bak/
mv ib_logfile* bak/
cp -a bak/ibdata1 ibdata1
cp -a bak/ib_logfile* .
rm -rf bak
```

****lsyncd làm mất file khi 1 node online trở lại***

Không nên để `systemctl enable lsyncd` (khởi động cùng hệ thống) vì khi 1 node chết và online trở lại thì node chết đó sẽ đồng bộ file của nó sang 2 node còn lại.

**Link tham khảo:**
