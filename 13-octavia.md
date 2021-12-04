# Octavia Install

## 1. CONTROLLER NODE

### Database setup

```bash
mysql
```

```bash
CREATE DATABASE octavia;
```

```bash
GRANT ALL PRIVILEGES ON octavia.* TO 'octavia'@'localhost' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON octavia.* TO 'octavia'@'%' IDENTIFIED BY 'password123';
exit
```

### Create Openstack resources

```bash
source .adminrc
```

```bash
openstack user create --domain default --password password123 octavia
```

```bash
openstack role add --project service --user octavia admin
```

```bash
openstack service create --name octavia --description "OpenStack Octavia" load-balancer
```

```bash
for i in public internal admin; do \
  openstack endpoint create --region RegionOne \
  load-balancer $i http://controller:9876; done
```

### Create octavia rc file

```bash
cat << EOF >> ~/.octaviarc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=service
export OS_USERNAME=octavia
export OS_PASSWORD=password123
export OS_AUTH_URL=http://controller:5000
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export OS_VOLUME_API_VERSION=3
EOF
```

### Install packages

```bash
apt install octavia-api octavia-health-manager octavia-housekeeping octavia-worker python3-octavia python3-octaviaclient -y
```

### Build amphora image

* Option 1. Uses a snap. Much less involved, but produces larger image

```bash
snap install octavia-diskimage-retrofit --beta --devmode

cd /var/snap/octavia-diskimage-retrofit/common/tmp

wget https://cloud-images.ubuntu.com/minimal/releases/focal/release/ubuntu-20.04-minimal-cloudimg-amd64.img

octavia-diskimage-retrofit ubuntu-20.04-minimal-cloudimg-amd64.img ubuntu-amphora-haproxy-amd64.qcow2
```

* Option 2. Uses build script. More control and produces smaller image

```bash

git clone https://opendev.org/openstack/octavia.git --branch stable/wallaby

# Use your preferred/system version of python
apt install python3.X-venv qemu-utils git kpartx debootstrap -y 

python3 -m venv octavia_disk_image_create

source octavia_disk_image_create/bin/activate

cd octavia/diskimage-create

pip install -r requirements.txt

./diskimage-create.sh
```

### Upload amphora image

```bash
source ~/.octavia-openrc
```

```bash
openstack image create --disk-format qcow2 --container-format bare \
  --private --tag amphora --file ubuntu-amphora-haproxy-amd64.qcow2 ubuntu-amphora-x64-haproxy
```

### Create amphora flavor (may need 5G size if used snap builder)

```bash
openstack flavor create --id 200 --vcpus 1 --ram 1024 \
  --disk 2 "lb.m1.small" --private
```

### Create certs

```bash
sudo mkdir -p /etc/octavia/certs/private
sudo chmod 755 /etc/octavia -R
git clone https://opendev.org/openstack/octavia.git
cd octavia/bin/
source create_dual_intermediate_CA.sh
sudo cp -p etc/octavia/certs/server_ca.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/server_ca-chain.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/server_ca.key.pem /etc/octavia/certs/private
sudo cp -p etc/octavia/certs/client_ca.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/client.cert-and-key.pem /etc/octavia/certs/private
chown -R octavia /etc/octavia/certs
```

### Create sec group and rules

```bash
source ~/.octavia-openrc
```

```bash
openstack security group create lb-mgmt-sec-grp
openstack security group rule create --protocol icmp lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 22 lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 80 lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 443 lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 9443 lb-mgmt-sec-grp
openstack security group create lb-health-mgr-sec-grp
openstack security group rule create --protocol udp --dst-port 5555 lb-health-mgr-sec-grp
```

### Create keypair for testing (optional)

```
openstack keypair create octavia-mgmt
```

### Network setup

1. Create dhcp config

```
sudo mkdir -m755 -p /etc/dhcp/octavia
sudo cp octavia/etc/dhcp/dhclient.conf /etc/dhcp/octavia
```

2. Create LB network (scary!)

