# Install Cinder

## 1: CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **cinder** database:

```sql
CREATE DATABASE cinder;
```

3. Grant proper access to **cinder** user:

```sql
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' identified by 'password123';
exit
```

### Create Openstack items

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **cinder** user and add role:

```bash
openstack user create --domain default --password password123 cinder

openstack role add --project service --user cinder admin
```

* Create **cinder v2** services:

```bash
openstack service create --name cinderv2 \
  --description "Openstack Block Storage" volumev2

openstack service create --name cinderv3 \
  --description "Openstack Block Storage" volumev3
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  volumev2 $i http://controller:8776/v2/%\(project_id\)s; \
  done

for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  volumev3 $i http://controller:8776/v3/%\(project_id\)s; \
  done
```

### Install Packages

* Install packages:

```bash
apt install cinder-api cinder-scheduler -y
```

### Configure server

1. Backup an sanitize **/etc/cinder/cinder.conf**:

```bash
cp -p /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
grep -Ev '^(#|$)' /etc/cinder/cinder.conf.bak > /etc/cinder/cinder.conf
```

2. Edit **/etc/cinder/cinder.conf** sections:

```yaml
[database]
# ...
connection = mysql+pymysql://cinder:password123@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:password123@controller
auth_strategy = keystone
my_ip = 10.10.10.11

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = password123

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```

* Fix packaging bug

```bash
install -d /var/lib/cinder/tmp -o cinder -g cinder
```

### Populate database

```bash
su -s /bin/sh -c "cinder-manage db sync" cinder
```

## Configure compute to use cinder:

* Edit **/etc/nova/nova.conf**:

```yaml
[cinder]
os_region_name = RegionOne
```

## Finalize install

```bash
service nova-api restart
service cinder-scheduler restart
service apache2 restart
```

## 2. STORAGE NODE

### Install Packages

* Install packages:

```bash
apt install lvm2 thin-provisioning-tools tgt -y
```

* NOTE: docs don't call for installing tgt, but should be required. tgt conf file for cinder wasn't created. maybe caused by coupling the installs. Decoupling here.

```bash
apt install cinder-volume -y
```

* NOTE: if **/etc/tgt/conf.d/cinder.conf** wasn't created:

```bash
echo 'include /var/lib/cinder/volumes/*' >> /etc/tgt/conf.d/cinder.conf
```

### Create LVM volumes

* Use appropriate block device name instead of /dev/vdb

```bash
pvcreate /dev/vdb

vgcreate cinder-volumes /dev/vdb
```

### Create LVM filter

```yaml
devices {
...
filter = [ "a/sdb/", "r/.*/" ]
```

* If LVM used on OS, include those vols. Do same on compute nodes:

```yaml
filter = [ "a/sda/", "a/sdb/", "r/.*/" ]
```

### Configure server

1. Backup an sanitize **/etc/cinder/cinder.conf**:

```bash
cp -p /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
grep -Ev '^(#|$)' /etc/cinder/cinder.conf.bak > /etc/cinder/cinder.conf
```

2. Edit **/etc/cinder/cinder.conf** sections:

```yaml
[database]
# ...
connection = mysql+pymysql://cinder:password123@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:password123@controller
auth_strategy = keystone
my_ip = 10.10.10.14
enabled_backends = lvm
glance_api_servers = http://controller:9292

# comment these out... using tgt here rather than lioadm
#iscsi_helper = lioadm
#volume_name_template = volume-%s
#volume_group = cinder-volumes

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = password123

[lvm]
# ...
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```

### Finalize Install

```bash
service tgt restart
service cinder-volume restart
```

### Verify Ops

```bash
source .adminrc

openstack volume service list
```





