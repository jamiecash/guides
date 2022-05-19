# Build a database server
This guide will walk you through the steps to install a Postgresql database server that authenticates through your IPA server.
1) Installing Postgresql;
2) Configuring Postgresql to authenticate using your IPA server;
3) Configuring Postgresql to create database roles using IPA roles; and
4) Integrating your database with your lab.

### References

* [Add secondary drive](https://devops.ionos.com/tutorials/add-secondary-drives-to-centos-rhel.html)
* [Setup postgresql14 on RHEL8](https://citizix.com/how-to-install-and-configure-postgres-14-on-centos-8/)
* [Configure Postgresql server to authenticate against IPA/IdM](https://access.redhat.com/solutions/674323_
* [Installing iPythion SQL Magic and jupyter-sql exteensions](https://docs.kyso.io/guides/sql-interface-within-jupyterlab)

### Guide Examples
The examples in this guide use:
* *192.168.86.6* as the IP address for your database server;
* *db.example.com* as the Fully Qualified Domain Name ("FQDN") of your database server;
* *ns1.example.com* as the FQDN of your IPA server;
* *lab.example.com* as the FQDN of your lab environment.

## Prerequisite Configuration
You will need:
* a [configured TrueNAS installation](./Configure%20TrueNAS%20to%20use%20FreeIPA%20for%20authentication.md); 
* a [configured FreeIPA Server](./Build%20IPA%20Server%20with%20Let's%20Encrypt!.md); and
* a fresh RHEL8 installation [configured to use your IPA server and NAS](./Client%20Setup.md).

## Install Postgresql

### If you are using a second drive to hold your database, you will need to install it first

Check if the drive has been installed and is available. Note the device name

```
sudo fdisk -l
```

Use *fdisk* to create the primary partition, passing the name of the device (e.g. /dev/sdb). When prompted for command, enter n, to create a new partition, then enter p to create a primary partition, then 1 to create the first partition. Use the default cylinder start and end values to use the full space available. Enter w to write. Check if it has been created and note the partition name. For */dev/sdb*, this will be */dev/sdb1*.

```
sudo fdisk /dev/vdb
sudo fdisk -l /dev/vdb
```

Create an ext4 filesystem on the partition

```
sudo mkfs.ext4 /dev/sdb1
```

Create a mount point and mount the secondary drive to this mount point:

```
sudo mkdir /mnt/data
sudo mount /dev/sdb1 /mnt/data/
```

Get the drive/partition UUID and note it

```
 ls -l /dev/disk/by-uuid/
```

Add it to */etc/fstab*. MYUUID should be replaced with the UUID of the partition that you noted earlier

```
sudo bash -c 'echo "UUID=MYUUID /mnt/data        xfs    defaults        0 0" >> /etc/fstab
sudo mount -a
```

Check that it has been mounted

```
df -h
```

### Install postgresql

Install the latest postgres module

```
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

Remove the default one to avoid any conflicts

```
sudo dnf -qy module disable postgresql
```

Install the server and contrib package

```
sudo dnf install -y postgresql14-server postgresql14-devel postgresql14-contrib
```

Install postgresql server

```
sudo /usr/pgsql-14/bin/postgresql-14-setup
```

If you have a separate disk for the database data, run initdb to create a new cluster in the new location. The location must be owned and the initdb command must be run as the postgres user. You will then need to override the service file to point to the new data location. You should then remove the contents of the old data directory and symlink to the new one. You can then follow the rest of this guide using the default locations.

```
sudo mkdir /mnt/data/pgdata
sudo chown postgres:postgres /mnt/data/pgdata
sudo su postgres
/usr/pgsql-14/bin/initdb -D /mnt/data/pgdata
exit
sudo mkdir postgresql-14.service.d
sudo bash -c 'echo -e "[Service]\nEnvironment=PGDATA=/mnt/data/pgdata/" > /etc/systemd/system/postgresql-14.service.d/override.conf'
sudo systemctl daemon-reload
sudo rm -R /var/lib/pgsql/14/data
ln -s  sudo ln -s /mnt/data/pgdata/ /var/lib/pgsql/14/data
```

Allow remote connections by editing */var/lib/pgsql/14/data/postgresql.conf*, uncommenting *listen_address* and setting it to '\*'. Uncomment the *port* line.

```
listen_addresses = '*'
port = 5432
```

Enable and start the service

```
sudo systemctl enable postgresql-14
sudo systemctl start postgresql-14
```

Check it is running and verify that it is working

```
sudo systemctl status postgresql-14
sudo -u postgres psql -c "SELECT version();"
```

Add a firewall rule to allow incoming conections

```
sudo firewall-cmd --add-service=postgresql --permanent
```

## Configure Postgresql to authenticate using your IPA server

We will enable Kerberos authentication against your free IPA server. This will enable other registered servers (e.g. your lab) to connect using a kerberos ticket for single sign-on.

Add a service in your IPA server

```
ipa service-add postgres/`hostname`
```

As postgres user, get a keytab for it.

```
sudo su postgres
ipa-getkeytab -s ns1.example.com -p postgres/`hostname`@EXAMPLE.COM -k /var/lib/pgsql/14/data/pg.keytab
```

Edit */var/lib/pgsql/14/data/pg_hba.conf* to enable GSSPI authentication. Add the following line, using the ip addrsss range for your internal network.

```
# Network connections to authenticate using kerberos
host    all             all             192.168.86.0/24         gss krb_realm=IPA.ALGOBUILDER.CO.UK include_realm=0
```

Edit */var/lib/pgsql/14/data/postgresql.conf* to add GSSAPI configuration settings

```
krb_server_keyfile = '/var/lib/pgsql/14/pg.keytab'
```

## Configure Postgresql to create database roles using IPA roles

Your IPA users will be able to connect to your database server, but not your database, as there won't be any roles for them. We will install and configure *ldap2pg* to automatically create roles on pgsql for users with the correct roles assigned on your IPA server.

Install ldap2pd. You will need to install some dependency packages and add the directory containing pg_config to the path first.

```
sudo su 
dnf install python3-devel openldap-devel
PATH=$PATH:/usr/pgsql-14/bin
pip3 install ldap2pg psycopg2-binary
```

Add database groups to your IPA server. The below will add the following four groups:

* dbadmin: Full admin access
* dbdev: CRUD and DDL
* dbuser: CRUD only
* dbreader: Read only access

```
kinit admin
ipa group-add dbadmin --desc="Database Administrators"
ipa group-add dbdev --desc="Database Developers. DDL & CRUD"
ipa group-add dbuser --desc="Database Users. CRUD only"
ipa group-add dbreader --desc="Database readers. Read only access to data"
```

Create a configuration file in the folder where ldap2pg has been installed and set its permissions so that it can only be read by the postgres user.

```
sudo touch /usr/local/bin/ldap2pg.yml
sudo chown postgres:postgres /usr/local/bin/ldap2pg.yml
sudo chmod 600 /usr/local/bin/ldap2pg.yml
```

Edit the config file and add the following contents, replacing [YOUR ADMIN PASSWORD] with your IPA admin users password. This will map your IPA groups to the corresponding database permissions.

```
postgres:
  dsn: postgres://postgres@localhost:5432/
ldap:
  uri: ldaps://ns1.example.com
  binddn: uid=admin,cn=users,cn=accounts,dc=example,dc=com
  password: [YOUR ADMIN PASSWORD]
privileges:
  ro:
  - __connect__
  - __usage_on_schemas__
  - __select_on_tables__
  crud:
  - ro
  - __insert_on_tables__
  - __update_on_tables__
  - __delete_on_tables__
  ddl:
  - crud
  - __all_on_schemas__
  - __all_on_sequences__
  - __all_on_tables__
sync_map:
- ldapsearch:
    base: cn=dbadmin,cn=groups,cn=accounts,dc=example,dc=com
  role:
    name: '{member.uid}'
    options: LOGIN SUPERUSER
  grant:
    privilege: ddl
    role: '{member.uid}'
    database: __all__
- ldapsearch:
    base: cn=dbdev,cn=groups,cn=accounts,dc=example,dc=com
  role:
    name: '{member.uid}'
    options: LOGIN
  grant:
    privilege: ddl
    role: '{member.uid}'
    database: __all__
- ldapsearch:
    base: cn=dbuser,cn=groups,cn=accounts,dc=example,dc=com
  role:
    name: '{member.uid}'
    options: LOGIN
  grant:
    privilege: crud
    role: '{member.uid}'
    database: __all__
- ldapsearch:
    base: cn=dbreader,cn=groups,cn=accounts,dc=example,dc=com
  role:
    name: '{member.uid}'
    options: LOGIN
  grant:
    privilege: ro
    role: '{member.uid}'
    database: __all__
```

Check that it works. Run as postgres. This wont make any changes but will output the changes that would be made as a dry-run. 

```
sudo su postgres
/usr/local/bin/ldap2pg
```

If there are no errors, run to make changes.

```
sudo su postgres
/usr/local/bin/ldap2pg -N
```

Create a crontab to run this every hour.

```
sudo bash -c 'echo "0 * * * * postgres cd /usr/local/bin && /usr/local/bin/ldap20g -N" >> /etc/crontab'
```

## Creating a database for your lab

Create a database for your lab

```
sudo su postgres
psql
create database lab;
exit
```

Edit */usr/local/bin/ldap2pg.yml* and add a rule to provide ddl access for members of the labusers group

```
- ldapsearch:
    base: cn=labusers,cn=groups,cn=accounts,dc=example,dc=com
  role:
    name: '{member.uid}'
    options: LOGIN
  grant:
    privilege: ddl
    role: '{member.uid}'
    database: lab
```

## Integrating your database with your lab

The following will install a sql extension to allow you to write SQL commands directly into JupyterLab, and an extension to manage your database through a GUI

ssh to your lab server

Upgrade NodeJS as the version from the RHEL8 repo is not compatable with some extensions

```
dnf remove nodejs
curl -fsSL https://rpm.nodesource.com/setup_17.x | bash -
dnf install nodejs -y
node -v
```

As root, activate your virtual environment

```
sudo su
cd /opt/jupyterhub/
. venv/bin/activate
```

Install the ipython-sql and jupyterlab-sql extensions; enable them; and rebuild the lab

**jupyterlab-sql is incompatable with the latest version of JupyterLab. Awaiting update from author.**

```
pip install ipython-sql
pip install jupyterlab_sql
jupyter serverextension enable jupyterlab_sql --py --sys-prefix
jupyter lab build
```

Restart your lab

```
sudo systemctl restart jupyter
```

From your lab, you should now be able to run sql commands. Connect to your lab database and list all tables

```
%load_ext sql
%%sql

postgresql://db.ipa.algobuilder.co.uk/lab
select * from pg_catalog.pg_tables;
```