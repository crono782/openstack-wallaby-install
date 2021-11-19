# Horizon

## Install packages

```bash
apt install openstack-dashboard -y
```

## Modify config file **/etc/openstack-dashboard/local_settings.py**:

```yaml
OPENSTACK_HOST = "controller"

ALLOWED_HOSTS = ['*']

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

# IF ONLY PROVIDER NEWORKS
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "TIME_ZONE"
```

* Modify file **/etc/apache2/conf-available/openstack-dashboard.conf**:

```yaml
WSGIApplicationGroup %{GLOBAL}
```

* Finalize install:

```bash
systemctl reload apache2
```


