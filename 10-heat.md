# heat install

## 1. CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **heat** database:

```sql
CREATE DATABASE heat;
```

3. Grant proper access to **heat** user and exit:

```sql
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' identified by 'password123';
exit
```

### Create openstack objects

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **heat** user and add role:

```bash
openstack user create --domain default --password password123 heat

openstack role add --project service --user heat admin
```

* Create **heat** services:

```bash
openstack service create --name heat \
  --description "Orchestration" orchestration

openstack service create --name heat-cfn \
  --description "Orchestration" cloudformation
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  orchestration $i http://controller:8004/v1/\(tenant_id\)s; \
  done

for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  cloudformation $i http://controller:8000/v1; \
  done
```

### Create heat domain

```
openstack domain create --description "Stack projects and users" heat

openstack user create --domain heat --password password123 heat_domain_admin

openstack role add --domain heat --user-domain heat --user heat_domain_admin admin

openstack role create heat_stack_owner

openstack role add --project demoproject --user demouser heat_stack_owner

openstack role create heat_stack_user
```

### Install and configure components

1. Install packages:

```bash
apt-get install heat-api heat-api-cfn heat-engine -y
```

2. Backup an sanitize **/etc/heat/heat.conf**:

```bash
cp -p /etc/heat/heat.conf /etc/heat/heat.conf.bak
grep -Ev '^(#|$)' /etc/heat/heat.conf.bak > /etc/heat/heat.conf
```

2. Edit **/etc/heat/heat.conf** sections:

```yaml
[DEFAULT]
#...
transport_url = rabbit://openstack:password123@controller
heat_metadata_server_url = http://controller:8000
heat_waitcondition_server_url = http://controller:8000/v1/waitcondition
stack_domain_admin = heat_domain_admin
stack_domain_admin_password = password123
stack_user_domain_name = heat

[database]
#...
connection = mysql+pymysql://heat:password123@controller/heat

[keystone_authtoken]
#...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = heat
password = password123

[trustee]
#...
auth_type = password
auth_url = http://controller:5000
username = heat
password = password123
user_domain_name = default

[clients_keystone]
#...
auth_uri = http://controller:5000
```

### Finalize install

1. Populate database

```
su -s /bin/sh -c "heat-manage db_sync" heat
```

2. Restart services

```
service heat-api restart
service heat-api-cfn restart
service heat-engine restart
```

### Verify ops

* Verify service list

```
openstack orchestration service list
```

### Install horizon UI

1. install packages (TODO verify this package)

```
apt heat-dashboard-common -y
```

2. restart apache

```
service apache2 restart
```


