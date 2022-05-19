# Configure TrueNAS to integrate with FreeIPA
This guide will walk you through the steps to configure your TrueNAS software to authenticate users through FreeIPA and automatically creare NAS folders on your NAS. This guide will cover:
1) Adding TrueNAS to FreeIPA to resolve DNS;
2) Configuring TrueNAS to authenticate users using FreeIPA;
3) Creating user home root folder on TrueNAS;
4) Configuring client machine to create and use NAS home folders for users; and
5) Issuing a Let's Encrypt! certificate to secure your web interface.

## About this Guide
This guide will modify a TrueNAS installation to use a FreeIPA server to authenticate users. Home folders for all domain users will be created on the NAS and the TrueNAS web interface will be secured with a HTTPS Let's Encrypt! certificate.

### References

* [Auto create home directories on RHEL8](https://unixcop.com/configuring-authselect-sssd-centos-rhel-8/)
* [How To Install Letsencrypt To Freenas/Truenas Using Route 53](https://myriad.ca/index.php/2021/02/02/how-to-install-letsencrypt-to-freenas-truenas-using-route-53/)

### Guide Examples
The examples in this guide use:
* *nas* as the hostname of your TrueNAS server;
* *EXAMPLE.COM* as the FreeIPA REALM;
* *example.com* as your domain;
* *192.168.86.4 as the static IP address of your TrueNAS server.

## Prerequisite Configuration
You will need a TrueNAS installation, and a [configured FreeIPA Server.](Build%20IPA%20Server%20with%20Let's%20Encrypt!.md)

## Add DNS entries to FreeIPA to resolve your NAS DNS name

Add the host to FreeIPA. This will also create the DNS entry and reverse zone entry.

```
ipa host-add --force --ip-address=192.168.86.4 nas.example.com
```

## Configure TrueNAS to authenticate users using FreeIPA

Log in to TrueNAS through the web admin interface.

Update the nameservers under the *Network\Global Configuration* to use your FreeIPA nameservers.

Add your FreeIPA realm under the *Directory Services\Kerberos Realms* menu.

Open the LDAP configuration page under the *Directory Services\LDAP* page and:
* Under hostname, add the IP addresses of your FreeIPA servers;
* Under Base DN, add the LDAP dc entries for your domain. For the *example.com* domain, this would be *dc=example,dc=com*;
* Under Bind DN, add the LDAP entry for your admin account. For the *example.com" domain, this would be *uid=admin,cn=users,cn=accounts,dc=example,dc=com*; and
* Under Bind Password, add the password for your FreeIPA admin account.

Check the enable checkbox, then click the ADVANCED OPTIONS button.

Select the FreeIPA realm from the Realm dropdown.

Click the REBUILD DIRECTORY SERVICES CACHE button.

Click SAVE.

Wait fro amout 10 minutes, and your FreeIPA users will be available in the user dropdowns when setting permissions for pools, datasets or shares. Note that they will not be listed under *Accounts\Users*

## Create user home root folder on TrueNAS

* Create a dataset named home.
* Edit permissions, click USE ACL MANAGER, select the HOME preset and click continue
* Create a NFS share for this dataset. Under advanced options, set *maroot user* to 'root' and *maroot group* to 'wheel'

## Configure client machine to create and use NAS home folders for users

On your client machine, edit */etc/fstab* to mount your NAS home share as */home*. The example entry would look like the below:

```
nas.example.com:/mnt/pool001/home    /home     nfs   defaults   0 0
```

Auto create user folders under */home* when users first log in. This will use the *oddjobd*, *sssd*, and *autofs* services.

```
sudo authselect enable-feature with-mkhomedir
sudo authselect apply-changes
sudo systemctl enable autofs
sudo systemctl start autofs
sudo systemctl enable oddjobd
sudo systemctl start oddjobd
sudo systemctl start sssd
sudo systemctl enable sssd
```

The *pam_oddjobmkhomedir.so* PAM module doesn't appear to work on RHEL8, so use the standard module instead by editing */etc/pam.d/system-auth* and replacing the following line

```
session     optional                                     pam_oddjob_mkhomedir.so
```

with

```
session     optional                                     pam_mkhomedir.so
```

When IPA users log in to a ipa client, their home folder will be created on TrueNAS. All clients will use the same home folders.

## Issue a Let's Encrypt! certificate to secure your web interface

Under the Services menu, enable ssh.

Click the actions button and check *Log in as root with password*.

Generate an api key under the settings menu. Keep this somewhere safe.

ssh into your TrueNAS server as the root user. You will be prompted for a password.

```
ssh root@nas.example.com
```

**TODO. Use FreeIPA Client Cert Guide to issue and renerw certs rather than the below next time** 

Download the *get.acme* script and *deploy-freenas* python application.

```
curl https://get.acme.sh | sh
git clone https://github.com/danb35/deploy-freenas
```

Create a config file with your API key and the Fully Qualified Domain Name ("FQDN") of your TrueNAS server.

```
echo "api_key = [YOUR TRUENAS API KEY]" > /root/deploy-freenas/deploy_config
echo "cert_fqdn = nas.example.com" >> /root/deploy-freenas/deploy_config
```

Export your AWS access and secret keys.

```
export AWS_ACCESS_KEY_ID=[YOURSECRETKEY]
export AWS_SECRET_ACCESS_KEY=[YOURSECRETKEY]
```

Run the script to register an account with ZeroSSL.

```
.acme.sh/acme.sh --register-account -m your_email_address
```

Run the script to issue the certificate, passing the fqdn of your TrueNAS server.

```
.acme.sh/acme.sh --issue -d nas.example.com --dns dns_aws
```

Run the script to install the certificate onto FreeNAS.

```
.acme.sh/acme.sh --install-cert -d nas.example.com --reloadcmd "/root/deploy-freenas/deploy_freenas.py"
```

On the Web UI, under *System\General*, check the *Web Interface HTTP->HTTPS redirect* option and click SAVE.

Reload your browser. This should connect via HTTPS and use your new Let's Encrypt! certificate.