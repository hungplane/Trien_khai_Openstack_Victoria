# Triển khai Cinder, Horizon

Bước 1: 
• Tạo Database cho cinder:
$ cat << EOF | mysql -pWelcome123
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'Welcome123';
EOF
• Tạo service và user cho Cinder:
$ source /root/admin-openrc
$ openstack user create --domain default --password Welcome123 cinder
$ openstack role add --project service --user cinder admin
$ openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
$ openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3

Bước 2: 
• Tạo endpoint cho Cinder:
$ openstack endpoint create --region Hanoi volumev2 public http://172.20.0.11:8776/v2/%\(project_id\)s
$ openstack endpoint create --region Hanoi volumev2 internal 
http://172.20.0.11:8776/v2/%\(project_id\)s
$ openstack endpoint create --region Hanoi volumev2 admin http://172.20.0.11:8776/v2/%\(project_id\)s
$ openstack endpoint create --region Hanoi volumev3 public http://172.20.0.11:8776/v3/%\(project_id\)s
$ openstack endpoint create --region Hanoi volumev3 internal 
http://172.20.0.11:8776/v3/%\(project_id\)s
$ openstack endpoint create --region Hanoi volumev3 admin http://172.20.0.11:8776/v3/%\(project_id\)s
• Cài đặt gói cho Cinder và LVM:
$ dnf --enablerepo=centos-openstack-victoria,powertools,epel -y install openstack-cinder lvm2 devicemapper-persistent-data targetcli

Bước 3: 
• Tạo hàm sửa đổi cấu hình
$ function ops_add {
crudini --set $1 $2 $3 $4
}
$ function ops_del {
crudini --del $1 $2 $3
}
• Thực hiện chỉnh sửa cấu hình cho Cinder:
$ ops_add /etc/cinder/cinder.conf DEFAULT my_ip 172.20.0.11
$ ops_add /etc/cinder/cinder.conf DEFAULT host controller
$ ops_add /etc/cinder/cinder.conf DEFAULT use_forwarded_for true
$ ops_add /etc/cinder/cinder.conf DEFAULT glance_api_servers http://172.20.0.11:9292
$ ops_add /etc/cinder/cinder.conf DEFAULT glance_api_version 2
$ ops_add /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:Welcome123@172.20.0.11:5672
$ ops_add /etc/cinder/cinder.conf DEFAULT log_dir /var/log/cinder
$ ops_add /etc/cinder/cinder.conf DEFAULT state_path /var/lib/cinder
$ ops_add /etc/cinder/cinder.conf DEFAULT auth_strategy keystone
$ ops_add /etc/cinder/cinder.conf DEFAULT volume_name_template volume-%s
$ ops_add /etc/cinder/cinder.conf DEFAULT api_paste_config /etc/cinder/api-paste.ini

Bước 4: 
• Tiếp tục thực hiện chỉnh sửa cấu hình cho Cinder:
$ ops_add /etc/cinder/cinder.conf DEFAULT rootwrap_config /etc/cinder/rootwrap.conf
$ ops_add /etc/cinder/cinder.conf DEFAULT allowed_direct_url_schemes cinder
$ ops_add /etc/cinder/cinder.conf DEFAULT enabled_backends lvm
$ ops_add /etc/cinder/cinder.conf lvm iscsi_ip_address 172.20.0.11
$ ops_add /etc/cinder/cinder.conf lvm volumes_dir /var/lib/cinder/volumes
$ ops_add /etc/cinder/cinder.conf lvm volume_driver cinder.volume.drivers.lvm.LVMVolumeDriver
$ ops_add /etc/cinder/cinder.conf lvm volume_group cinder-volumes
$ ops_add /etc/cinder/cinder.conf lvm volume_backend_name lvm
$ ops_add /etc/cinder/cinder.conf lvm target_helper lioadm
$ ops_add /etc/cinder/cinder.conf lvm target_protocol iscsi
$ ops_add /etc/cinder/cinder.conf database connection mysql+pymysql://cinder:Welcome123@172.20.0.11/cinder
$ ops_add /etc/cinder/cinder.conf keystone_authtoken www_authenticate_uri http://172.20.0.11:5000
$ ops_add /etc/cinder/cinder.conf keystone_authtoken auth_url http://172.20.0.11:5000
$ ops_add /etc/cinder/cinder.conf keystone_authtoken memcached_servers 172.20.0.11:11211
$ ops_add /etc/cinder/cinder.conf keystone_authtoken auth_type password
$ ops_add /etc/cinder/cinder.conf keystone_authtoken project_domain_id default
$ ops_add /etc/cinder/cinder.conf keystone_authtoken user_domain_id default
$ ops_add /etc/cinder/cinder.conf keystone_authtoken project_name service

