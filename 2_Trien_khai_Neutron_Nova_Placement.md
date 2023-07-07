# Triển khai Neutron, Nova, Placement

## Cài đặt trên Controller

Bước 1: 
• Tạo database cho Placement:
$ cat << EOF | mysql -uroot -pWelcome123
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'Welcome123';
EOF
• Tạo user cho Placement:
$ source /root/admin-openrc
$ openstack user create --domain default --password Welcome123 placement
$ openstack role add --project service --user placement admin
$ openstack service create --name placement --description "Placement API" placement

Bước 2: 
• Tạo endpoint cho Placement:
$ openstack endpoint create --region Hanoi placement public http://172.20.0.11:8778
$ openstack endpoint create --region Hanoi placement internal http://172.20.0.11:8778
$ openstack endpoint create --region Hanoi placement admin http://172.20.0.11:8778
• Cài đặt gói cho Placement:
$ dnf --enablerepo=centos-openstack-victoria,powertools,epel -y install openstack-placement-api
• Tạo function để sửa cấu hình:
$ function ops_add {
crudini --set $1 $2 $3 $4
}
$ function ops_del {
crudini --del $1 $2 $3
}

Bước 3: 
• Chỉnh sửa tập tin cấu hình /etc/placement/placement.conf
$ ops_add /etc/placement/placement.conf DEFAULT debug false
$ ops_add /etc/placement/placement.conf keystone_authtoken www_authenticate_uri http://172.20.0.11:5000
$ ops_add /etc/placement/placement.conf keystone_authtoken auth_url http://172.20.0.11:5000
$ ops_add /etc/placement/placement.conf keystone_authtoken memcached_servers 172.20.0.11:11211
$ ops_add /etc/placement/placement.conf keystone_authtoken auth_type password
$ ops_add /etc/placement/placement.conf keystone_authtoken project_domain_name default
$ ops_add /etc/placement/placement.conf keystone_authtoken user_domain_name default
$ ops_add /etc/placement/placement.conf keystone_authtoken project_name service
$ ops_add /etc/placement/placement.conf keystone_authtoken username placement
$ ops_add /etc/placement/placement.conf keystone_authtoken password Welcome123
$ ops_add /etc/placement/placement.conf placement_database connection 
mysql+pymysql://placement:Welcome123@172.20.0.11/placement
$ ops_add /etc/placement/placement.conf api auth_strategy keystone

Bước 4: 
• Thực hiện đồng bộ Database:
$ su -s /bin/sh -c "placement-manage db sync" placement
• Thêm cấu hình của placement vào Apache:
cat << 'EOF' >> /etc/httpd/conf.d/00-placement-api.conf
<Directory /usr/bin>
<IfVersion >= 2.4>
Require all granted
</IfVersion>
<IfVersion < 2.4>
Order allow,deny
Allow from all
</IfVersion>
</Directory>
EOF
• Khởi động lại dịch vụ Apache:
$ systemctl restart httpd

Bước 5: 
• Tạo Database cho Nova:
$ cat << EOF | mysql -uroot -pWelcome123
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'Welcome123';
EOF

Bước 6: 
• Tạo service và user cho Nova:
$ source /root/admin-openrc
$ openstack user create --domain default --password Welcome123 nova
$ openstack role add --project service --user nova admin
$ openstack service create --name nova --description "OpenStack Compute" compute
• Tạo endpoint cho Nova:
$ openstack endpoint create --region Hanoi compute public http://172.20.0.11:8774/v2.1
$ openstack endpoint create --region Hanoi compute internal http://172.20.0.11:8774/v2.1
$ openstack endpoint create --region Hanoi compute admin http://172.20.0.11:8774/v2.1
• Cài đặt gói cho Nova:
$ dnf --enablerepo=centos-openstack-victoria,powertools,epel -y install openstack-nova

