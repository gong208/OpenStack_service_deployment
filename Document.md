This document records the process of installing the OpenStack service of the Yoga version on a virtual machine with a system of Ubuntu 22.04.

- [Environment](#environment)
  - [OpenStack Packages](#openstack-packages)
  - [SQL database for Ubuntu](#sql-database-for-ubuntu)
  - [Message queue for Ubuntu](#message-queue-for-ubuntu)
  - [Memcached for Ubuntu](#memcached-for-ubuntu)
  - [Etcd for Ubuntu](#etcd-for-ubuntu)
- [Minimum deployment for Yoga](#minimum-deployment-for-yoga)
  - [Identity service](#identity-service)
    - [Prerequisite](#prerequisite)
    - [Install and configure components](#install-and-configure-components)
    - [Create a domain, project, and user](#create-a-domain-project-and-user)
    - [Creating scripts and using scripts](#creating-scripts-and-using-scripts)
    - [Verify operation](#verify-operation)
  - [Image service](#image-service)
    - [Prerequisites](#prerequisites)
    - [Installations and configurations](#installations-and-configurations)
    - [Finalize installation¶](#finalize-installation)
    - [Verify operation](#verify-operation-1)
  - [Placement service](#placement-service)
    - [Prerequisites](#prerequisites-1)
    - [Installation and configurations](#installation-and-configurations)
    - [Finalize installation¶](#finalize-installation-1)
    - [Verify operation](#verify-operation-2)
  - [Compute service](#compute-service)
    - [Prerequisite](#prerequisite-1)
    - [Installation and configurations](#installation-and-configurations-1)
    - [Finalize installation](#finalize-installation-2)

# Environment


## OpenStack Packages

OpenStack Yoga is available by default using Ubuntu 22.04 LTS.


## SQL database for Ubuntu

1. Install package for Ubuntu

Check the version of the system using:

``` zsh
# cat /etc/issue
```

For Ubuntu 22.04, run:

```zsh
# apt install mariadb-server python3-pymysql
```



2. Setup management IP address for accesses from other nodes

Create and edit the /etc/mysql/mariadb.conf.d/99-openstack.cnf file.

```zsh
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

```zsh
$ ip address
```



3. Restart the database service:

```zsh
# service mysql restart
```



4. Secure the database service by running the mysql_secure_installation script. In particular, choose a suitable password for the database root account:

```zsh
# mysql_secure_installation

```


## Message queue for Ubuntu

1. Install the package:

``` zsh
# apt install rabbitmq-server

```

2. Add the openstack user, and **replace RABBIT_PASS with a suitable password**.

```zsh
# rabbitmqctl add_user openstack RABBIT_PASS
```
3. Permit configuration, write, and read access for the openstack user:

```zsh
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## Memcached for Ubuntu
1. Install the packages: For Ubuntu 18.04 and newer versions use:

```zsh
# apt install memcached python3-memcache
```

2. Edit the /etc/memcached.conf file and **configure the service to use the management IP** 

**address of the controller node**. This is to enable access by other nodes via the management network: \
```
-l MANAGEMENT_IP_ADDRESS
```

3. Finalize installation: Restart the Memcached service:

``` zsh
# service memcached restart
```

## Etcd for Ubuntu

1. Install the etcd package:

```zsh
# apt install etcd
```

2. Edit the /etc/default/etcd file and set the ETCD_INITIAL_CLUSTER, ETCD_INITIAL_ADVERTISE_PEER_URLS, ETCD_ADVERTISE_CLIENT_URLS, ETCD_LISTEN_CLIENT_URLS to **the management IP address of the controller node** to enable access by other nodes via the management network:

```
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

3. Enable and restart the etcd service:

```zsh
# systemctl enable etcd
# systemctl restart etcd
```


# Minimum deployment for Yoga

## Identity service

### Prerequisite

1. Use the database access client to connect to the database server as the **<code>root</code></strong> user:

```zsh
# mysql
```

2. Create the keystone database:

``` \
MariaDB [(none)]> CREATE DATABASE keystone;

```

3. Grant proper access to the keystone database, **replace KEYSTONE_DBPASS** with a suitable password:

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
```

4. Exit the database access client.

### Install and configure components

1. Register domain name “controller” with the **management IP address** in the file /etc/hosts so that controller can be resolved to the **management IP address**.

```
MANAGEMENT_IP_ADDRESS controller
```

2. Run the following command to install the packages:

```zsh
# apt install keystone
```

3. Edit the /etc/keystone/keystone.conf file. In the [database] section, configure database access, replace **KEYSTONE_DBPASS** with the password you chose for the database:

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

``` zsh
# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

5. Initialize Fernet key repositories:

```zsh
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

6. Bootstrap the Identity service, replace **ADMIN_PASS** with a suitable password for an administrative user.:

```zsh
# keystone-manage bootstrap --bootstrap-password ADMIN_PASS  --bootstrap-admin-url http://controller:5000/v3/  --bootstrap-internal-url http://controller:5000/v3/  --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne
```

7. Edit the /etc/apache2/apache2.conf file and configure the ServerName option to reference the controller node:

```
ServerName controller
```

The ServerName entry will need to be added if it does not already exist.

8. Restart the Apache service:

```zsh
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

### Create a domain, project, and user
1. Although the “default” domain already exists from the keystone-manage bootstrap step in this guide, a formal way to create a new domain would be:

```zsh
$ openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 2f4f80574fd84fe6ba9067228ae0a50c |
| name        | example                          |
| tags        | []                               |
+-------------+----------------------------------+
```

2. This guide uses a service project that contains a unique user for each service that you add to your environment. Create the service project:

``` zsh
$ openstack project create --domain default --description "Service Project" service

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 24ac7f19cd944f4cba1d77469b2a73ed |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

3. Regular (non-admin) tasks should use an unprivileged project and user. As an example, this guide creates the myproject project and myuser user.

Create the myproject project:

```zsh
$ openstack project create --domain default --description "Demo Project" myproject

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 231ad6e7ebba47d6a1e57e1cc07ae446 |
| is_domain   | False                            |
| name        | myproject                        |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

4. Create the myuser user:

```zsh
$ openstack user create --domain default --password-prompt myuser

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | aeda23aa78f44e859900e22c24817832 |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

5. Create the myrole role:

```zsh
$ openstack role create myrole

+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 997ce8d05fc143ac97d83fdfb5998552 |
| name      | myrole                           |
+-----------+----------------------------------+
```

6. Add the myrole role to the myproject project and myuser user:

```zsh
$ openstack role add --project myproject --user myuser myrole
```

### Creating scripts and using scripts
1. Creating the scripts¶
* Create client environment scripts for the admin and demo projects and users. Future portions of this guide reference these scripts to load appropriate credentials for client operations.
* Create and edit the `admin-openrc` file and add the following content, replace ADMIN_PASS with the password you chose for the admin user in the Identity service.
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

* Create and edit the `demo-openrc` file and add the following content, replace DEMO_PASS with the password you chose for the demo user in the Identity service.
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=DEMO_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
2. Using the script
* To run clients as a specific project and user, you can simply load the associated client environment script prior to running them. For example:
* Load the admin-openrc file to populate environment variables with the location of the Identity service and the admin project and user credentials:
```
$ . admin-openrc
```

* Request an authentication token:
```zsh
$ openstack token issue

+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:44:35.659723Z                                     |
| id         | gAAAAABWvjYj-Zjfg8WXFaQnUd1DMYTBVrKw4h3fIagi5NoEmh21U72SrRv2trl |
|            | JWFYhLi2_uPR31Igf6A8mH2Rw9kv_bxNo1jbLNPLGzW_u5FC7InFqx0yYtTwa1e |
|            | eq2b0f6-18KZyQhs7F3teAta143kJEWuNEYET-y7u29y0be1_64KYkM7E       |
| project_id | 343d245e850143a096806dfaefa9afdc                                |
| user_id    | ac3377633149401296f6c0d92d79dc16                                |
+------------+-----------------------------------------------------------------+
```


### Verify operation
1. Unset the temporary OS_AUTH_URL and OS_PASSWORD environment variable:
```zsh
$ unset OS_AUTH_URL OS_PASSWORD
```

1. As the admin user, request an authentication token:
```zsh
$ openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue
Password:
+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:14:07.056119Z                                     |
| id         | gAAAAABWvi7_B8kKQD9wdXac8MoZiQldmjEO643d-e_j-XXq9AmIegIbA7UHGPv |
|            | atnN21qtOMjCFWX7BReJEQnVOAj3nclRQgAYRsfSU_MrsuWb4EDtnjU7HEpoBb4 |
|            | o6ozsA_NmFWEpLeKy0uNn_WeKbAhYygrsmQGA49dclHVnz-OMVLiyM9ws       |
| project_id | 343d245e850143a096806dfaefa9afdc                                |
| user_id    | ac3377633149401296f6c0d92d79dc16                                |
+------------+-----------------------------------------------------------------+
```
This command uses the password for the admin user.

1. As the myuser user created in the previous, request an authentication token:
```zsh
$ openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name myproject --os-username myuser token issue

Password:
+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:15:39.014479Z                                     |
| id         | gAAAAABWvi9bsh7vkiby5BpCCnc-JkbGhm9wH3fabS_cY7uabOubesi-Me6IGWW |
|            | yQqNegDDZ5jw7grI26vvgy1J5nCVwZ_zFRqPiz_qhbq29mgbQLglbkq6FQvzBRQ |
|            | JcOzq3uwhzNxszJWmzGC7rJE_H0A_a3UFhqv8M4zMRYSbS2YF0MyFmp_U       |
| project_id | ed0b60bf607743088218b0a533d5943f                                |
| user_id    | 58126687cbcc4888bfa9ab73a2256f27                                |
+------------+-----------------------------------------------------------------+
```

## Image service
### Prerequisites
1. Create database for glance

```
# mysql
```
Create the glance database:

```
MariaDB [(none)]> CREATE DATABASE glance;
```

Grant proper access to the glance database, replace **GLANCE_DBPASS** with a suitable password.

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
```

2. Source the admin credentials to gain access to admin-only CLI commands:

```zsh
$ . admin-openrc
```

3. Create the user for glance and the service entity.

* create glance user
```zsh
$ openstack user create --domain default --password-prompt glance

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3f4e777c4062483ab8d9edd7dff829df |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

* Add the admin role to user glance under service project:
```zsh
$ openstack role add --project service --user glance admin
```

* Create the glance service entity:
```zsh
$ openstack service create --name glance --description "OpenStack Image" image

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

1. Create the image service and api endpoints:
```zsh
$ openstack endpoint create --region RegionOne image public http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 340be3625e9b4239a6415d034e98aace |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a6e4b153c2ae4c919eccfdbb7dceb5d2 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0c37ed58103f4300a84ff125a539032d |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

### Installations and configurations
1. Install the packages:
```zsh
# apt install glance
```
2. Edit the `/etc/glance/glance-api.conf` file and complete the following actions:

* In the [database] section, configure database access, replace **GLANCE_DBPASS** with the password you chose for the Image service database.
```
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```
* In the [keystone_authtoken] and [paste_deploy] sections, configure Identity service access, replace **GLANCE_PASS** with the password you chose for the glance user in the Identity service.
```
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
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone
```
Comment out or remove any other options in the [keystone_authtoken] section.

* In the [glance_store] section, configure the local file system store and location of image files:
```
[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
* In the [oslo_limit] section, configure access to keystone:

```
[oslo_limit]
auth_url = http://controller:5000
auth_type = password
user_domain_id = default
username = MY_SERVICE
system_scope = all
password = MY_PASSWORD
endpoint_id = ENDPOINT_ID
region_name = RegionOne
```

As for the `[oslo_limit]` section, use the username and password assigned to glance service. The endpoint_id should be the internal endpoint. Either it should be the endpoint of keystone or of glance is yet to be clarified. I used the internal endpoint of glance and the service operation runs correctly.

* Make sure that the MY_SERVICE account has reader access to system-scope resources (like limits):

```zsh
$ openstack role add --user MY_SERVICE --user-domain Default --system all reader
```
See the oslo_limit docs for more information about configuring the unified limits client.

1. Populate the Image service database:
```zsh
# su -s /bin/sh -c "glance-manage db_sync" glance
```

### Finalize installation¶
Restart the Image services:
```zsh
# service glance-api restart
```

### Verify operation
1. Source the admin credentials to gain access to admin-only CLI commands:
```zsh
$ . admin-openrc
```

2. Download the source image:
```zsh
$ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```
Install wget if your distribution does not include it.

3. Upload the image to the Image service using the QCOW2 disk format, bare container format, and public visibility so all projects can access it:
```zsh
$ glance image-create --name "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility=public

+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 133eae9fb1c98f45894a4e60d8736619                     |
| container_format | bare                                                 |
| created_at       | 2015-03-26T16:52:10Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/cc5c6982-4910-471e-b864-1098015901b5/file |
| id               | cc5c6982-4910-471e-b864-1098015901b5                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | ae7a98326b9c455588edd2656d723b9d                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13200896                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2015-03-26T16:52:10Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```
OpenStack generates IDs dynamically, so you will see different values in the example command output.

4. Confirm upload of the image and validate attributes:
```zsh
$ glance image-list

+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 38047887-61a7-41ea-9b49-27987d5e8bb9 | cirros | active |
+--------------------------------------+--------+--------+
```
`openstack image list` is an alternative command

## Placement service
### Prerequisites
1. Create database for placement service
* Use the database access client to connect to the database server as the root user:
```zsh
# mysql
```

* Create the placement database:
```
MariaDB [(none)]> CREATE DATABASE placement;
```

* Grant proper access to the database, replace **PLACEMENT_DBPASS** with a suitable password:

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';
```

* Exit the database access client.

1. Create user, service entity, and endpoint for placement service.
* Source the admin credentials to gain access to admin-only CLI commands:
```zsh
$ . admin-openrc
```

* Create a Placement service user using your chosen **PLACEMENT_PASS**:
```zsh
$ openstack user create --domain default --password-prompt placement

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | fa742015a6494a949f67629884fc7ec8 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

* Add the Placement user to the service project with the admin role:
```zsh
$ openstack role add --project service --user placement admin
```
This command provides no output.

* Create the Placement API entry in the service catalog:
  
```zsh
$ openstack service create --name placement --description "Placement API" placement

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | 2d1a27022e6e4185b86adac4444c495f |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
```

* Create the Placement API service endpoints:
Depending on your environment, the URL for the endpoint will vary by port (possibly 8780 instead of 8778, or no port at all) and hostname. You are responsible for determining the correct URL.

```zsh
$ openstack endpoint create --region RegionOne placement public http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2b1b2637908b4137a9c2e0470487cbc0 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2d1a27022e6e4185b86adac4444c495f |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
placement internal http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 02bcda9a150a4bd7993ff4879df971ab |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2d1a27022e6e4185b86adac4444c495f |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
placement admin http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 3d71177b9e0f406f98cbff198d74b182 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2d1a27022e6e4185b86adac4444c495f |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

### Installation and configurations
1. Install the packages:

```zsh
# apt install placement-api
```

2. Edit the `/etc/placement/placement.conf` file and complete the following actions:

* In the [placement_database] section, configure database access, replace **PLACEMENT_DBPASS** with the password you chose for the placement database.

```
[placement_database]
# ...
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
```

* In the [api] and [keystone_authtoken] sections, configure Identity service access, comment out or remove any other options in the [keystone_authtoken] section. The value of user_name, password, project_domain_name and user_domain_name need to be in sync with your keystone config.

```
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS
Replace PLACEMENT_PASS with the password you chose for the placement user in the Identity service.
```

* Populate the placement database:

```zsh
# su -s /bin/sh -c "placement-manage db sync" placement
```

### Finalize installation¶
Reload the web server to adjust to get new configuration settings for placement.

```zsh
# service apache2 restart
```

### Verify operation

1. Source the admin credentials to gain access to admin-only CLI commands:
```zsh
$ . admin-openrc
```

2. Perform status checks to make sure everything is in order:
```zsh
$ placement-status upgrade check
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
```

3. Run some commands against the placement API:
* Install the osc-placement plugin:
```zsh
$ pip3 install osc-placement
```
* List available resource classes and traits:

```zsh
$ openstack --os-placement-api-version 1.2 resource class list --sort-column name
+----------------------------+
| name                       |
+----------------------------+
| DISK_GB                    |
| IPV4_ADDRESS               |
| ...                        |

$ openstack --os-placement-api-version 1.6 trait list --sort-column name
+---------------------------------------+
| name                                  |
+---------------------------------------+
| COMPUTE_DEVICE_TAGGING                |
| COMPUTE_NET_ATTACH_INTERFACE          |
| ...                                   |
```

## Compute service
### Prerequisite
1. Create the databases:
* Use the database access client to connect to the database server as the root user:
* 
```zsh
# mysql
```
* Create the nova_api, nova, and nova_cell0 databases:

```
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;
```

* Grant proper access to the databases, replace **NOVA_DBPASS** with a suitable password.

``` \
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
```

* Exit the database access client.

2. Create user, service entity, and endpoints for nova.
* Source the admin credentials to gain access to admin-only CLI commands:

```zsh
$ . admin-openrc
```

* Create the nova user:

```zsh
$ openstack user create --domain default --password-prompt nova

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8a7dbf5279404537b1c7b86c033620fe |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

* Add the admin role to the nova user:

```zsh
$ openstack role add --project service --user nova admin
```

* Create the nova service entity:

```zsh
$ openstack service create --name nova --description "OpenStack Compute" compute

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 060d59eac51b4594815603d75a00aba2 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

* Create the Compute API service endpoints:

```zsh
$ openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | 3c1caa473bfe4390a11e7177894bcc7b          |
| interface    | public                                    |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 060d59eac51b4594815603d75a00aba2          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1               |
+--------------+-------------------------------------------+

$ openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | e3c918de680746a586eac1f2d9bc10ab          |
| interface    | internal                                  |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 060d59eac51b4594815603d75a00aba2          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1               |
+--------------+-------------------------------------------+

$ openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | 38f7af91666a47cfb97b4dc790b94424          |
| interface    | admin                                     |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 060d59eac51b4594815603d75a00aba2          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1               |
+--------------+-------------------------------------------+
```

### Installation and configurations
1. Install the packages:

```zsh
# apt install nova-api nova-conductor nova-novncproxy nova-scheduler
```

2. Edit the `/etc/nova/nova.conf` file and complete the following actions:
* In the `[api_database]` and `[database]` sections, configure database access:

```
[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
Replace NOVA_DBPASS with the password you chose for the Compute databases.
```

* In the `[DEFAULT]` section, configure RabbitMQ message queue access, replace **RABBIT_PASS** with the password you chose for the openstack account in RabbitMQ.

```
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
```

* In the `[api]` and `[keystone_authtoken]` sections, configure Identity service access, replace **NOVA_PASS** with the password you chose for the nova user in the Identity service, comment out or remove any other options in the `[keystone_authtoken]` section.

```
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS
```

* In the `[service_user]` section, configure service user tokens, replace **NOVA_PASS** with the password you chose for the nova user in the Identity service.

```
[service_user]
send_service_user_token = true
auth_url = https://controller/identity
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = NOVA_PASS
```

* In the `[DEFAULT]` section, configure the my_ip option to use the **management interface IP** address of the controller node:

```
[DEFAULT]
# ...
my_ip =MANAGEMENT_IP_ADDRESS
```

* Configure the `[neutron]` section of `/etc/nova/nova.conf`. Refer to the [Networking service install guide](https://docs.openstack.org/neutron/yoga/install/controller-install-ubuntu.html#configure-the-compute-service-to-use-the-networking-service) for more information.

* In the `[vnc]` section, configure the VNC proxy to use the **management interface IP address** of the controller node:

```
[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

* In the `[glance]` section, configure the location of the Image service API:

```
[glance]
# ...
api_servers = http://controller:9292
```

* In the [oslo_concurrency] section, configure the lock path:

```
[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp
```

* **Due to a packaging bug, remove the log_dir option from the `[DEFAULT]` section.**

* In the `[placement]` section, configure access to the Placement service, replace **PLACEMENT_PASS** with the password you choose for the placement service user created when installing Placement. Comment out or remove any other options in the `[placement]` section.

```
[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

* Populate the nova-api database:

```zsh
# su -s /bin/sh -c "nova-manage api_db sync" nova
```

* Register the cell0 database:

```zsh
# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

* Create the cell1 cell:

```zsh
# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

* Populate the nova database:

```zsh
# su -s /bin/sh -c "nova-manage db sync" nova
```

* Verify nova cell0 and cell1 are registered correctly:

```
# su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
|  Name |                 UUID                 |                   Transport URL                    |                     Database Connection                      | Disabled |
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                       none:/                       | mysql+pymysql://nova:****@controller/nova_cell0?charset=utf8 |  False   |
| cell1 | f690f4fd-2bc5-4f15-8145-db561a7b9d3d | rabbit://openstack:****@controller:5672/nova_cell1 | mysql+pymysql://nova:****@controller/nova_cell1?charset=utf8 |  False   |
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
```

### Finalize installation
1. Restart the Compute services:

```zsh
# service nova-api restart
# service nova-scheduler restart
# service nova-conductor restart
# service nova-novncproxy restart
```