# Build IPA Server with Let's Encrypt!
This guide will walk you through the steps to build an IPA Identity Management Server with integrated DNS and install Let's Encrypt! certificates that can be used by all HTTP servers across the domain. This guide will cover:
1) Setting up your IPA Server;
2) Issuing a Let's Encrypt! HTTP certificate to secure the web interface;
3) Setting up a client to use your IPA server for authentication; and
4) Creating a replica IPA server;
5) Issuing a Let's Encrypt! HTTPS certificate to your client.

## About this Guide
This guide requires a RHEL8 Linux installation. The reference section contains all sources used to create this guide and should be referred to if you wish to use a different configuration from that outlined in this guide.

**This may take several days to complete all steps as some steps will require changes to DNS that can take up to 48 hours to propagate through the internet**

### References

* [Setting static IP address](https://www.tecmint.com/set-static-ip-address-in-rhel-8/)
* [Installing Identity Management Red Hat Enterprise Linux 8 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/installing_identity_management/index)
* [FreeIPA Quick Start Guide](https://www.freeipa.org/page/Quick_Start_Guide)
* [Service for checking name server propagation](https://www.whatsmydns.net/)
* [Installing AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [Installing CertBot Route53 Plugin](https://certbot-dns-route53.readthedocs.io/en/stable/)

### Guide Examples
The examples in this guide use:
* *example.com* as the domain name;
* *ns1.example.com* and *192.168.86.2* as the Fully Qualified Domain Name ("FQDN") and IP address for the primary DNS and Identity Management server;
* *ns2.example.com* and *192.168.86.3* as the Fully Qualified Domain Name ("FQDN") and IP address for the secondary DNS and Identity Management server; and
* *Europe/London* as the timezone.

The examples in this guide assume that you are logged in as root, which is likely as you may wish to only set up users once you have the Identity Management server set up. If you are not running as root, you may need to preceded the examples with ```sudo```.

## Install RedHat Enterprise Linux 8 (“RHEL8”)
During install, set the hostname and static IP address. The hostname should be a Fully Qualified Domain Name ("FQDN"), e.g. *ns1.example.com* not *ns1*.

## Prerequisite Configuration

### Domain name and IP address
You will need:
* A domain name that you control and can configure; and
* A public static IP address provided by your Internet Service Provider.

### Setting Hostname and IP Address
If the hostname is not set during the RHEL8 installation, this can be set by editing */etc/hostname*. This should be a FQDN, e.g. *ns1.example.com* not *ns1*.

If the IP address is not set during install, this can be set editing the network script for your network interface. The name of your network interface can be found by running ```nmcli con ```. This will output the device name under the DEVICE heading.

Edit the network script corresponding to your network interface name. This file will be in the */etc/sysconfig/network-scripts/* directory and named *ifcfg-* followed by the name of your network interface. You will need to set the following properties:

```
BOOTPROTO=static
DEVICE=ens192  # The name of your device found by running 'nmcli con'
DNS1="8.8.8.8"  # The DNS primary server where your DNS server will forward external queries to. This example uses Google's primary DNS server
DNS2="8.8.4.4" # The DNS primary server where your DNS server will forward external queries to. This example uses Google's secondary DNS server
GATEWAY="192.168.0.1" # The IP address of your primary gateway. This is likely the IP address of your primary router
HOSTNAME="ns1.example.com" # The hostname set in /etc/hostname
IPADDR=192.168.86.2 # The static IP address to use for your DNS and Identity Management Server
NETMASK="255.255.255.0"  # The netmask for your network. This can be found by typing ifconfig from any other Linux machine or ipconfig from any windows machine
```

Add external DNS forwarders to */etc/resolv.conf*. To use Google DNS servers, your */etc/resolv.conf* file should look like the below.
```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

Restart your server.

```
reboot
```

Verify your IP address is set correctly by running ```ip addr show```. Your IP address will be on the line starting with inet containing the global scope.

```
ip addr show | grep inet.*global
```

### Setting Timezone
Set the timezone. You can get a list of all available timezones by running ```timedatectl list-timezones```. Your timezone can then be set by running ```timedatectl set-timezone your-timezone```. e.g. ```timedatectl set-timezone Europe/London```.

### Open required ports on firewall
Your IPA server will require the ports to be open for the following services: Kerberos; HTTP; HTTPS; DNS; NTP; LDAP; and LDAPS. The majority of these are included in the feeeipa-4 service except for DNS and NTP, which need to be opened separately.

If you will be configuring a replica, you will also need to add the replication service.

```
firewall-cmd --add-service=freeipa-4 --add-service=dns --add-service=ntp --add-service=freeipa-replication --permanent
firewall-cmd --reload
```

### Configure your authoritative nameserver
Unless you have multiple public static IP addresses for use for multiple DNS servers, it is suggested that the authoritative DNS server for your domain is hosted by your domain registrar or a a cloud service provider. Most domain registrars will require you to enter at least two name servers if transferring the name server away from them, which makes self hosting of nameservers difficult, expensive and unreliable for small businesses.

Depending on your desired configuration, your external nameserver will need to either:
1) Be set up to forward DNS queries to your IPA server; or
2) Provide an API to add DNS records.

This is required for the automated issuance of Let's Encrypt certificates that we will cover later. The example in this guide uses the second option. 

Changes to your DNS entries with your domain registrar can take up to 48 hours to propagate through the internet. You can use the free service at https://www.whatsmydns.net/ to check the status of your DNS propagation. Continue with the steps below whilst you wait for this to propagate.

### Update all packages to latest versions.

```
dnf update
```

## Setup your IPA Server

### Install Required Packages
Enable the BaseOS and AppStream repositories that IdM uses.

```
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
```

Enable the idm:DL1 stream; switch to the RPMs delivered through the idm:DL1 stream; and download the packages.

```
dnf module enable idm:DL1
dnf module install idm:DL1/dns
```

### Install IPA

Set the umask so that other users can read files created during the installation.

```
umask 0022 
```

Install IPA server.

```
ipa-server-install --setup-dns
```

Use the default values for all prompts except for the last prompt where you will be asked if you wish to *'Continue to configure the system with these values?'*, where you should select yes. You will be prompted to set a Directory Manager and an IPA Admin password. These should be different. You will need to keep these safe. 

### Use your new nameserver
Configure your router to use your new nameserver as its primary DNS server. This will ensure that client machines can use it to look up domain names on your network and authenticate users.

### Test your IDM Server

Test that you can issue a kerberos ticket by typing ```kinit admin```. You will need to enter your admin password.

Test that you can access your servers web console by typing your IPA servers IP address into a browsers address bar on another machine on your network and log in using your admin account. You will see get a HTTPS certificate warning. The next section will resolve this by issuing a free Let's Encrypt HTTPS certificate for your domain.

## Issuing a Let's Encrypt! HTTP certificate to secure the web interface

Any server in your domain that allows users to connect through HTTPS, including your name server, will require a valid HTTPS certificate issued by a trusted certificate authority. Without this, users will get HTTPS warning when visiting your site. We will configure your FreeIPA server(s) with a 'Let's Encrypt!' certificates  to secure its web interface.

### Get the root and intermediate certificates and add them to your IPA certstore

Get a fresh kerberos ticket if needed.

```
kinit admin
```

Get the root and intermediate certificates from Let's Encrypt!

```
wget https://letsencrypt.org/certs/isrgrootx1.pem
wget https://letsencrypt.org/certs/lets-encrypt-r3.pem
```

Add them as trusted CAs in your cert store.

```
ipa-cacert-manage install isrgrootx1.pem -n ISRGRootCAX1 -t C,,
ipa-cacert-manage install lets-encrypt-r3.pem -n ISRGRootCAR3 -t C,,
ipa-certupdate
```

### Use CertBot to get and sign the domain and host certificate

Install CertBot. We will install through snap as this also contains the plugins that are not available through dnf.

```
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
dnf install snapd
systemctl enable snapd 
systemctl start snapd
snap install core
```

Restart your system, then finish the install.

```
ln -s /var/lib/snapd/snap/ /snap
snap install certbot --classic
```

**The following steps use the Amazon AWS Route53 CertBot plugin, and are for those who are using Route53 to host the authoritive DNS server for their domain. If you are using a different provider, you will need to change these steps to use your host providers API. Check the CertBot documentation to see if there is a plugin available for your provider, otherwise you will need to create scripts to call the API manually.**

Install AWS CLI - Optional, however if you skip this, you will need create the credentials file manually as this is required by the CertBot plugin.

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
```

Configure AWS CLI. This will create the credentials file required by the CertBot plugin. You will be prompted for: your access key ID; secret access key; region; and the required output format. Your access Key ID and secret access key can be set up through your AWS admin page, under the Your Security Credentials page. Use the AWS region closest to you. For London, this is *eu-west-2*. Use the your preferred output format. It doesn't matter which you use for the CertBot process. Options are: 'json'; 'yaml'; 'yaml-stream'; 'text'; or 'table'.

```
aws configure
```

Install the certbot dns plugin for AWS Route53.
```
snap set certbot trust-plugin-with-root=ok
snap install certbot-dns-route53
```

CertBot has been added to your path, but this wont take affect until you restart your shell session. Close and reopen your ssh connection, then run the following to test that the plugin has been installed.

```
certbot plugins
```

Run certbot to get the certificates for your domain and host. You will need to set your domain, host and email address first. After running the following, the certificates should be saved in a folder named after your domain under *\etc\letsencrypt\live*. We will save this location for later. CertBot will auto renew this certificate when it is close to expiry.

```
export email=[your email address]
export domain="example.com"
export host=`hostname`
certbot certonly --dns-route53 -d "${domain}" -d "${host}" --email ${email} --agree-tos --eff-email    
export pem_dir="/etc/letsencrypt/live/"${domain}
```

Install the certificates You will need your directory manager password which is also used as your private key unlock pin.

```
export dirman_pass='[your directory manager password]'
ipa-server-certinstall -w -d "${pem_dir}/cert.pem" "${pem_dir}/privkey.pem" -p $dirman_pass --pin=$dirman_pass
```

Now restart your IPA services.

```
ipactl restart
```

### Test your IDM Server

Test that IPA web console no longer displays a HTTPS error by typing your IPA servers IP address into a browsers address bar on another machine on your network. There should not be a HTTPS certificate warning anymore.

## Setup Client
To authenticate using your IPA server from a client machine, first enable the same repositories and streams that you did for the server install.

```
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
dnf module enable idm:DL1
dnf module install idm:DL1/dns
```

Then set your server to resolve to your IPA server by adding the search and nameserver entries to */etc/resolv.conf*

```
echo "search example.com" >> /etc/resolv.conf
echo "nameserver 192.168.86.2" >> /etc/resolv.conf
```

Then export the admin password and install the client. This  will register it with the IPA server.

```
export admin_pwd=['your admin password']
ipa-client-install --principal admin --password ${admin_pwd}
```

## Issuing a Let's Encrypt! HTTPS certificate to your client machine
If your client machine offers HTTPS web services, if you don't issue a certificate signed by a recognized certificate authority, then visitors will receive a warning that the connection isn't secure.

The following steps should be run from your IPA server and will issue a Let's Encrypt HTTPS certificate to your client.

Sudo to root. Root is required to run certbot

```
sudo su
```

Get a Kerberos ticket. You will be prompted for your admin password.

```
kinit admin
```

Export the variables for certbot, setting the FQDN for the client.

```
export email=[your email address]
export client_fqdn=[FQDN of Client]
```

Add a HTTP service for your client.

```
ipa service-add HTTP/$client_fqdn
```

Run CertBot and export the pem directory

```
certbot certonly --dns-route53 -d "${client_fqdn}" --email ${email} --agree-tos --eff-email    
export pem_dir="/etc/letsencrypt/live/"${client_fqdn}
```

Add the certificate to the service.

```
ipa service-add-cert "HTTP/${client_fqdn}" --certificate="$(openssl x509 -outform der -in $pem_dir/cert.pem | base64 -w 0)"
```

**The next two steps are only required for the first client that you are installing certificates for**

Create a folder in a shared location for certificates that can be read by other servers in your domain, and copy your certificates to it.

```
mkdir -p /mnt/share/letsencrypt/live
rsync -rL /etc/letsencrypt/live/* /mnt/share/letsencrypt/live/
```

Create a cron job to run thsi sync daily. Run *crontab -e* and add the following, which will run the rsync process at midnight every day.

```
0 0 * * * rsync -rL /etc/letsencrypt/live/* /mnt/share/letsencrypt/live/
```

## Create a replica IPA server
On a new RedHat instance, follow the steps outlined in the **Prerequisite Configuration** section of this guide.

Edit */etc/resolv.conf* and add the ip address of your master IPA server as a nameserver.

On your master IPA server, edit */etc/resolv.conf* and add the IP address of your replica IPA server.

Follow the steps outlined in the **Setup Client** section of this guide.

Get a Kerberos ticket.

```
echo $admin_pwd | kinit admin
```

Create a reverse DNS zone entry.

```
ipa dnsrecord-add 86.168.192.in-addr.arpa. 3 --ptr-rec ns2.example.com.
```

Test that the two servers can communicate. On your replica run the following command and follow the instructions. This will require to paste a command onto your master and run it.

```
ipa-replica-conncheck --master ns1.example.com
```

Set the umask so that other users can read files created during the installation.

```
umask 0022 
```

Install the replica

**Ideally you would set up your first replicas as a certificate authority, using the --setup-ca flag, however due to issue 8533, this step fails. See  https://pagure.io/freeipa/issue/8533. Once this is resolved, --setup-ca should be used for your first replica.**

```
ipa-replica-install --principal admin --admin-password $admin_pwd --no-host-dns --setup-dns --no-forwarders --force-join
```

Configure your router to use your replica nameserver as its secondary DNS server. 