Bước 7: 
• Thực hiện chỉnh sửa cấu hình cho Nova:
$ ops_add /etc/nova/nova.conf DEFAULT my_ip 172.20.0.11
$ ops_add /etc/nova/nova.conf DEFAULT state_path /var/lib/nova
$ ops_add /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
$ ops_add /etc/nova/nova.conf DEFAULT log_dir /var/log/nova
$ ops_add /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:Welcome123@172.20.0.11:5672/
$ ops_add /etc/nova/nova.conf api auth_strategy keystone
$ ops_add /etc/nova/nova.conf glance api_servers http://172.20.0.11:9292
$ ops_add /etc/nova/nova.conf oslo_concurrency lock_path $state_path/tmp
$ ops_add /etc/nova/nova.conf api_database connection mysql+pymysql://nova:Welcome123@172.20.0.11/nova_api
$ ops_add /etc/nova/nova.conf database connection mysql+pymysql://nova:Welcome123@172.20.0.11/nova
$ ops_add /etc/nova/nova.conf keystone_authtoken www_authenticate_uri http://172.20.0.11:5000
$ ops_add /etc/nova/nova.conf keystone_authtoken auth_url http://172.20.0.11:5000
$ ops_add /etc/nova/nova.conf keystone_authtoken memcached_servers 172.20.0.11:11211
$ ops_add /etc/nova/nova.conf keystone_authtoken auth_type password
$ ops_add /etc/nova/nova.conf keystone_authtoken project_domain_name default
$ ops_add /etc/nova/nova.conf keystone_authtoken user_domain_name default
$ ops_add /etc/nova/nova.conf keystone_authtoken project_name service
$ ops_add /etc/nova/nova.conf keystone_authtoken username nova
$ ops_add /etc/nova/nova.conf keystone_authtoken password Welcome123

Bước 8: 
• Tiếp tục chỉnh sửa cấu hình Nova:
$ ops_add /etc/nova/nova.conf placement auth_url http://172.20.0.11:5000
$ ops_add /etc/nova/nova.conf placement os_region_name Hanoi
$ ops_add /etc/nova/nova.conf placement auth_type password
$ ops_add /etc/nova/nova.conf placement project_domain_name default
$ ops_add /etc/nova/nova.conf placement user_domain_name default
$ ops_add /etc/nova/nova.conf placement project_name service
$ ops_add /etc/nova/nova.conf placement username placement
$ ops_add /etc/nova/nova.conf placement password Welcome123
$ ops_add /etc/nova/nova.conf wsgi api_paste_config /etc/nova/api-paste.ini
$ ops_add /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
$ ops_add /etc/nova/nova.conf vnc enabled true
$ ops_add /etc/nova/nova.conf vnc server_listen 172.20.0.11
$ ops_add /etc/nova/nova.conf vnc server_proxyclient_address 172.20.0.11

Bước 9: 
• Thực hiện đồng bộ database:
$ su -s /bin/sh -c "nova-manage api_db sync" nova
$ su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
$ su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
$ su -s /bin/sh -c "nova-manage db sync" nova
• Kiểm tra việc cài đặt và cấu hình Nova:
$ nova-manage cell_v2 list_cells
• Khởi động dịch vụ:
$ systemctl start openstack-nova-api.service openstack-nova-scheduler.service openstack-novaconductor.service openstack-nova-novncproxy.service
$ systemctl enable openstack-nova-api.service openstack-nova-scheduler.service openstack-novaconductor.service openstack-nova-novncproxy.service

Bước 10:
• Thực hiện tạo Database cho Neutron:
$ cat << EOF | mysql -uroot -pWelcome123
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'Welcome123';
EOF
• Tạo service và user cho neutron:
$ openstack user create --domain default --password Welcome123 neutron
$ openstack role add --project service --user neutron admin
$ openstack service create --name neutron --description "OpenStack Networking" network

Bước 11:
• Tạo endpoint:
$ openstack endpoint create --region Hanoi network public http://172.20.0.11:9696
$ openstack endpoint create --region Hanoi network internal http://172.20.0.11:9696
$ openstack endpoint create --region Hanoi network admin http://172.20.0.11:9696
• Cài đặt gói Neutron:
$ dnf --enablerepo=centos-openstack-victoria,powertools,epel -y install openstack-neutron openstackneutron-ml2 openstack-neutron-openvswitch ebtables libibverbs

