#Build Openstack
```
for i in ROOT_DBPASS ADMIN_PASS CINDER_DBPASS CINDER_PASS DASH_DBPASS  \
DEMO_PASS GLANCE_DBPASS GLANCE_PASS KEYSTONE_DBPASS NEUTRON_DBPASS NEUTRON_PASS \ 
NOVA_DBPASS NOVA_PASS  RABBIT_PASS; do echo "export $i=$(openssl rand -hex 10)" >> password.txt ;done
```

```
[root@controller ~]# cat password.txt
export ROOT_DBPASS=eb8667805b512ea32c17
export ADMIN_PASS=34a40917f0b5c14d0d36
export CINDER_DBPASS=bb542f421fb3e82cb963
export CINDER_PASS=32042cf4538d88bbfe81
export DASH_DBPASS=265aea2dc4347aa6594c
export DEMO_PASS=7be794e716af532ab0c0
export GLANCE_DBPASS=36c42dd8dc2c2e627bdd
export GLANCE_PASS=a559d37a0b339df09733
export KEYSTONE_DBPASS=c78dd5b934dde9c78d1f
export NEUTRON_DBPASS=4437b0907473c636475a
export NEUTRON_PASS=846eba05ffe6f90fe13e
export NOVA_DBPASS=c68593bb1ae83dfce9c3
export NOVA_PASS=e921884b74c7417beba3
export RABBIT_PASS=f86b659dc9fad20a9194
```
```
export PATH
source /root/password.txt
echo $ROOT_DBPASS
alias db="mysql -uroot -p$ROOT_DBPASS"
```
## 1 Add database user
```
db -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '$KEYSTONE_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '$KEYSTONE_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'192.168.10.10' IDENTIFIED BY '$KEYSTONE_DBPASS';"

## 2 Install package
```
yum install openstack-keystone httpd mod_wsgi

yum install openstack-utils
```
## 3 config
```
keystonconf=/etc/keystone/keystone.conf
crudini --set $keystonconf database connection mysql+pymysql://keystone:$KEYSTONE_DBPASS@controller/keystone
crudini --set $keystonconf token provider fernet
```
## 4 Sync Database
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
cat /etc/password
tail -f /var/log/keystone/keystone.log

## Fenet
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

## Bootstrap 
```
echo $ADMIN_PASS
keystone-manage bootstrap --bootstrap-password $ADMIN_PASS \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:35357/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

## Configure the Apache HTTP server
```
/etc/httpd/conf/httpd.conf 
ServerName controller
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
systemctl enable httpd
systemctl start httpd
ss -tulan | grep 5000
ss -tulan | grep 35357
```
## create admin_rc in /root
```
unset OS_USERNAME
unset OS_PASSWORD
unset OS_PROJECT_NAME
unset OS_USER_DOMAIN_NAME
unset OS_AUTH_URL
unset OS_IDENTITY_API_VERSION
export OS_USERNAME=admin
export OS_PASSWORD=$ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```

openstack project create --domain default --description "Service Project" service
```
[root@controller ~]# systemctl start httpd
[root@controller ~]# openstack project create --domain default --description "Service Project"  service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 316d7b045b4f4ae7a955b47e2a5d302c |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+

[root@controller ~]# openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | d4684bdc03ac4a89acfe04ce0f08817d |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+

[root@controller ~]# openstack user create --domain default --password-prompt demo
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 6240c4a5580047e8be08d90fb3123b56 |
| name                | demo                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+

[root@controller ~]# openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 4a695e2598a3462285354d23588bf084 |
| name      | user                             |
+-----------+----------------------------------+

```
```
2017-01-25 08:56:46.208 15337 INFO keystone.cmd.cli [req-8a63237b-b2c2-4013-abe3-90ef1fa5c
08a - - - - -] Created role admin
2017-01-25 08:56:46.219 15337 INFO keystone.cmd.cli [req-8a63237b-b2c2-4013-abe3-90ef1fa5c
08a - - - - -] Granted admin on admin to user admin.
2017-01-25 08:56:46.229 15337 INFO keystone.cmd.cli [req-8a63237b-b2c2-4013-abe3-90ef1fa5c
08a - - - - -] Created region RegionOne
```
```
[root@controller ~]# openstack role add --project demo --user demo user
```

## Verify operation

/etc/keystone/keystone-paste.ini 

unset OS_AUTH_URL OS_PASSWORD

openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue


