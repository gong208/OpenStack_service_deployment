This document records the process of installing the OpenStack service of the Yoga version on a virtual machine with a system of Ubuntu 22.04.


# Environment


## OpenStack Packages

OpenStack Yoga is available by default using Ubuntu 22.04 LTS.


## SQL database for Ubuntu



1. Install package for Ubuntu

Check the version of the system using:

```

# cat /etc/issue

```

For Ubuntu 22.04, run:

```

# apt install mariadb-server python3-pymysql

```



2. Setup management IP address for accesses from other nodes

Create and edit the /etc/mysql/mariadb.conf.d/99-openstack.cnf file.

```

# touch /etc/mysql/mariadb.conf.d/99-openstack.cnf

```

Create a [mysqld] section, and **set the bind-address key to the management IP address of the controller node**:

```

[mysqld]

bind-address = MANAGEMENT_IP_ADDRESS

default-storage-engine = innodb

innodb_file_per_table = on

max_connections = 4096

collation-server = utf8_general_ci

character-set-server = utf8

```

The management IP address is determined when setting up the virtual machine system. You can check all the interfaces using following command

```

ip address

```



3. Restart the database service:

```

# service mysql restart

```



4. Secure the database service by running the mysql_secure_installation script. In particular, choose a suitable password for the database root account:

``` \
# mysql_secure_installation

```


## Message queue for Ubuntu



1. Install the package:

``` \
# apt install rabbitmq-server

```



2. Add the openstack user, and **replace RABBIT_PASS with a suitable password**.

```

# rabbitmqctl add_user openstack RABBIT_PASS

```



3. Permit configuration, write, and read access for the openstack user:

```

# rabbitmqctl set_permissions openstack ".*" ".*" ".*"

```


## Memcached for Ubuntu



1. Install the packages: For Ubuntu 18.04 and newer versions use:

```

# apt install memcached python3-memcache

```



1. Edit the /etc/memcached.conf file and **configure the service to use the management IP** 

**address of the controller node**. This is to enable access by other nodes via the management network: \
```

-l MANAGEMENT_IP_ADDRESS

```



2. Finalize installation: Restart the Memcached service:

``` \
# service memcached restart

```


## Etcd for Ubuntu



1. Install the etcd package:

```

# apt install etcd

```

Edit the /etc/default/etcd file and set the ETCD_INITIAL_CLUSTER, ETCD_INITIAL_ADVERTISE_PEER_URLS, ETCD_ADVERTISE_CLIENT_URLS, ETCD_LISTEN_CLIENT_URLS to **the management IP address of the controller node** to enable access by other nodes via the management network:

``` \
ETCD_NAME="controller"

ETCD_DATA_DIR="/var/lib/etcd"

ETCD_INITIAL_CLUSTER_STATE="new"

ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"

ETCD_INITIAL_CLUSTER="controller=http://MANAGEMENT_IP_ADDRESS:2380"

ETCD_INITIAL_ADVERTISE_PEER_URLS="http://MANAGEMENT_IP_ADDRESS:2380"

ETCD_ADVERTISE_CLIENT_URLS="http://MANAGEMENT_IP_ADDRESS:2379"

ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"

ETCD_LISTEN_CLIENT_URLS="http://MANAGEMENT_IP_ADDRESS:2379"

```



2. Enable and restart the etcd service:

```

# systemctl enable etcd

# systemctl restart etcd

```


# Minimum deployment for Yoga


## Identity service


### Prerequisite



1. Use the database access client to connect to the database server as the **<code>root</code></strong> user:

```

# mysql

```



2. Create the keystone database:

``` \
MariaDB [(none)]> CREATE DATABASE keystone;

```



3. Grant proper access to the keystone database, **replace KEYSTONE_DBPASS** with a suitable password:

``` \
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \

IDENTIFIED BY 'KEYSTONE_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \

IDENTIFIED BY 'KEYSTONE_DBPASS';

```



4. Exit the database access client.


### Install and configure components



1. Register domain name “controller” with the management IP address in the file /etc/hosts so that controller can be resolved to the management IP address.

```

MANAGEMENT_IP_ADDRESS controller

```



2. Run the following command to install the packages:

```

# apt install keystone

```



3. Edit the /etc/keystone/keystone.conf file. In the [database] section, configure database access, replace KEYSTONE_DBPASS with the password you chose for the database:

```

[database]

connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

```

Comment out or remove any other connection options in the [database] section.

In the [token] section, configure the Fernet token provider:

``` \
[token]

provider = fernet

```



4. Populate the Identity service database:

``` \
# su -s /bin/sh -c "keystone-manage db_sync" keystone

```



5. Initialize Fernet key repositories:

```

# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

```



6. Bootstrap the Identity service, replace ADMIN_PASS with a suitable password for an administrative user.:

```

# keystone-manage bootstrap --bootstrap-password ADMIN_PASS  --bootstrap-admin-url http://controller:5000/v3/  --bootstrap-internal-url http://controller:5000/v3/  --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne

```



7. Edit the /etc/apache2/apache2.conf file and configure the ServerName option to reference the controller node:

```

ServerName controller

```

The ServerName entry will need to be added if it does not already exist.



8. Restart the Apache service:

```

# service apache2 restart

```



9. Configure the administrative account by setting the proper environmental variables, replace ADMIN_PASS with the password used in the keystone-manage bootstrap command in [keystone-install-configure-ubuntu](https://docs.openstack.org/keystone/ussuri/install/keystone-install-ubuntu.html#keystone-install-configure-ubuntu):

``` \
export OS_USERNAME=admin

export OS_PASSWORD=ADMIN_PASS

export OS_PROJECT_NAME=admin

export OS_USER_DOMAIN_NAME=Default

export OS_PROJECT_DOMAIN_NAME=Default

export OS_AUTH_URL=http://controller:5000/v3

export OS_IDENTITY_API_VERSION=3

```

These values shown here are the default ones created from keystone-manage bootstrap. For convenience, running the commands in a script file is recommended. 