Bước 12:
• Chỉnh sửa cấu hình tập tin /etc/neutron/neutron.conf:
$ ops_add /etc/neutron/neutron.conf DEFAULT bind_host 172.20.0.11
$ ops_add /etc/neutron/neutron.conf DEFAULT bind_port 9696
$ ops_add /etc/neutron/neutron.conf DEFAULT core_plugin ml2
$ ops_add /etc/neutron/neutron.conf DEFAULT service_plugins router,qos,neutron.services.metering.metering_plugin.MeteringPlugin
$ ops_add /etc/neutron/neutron.conf DEFAULT advertise_mtu true
$ ops_add /etc/neutron/neutron.conf DEFAULT allow_overlapping_ips true
$ ops_add /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes true
$ ops_add /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes true
$ ops_add /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:Welcome123@172.20.0.11:5672
$ ops_add /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
$ ops_add /etc/neutron/neutron.conf DEFAULT metadata_proxy_socket /var/lib/neutron/metadata_proxy
$ ops_add /etc/neutron/neutron.conf database connection mysql+pymysql://neutron:Welcome123@172.20.0.11/neutron
$ ops_add /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://172.20.0.11:5000
$ ops_add /etc/neutron/neutron.conf keystone_authtoken auth_url http://172.20.0.11:5000
$ ops_add /etc/neutron/neutron.conf keystone_authtoken memcached_servers 172.20.0.11:11211
$ ops_add /etc/neutron/neutron.conf keystone_authtoken auth_type password

Bước 13:
• Chỉnh sửa cấu hình tập tin /etc/neutron/neutron.conf:
$ ops_add /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
$ ops_add /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
$ ops_add /etc/neutron/neutron.conf keystone_authtoken project_name service
$ ops_add /etc/neutron/neutron.conf keystone_authtoken username neutron
$ ops_add /etc/neutron/neutron.conf keystone_authtoken password Welcome123
$ ops_add /etc/neutron/neutron.conf nova auth_url http://172.20.0.11:5000
$ ops_add /etc/neutron/neutron.conf nova region_name Hanoi
$ ops_add /etc/neutron/neutron.conf nova auth_type password
$ ops_add /etc/neutron/neutron.conf nova project_domain_name default
$ ops_add /etc/neutron/neutron.conf nova user_domain_name default
$ ops_add /etc/neutron/neutron.conf nova project_name service
$ ops_add /etc/neutron/neutron.conf nova username nova
$ ops_add /etc/neutron/neutron.conf nova password Welcome123
$ ops_add /etc/neutron/neutron.conf qos notification_drivers message_queue

Bước 14:
• Cấu hình tập /etc/neutron/l3_agent.ini
$ ops_add /etc/neutron/l3_agent.ini DEFAULT interface_driver openvswitch
$ ops_add /etc/neutron/l3_agent.ini DEFAULT external_network_bridge
$ ops_add /etc/neutron/l3_agent.ini DEFAULT agent_mode legacy
• Cấu hình tập tin /etc/neutron/plugins/ml2/ml2_conf.ini
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers flat,vlan,vxlan,gre
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan,gre,vlan
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers openvswitch,l2population
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security,qos
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini ml2 path_mtu 1500
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks physnet1
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 200:2000
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_gre tunnel_id_ranges 2000:4000
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset true
$ ops_add /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup firewall_driver 
neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

Bước 15:
• Cấu hình tập tin /etc/neutron/plugins/ml2/openvswitch_agent.ini
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini agent tunnel_types vxlan,gre
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini agent l2_population True
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini agent extensions qos
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini agent arp_responder true
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings physnet1:br-provider
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs local_ip 172.20.0.11
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs datapath_type system
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup firewall_driver 
neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
• Cấu hình tập tin /etc/neutron/dhcp_agent.ini:
$ ops_add /etc/neutron/dhcp_agent.ini DEFAULT interface_driver openvswitch
$ ops_add /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
$ ops_add /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata true

