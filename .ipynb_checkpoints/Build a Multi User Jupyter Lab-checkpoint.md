# Build a multi user Jupyter Lab
This guide will walk you through the steps to install Jupyter Hub and configure to provide a Jupyter Lab environment to your IPA users.
1) Installing JupyterHub;
2) Configuring JupyterHub to authenticate using your IPA server; and
3) Installing core set of Jupyter notebooks to all users.

### References

* [JupyterHub Installation Guide](https://jupyterhub.readthedocs.io/en/stable/installation-guide.html)

### Guide Examples
The examples in this guide use:
* *192.168.86.5* as the IP address for your lab server;
* *lab.example.com* as the Fully Qualified Domain Name ("FQDN") of your lab server; and
* *ns1.example.com* as the FQDN of your IPA server

## Prerequisite Configuration
You will need:
* a [configured TrueNAS installation](./Configure%20TrueNAS%20to%20use%20FreeIPA%20for%20authentication.md); 
* a [configured FreeIPA Server](./Build%20IPA%20Server%20with%20Let's%20Encrypt!.md);
* a fresh RHEL8 installation [configured to use your IPA server and NAS](./Client%20Setup.md); and
* a [client SSH certificate issued to your lab]. See the *Issuing a Let's Encrypt! HTTPS certificate to your client machine* section in the in the [IPA Server](./Build%20IPA%20Server%20with%20Let's%20Encrypt!.md) guide.

## Install JupyterHub

Install latest version of python

```
sudo dnf install python39
```

Install NodeJS and npm

```
sudo dnf install nodejs
sudo dnf install npm
```

Create a virtual environment for your JupyterHub install, set permissions and activate it

```
sudo mkdir /opt/jupyterhub
sudo python3.9 -m venv /opt/jupyterhub/venv
sudo chmod -R 777 /opt/jupyterhub
cd /opt/jupyterhub
. venv/bin/activate
```

Install JupyterHub and npm configurable http proxy

```
pip install jupyterhub jupyterlab notebook
npm install -g configurable-http-proxy
```

Install common packages for all users. e.g.

```
pip install pandas
pip install scipi
pip install statsmodels
pip install jupyterlab-spellchecker
... any others that you want your users to have access to.
```

Generate the jupyter hub configuration file

```
jupyterhub --generate-config
```

Edit the *jupyterhub_config.py*  file and:
* un-comment the *c.JupyterHub.ip* line and set to '';
* Set the port to 443 by adding *c.JupyterHub.port = 443*.

```
c.JupyterHub.ip = ''
c.JupyterHub.port = 443
```

Create a service file to run JupyterHub as a daemon. Create file named *jupyer.service* under */etc/systemd/system* with the following contents.

```
[Unit]
Description=Jupyter Hub

[Service]
Type=simple
PIDFile=/run/jupyter.pid
Environment=PATH=$PATH:/usr/bin:/usr/local/bin:/opt/jupyterhub/venv/bin
ExecStart=/bin/bash -c "/opt/jupyterhub/venv/bin/jupyterhub --debug"
User=root
Group=root
WorkingDirectory=/opt/jupyterhub
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Open the https port

```
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --reload
```

Enable and start the JupyterHub service

```
sudo systemctl enable jupyter
sudo systemctl start jupyter
```

You should now be able to access Jupyter lab by typing the address of your lab server from another machine in your network. You won't be able to log in yet.

## Configure JupyterHub to authenticate using your IPA server


Stop the jupyter service

```
sudo systemctl stop jupyter
```

Install the Jupyter Hub LDAB Authenticator from the JupyterHyb venv

```
pip install jupyterhub-ldapauthenticator
```

Edit */opt/jupyterhub/jupyterhub_config.py* and configure LDAP authentication by adding the following lines using your ipa serveer in place of ipa.example.com and your domain dc ldap records in place of dc=example,dc=com:

```
# We are authenticating using ldap
c.JupyterHub.authenticator_class = 'ldapauthenticator.LDAPAuthenticator'
c.LDAPAuthenticator.server_address='ns1.example.com'
c.LDAPAuthenticator.bind_dn_template='uid={username},cn=users,cn=accounts,dc=example,dc=com'
c.LDAPAuthenticator.use_ssl = True
c.LDAPAuthenticator.allowed_groups = ['cn=labusers,cn=groups,cn=accounts,dc=example,dc=com']
```

Create the labusers group and add your users to it.

```
kinit admin
group-add labusers--desc="Users of lab environment"
ipa group-add-member labusers --users=exampleuser
```

Restart the jupyter service

```
sudo systemctl start jupyter
```

## Configure JupyterHub to use your Let's Encrypt! certificates

Stop the jupyter service

```
sudo systemctl stop jupyter
```

Edit */opt/jupyterhub/jupyterhub_config.py* and configure the ssl_cert and ssl_key parameters to point to your lets encrypt certificates.

```
c.JupyterHub.ssl_cert = '/etc/letsencrypt/lab.example.com/cert.pem'
c.JupyterHub.ssl_key = '/etc/letsencrypt/lab.example.com/privkey.pem'
```

Restart the jupyter service

```
sudo systemctl start jupyter
```

##  Install core set of Jupyter notebooks to all users