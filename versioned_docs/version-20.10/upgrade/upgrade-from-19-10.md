---
id: upgrade-from-19-10
title: Upgrade from Centreon 19.10
---

This chapter describes how to upgrade your Centreon platform from version 19.10
to version 20.10.

## Perform a backup

Be sure that you have fully backed up your environment for the following
servers:

- Central server
- Database server

## Update the RPM signing key

For security reasons, the keys used to sign Centreon RPMs are rotated regularly. The last change occurred on October 14, 2021. When upgrading from an older version, you need to go through the [key rotation procedure](../security/key-rotation#existing-installation), to remove the old key and install the new one.

## Upgrade the Centreon Central server

### Update the Centreon repository

Run the following commands:

```shell
yum install -y https://yum.centreon.com/standard/20.10/el7/stable/noarch/RPMS/centreon-release-20.10-4.el7.centos.noarch.rpm
```

> If you are using a CentOS environment, you must install the *Software
> Collections* repositories with the following command:
>
> ```shell
> yum install -y centos-release-scl-rh
> ```

### Upgrade the Centreon solution

> Since 20.04, Centreon uses **MariaDB 10.3**.
>
> This upgrade process will only upgrade Centreon components first.
>
> MariaDB will be upgraded afterwards.

Clean yum cache:

```shell
yum clean all --enablerepo=*
```

Then upgrade all the components with the following command:

```shell
yum update centreon\*
```

> Accept new GPG keys from the repositories as needed.

### Additional actions

#### Configure Apache API access

If you had a custom apache configuration, upgrade process through RPM did not
update it.

> If you use https, you can follow [this
> procedure](../administration/secure-platform#securing-the-apache-web-server)

You'll then need to add API access section to your configuration file:
**/opt/rh/httpd24/root/etc/httpd/conf.d/10-centreon.conf**

Only lines with "+" symbol must be taken into account.

```diff
+Alias /centreon/api /usr/share/centreon
Alias /centreon /usr/share/centreon/www/

+<LocationMatch ^/centreon/(?!api/latest/|api/beta/|api/v[0-9]+/|api/v[0-9]+\.[0-9]+/)(.*\.php(/.*)?)$>
+  ProxyPassMatch fcgi://127.0.0.1:9042/usr/share/centreon/www/$1
+</LocationMatch>

+<LocationMatch ^/centreon/api/(latest/|beta/|v[0-9]+/|v[0-9]+\.[0-9]+/)(.*)$>
+  ProxyPassMatch fcgi://127.0.0.1:9042/usr/share/centreon/api/index.php/$1
+</LocationMatch>

ProxyTimeout 300

<Directory "/usr/share/centreon/www">
DirectoryIndex index.php
Options Indexes
AllowOverride all
Order allow,deny
Allow from all
Require all granted
<IfModule mod_php5.c>
php_admin_value engine Off
</IfModule>

+    FallbackResource /centreon/index

AddType text/plain hbs
</Directory>

+<Directory "/usr/share/centreon/api">
+    Options Indexes
+    AllowOverride all
+    Order allow,deny
+    Allow from all
+    Require all granted
+    <IfModule mod_php5.c>
+        php_admin_value engine Off
+    </IfModule>
+
+    AddType text/plain hbs
+</Directory>

RedirectMatch ^/$ /centreon
```

Then, restart apache service :

```shell
systemctl restart httpd24-httpd
```

### Finalizing the upgrade

Before starting the web upgrade process, reload the Apache server with the
following command:

```shell
systemctl reload httpd24-httpd
```

Then log on to the Centreon web interface to continue the upgrade process:

Click on **Next**:

![image](../assets/upgrade/web_update_1.png)

Click on **Next**:

![image](../assets/upgrade/web_update_2.png)

The release notes describe the main changes. Click on **Next**:

![image](../assets/upgrade/web_update_3.png)

This process performs the various upgrades. Click on **Next**:

![image](../assets/upgrade/web_update_4.png)

Your Centreon server is now up to date. Click on **Finish** to access the login
page:

![image](../assets/upgrade/web_update_5.png)

If the Centreon BAM module is installed, refer to the
[upgrade procedure](../service-mapping/upgrade).

### Post-upgrade actions

#### Upgrade extensions

From `Administration > Extensions > Manager`, upgrade all extensions, starting
with the following:

- License Manager,
- Plugin Packs Manager,
- Auto Discovery.

Then you can upgrade all other commercial extensions.

#### Start the tasks manager

Since 20.04, Centreon has changed his tasks manager from *Centcore* to
*Gorgone*.

To act this change, run the following commands:

```shell
systemctl stop centcore
systemctl enable gorgoned
systemctl start gorgoned
```

Engine statistics that have been collected by *Centcore* will know be collected
by *Gorgone*.

Change the rights on the statistics RRD files by running the following command:

```shell
chown -R centreon-gorgone /var/lib/centreon/nagios-perf/*
```

#### Restart monitoring processes

Centreon Broker component has changed its configuration file format.

It now uses JSON instead of XML.

To make sure Broker and Engine's Broker module are using new configuration
files, follow this steps:

1. Deploy Central's configuration from the Centreon web UI by following [this
procedure](../monitoring/monitoring-servers/deploying-a-configuration),
2. Restart both Broker and Engine on the Central server by running this
command:

```shell
systemctl restart cbd centengine
```

### Upgrade MariaDB server

The MariaDB components can now be upgraded.

Be aware that MariaDB strongly recommends to upgrade the server through each
major release. Please refer to the [official MariaDB
documentation](https://mariadb.com/kb/en/upgrading/) for further information.

You then need to upgrade from 10.1 to 10.2 and from 10.2 to 10.3.

That is why Centreon provides both 10.2 and 10.3 versions on its stable
repositories.

> Refer to the official MariaDB documentation to know more about this process:
>
> - https://mariadb.com/kb/en/upgrading-from-mariadb-101-to-mariadb-102/#how-to-upgrade
> - https://mariadb.com/kb/en/upgrading-from-mariadb-102-to-mariadb-103/#how-to-upgrade

#### Configuration

`innodb_additional_mem_pool_size` parameter has been removed since MariaDB 10.2,
so you should remove it Zfrom file **/etc/my.cnf.d/centreon.cnf**

```diff
#
# Custom MySQL/MariaDB server configuration for Centreon
#
[server]
innodb_file_per_table=1

open_files_limit = 32000

key_buffer_size = 256M
sort_buffer_size = 32M
join_buffer_size = 4M
thread_cache_size = 64
read_buffer_size = 512K
read_rnd_buffer_size = 256K
max_allowed_packet = 8M

# For 4 Go Ram
-#innodb_additional_mem_pool_size=512M
#innodb_buffer_pool_size=512M

# For 8 Go Ram
-#innodb_additional_mem_pool_size=1G
#innodb_buffer_pool_size=1G
```

#### Upgrade from 10.1 to 10.2

Follow those summarized steps to perform the upgrade in the way recommended by
MariaDB:

1. Stop the mariadb service:

```shell
systemctl stop mariadb
```

2. Uninstall current 10.1 version:

```shell
rpm --erase --nodeps --verbose MariaDB-server MariaDB-client MariaDB-shared MariaDB-compat MariaDB-common
```

3. Install 10.2 version:

```shell
yum install MariaDB-server-10.2\* MariaDB-client-10.2\* MariaDB-shared-10.2\* MariaDB-compat-10.2\* MariaDB-common-10.2\*
```

4. Start the mariadb service:

```shell
systemctl start mariadb
```

5. Launch the MariaDB upgrade process:

```shell
mysql_upgrade
```

> Refer to the [official documentation](https://mariadb.com/kb/en/mysql_upgrade/)
> if errors occur during this last step.

#### Upgrade from 10.2 to 10.3

Follow those summarized steps to perform the upgrade in the way recommended by
MariaDB:

1. Stop the mariadb service:

```shell
systemctl stop mariadb
```

2. Uninstall current 10.2 version:

```shell
rpm --erase --nodeps --verbose MariaDB-server MariaDB-client MariaDB-shared MariaDB-compat MariaDB-common
```

3. Install 10.3 version:

```shell
yum install MariaDB-server-10.3\* MariaDB-client-10.3\* MariaDB-shared-10.3\* MariaDB-compat-10.3\* MariaDB-common-10.3\*
```

4. Start the mariadb service:

```shell
systemctl start mariadb
```

5. Launch the MariaDB upgrade process:

```shell
mysql_upgrade
```

> Refer to the [official documentation](https://mariadb.com/kb/en/mysql_upgrade/)
> if errors occur during this last step.

#### Enable MariaDB on startup

Execute the following command:

```shell
systemctl enable mariadb
```

## Upgrade the Remote Servers

This procedure is the same than to upgrade a Centreon Central server.

## Upgrade the Pollers

### Update the Centreon repository

Run the following command:

```shell
yum install -y https://yum.centreon.com/standard/20.10/el7/stable/noarch/RPMS/centreon-release-20.10-4.el7.centos.noarch.rpm
```

> If you are using a CentOS environment, you must install the *Software
> Collections* repositories with the following command:
>
> ```shell
> yum install -y centos-release-scl-rh
> ```

### Upgrade the Centreon solution

Clean yum cache:

```shell
yum clean all --enablerepo=*
```

Upgrade all the components with the following command:

```shell
yum update centreon\*
```

> Accept new GPG keys from the repositories as needed.

Start and enable **gorgoned**:

```shell
systemctl start gorgoned
systemctl enable gorgoned
```

### Post-upgrade actions

Due to new configuration file format for Engine's Broker module, the
configuration needs to be re-deployed.

Deploy Poller's configuration from the Centreon web UI by following
[this procedure](../monitoring/monitoring-servers/deploying-a-configuration),
and choose *Restart* method for Engine process.

## Communications

By default, the communication between Central and Pollers or Remote Servers
will still be using SSH protocol.

Consider changing the communication protocol by following the
[Change communication from SSH to ZMQ](../monitoring/monitoring-servers/communications#change-communication-from-ssh-to-zmq)
procedure.

## Secure your platform

Don't forget to secure your Centreon platform following our
[recommendations](../administration/secure-platform)