Bước 16:
• Cấu hình tập tin /etc/neutron/metadata_agent.ini
$ ops_add /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret 5b721855305f81836b04
$ ops_add /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_port 8775
$ ops_add /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host 172.20.0.11
• Sửa tập tin /etc/nova/nova.conf trên controller:
$ ops_add /etc/nova/nova.conf neutron url http://172.20.0.11:9696
$ ops_add /etc/nova/nova.conf neutron auth_url http://172.20.0.11:5000
$ ops_add /etc/nova/nova.conf neutron region_name Hanoi
$ ops_add /etc/nova/nova.conf neutron auth_type password
$ ops_add /etc/nova/nova.conf neutron project_domain_name default
$ ops_add /etc/nova/nova.conf neutron user_domain_name default
$ ops_add /etc/nova/nova.conf neutron project_name service
$ ops_add /etc/nova/nova.conf neutron username neutron
$ ops_add /etc/nova/nova.conf neutron password Welcome123
$ ops_add /etc/nova/nova.conf neutron service_metadata_proxy true
$ ops_add /etc/nova/nova.conf neutron metadata_proxy_shared_secret 5b721855305f81836b04

Bước 17:
• Tạo liên kết tắt cho tập tin cấu hình plugin:
$ ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
• Thực hiện đồng bộ Database Neutron:
$ su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file 
/etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
• Khởi động lại dịch vụ Nova:
$ systemctl restart openstack-nova-api.service
$ systemctl restart openstack-nova-scheduler openstack-nova-conductor
• Khởi động dịch vụ Neutron:
$ systemctl enable neutron-server.service neutron-openvswitch-agent.service neutron-dhcp-agent.service neutronmetadata-agent.service neutron-l3-agent.service
$ systemctl start neutron-server.service neutron-openvswitch-agent.service neutron-dhcp-agent.service neutronmetadata-agent.service neutron-l3-agent.service

Bước 18:
• Khởi động dịch vụ OpenvSwitch
$ systemctl enable openvswitch
$ systemctl start openvswitch
• Tạo switch và gắn NIC:
$ ovs-vsctl add-br br-provider
$ ovs-vsctl add-port br-provider ens224
• Xóa configuration file của ens224 (nếu có):
$ rm -rf /etc/sysconfig/network-scripts/ifcfg-Intranet

Bước 19:
• Chỉnh sửa cấu hình cho Interface:
$ cat << EOF > /etc/sysconfig/network-scripts/ifcfg-ens224
DEVICE=ens224
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-provider
BOOTPROTO=none
EOF
$ cat << EOF > /etc/sysconfig/network-scripts/ifcfg-br-provider
DEVICE=br-provider
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=none
IPADDR=172.20.10.10
NETMASK=255.255.255.0 
EOF
• Khởi động lại network:
$ systemctl restart network




## Triển khai trên Compute node

Bước 20:
• Cập nhật OS:
$ dnf -y update
• Cài đặt repo epel:
$ dnf -y install epel-release
$ dnf -y update
NOTE: 
• Gặp lỗi “Error: GPG check FAILED” khi chạy lệnh update thì xử lý như sau rồi chạy lại:
$ rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
$ rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-*
• Gặp lỗi “SSL certificate problem: certificate is not yet valid” khi chạy lệnh update thì xử lý như sau rồi chạy lại:
$ sed -i 's/https/http/g' /etc/yum.repos.d/epel.repo
$ sed -i 's/https/http/g' /etc/yum.repos.d/epel-modular.repo

Bước 21:
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

Bước 22:
• Dùng GUI bằng lệnh: nmtui
• Dùng cách sửa file tại: /etc/sysconfig/network-scripts/ifcfg-ensXX
• Dùng CLI:
• IP cho ens192 trên controller:
$ nmcli con modify ens192 ipv4.addresses 172.20.0.12/24
$ nmcli con modify ens192 ipv4.gateway 172.20.0.1
$ nmcli con modify ens192 ipv4.dns 1.1.1.1
$ nmcli con modify ens192 ipv4.method manual
$ nmcli con mod ens3 connection.autoconnect yes
• IP cho ens224 trên controller:
$ nmcli c modify ens224 ipv4.addresses 172.20.10.11/24
$ nmcli c modify ens224 ipv4.method manual
$ nmcli con mod ens224 connection.autoconnect yes
• Đặt hostname: 
$ hostnamectl set-hostname compute1

