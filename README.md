# 1. Keepalived

- Keepalived cung cấp khả năng tạo độ sẵn sàng cao (High Availability) cho hệ thống dịch vụ và khả năng cân bằng tải (Load Balancing) đơn giản. 
- Hoạt động: Keepalived sẽ gom nhóm các máy chủ dịch vụ nào tham gia cụm HA, khởi tạo một Virtual Server đại diện cho một nhóm thiết bị đó với một Virtual IP (VIP) và một địa chỉ MAC vật lý của máy chủ dịch vụ đang giữ Virtual IP đó. Vào mỗi thời điểm nhất định, chỉ có một server dịch vụ dùng địa chỉ MAC này tương ứng Virtual IP. 
- Các máy chủ dịch vụ sử dụng chung VIP phải liên lạc với nhau bằng địa chỉ multicast 224.0.0.18 bằng giao thức VRRP. Các máy chủ sẽ có độ ưu tiên (priority) trong khoảng từ 1 – 254, và máy chủ nào có độ ưu tiên cao nhất sẽ thành Master, các máy chủ còn lại sẽ thành các Slave/Backup, hoạt động ở chế độ chờ.


## configure

### Global definitions

``` sh
global_defs {
  notification_email {
    email
    email
  }
  notification_email_from email
  smtp_server host
  smtp_connect_timeout num
  lvs_id string
}
```

- `global_defs` : cho biết đây là block cấu hình của global def
- `notification_email` : email nhận được thông báo
- `notification_email_from` : email được sử dụng khi xử lí câu lệnh “MAIL FROM:” SMTP
- `smtp_server` : SMTP server dùng để gửi mail thông báo
- `smtp_connection_timeout` : timeout cho tiến trình xử lí SMTP


### Virtual server definitions

``` sh
virtual_server (@IP PORT)|(fwmark num) {
    delay_loop num
    lb_algo rr|wrr|lc|wlc|sh|dh|lblc
    lb_kind NAT|DR|TUN
    (nat_mask @IP)
    persistence_timeout num
    persistence_granularity @IP
    virtualhost string
    protocol TCP|UDP

    sorry_server @IP PORT

    real_server @IP PORT {
      weight num
      TCP_CHECK {
        connect_port num
        connect_timeout num
      }
    }
    real_server @IP PORT {
      weight num
      MISC_CHECK {
        misc_path /path_to_script/script.sh
        (or misc_path “/path_to_script/script.sh <arg_list>”)
      }
    }
    real_server @IP PORT {
        weight num
        HTTP_GET|SSL_GET {
            url { # You can add multiple url block
              path alphanum
              digest alphanum
            }
            connect_port num
            connect_timeout num
            nb_get_retry num
            delay_before_retry num
        }
    }
}
```

- `delay_loop` : số giây giữa các lần check
- `lb_algo` : load balancing algorithm (rr|wrr|lc|wlc…)
- `lb_kind` : method dùng để forwarding (NAT|DR|TUN)
- `persistence_timeout` : thời gian timeout cho các persistence connection
- `Virtualhost` : HTTP virtualhost để dùng cho  HTTP|SSL_GET
- `protocol` :  (TCP|UDP)
- `sorry_server` : server được add vào pool nếu mọi server đều bị down
- `Weight` : trọng số cho load balancing
- `TCP_CHECK` : check bằng tcp connect
- `MISC_CHECK` : check bằng user defined script
- `misc_path` : Đường dẫn tới script
- `HTTP_GET` : check bằng HTTP GET request
- `SSL_GET` : check bằng SSL GET request

### VRRP Instance definitions

``` sh
vrrp_sync_group string {
  group {
    string
    string
  }
  notify_master /path_to_script/script_master.sh
      (or notify_master “/path_to_script/script_master.sh <arg_list>”)
  notify_backup /path_to_script/script_backup.sh
      (or notify_backup “/path_to_script/script_backup.sh <arg_list>”)
  notify_fault /path_to_script/script_fault.sh
      (or notify_fault “/path_to_script/script_fault.sh <arg_list>”)
}
vrrp_instance string {
  state MASTER|BACKUP
  interface string
  mcast_src_ip @IP
  lvs_sync_daemon_interface string
  virtual_router_id num
  priority num
  advert_int num
  smtp_alert
  authentication {
    auth_type PASS|AH
    auth_pass string
  }
  virtual_ipaddress { # Block limited to 20 IP addresses
    @IP
    @IP
    @IP
  }
  virtual_ipaddress_excluded { # Unlimited IP addresses number
    @IP
    @IP
    @IP
  }
  notify_master /path_to_script/script_master.sh
    (or notify_master “/path_to_script/script_master.sh <arg_list>”)
  notify_backup /path_to_script/script_backup.sh
    (or notify_backup “/path_to_script/script_backup.sh <arg_list>”)
  notify_fault /path_to_script/script_fault.sh
    (or notify_fault “/path_to_script/script_fault.sh <arg_list>”)
}
```

- `State` : trạng thái của Instance
- `Interface` : Interface mà Instance đang chạy
- `mcast_src_ip` : địa chỉ Multicast
- `lvs_sync_daemon_inteface` : Interface cho LVS sync_daemon
- `Virtual_router_id` :  VRRP router id
- `Priority` : thứ tự ưu tiên trong VRRP router
- `advert_int` : số  advertisement interval trong 1 giây
- `smtp_aler` : kích hoạt thông báo SMTP cho MASTER
- `authentication` :  VRRP authentication
- `virtual_ipaddress` : VRRP VIP


