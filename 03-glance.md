# Glance Install

> ![Glance logo](/images/glance.png)

## CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **keystone** database:

```sql
CREATE DATABASE glance;
```

3. Grant proper access to **glance** user:

```sql
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' identified by 'password123';
```

### Prerequisites

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **glance** user and add role:

```bash
openstack user create --domain default --password-prompt glance

openstack role add --project service --user glance admin
```

* Create **glance** service:

```bash
openstack service create --name glance \
  --description "OpenStack Image" image
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  image $i http://controller:9292; \
  done
```

### Install and configure componenets

1. Install packages:

```bash
apt install glance python3-boto3 -y
```

2. Backup an sanitize **/etc/glance/glance-api.conf**:

```bash
cp -p /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bak
grep -Ev '^(#|$)' /etc/glance/glance-api.conf.bak > /etc/glance/glance-api.conf
```

2. Edit **/etc/glance/glance-api.conf** sections:

```yaml
[DEFAULT]
# ...
enabled_backends = file:file

[database]
# ...
connection = mysql+pymysql://glance:password123@controller/glance

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = password123

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
default_store = file

[file]
# ...
filesystem_store_datadir = /var/lib/glance/images/
```

3. Populate database:

```bash
su -s /bin/sh -c "glance-manage db_sync" glance
```

### Finalize installation

1. Restart image service:

```bash
service glance-api restart
```

### Verify ops

1. Source .adminrc

```bash
source .adminrc
```

2. Download CirrOS image:

```bash
wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img
```

3. Upload image to glance:

```bash
openstack image create \
  --file cirros-0.5.2-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public cirros
```

4. Verify

```bash
openstack image list
```