Bước 23:
• Cấu hình DNS local:
$ cat >> /etc/hosts << "EOF"
172.20.0.11 controller
172.20.0.12 compute1
EOF
• Khởi động lại máy
$ init 6

Bước 24:
• Cài đặt chrony:
$ dnf -y install chrony
• Sửa file /etc/chrony.conf
$ sed -i 's/2.centos.pool.ntp.org/0.asia.pool.ntp.org/' /etc/chrony.conf
• Khởi động lại dịch vụ:
$ systemctl enable chronyd.service
$ systemctl restart chronyd.service
• Kiểm tra hoạt động:
$ chronyc sources -v

Bước 25:
• Cài đặt repository về OpenStack Victoria:
$ dnf config-manager --enable powertools
$ dnf install -y centos-release-openstack-victoria
$ sed -i -e "s/enabled=1/enabled=0/g" /etc/yum.repos.d/CentOS-OpenStack-victoria.repo
$ dnf --enablerepo=centos-openstack-victoria -y upgrade
• Cài đặt OpenStack SELinux
$ dnf --enablerepo=centos-openstack-victoria,epel,powertools -y install openstack-selinux

Bước 26:
• Cài đặt gói Nova compute
$ dnf --enablerepo=centos-openstack-victoria,powertools,epel -y install openstack-novacompute libvirt-client
• Tạo hàm sửa đổi cấu hình
$ function ops_add {
crudini --set $1 $2 $3 $4
}
$ function ops_del {
crudini --del $1 $2 $3
}

Bước 27:
• Chỉnh sửa cấu hình Nova compute
$ ops_add /etc/nova/nova.conf DEFAULT my_ip 172.20.0.12
$ ops_add /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:Welcome123@172.20.0.11:5672
$ ops_add /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
$ ops_add /etc/nova/nova.conf DEFAULT use_neutron True
$ ops_add /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
$ ops_add /etc/nova/nova.conf api auth_strategy keystone
$ ops_add /etc/nova/nova.conf glance api_servers http://172.20.0.11:9292
$ ops_add /etc/nova/nova.conf keystone_authtoken www_authenticate_uri http://172.20.0.11:5000
$ ops_add /etc/nova/nova.conf keystone_authtoken auth_url http://172.20.0.11:5000
$ ops_add /etc/nova/nova.conf keystone_authtoken memcached_servers 172.20.0.11:11211
$ ops_add /etc/nova/nova.conf keystone_authtoken auth_type password
$ ops_add /etc/nova/nova.conf keystone_authtoken project_domain_name default
$ ops_add /etc/nova/nova.conf keystone_authtoken user_domain_name default
$ ops_add /etc/nova/nova.conf keystone_authtoken project_name service
$ ops_add /etc/nova/nova.conf keystone_authtoken username nova
$ ops_add /etc/nova/nova.conf keystone_authtoken password Welcome123

Bước 28:
• Tiếp tục chỉnh sửa cấu hình Nova compute
$ ops_add /etc/nova/nova.conf placement auth_url http://172.20.0.11:5000
$ ops_add /etc/nova/nova.conf placement os_region_name Hanoi
$ ops_add /etc/nova/nova.conf placement auth_type password
$ ops_add /etc/nova/nova.conf placement project_domain_name default
$ ops_add /etc/nova/nova.conf placement user_domain_name default
$ ops_add /etc/nova/nova.conf placement project_name service
$ ops_add /etc/nova/nova.conf placement username placement
$ ops_add /etc/nova/nova.conf placement password Welcome123
$ ops_add /etc/nova/nova.conf libvirt virt_type qemu
$ ops_add /etc/nova/nova.conf vnc enabled true
$ ops_add /etc/nova/nova.conf vnc server_listen 0.0.0.0
$ ops_add /etc/nova/nova.conf vnc server_proxyclient_address 172.20.0.12
$ ops_add /etc/nova/nova.conf vnc novncproxy_base_url http://172.20.0.11:6080/vnc_auto.html
• Khởi động lại dịch vụ:
$ systemctl enable libvirtd.service openstack-nova-compute.service
$ systemctl restart libvirtd.service openstack-nova-compute.service

