# Controller setup

# CONTROLLER NODE

## SQL Database

### Install and configure componenets

1. Install packages:

```bash
apt install mariadb-server python3-pymysql -y
```

2. Run mysql setup:

```bash
mysql_secure_installation
```

3. Create and edit **/etc/mysql/mariadb.conf.d/99-openstack.cnf**

```yaml
[mysqld]
bind-address = 10.10.10.11
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

### Finalize installation

1. Restart database service:

```
service mysql restart
```

## Message Queue

### Install and configure components

1. Install packages:

```
apt install rabbitmq-server -y
```

2. Add **openstack** user:

```
rabbitmqctl add_user openstack password123
```

3. Apply permissions for **openstack** user:
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## Memcached

### Install and configure components

1. Install packages:

```
apt install memcached python3-memcache -y
```

2. Edit **/etc/memcached.conf** and change following setting:

```
-l 10.10.10.11
```

### Finalize installation

1. Restart memcached service:

```
service memcached restart
```

## Etcd

### Install and configure components

1. Install packages:

```
apt install etcd -y
```

2. Edit **/etc/default/etcd** and change following setting:

```yaml
ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://10.10.10.11:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.11:2379"
ETCD_LISTEN_PEER_URLS="http://10.10.10.11:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.10.11:2379"
```

### Finalize installation

1. Enable and restart etcd service:

```
systemctl enable etcd
systemctl restart etcd
```