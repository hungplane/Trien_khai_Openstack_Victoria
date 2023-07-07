# Triển khai Keystone

** Lưu ý: Keystone chỉ triển khai trên Controller node **

- Bước 1:
• Cập nhật OS:
$ dnf -y update
• Cài đặt repo epel:
$ dnf -y install epel-release
$ dnf -y update

- Bước 2:
• Cài đặt các công cụ cần thiết:
$ dnf -y install vim wget curl telnet bash-completion dnf-utils crudini
• Tắt firewall:
$ systemctl disable firewalld
$ systemctl stop firewalld
• Tắt SELinux:
$ setenforce 0
$ sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
• Cài đặt network-scripts để quản lý network dưới dạng file:
$ dnf install -y network-scripts
$ systemctl enable network && systemctl start network
$ touch /etc/sysconfig/disable-deprecation-warnings

Bước 3:
• Dùng GUI bằng lệnh: nmtui
• Dùng cách sửa file tại: /etc/sysconfig/network-scripts/ifcfg-ensXX
• Dùng CLI:
• IP cho ens192 trên controller:
$ nmcli con modify ens192 ipv4.addresses 172.20.0.11/24
$ nmcli con modify ens192 ipv4.gateway 172.20.0.1
$ nmcli con modify ens192 ipv4.dns 1.1.1.1
$ nmcli con modify ens192 ipv4.method manual
$ nmcli con mod ens3 connection.autoconnect yes
• IP cho ens224 trên controller:
$ nmcli con modify ens224 ipv4.addresses 172.20.10.10/24
$ nmcli con modify ens224 ipv4.method manual
$ nmcli con modify ens224 connection.autoconnect yes
• Đặt hostname: 
$ hostnamectl set-hostname controller

Bước 4:
• Cấu hình DNS local:
$ cat >> /etc/hosts << "EOF"
172.20.0.11 controller
172.20.0.12 compute1
EOF
• Khởi động lại máy
$ init 6

Bước 5: 
• Cài đặt chrony:
$ dnf -y install chrony
• Sửa file /etc/chrony.conf
$ sed -i 's/2.centos.pool.ntp.org/0.asia.pool.ntp.org/' /etc/chrony.conf
• Khởi động lại dịch vụ:
$ systemctl enable chronyd.service
$ systemctl restart chronyd.service
• Kiểm tra hoạt động:
$ chronyc sources -v

Bước 6:
• Cài đặt repository về OpenStack Victoria:
$ dnf config-manager --enable powertools
$ dnf install -y centos-release-openstack-victoria
$ sed -i -e "s/enabled=1/enabled=0/g" /etc/yum.repos.d/CentOS-OpenStack-victoria.repo
$ dnf --enablerepo=centos-openstack-victoria -y upgrade
• Cài đặt OpenStack SELinux
$ dnf --enablerepo=centos-openstack-victoria,epel,powertools -y install openstack-selinux

Bước 7:
• Cài đặt gói mariadb, chỉ định phiên bản 10.3
dnf module -y install mariadb:10.3
• Cấu hình Mariadb:
cat << EOF > /etc/my.cnf.d/openstack.cnf
[server]
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mariadb/mariadb.log
pid-file=/run/mariadb/mariadb.pid
character-set-server=utf8
bind-address = 0.0.0.0
max_connections = 102400
[galera]
[embedded]
[mariadb]
[mariadb-10.3]
EOF

Bước 8:
• Khởi chạy MariaDB
$ systemctl enable --now mariadb
• Thiết lập mật khẩu cho tài khoản ‘root’. Do chưa có mật khẩu nên chỉ cần Enter để truy cập. Sau đó 
đặt mật khẩu mới là ‘Welcome123’
$ mysql_secure_installation

Bước 9:
• Cài đặt gói:
$ dnf --enablerepo=powertools -y install rabbitmq-server
• Khởi chạy dịch vụ RabbitMQ:
$ systemctl enable --now rabbitmq-server.service
• Tạo user và phân quyền:
$ rabbitmqctl add_user openstack Welcome123
$ rabbitmqctl set_permissions openstack ".*" ".*" ".*"

Bước 10:
• Cài đặt gói:
$ dnf --enablerepo=powertools -y install memcached
• Thiết lập cấu hình:
$ sed -i -e "s/-l 127.0.0.1,::1/-l 127.0.0.1,::1,172.20.0.11/g" /etc/sysconfig/memcached
• Khởi chạy dịch vụ:
$ systemctl enable --now memcached.service

Bước 11:
• Khởi tạo DB
$ cat << EOF | mysql -uroot -pWelcome123
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Welcome123';
EOF
• Cài đặt gói:
$ dnf --enablerepo=centos-openstack-victoria,epel,powertools -y install openstack-keystone python3-
openstackclient httpd mod_ssl python3-mod_wsgi python3-oauth2client
• Thiết lập hàm sửa đổi cấu hình:
$ function ops_add {
crudini --set $1 $2 $3 $4
}
$ function ops_del {
crudini --del $1 $2 $3
}
• NOTE: Không đặt password có ký tự đặc biệt như @ ! #