Bước 29:
• Cài đặt các package Neutron:
$ dnf --enablerepo=centos-openstack-victoria,powertools,epel -y install openstack-neutron-openvswitch ebtables ipset
• Cấu hình /etc/neutron/neutron.conf
$ ops_add /etc/neutron/neutron.conf DEFAULT bind_host 172.20.0.12
$ ops_add /etc/neutron/neutron.conf DEFAULT interface_driver openvswitch
$ ops_add /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
$ ops_add /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:Welcome123@172.20.0.11:5672
$ ops_add /etc/neutron/neutron.conf DEFAULT core_plugin ml2
$ ops_add /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://172.20.0.11:5000
$ ops_add /etc/neutron/neutron.conf keystone_authtoken auth_url http://172.20.0.11:5000
$ ops_add /etc/neutron/neutron.conf keystone_authtoken memcached_servers 172.20.0.11:11211
$ ops_add /etc/neutron/neutron.conf keystone_authtoken auth_type password
$ ops_add /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
$ ops_add /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
$ ops_add /etc/neutron/neutron.conf keystone_authtoken project_name service
$ ops_add /etc/neutron/neutron.conf keystone_authtoken username neutron
$ ops_add /etc/neutron/neutron.conf keystone_authtoken password Welcome123
$ ops_add /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp

Bước 30:
• Chỉnh sửa /etc/neutron/plugins/ml2/openvswitch_agent.ini
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini agent tunnel_types vxlan,gre
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini agent l2_population True
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini agent extensions qos
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs local_ip 172.20.0.12
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings physnet1:br-provider
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup firewall_driver openvswitch
$ ops_add /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup enable_security_group true
• Chỉnh sửa cấu hình /etc/nova/nova.conf:
$ ops_add /etc/nova/nova.conf neutron url http://172.20.0.11:9696
$ ops_add /etc/nova/nova.conf neutron auth_url http://172.20.0.11:5000
$ ops_add /etc/nova/nova.conf neutron region_name Hanoi
$ ops_add /etc/nova/nova.conf neutron auth_type password
$ ops_add /etc/nova/nova.conf neutron project_domain_name default
$ ops_add /etc/nova/nova.conf neutron user_domain_name default
$ ops_add /etc/nova/nova.conf neutron project_name service
$ ops_add /etc/nova/nova.conf neutron username neutron
$ ops_add /etc/nova/nova.conf neutron password Welcome123
$ ops_add /etc/nova/nova.conf neutron service_metadata_proxy true
$ ops_add /etc/nova/nova.conf neutron metadata_proxy_shared_secret 5b721855305f81836b04

Bước 31:
• Khởi động lại dịch vụ Nova, Neutron
$ systemctl restart openstack-nova-compute.service
$ systemctl enable neutron-openvswitch-agent.service 
$ systemctl restart neutron-openvswitch-agent.service
• Khởi động dịch vụ OpenvSwitch
$ systemctl enable openvswitch
$ systemctl start openvswitch
• Tạo switch và gắn NIC:
$ ovs-vsctl add-br br-provider
$ ovs-vsctl add-port br-provider ens224

Bước 32:
• Chỉnh sửa cấu hình cho Interface:
$ cat << EOF > /etc/sysconfig/network-scripts/ifcfg-ens224
DEVICE=ens224
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-provider
BOOTPROTO=none
EOF
$ cat << EOF > /etc/sysconfig/network-scripts/ifcfg-br-provider
DEVICE=br-provider
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=none
IPADDR=172.20.10.11
NETMASK=255.255.255.0 
EOF
• Khởi động lại network:
$ systemctl restart network

Bước 33:
• Thực hiện cập nhật Compute node vào database:
$ su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
• Lệnh kiểm tra sau khi cài đặt placement:
$ placement-status upgrade check
• Lệnh kiểm tra sau khi cài đặt nova:
$ openstack compute service list
$ openstack hypervisor list
$ nova-status upgrade check
• Thực hiện kiểm tra việc cài đặt Neutron:
$ openstack network agent list