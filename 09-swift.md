# Install swift

> ![Swift logo](/images/swift.png)

## 1. CONTROLLER NODE

### Create Openstack objects

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **swift** user and add role:

```bash
openstack user create --domain default --password password123 swift

openstack role add --project service --user swift admin
```

* Create **swift** service:

```bash
openstack service create --name swift \
  --description "Openstack Object Storage" object-store
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  object-store $i http://controller:8080/v1/AUTH_%\(project_id\)s; \
  done
```

### Install and configure componenets

1. Install packages:

```bash
apt install swift swift-proxy python-swiftclient \
  python-keystoneclient python-keystonemiddleware \
  memcached -y
```

2. Create **/etc/swift** dir:

```bash
install -d /etc/swift -o swift -g swift
```

3. Get copy of **proxy-server.conf**:

```bash
curl -o /etc/swift/proxy-server.conf https://opendev.org/openstack/swift/raw/branch/stable/wallaby/etc/proxy-server.conf-sample
```

4. Backup an sanitize **/etc/swift/proxy-server.conf**:

```bash
cp -p /etc/swift/proxy-server.conf /etc/swift/proxy-server.conf.bak
grep -Ev '^(#|$)' /etc/swift/proxy-server.conf.bak > /etc/swift/proxy-server.conf
```

3. Edit **/etc/swift/proxy-server.conf** sections:

```yaml
[DEFAULT]
#...
bind_port = 8080
user = swift
swift_dir = /etc/swift

[pipeline:main]
# remove tempurl and tempauth, add authtoken and keystoneauth
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server

[app:proxy-server]
use = egg:swift#proxy
#...
account_autocreate = True

[filter:keystoneauth]
use = egg:swift#keystoneauth
#...
operator_roles = admin,user

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
#...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = swift
password = password123
delay_auth_decision = True

[filter:cache]
use = egg:swift#memcache
#...
memcache_servers = controller:11211
```

## 1. OBJECT STORAGE NODE