[root@controller ~]# openstack --os-auth-url http://controller:35357/v3 \
>   --os-project-domain-name Default --os-user-domain-name Default \
>   --os-project-name admin --os-username admin token issue
Password:
+------------+------------------------------------------------------------------------------------------+
| Field      | Value                                                                                    |
+------------+------------------------------------------------------------------------------------------+
| expires    | 2017-01-25 10:37:14+00:00                                                                |
| id         | gAAAAABYiHHKsR34xQB6WqbU17-grz5CvRiJ6RY6o93fgwRVi4hEvqEd3ypUW47lSaUTa8zfX8KF1adYHeb4DUEI |
|            | f86-MNjYjkBVStgTgDDy-JRamw2do5BPK1s6hw1L7qrs4j4s-uQokR6gFCZ6qG_NhX14WguWRJQve-           |
|            | hCihUUbkPPtQgYn5w                                                                        |
| project_id | 7af7520ae92047fb87722fc7c58d8951                                                         |
| user_id    | 897a748e344f45a8a5d06d2398c3c24b                                                         |
+------------+------------------------------------------------------------------------------------------+

openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name demo --os-username demo token issue

```
[root@controller ~]# openstack --os-auth-url http://controller:5000/v3 \
>   --os-project-domain-name Default --os-user-domain-name Default \
>   --os-project-name demo --os-username demo token issue
Password:
+------------+------------------------------------------------------------------------------------------+
| Field      | Value                                                                                    |
+------------+------------------------------------------------------------------------------------------+
| expires    | 2017-01-25 10:44:42+00:00                                                                |
| id         | gAAAAABYiHOKQS-QjpsAofkK-mp3nuOQkoFWREvegBRvdkQKZxxy0MFpaFNQbzSEv1CptHEEXKc5LEL5LOflU9th |
|            | Td6wYOpTEFBNTd56mQ7EnUQgD-MF_34bqyx-hsmZpVBPhFrn1C6Vi5vh_gwNvcWqIAuU-                    |
|            | IQmPH4pNZNhf0jYSywQpGlbZ_c                                                               |
| project_id | d4684bdc03ac4a89acfe04ce0f08817d                                                         |
| user_id    | 6240c4a5580047e8be08d90fb3123b56                                                         |
+------------+------------------------------------------------------------------------------------------+
```

## DB
```
db -e "CREATE DATABASE glance;"
echo $GLANCE_DBPASS

db -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '$GLANCE_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '$GLANCE_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'192.168.10.10' IDENTIFIED BY '$GLANCE_DBPASS';"

[root@controller ~]# mysql -uglance -p$GLANCE_DBPASS
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 41
Server version: 10.1.18-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

$ . admin-openrc

[root@controller ~]# openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 6bd9904c11204594a2f538b856335595 |
| name                | glance                           |
| password_expires_at | None                             |
+---------------------+----------------------------------+

[root@controller ~]# openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image

[root@controller ~]# openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 710cadce24664964ad943cf6ef591d25 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+


openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292

## /etc/glance/glance-api.conf
```
glanceconf=/etc/glance/glance-api.conf 
crudini --set $glanceconf database connection mysql+pymysql://glance:$GLANCE_DBPASS@controller/glance
crudini --set $glanceconf keystone_authtoken auth_uri  http://controller:5000
crudini --set $glanceconf keystone_authtoken auth_url  http://controller:35357
crudini --set $glanceconf keystone_authtoken memcached_servers  controller:11211
crudini --set $glanceconf keystone_authtoken auth_type  password
crudini --set $glanceconf keystone_authtoken project_domain_name  Default
crudini --set $glanceconf keystone_authtoken user_domain_name  Default
crudini --set $glanceconf keystone_authtoken project_name  service
crudini --set $glanceconf keystone_authtoken username  glance
crudini --set $glanceconf keystone_authtoken password  $GLANCE_PASS            
crudini --set $glanceconf paste_deploy flavor  keystone
crudini --set $glanceconf glance_store stores  file,http
crudini --set $glanceconf glance_store default_store  file
crudini --set $glanceconf glance_store filesystem_store_datadir  /var/lib/glance/images/
```

## /etc/glance/glance-registry.conf 
```
glance_registry=/etc/glance/glance-registry.conf
crudini --set $glance_registry database connection  mysql+pymysql://glance:$GLANCE_DBPASS@controller/glance
crudini --set $glance_registry keystone_authtoken auth_uri  http://controller:5000
crudini --set $glance_registry keystone_authtoken auth_url  http://controller:35357
crudini --set $glance_registry keystone_authtoken memcached_servers  controller:11211
crudini --set $glance_registry keystone_authtoken auth_type  password
crudini --set $glance_registry keystone_authtoken project_domain_name  Default
crudini --set $glance_registry keystone_authtoken user_domain_name  Default
crudini --set $glance_registry keystone_authtoken project_name  service
crudini --set $glance_registry keystone_authtoken username  glance
crudini --set $glance_registry keystone_authtoken password  $GLANCE_PASS

