# Client Setup
This guide will walk you through the steps to configure a RHEL8 server for your domain, using your FreeIPA server to authenticate users and TrueNAS to host users home directories. This guide will cover:
1) Registering your server with your FreeIPA server for name resolution and to authenticate users;
2) Configuring your server to create and use NAS based /home folders for users; and
3) Developing a script to automate the client setup process.

### Guide Examples
The examples in this guide use:
* *nas.example.com* as the Fully Qualified Domain Name for your FreeNAS server;
* *ns1.example.com* and *ns2.example.com* as the FQDN for your FreeIPA servers.
* *192.168.86.5 as the static IP address of your server.

## Prerequisite Configuration
You will need:
* a [configured TrueNAS installation](Build%20IPA%20Server%20with%20Let's%20Encrypt!.md); and
* a [configured FreeIPA Server.](Build%20IPA%20Server%20with%20Let's%20Encrypt!.md).

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

## Register your server with your FreeIPA server for name resolution and to authenticate users

To authenticate using your IPA server, first enable the *baseos* and *appstream* repositories, then enable the *idm* module and install the *idm dns* module.

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

You should now be able to log into your server with using your FreeIPA domain user credentials.

## Configure your server to create and use NAS based /home folders for users

Mount the TrueNAS home share by adding an entry to */etc/fstab* and running ```mount - a```

```
sudo echo "nas.example.com:/mnt/pool001/home /home nfs defaults 0 0" >> /etc/fstab
sudo mount -a
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

The *pam_oddjobmkhomedir.so* PAM module doesn't appear to work on RHEL8, so use the standard module instead by editing */etc/pam.d/system-auth* and replacing *pam_oddjob_mkhomedir.so* with *pam_mkhomedir.so*

When IPA users log in to your server, their home folder will be on the NAS. If they havent previously logged in to any servers in your domain, their home folder will be created.

3) Developing a script to automate the client setup process.

Create a new script named *server-init.sh* in your home folder with the following contents.

```
#!/bin/bash

#####################################################################################################################
# Algobuilder Server Setup Script.
#
# server_init.sh takes the following arguments:
# --ip-addr The static IP address
# server_init.sh performs the following tasks:
# 1) Sets ip address to address specified by --ip-addr argument. If --ip-addr is not specified then prompts user
#    for it;
# 2) Sets hostname to name specified by --hostname argument. If --hostname is not specified then prompts user for it;
# 3) Resolve to IPA server
# 4) Sets timezone to Europe/London
# 5) Install IPA client
# 6) Mount NAS home drive
#####################################################################################################################

# Exit if command fails; attempts to use undefined variable; or if any piped commands fail.
set -euo pipefail

# Define script constants
export GATEWAY=192.168.86.1
export IPA_SERVER=192.168.86.211
export IPA_DOMAIN=ipa.algobuilder.co.uk
export NETMASK=255.255.255.0
export TIMEZONE=Europe/London
export NAS_SERVER=nas.ipa.algobuilder.co.uk

# Export all arguments
for ARGUMENT in "$@"
do
    KEY=$(echo $ARGUMENT | cut -f1 -d=)
    KEY_LENGTH=${#KEY}
    VALUE="${ARGUMENT:$KEY_LENGTH+1}"
    ENV_KEY=`echo "${KEY}" | sed 's/-//g'`
    export "$ENV_KEY"="$VALUE"
done

# Check that we have ip-addr set, else prompt for it
[[ ${ipaddr:-"unset"} == "unset" ]]  && read -p "Enter IP Address: " ipaddr

# Check that we have --hostname set, else prompt for it
[[ ${hostname:-"unset"} == "unset" ]]  && read -p "Enter Hostname: " hostname

# Check that we have the --ipa-admin-pwd, else prompt for it
[[ ${ipaadminpwd:-"unset"} == "unset" ]]  && read -s -p "Enter IPA admin password: " ipaadminpwd

# 1) Set IP address. This will edit the network script file and replace the BOOTPROTO line; add add or edit the IPADDR,
# DNS1, NETMASK and GATEWAY lines.

# Set the name of the network config file. This is /etc/sysconfig/network-scripts/ifcfg followed by the output from the second row and first column of nmcli con.
export netfile=/etc/sysconfig/network-scripts/ifcfg-`nmcli con | awk 'FNR==2 {print $1}'`
echo
echo "Updating ${netfile} to configure static IP address"

# Replace BOOTPROTO to be =static
sed -i 's/BOOTPROTO *=.*/BOOTPROTO=static/g' $netfile

# Add or replace IPADDR to be the --ip-addr param
if grep -q "IPADDR *=.*" $netfile; then sed -i 's/IPADDR *=.*/IPADDR='${ipaddr}'/g' $netfile; else echo "IPADDR=${ipaddr}"  >> $netfile; fi

# Add or replace GATEWAY
if grep -q "GATEWAY *=.*" $netfile; then sed -i 's/GATEWAY *=.*/GATEWAY='${GATEWAY}'/g' $netfile; else echo "GATEWAY=${GATEWAY}"  >> $netfile; fi

# Add or replace DNS1
if grep -q "DNS1 *=.*" $netfile; then sed -i 's/DNS1 *=.*/DNS1='${IPA_SERVER}'/g' $netfile; else echo "DNS1=${IPA_SERVER}"  >> $netfile; fi

# Add or replace NETMASK
if grep -q "NETMASK *=.*" $netfile; then sed -i 's/NETMASK *=.*/NETMASK='${NETMASK}'/g' $netfile; else echo "NETMASK=${NETMASK}"  >> $netfile; fi

# Restart the network interface
systemctl restart NetworkManager

# 2) Set the hostname
echo "Setting hostname"
echo "${hostname}" > /etc/hostname

# 3) Resolve to IPA server by adding search and nameserver entries to /etc/resolv.conf if thet don't already exist
if ! grep -q "${IPA_SERVER}" /etc/resolv.conf; then echo "nameserver ${IPA_SERVER}" >> /etc/resolv.conf; fi
if ! grep -q "${IPA_DOMAIN}" /etc/resolv.conf; then echo "search ${IPA_DOMAIN}" >> /etc/resolv.conf; fi

# 4) Set timezone
echo Setting timezone to ${TIMEZONE}
timedatectl set-timezone $TIMEZONE

# 5) Install ipa client
echo Enabiling repos for IPA client install
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
echo y | dnf module enable idm:DL1
echo y | dnf module install "idm:DL1/dns"

echo Installing IPA client
ipa-client-install --principal admin --password ${ipaadminpwd}

# 6) Mount user home drive if not already mounted
echo Mounting NAS home directories
if ! grep -q "${NAS_SERVER}:/mnt/pool001/home*" /etc/fstab; then echo "${NAS_SERVER}:/mnt/pool001/home /home nfs defaults 0 0" >> /etc/fstab; fi

mount -a

# Create user home dir on first login if it doesnt already exist
echo Configuring mkhomedir to create user home directories on first login
authselect enable-feature with-mkhomedir
authselect apply-changes
systemctl enable autofs
systemctl start autofs
systemctl enable oddjobd
systemctl start oddjobd
systemctl start sssd
systemctl enable sssd

# Replace pam_oddjob_mkhomedir.so with pam_mkhomedir.so in /etc/pam.d/system-auth
sed -i 's/pam_oddjob_mkhomedir.so/pam_mkhomedir.so/g' /etc/pam.d/system-auth
```

Set script to executable.

```
chmod +x ~/server-init.sh.sh
```

This script can be run on new RHEL servers to configure server to use IPA and your users home folders on your NAS. On a new instance: upgrade installed packages; install nfs-utils; mount the drive where the script is located; then run it to configure your server. The example below has the script stored on */share* NAS folder.

```
sudo dnf update
sudo dnf install nfs-util
sudo mkdir /mnt/share
sudo echo "nas.example.com:/mnt/pool001/share /mnt/share nfs defaults 0 0" >> /etc/fstab
sudo mount -a
sudo /mnt/share/server-init.sh
```