# Proxy

### Khái niệm
- Proxy server là một máy chủ đóng vai trò như một trạm trung chuyển giữa người dùng nội bộ và các host bên ngoài.
- Proxy server bảo vệ và ẩn máy tính đối với mạng bên ngoài.
- Theo dõi và giám sát lưu lượng vào ra của mỗi port.
- Proxy server cũng có thể được sử dụng cho việc lọc request.

### Hoạt động

- Proxy server xác định những yêu cầu từ phía client và quyết định đáp ứng hay không đáp ứng, nếu yêu cầu được đáp ứng, proxy server sẽ kết nối tới server thật thay cho client và tiếp tục chuyển tiếp đến những yêu cầu từ client đến server, cũng như đáp ứng những yêu cầu của server đến client. 


### Configure

- Load Balance

```sh
http   {

upstream server_group   {

server my.server1.com weight=3;

server my.server2.com;

}

server  {

location / {

proxy_pass http://server_group;

}

}

}
```

- Reverse Proxy

```sh
upstream server_group   {
server 10.5.9.161;
server 10.5.10.176;
}

...

location / {
        proxy_send_timeout   90;
        proxy_read_timeout   90;
        proxy_connect_timeout 30s;
 
        proxy_redirect  http://server_group  http://server_group;
        proxy_pass   http://server_group;
 
        proxy_set_header   Host   $host;
        proxy_set_header   X-Real-IP  $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }
 ```
 proxy_send_timeout: Thời gian chờ để request đến được proxy server
 proxy_read_timeout: Thời gian chờ để đọc reponse từ proxy server
 proxy_connect_timeout: Thời gian chờ để thiết lập kết nối với proxy server
 proxy_set_header: Sửa đổi hoặc thêm các trường vào header của request được chuyển đến proxy server.
 
# 3. Mô hình triên khai


```sh
                                          | eth0
                            +----------------------------+
                            |                            |                       
                            |10.5.10.112                 | 10.5.10.156              
                +-----------+-----------+    +-----------+-----------+   
                |      [keepalived1]    |    |     [keepalived2]     | 
                |       keepalived      |    |       keepalived      | 
                |         nginx         |    |         nginx         | 
                |                       |    |                       |      
                |                       |    |                       |    
                |                       |    |                       |    
                +-----------+-----------+    +-----------------------+    
                            |             eth1            |   
                            +-----------------------------+
                                          |
                                          |
                               +----------+----------+
                               |      [webserver]    |
                               |        nginx        |
                               |                     |
                               |                     |
                               |                     |
                               +---------------------+
 
```
 
 
 
 #### Cài đặt nginx trên tất cả các node
 
```sh
 root@quynv-keepalive2:~# apt install -y nginx
```
 
 #### Cài đặt keepalived trên các node keepalived
 
 ```sh
  root@quynv-keepalive2:~# apt install -y keepalived
```
#### Cấu hình reverse proxy
  
```sh
  root@quynv-keepalive2:~# vim /etc/nginx/sites-available/default 

...
upstream server_group   {

server 10.5.9.161;

server 10.5.10.176;

}

server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;
        server_name _;
        proxy_set_header        Host $host;
        proxy_set_header        X-Real-IP $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;
        location / {
                proxy_pass          http://10.5.9.161;
        }

 ```
 
#### Cấu hình keepalived

- Trên keepalive1

```sh
global_defs {
script_user root
enable_script_security
}
vrrp_script check_nginx {
  script "/bin/bash /etc/keepalived/check_nginx.sh"
  interval 2
  weight 50
}
vrrp_instance VI_1 {
  virtual_router_id 53
  advert_int 1
  priority 100
  mcast_src_ip 10.5.10.112
  state MASTER
  interface eth0
  virtual_ipaddress {
    10.5.9.125 dev eth0
  }
 authentication {
     auth_type PASS
     auth_pass 123456
     }
 track_script {
    check_nginx
  }
}
```

- Trên keepalived2

```sh
root@quynv-keepalive2:~# vim /etc/keepalived/keepalived.conf

...
global_defs {
}
vrrp_script check_nginx {
  script "/bin/bash /etc/keepalived/check_nginx.sh"
  interval 2
  weight 50
}
vrrp_instance VI_1 {
  virtual_router_id 53
  advert_int 1
  mcast_src_ip 10.5.10.156
  priority 99
  state BACKUP
  interface eth0
  virtual_ipaddress {
    10.5.9.125 dev eth0
  }
authentication {
        auth_type PASS
        auth_pass 123456
        }
  track_script {
    check_nginx
  }
}
```
 
- Script check_nginx.sh

```sh
root@quynv-keepalive1:~# cat /etc/keepalived/check_nginx.sh 
#!/bin/sh

if [ -z "`/bin/pidof nginx`" ]; then
  exit 1
fi
```
- Khởi động lại keepalived

```sh
root@quynv-keepalive1:~# systemctl restart keepalived
```

- Kiểm tra ip trên node master

```sh
 root@quynv-keepalive1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:d8:7e:63 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    altname ens3
    inet 10.5.10.112/22 metric 100 brd 10.5.11.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.5.9.125/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fed8:7e63/64 scope link 
```

- Kết nối đến vip

```sh
root@quynv-keepalive1:~# curl 10.5.9.125
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx-server-1!</h1>
</body>
</html>
```