crudini --set $glance_registry paste_deploy flavor  keystone
```

su -s /bin/sh -c "glance-manage db_sync" glance


systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service

wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

  [root@controller ~]# openstack image create "cirros" \
>   --file cirros-0.3.4-x86_64-disk.img \
>   --disk-format qcow2 --container-format bare \
>   --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2017-01-25T10:46:07Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/5c74436a-f0c7-4da1-85ff-c540af71ed18/file |
| id               | 5c74436a-f0c7-4da1-85ff-c540af71ed18                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 7af7520ae92047fb87722fc7c58d8951                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-01-25T10:46:08Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
## Nova
db -e "CREATE DATABASE nova_api;"
db -e "CREATE DATABASE nova;"
echo $NOVA_DBPASS
db -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY '$NOVA_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '$NOVA_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'192.168.10.10' IDENTIFIED BY '$NOVA_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '$NOVA_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '$NOVA_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'192.168.10.10' IDENTIFIED BY '$NOVA_DBPASS';"

openstack user create --domain default \
  --password-prompt nova
  
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute


openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1/%\(tenant_id\)s

yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler

# Install and configure components

novaconf=/etc/nova/nova.conf 

crudini --set $novaconf DEFAULT  enabled_apis  osapi_compute,metadata
crudini --set $novaconf api_database connection  mysql+pymysql://nova:$NOVA_DBPASS@controller/nova_api
crudini --set $novaconf database connection mysql+pymysql://nova:$NOVA_DBPASS@controller/nova


crudini --set $novaconf DEFAULT transport_url rabbit://openstack:$RABBIT_PASS@controller
crudini --set $novaconf DEFAULT auth_strategy keystone
crudini --set $novaconf keystone_authtoken auth_uri http://controller:5000
crudini --set $novaconf keystone_authtoken auth_url  http://controller:35357
crudini --set $novaconf keystone_authtoken memcached_servers controller:11211
crudini --set $novaconf keystone_authtoken auth_type password
crudini --set $novaconf keystone_authtoken project_domain_name Default
crudini --set $novaconf keystone_authtoken user_domain_name Default
crudini --set $novaconf keystone_authtoken project_name  service
crudini --set $novaconf keystone_authtoken username nova
crudini --set $novaconf keystone_authtoken password $NOVA_PASS


crudini --set $novaconf DEFAULT my_ip 192.168.10.10
crudini --set $novaconf DEFAULT use_neutron  True
crudini --set $novaconf firewall_driver nova.virt.firewall.NoopFirewallDriver

crudini --set $novaconf vnc vncserver_listen \$my_ip
crudini --set $novaconf vnc vncserver_proxyclient_address \$my_ip

crudini --set $novaconf  glance api_servers http://controller:9292
crudini --set $novaconf lock_path  /var/lib/nova/tmp

su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage db sync" nova

systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service


  
  
#nova compute

yum install openstack-nova-compute
echo $RABBIT_PASS
novaconf=/etc/nova/nova.conf 
crudini --set $novaconf DEFAULT enabled_apis osapi_compute,metadata
crudini --set $novaconf DEFAULT transport_url rabbit://openstack:$RABBIT_PASS@controller

crudini --set $novaconf DEFAULT auth_strategy keystone
crudini --set $novaconf keystone_authtoken auth_uri http://controller:5000
crudini --set $novaconf keystone_authtoken auth_url  http://controller:35357
crudini --set $novaconf keystone_authtoken memcached_servers controller:11211
crudini --set $novaconf keystone_authtoken auth_type password
crudini --set $novaconf keystone_authtoken project_domain_name Default
crudini --set $novaconf keystone_authtoken user_domain_name Default
crudini --set $novaconf keystone_authtoken project_name  service
crudini --set $novaconf keystone_authtoken username nova
crudini --set $novaconf keystone_authtoken password $NOVA_PASS
crudini --set $novaconf DEFAULT my_ip 192.168.10.12
crudini --set $novaconf DEFAULT use_neutron  True
crudini --set $novaconf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver 

crudini --set $novaconf vnc enabled  True
crudini --set $novaconf vnc vncserver_listen 0.0.0.0
crudini --set $novaconf vnc vncserver_proxyclient_address \$my_ip
crudini --set $novaconf vnc novncproxy_base_url  http://controller:6080/vnc_auto.html
crudini --set $novaconf glance api_servers  http://controller:9292
crudini --set $novaconf glance oslo_concurrency lock_path /var/lib/nova/tmp
crudini --set $novaconf libvirt virt_type  qemu

crudini --set $novaconf DEFAULT transport_url  rabbit://openstack:$RABBIT_PASS@controller

systemctl enable libvirtd.service openstack-nova-compute.service


91e05d15281] AMQP server on 127.0.0.1:5672 is unreachable: [Errno 111] ECONNREFUSED.
 Trying again in 32 seconds. Client port: None
 
 
# Install Networknode on Networknode
### Create neutron user in database

```
db -e "CREATE DATABASE neutron;"
db -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY '$NEUTRON_DBPASS';"
db -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '$NEUTRON_DBPASS';"
source admin_openrc
openstack user create --domain default --password-prompt neutron

openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network

openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696

##Networking Option 1: Provider networks
On Networknode
```
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables

neutronconf=/etc/neutron/neutron.conf
crudini --set $neutronconf database connection mysql+pymysql://neutron:$NEUTRON_DBPASS@controller/neutron  
crudini --set $neutronconf DEFAULT core_plugin ml2 core_plugin ml2
crudini --set $neutronconf DEFAULT core_plugin ml2 service_plugins 
crudini --set $neutronconf DEFAULT transport_url rabbit://openstack:$RABBIT_PASS@controller
crudini --set $neutronconf DEFAULT auth_strategy keystone
crudini --set $neutronconf keystone_authtoken auth_uri http://controller:5000
crudini --set $neutronconf keystone_authtoken auth_url  http://controller:35357
crudini --set $neutronconf keystone_authtoken memcached_servers controller:11211
crudini --set $neutronconf keystone_authtoken auth_type password
crudini --set $neutronconf keystone_authtoken project_domain_name Default
crudini --set $neutronconf keystone_authtoken user_domain_name Default
crudini --set $neutronconf keystone_authtoken project_name  service
crudini --set $neutronconf keystone_authtoken username neutron
crudini --set $neutronconf keystone_authtoken password $NEUTRON_PASS


crudini --set $neutronconf DEFAULT notify_nova_on_port_status_changes True
crudini --set $neutronconf DEFAULT notify_nova_on_port_data_changes True
crudini --set $neutronconf nova 

crudini --set $neutronconf nova auth_url  http://controller:35357
crudini --set $neutronconf nova auth_type  password
crudini --set $neutronconf nova project_domain_name  Default
crudini --set $neutronconf nova user_domain_name  Default
crudini --set $neutronconf nova region_name  RegionOne
crudini --set $neutronconf nova project_name  service
crudini --set $neutronconf nova username  nova
crudini --set $neutronconf nova password  $NOVA_PASS

crudini --set $neutronconf oslo_concurrency lock_path /var/lib/neutron/tmp

ml2conf=/etc/neutron/plugins/ml2/ml2_conf.ini
crudini --set $ml2conf ml2 type_drivers flat,vlan
crudini --set $ml2conf ml2 tenant_network_types 
crudini --set $ml2conf ml2 mechanism_drivers linuxbridge
crudini --set $ml2conf ml2 extension_drivers port_security
crudini --set $ml2conf ml2_type_flat flat_networks provider
crudini --set $ml2conf securitygroup enable_ipset  True

bridgeconf=/etc/neutron/plugins/ml2/linuxbridge_agent.ini
crudini --set $bridgeconf linux_bridge physical_interface_mappings provider:eth2
crudini --set $bridgeconf vxlan enable_vxlan False
crudini --set $bridgeconf securitygroup enable_security_group  True
crudini --set $bridgeconf securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

dhcpconf=/etc/neutron/dhcp_agent.ini 
crudini --set $dhcpconf DEFAULT interface_driver neutron.agent.linux.interface.BridgeInterfaceDriver
crudini --set $dhcpconf DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
crudini --set $dhcpconf DEFAULT enable_isolated_metadata True

## on controller
metaconf=/etc/neutron/metadata_agent.ini 
crudini --set $metaconf DEFAULT nova_metadata_ip  controller
crudini --set $metaconf DEFAULT metadata_proxy_shared_secret  $METADATA_SECRET

novaconf=/etc/nova/nova.conf 

crudini --set $novaconf neutron
crudini --set $novaconf neutron url  http://controller:9696
crudini --set $novaconf neutron auth_url  http://controller:35357
crudini --set $novaconf neutron auth_type  password
crudini --set $novaconf neutron project_domain_name  Default
crudini --set $novaconf neutron user_domain_name  Default
crudini --set $novaconf neutron region_name  RegionOne
crudini --set $novaconf neutron project_name  service
crudini --set $novaconf neutron username  neutron
crudini --set $novaconf neutron password  $NEUTRON_PASS
crudini --set $novaconf neutron service_metadata_proxy  True
crudini --set $novaconf neutron metadata_proxy_shared_secret $METADATA_SECRET
