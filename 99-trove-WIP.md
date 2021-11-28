mysql

CREATE DATABASE trove;

GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'localhost' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'%' IDENTIFIED BY 'password123';
exit

source .adminrc

openstack user create --domain default --password password123 trove

openstack role add --project service --user trove admin

openstack service create --name trove --description "Database" database

for i in public internal admin; do \
  openstack endpoint create --region RegionOne \
  database $i http://controller:8779/v1.0/%\(tenant_id\)s; done
  
apt-get install trove-api trove-taskmanager trove-conductor python3-troveclient -y


/etc/trove/trove.conf

[DEFAULT]
log_dir = /var/log/trove
transport_url = rabbit://openstack:password123@controller:5672
control_exchange = trove
trove_api_workers = 5
network_driver = trove.network.neutron.NeutronDriver
taskmanager_manager = trove.taskmanager.manager.Manager
default_datastore = mysql
cinder_volume_type = lvm-trove
reboot_time_out = 300
usage_timeout = 900
agent_call_high_timeout = 1200

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = trove
password = password123

[service_credentials]
auth_url = http://controller:5000
region_name = RegionOne
project_domain_name = Default
user_domain_name = Default
project_name = service
username = trove
password = password123

[database]
connection = mysql+pymysql://trove:password123@controller/trove

[mariadb]
tcp_ports = 3306,4444,4567,4568

[mysql]
tcp_ports = 3306

[postgresql]
tcp_ports = 5432

[redis]
tcp_ports = 6379,16379


/etc/trove/trove-guestagent.conf

[DEFAULT]
log_file = trove-guestagent.log
log_dir = /var/log/trove/
ignore_users = os_admin
control_exchange = trove
transport_url = rabbit://openstack:password123@controller:5672/
command_process_timeout = 60
use_syslog = False

[service_credentials]
auth_url = http://controller:5000
region_name = RegionOne
project_domain_name = Default
user_domain_name = Default
project_name = service
username = trove
password = password123




su -s /bin/sh -c "trove-manage db_sync" trove



service trove-api restart
service trove-taskmanager restart
service trove-conductor restart



wget https://tarballs.opendev.org/openstack/trove/images/trove-wallaby-guest-ubuntu-bionic.qcow2

openstack image create trove-guest-ubuntu-bionic --file=trove-wallaby-guest-ubuntu-bionic.qcow2 --disk-format=qcow2 --container-format=bare --tag=trove --private 

openstack image list

openstack volume type create lvm-trove --private



openstack volume type create lvm-trove --private 



openstack datastore version create 10.3 mariadb mariadb abe2a0a0-c1b7-47f1-8983-d9baf23edc9f


su -s /bin/sh -c "trove-manage db_load_datastore_config_parameters mariadb 10.3 /usr/lib/python3/dist-packages/trove/templates/mariadb/validation-rules.json"


mkdir /etc/trove/cloudinit

/etc/trove/cloudinit/mariadb.cloudinit

#cloud-config
write_files:
  - path: /etc/trove/controller.conf
    content: |
      CONTROLLER=10.10.10.11
  - path: /ec/hosts
    content: |
      10.10.10.11 controller
    append: true
runcmd:
  - chmod 644 /etc/trove/controller.conf



chown -R trove /etc/trove/cloudinit


openstack database instance create mariadb-103 \
  --flavor m1.small \
  --size 1 \
  --nic net-id=270ab477-71c1-4fe9-b7c4-674d7dd38630 \
  --database mydb --users dqueen:password123 \
  --datastore mariadb --datastore-version 10.3 \
  --is-public \
  --allowed-cidr 10.10.10.0/24 --allowed-cidr 203.0.113.0/24 
  
 openstack database instance list
 
 mysql -h <floating ip> -u user -p


# notes
if need to debug, create keypair in service project, add "nova_keypair = <name> in trove.conf under [DEFAULT]. update security group rules on generated group in service project to allow ping/ssh. node cannot resolve controller so add to hosts file via cloud init... dont forget to allow traffic via iptables if needed

# some to-dos, investigate better tag based approach to image selection. maybe dedicated trove mgmt network and sec group for better handling. add postgresql and mysql... cinder-scheduler doesn't enable by default