Bước 5: 
• Tiếp tục thực hiện chỉnh sửa cấu hình cho Cinder:
$ ops_add /etc/cinder/cinder.conf keystone_authtoken username cinder
$ ops_add /etc/cinder/cinder.conf keystone_authtoken password Welcome123
$ ops_add /etc/cinder/cinder.conf keystone_authtoken region_name Hanoi
$ ops_add /etc/cinder/cinder.conf nova interface internal
$ ops_add /etc/cinder/cinder.conf nova auth_url http://172.20.0.11:5000
$ ops_add /etc/cinder/cinder.conf nova auth_type password
$ ops_add /etc/cinder/cinder.conf nova project_domain_name default
$ ops_add /etc/cinder/cinder.conf nova user_domain_name default
$ ops_add /etc/cinder/cinder.conf nova region_name Hanoi
$ ops_add /etc/cinder/cinder.conf nova project_name service
$ ops_add /etc/cinder/cinder.conf nova username nova
$ ops_add /etc/cinder/cinder.conf nova password Welcome123
$ ops_add /etc/cinder/cinder.conf oslo_concurrency lock_path /var/lib/cinder/tmp
$ ops_add /etc/cinder/cinder.conf oslo_messaging_notifications transport_url 
rabbit://openstack:Welcome123@172.20.0.11:5672

Bước 6: 
• Thực hiện đồng bộ database Cinder:
$ su -s /bin/sh -c "cinder-manage db sync" cinder
• Cấu hình LVM:
$ lsblk
$ pvcreate /dev/sdb
$ vgcreate cinder-volumes /dev/sdb
$ string="filter = [ \"a/sdb/\", \"r/.*/\"]"
$ sed -i 's|# Accept every block device:|'"$string"'|g' /etc/lvm/lvm.conf
• Khởi động service Cinder:
$ systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service openstack-cindervolume.service
$ systemctl restart openstack-cinder-api.service openstack-cinder-scheduler.service openstack-cindervolume.service
• Tạo volume type LVM:
$ openstack volume type create lvm
$ openstack volume type set lvm --property volume_backend_name=lvm



Bước 7: 
• Cài đặt gói cung cấp giao diện Dashboard:
$ dnf --enablerepo=centos-openstack-victoria,powertools,epel -y install openstack-dashboard
• Chỉnh sửa cấu hình cho horizon:
$ sed -i 's|127.0.0.1|'"172.20.0.11"'|g' /etc/openstack-dashboard/local_settings
$ sed -i 's|"http://%s/identity/v3"|"'http://%s:5000/v3'"|g' /etc/openstack-dashboard/local_settings
$ sed -i -e "s/ALLOWED_HOSTS.*/ALLOWED_HOSTS = ['*',]/g" /etc/openstack-dashboard/local_settings
$ echo "SESSION_ENGINE = 'django.contrib.sessions.backends.cache'" >> /etc/openstackdashboard/local_settings
$ echo "OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True" >> /etc/openstackdashboard/local_settings
$ echo "WEBROOT = '/dashboard/'" >> /etc/openstack-dashboard/local_settings


Bước 8: 
• Tiếp tục thực hiện chỉnh sửa cấu hình cho Horizon:
$ cat << EOF >> /etc/openstack-dashboard/local_settings
OPENSTACK_API_VERSIONS = {
"identity": 3,
"image": 2,
"volume": 2,
}
CACHES = {
'default': {
'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
'LOCATION': ['172.20.0.11']
}
}
POLICY_FILES_PATH = '/etc/openstack-dashboard'
EOF

Bước 9: 
• Tiếp tục thực hiện chỉnh sửa cấu hình cho Horizon:
$ echo 'OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"' >> /etc/openstackdashboard/local_settings
$ echo 'OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"' >> /etc/openstack-dashboard/local_settings
$ sed -i 's/TIME_ZONE = "UTC"/TIME_ZONE = "Asia\/Ho_Chi_Minh"/g' /etc/openstackdashboard/local_settings
$ echo "WSGIApplicationGroup %{GLOBAL}" >> /etc/httpd/conf.d/openstack-dashboard.conf
• Khởi động lại dịch vụ httpd và Memcached:
$ systemctl daemon-reload
$ systemctl restart httpd memcached
• Truy cập vào giao diện Horizon bằng địa chỉ:
$ http://172.20.0.11/dashboard