```
OCTAVIA_MGMT_SUBNET=172.16.0.0/24
OCTAVIA_MGMT_SUBNET_START=172.16.0.100
OCTAVIA_MGMT_SUBNET_END=172.16.0.254
OCTAVIA_MGMT_PORT_IP=172.16.0.2

openstack network create lb-mgmt-net
openstack subnet create --subnet-range $OCTAVIA_MGMT_SUBNET --allocation-pool \
  start=$OCTAVIA_MGMT_SUBNET_START,end=$OCTAVIA_MGMT_SUBNET_END \
  --network lb-mgmt-net lb-mgmt-subnet

SUBNET_ID=$(openstack subnet show lb-mgmt-subnet -f value -c id)
PORT_FIXED_IP="--fixed-ip subnet=$SUBNET_ID,ip-address=$OCTAVIA_MGMT_PORT_IP"

MGMT_PORT_ID=$(openstack port create --security-group \
  lb-health-mgr-sec-grp --device-owner Octavia:health-mgr \
  --host=$(hostname) -c id -f value --network lb-mgmt-net \
  $PORT_FIXED_IP octavia-health-manager-listen-port)

MGMT_PORT_MAC=$(openstack port show -c mac_address -f value \
  $MGMT_PORT_ID)

sudo ip link add o-hm0 type veth peer name o-bhm0
NETID=$(openstack network show lb-mgmt-net -c id -f value)
BRNAME=brq$(echo $NETID|cut -c 1-11)
sudo brctl addif $BRNAME o-bhm0
sudo ip link set o-bhm0 up

sudo ip link set dev o-hm0 address $MGMT_PORT_MAC
sudo iptables -I INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT
sudo dhclient -v o-hm0 -cf /etc/dhcp/octavia
```

* perhaps a better method here?

```
NETID=$(openstack network show lb-mgmt-net -c id -f value)

BRNAME=brq$(echo $NETID|cut -c 1-11)

ip link add o-hm0 type veth peer name o-bhm0

ip link set dev o-hm0 address $MGMT_PORT_MAC

ip link set o-hm0 up
ip link set o-bhm0 up

ip addr add $OCTAVIA_MGMT_PORT_IP/24 dev o-hm0

brctl addif $BRNAME o-bhm0

iptables -I INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT
```

* Things tend to go screwy here after running dhclient. Still need to figure out real fix. rebooting then restarting neutron services seems to help. Investigate further.

### config files

```yaml
[DEFAULT]
transport_url = rabbit://openstack:password123@controller:5672

[api_settings]
bind_host = 0.0.0.0
bind_port = 9876
auth_strategy = keystone
api_base_uri = http://controller:9876

[certificates]
server_certs_key_passphrase = insecure-key-do-not-use-this-key
ca_private_key_passphrase = not-secure-passphrase
ca_private_key = /etc/octavia/certs/private/server_ca.key.pem
ca_certificate = /etc/octavia/certs/server_ca.cert.pem

[controller_worker]
client_ca = /etc/octavia/certs/client_ca.cert.pem

amp_image_owner_id = 86e49ccf2d874fff8981f746fcc05198
amp_image_tag = amphora
amp_ssh_key_name = octavia-mgmt # for testing
amp_secgroup_list = <lb-mgmt-sec-grp-id>
amp_boot_network_list = <lb-mgmt-net-id>
amp_flavor_id = 200
network_driver = allowed_address_pairs_driver
compute_driver = compute_nova_driver
amphora_driver = amphora_haproxy_rest_driver

[database]
connection = mysql+pymysql://octavia:password123@controller/octavia

[haproxy_amphora]
server_ca = /etc/octavia/certs/server_ca-chain.cert.pem
client_cert = /etc/octavia/certs/private/client.cert-and-key.pem

[health_manager]
bind_ip = 0.0.0.0
bind_port = 5555
controller_ip_port_list = 172.16.0.2:5555

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = octavia
password = password123

[oslo_messaging]
topic = octavia_prov

[service_auth]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = octavia
password = password123
```

### Run database migrations

```
octavia-db-manage --config-file /etc/octavia/octavia.conf upgrade head
```

### Finalize install

```
systemctl restart octavia-api octavia-health-manager octavia-housekeeping octavia-worker
```