Bước 12:
• Cấu hình DB
$ ops_add /etc/keystone/keystone.conf database connection 
mysql+pymysql://keystone:Welcome123@172.20.0.11/keystone
• Cấu hình sử dụng fernet key
$ ops_add /etc/keystone/keystone.conf token provider fernet
• Đồng bộ DB cho Keystone:
$ su -s /bin/sh -c "keystone-manage db_sync" keystone
• Tạo lập fernet key:
$ keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
• Thực hiện Bootstrap Keystone:
$ keystone-manage bootstrap --bootstrap-password Welcome123 --bootstrap-admin-url 
http://172.20.0.11:5000/v3/ --bootstrap-internal-url http://172.20.0.11:5000/v3/ --bootstrap-public-url 
http://172.20.0.11:5000/v3/ --bootstrap-region-id Hanoi


Bước 13:
• Sửa file /etc/httpd/conf/httpd.conf:
$ sed -i -e "s/^Listen.*/Listen controller:80/g" /etc/httpd/conf/httpd.conf
• Tạo liên kết tắt cho file cấu hình:
$ ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
• Khởi động lại dịch vụ Apache:
$ systemctl enable httpd.service
$ systemctl restart httpd.service

Bước 14:
• Tạo file khai báo biến môi trường để sử dụng tập lệnh openstack :
$ cat << EOF > /root/admin-openrc
export OS_USERNAME=admin
export OS_PASSWORD=Welcome123
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://172.20.0.11:5000/v3
export OS_IDENTITY_API_VERSION=3
EOF
• Import biến môi trường:
$ source /root/admin-openrc
• Chạy lệnh kiểm tra:
$ openstack token issue

Bước 15:
• Tạo project service
$ openstack project create --domain default --description "Service Project" service
• Tạo project demo:
$ openstack project create --domain default --description "Demo Project" demo
• Tạo user demo:
$ openstack user create --domain default --password Welcome123 demo
• Tạo role user:
$ openstack role create user
• Gán user demo vào role user và project demo:
$ openstack role add --project demo --user demo user

Bước 16:
• Tạo Database cho Glance:
$ cat << EOF | mysql -uroot -pWelcome123
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Welcome123';
EOF
• Tạo user glance:
$ openstack user create --domain default --password Welcome123 glance
• Thêm role admin cho user glance:
$ openstack role add --project service --user glance admin
• Tạo dịch vụ có tên glance:
$ openstack service create --name glance --description "OpenStack Image" image

Bước 17:
• Tạo các endpoint cho dịch vụ glance
$ openstack endpoint create --region Hanoi image public http://172.20.0.11:9292
$ openstack endpoint create --region Hanoi image internal http://172.20.0.11:9292
$ openstack endpoint create --region Hanoi image admin http://172.20.0.11:9292
• Cài đặt gói glance:
$ dnf --enablerepo=centos-openstack-victoria,powertools,epel -y install openstack-glance
• Sửa đổi cấu hình glance:
$ ops_add /etc/glance/glance-api.conf database connection 
mysql+pymysql://glance:Welcome123@172.20.0.11/glance
$ ops_add /etc/glance/glance-api.conf DEFAULT bind_host 0.0.0.0
$ ops_add /etc/glance/glance-api.conf glance_store stores file,http
$ ops_add /etc/glance/glance-api.conf glance_store default_store file
$ ops_add /etc/glance/glance-api.conf glance_store filesystem_store_datadir /var/lib/glance/images/
$ ops_add /etc/glance/glance-api.conf paste_deploy flavor keystone

Bước 18:
$ ops_add /etc/glance/glance-api.conf keystone_authtoken www_authenticate_uri 
http://172.20.0.11:5000
$ ops_add /etc/glance/glance-api.conf keystone_authtoken auth_url http://172.20.0.11:5000
$ ops_add /etc/glance/glance-api.conf keystone_authtoken memcached_servers 172.20.0.11:11211
$ ops_add /etc/glance/glance-api.conf keystone_authtoken auth_type password
$ ops_add /etc/glance/glance-api.conf keystone_authtoken project_domain_name default
$ ops_add /etc/glance/glance-api.conf keystone_authtoken user_domain_name default
$ ops_add /etc/glance/glance-api.conf keystone_authtoken project_name service
$ ops_add /etc/glance/glance-api.conf keystone_authtoken username glance
$ ops_add /etc/glance/glance-api.conf keystone_authtoken password Welcome123
• Thực hiện đồng bộ DB cho glance:
$ su -s /bin/sh -c "glance-manage db_sync" glance
• Khởi động dịch vụ glance:
$ systemctl enable openstack-glance-api.service 
$ systemctl restart openstack-glance-api.service

Bước 18:
$ ops_add /etc/glance/glance-api.conf keystone_authtoken www_authenticate_uri 
http://172.20.0.11:5000
$ ops_add /etc/glance/glance-api.conf keystone_authtoken auth_url http://172.20.0.11:5000
$ ops_add /etc/glance/glance-api.conf keystone_authtoken memcached_servers 172.20.0.11:11211
$ ops_add /etc/glance/glance-api.conf keystone_authtoken auth_type password
$ ops_add /etc/glance/glance-api.conf keystone_authtoken project_domain_name default
$ ops_add /etc/glance/glance-api.conf keystone_authtoken user_domain_name default
$ ops_add /etc/glance/glance-api.conf keystone_authtoken project_name service
$ ops_add /etc/glance/glance-api.conf keystone_authtoken username glance
$ ops_add /etc/glance/glance-api.conf keystone_authtoken password Welcome123
• Thực hiện đồng bộ DB cho glance:
$ su -s /bin/sh -c "glance-manage db_sync" glance
• Khởi động dịch vụ glance:
$ systemctl enable openstack-glance-api.service 
$ systemctl restart openstack-glance-api.service

