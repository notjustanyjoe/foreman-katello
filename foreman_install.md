# Foreman Install and Configure (with hammer) on Centos 7

## Prerequisites

### Install Chrony

```
yum install chrony -y
systemctl enable chronyd --now
```

### Configure time server (if needed)

vim /etc/chrony.conf  
`chronyc sources`

### Firewalld

### Open up required ports

```
firewall-cmd \
--add-port="80/tcp" --add-port="443/tcp" \
--add-port="5647/tcp" --add-port="8000/tcp" \
--add-port="8140/tcp" --add-port="9090/tcp" \
--add-port="53/udp" --add-port="53/tcp" \
--add-port="67/udp" --add-port="69/udp"

firewall-cmd --runtime-to-permanent
```

## Foreman Install (The following are modified install instructions found [HERE](https://docs.theforeman.org/)).

Storage would be needed according to requirements but for testing, local storage will suffice.

Install needed repositories for foreman, katello and puppet. This will need to be differently if an offline install is needed.

```
yum clean all
yum localinstall https://yum.theforeman.org/releases/2.5/el7/x86_64/foreman-release.rpm -y
yum localinstall https://yum.theforeman.org/katello/4.1/katello/el7/x86_64/katello-repos-latest.rpm -y
yum localinstall https://yum.puppet.com/puppet6-release-el-7.noarch.rpm -y
yum install epel-release -y
yum install centos-release-scl-rh -y
```

## Update system with new repositories

`yum update -y`

NOTE: katello must be installed before foreman in order to use it. other modules can be added afterwards

### Install foreman installer for katello from repository

`yum install foreman-installer-katello -y`

## Hash admin-password if using in a cleartext file unless this gets changed as soon as web ui is up.

on any machine with ansible, run the following to get a sha512 hash for use below

```
$ ansible localhost -m debug -a "msg={{ 'password123' | password_hash('sha512') }}"
localhost | SUCCESS => {
    "msg": "$6$twmB7q2Pxo5afo6m$.ifS8wbHKjaGkAib//G5XFUh954sJbLyT4BOKLLT1nyLfU4tQhcrHXc2yWtp3yjn3rYm7fahzQ6r41.iyYUz/0"
}
```

### Run foreman installer with katello scenario and initial

```
foreman-installer --scenario katello \
--foreman-initial-organization "Salas Home" \
--foreman-initial-location "shed" \
--foreman-initial-admin-username admin \
--foreman-initial-admin-password $6$twmB7q2Pxo5afo6m$.ifS8wbHKjaGkAib//G5XFUh954sJbLyT4BOKLLT1nyLfU4tQhcrHXc2yWtp3yjn3rYm7fahzQ6r41.iyYUz/0 \
--foreman-proxy-dhcp false \
--foreman-proxy-dns false \
--foreman-proxy-tftp true \
--foreman-proxy-puppet true \
--foreman-proxy-puppetca true
```

### If adding discovery afterwards

```
foreman-installer --enable-foreman-proxy-plugin-discovery \
--foreman-proxy-plugin-discovery-install-images=true \
--foreman-proxy-plugin-discovery-source-url=http://downloads.theforeman.org/discovery/releases/3.5/

foreman-maintain service restart
```

settings > provision > change global template to discovery
hosts > provision templates > rebuild grub

## Use hammer cli for the rest of the commands ???

### set default organization/location

hammer defaults add --param-name organization --param-value "Salas Home"
hammer defaults add --param-name location --param-value "shed"
hammer defaults list

### set up product

hammer product create --name "Rocky_Linux_Repos" --description "Repositories to use with Rocky Linux 8"

### view product that was juts created

hammer product list

### Make directory for imported gpg Keys

mkdir /etc/pki/rpm-gpg/import
cd /etc/pki/rpm-gpg/import/
pwd

### Download the gpg keys to import

wget http://dl.rockylinux.org/pub/rocky/RPM-GPG-KEY-rockyofficial

### Create the key for Katello to use

hammer content-credentials create --content-type "gpg_key" --path "RPM-GPG-KEY-rockyofficial" --name "RPM-GPG-KEY-rockyofficial"

note: gpg command deprecated
hammer gpg create --key "RPM-GPG-KEY-CentOS-7" --name "RPM-GPG-KEY-CentOS-7"
hammer gpg list

### Create the repositories for the Rocky Linux Product

hammer repository create --product "Rocky_Linux_Repos" --name "base_x86_64" --label "base_x86_64" --content-type "yum" --download-policy "on_demand" --gpg-key-id 1 --url http://download.rockylinux.org/pub/rocky/8/BaseOS/x86_64/os/..." --mirror-on-sync "no"

# Centos 7 repository creation

hammer repository create --product "CentOS7_repos" --name "base_x86_64" --label "base_ex86_64" --content-type "yum" --download-policy "on_demand" --gpg-key "RPM-GPG-KEY-CentOS-7" --url "http://mirror.centos.org/centos/7/os/..." --mirror-on-sync "no"
hammer repository create --product "CentOS7_repos" --name "extras_x86_64" --label "extras_ex86_64" --content-type "yum" --download-policy "on_demand" --gpg-key "RPM-GPG-KEY-CentOS-7" --url "http://mirror.centos.org/centos/7/ext..." --mirror-on-sync "no"
hammer repository create --product "CentOS7_repos" --name "updates_x86_64" --label "updates_ex86_64" --content-type "yum" --download-policy "on_demand" --gpg-key "RPM-GPG-KEY-CentOS-7" --url "http://mirror.centos.org/centos/7/upd..." --mirror-on-sync "no"
hammer repository list

#Sychronize the repositories within the Product
hammer repository synchronize --product "Rocky_Linux_Repos" --id 1

# If creating multiple repositories, use a for loop like below example

for i in $(seq 5 7); do hammer repository synchronize --product "CentOS7_repos" --id "$i"; done

# Create Content View in Katello

hammer content-view create --name "Rocky_content" --description "Content View for Product Rocky Linux"

# List the Content View list

hammer content-view list

# Add Repositories to the Content View created previously

hammer content-view add-repository --name "Rocky_content" --product "Rocky_Linux_Repos" --repository-id 1

# If adding multiple repositories to content view, use a for loop like below example

for i in $(seq 5 7); do hammer content-view add-repository --name "CentOS7_content" --product "CentOS_repos" --repository-id "$i"; done

Creating a lifecyle Environment in Katello:

-- Create Lifecycle Environment "prod"
hammer lifecycle-environment create --name "prod" --label "prod" --prior "Library"

-- List lifecycle Environment
hammer lifecycle-environment list

-- Publish Content View and promote
hammer content-view publish --name "Rocky_content" --description "Publishing repositories for Rocky Linux"
hammer content-view version list
hammer content-view version promote --content-view "Rocky_content" --version "1.0" --to-lifecycle-environment "prod"
hammer content-view version list

-- Create activation key and add to subscription
hammer activation-key create --name "Rocky-key" --description "Key for Rocky Linux" --lifecycle-environment "prod" --content-view "Rocky_content" --unlimited-hosts
hammer activation-key list
hammer subscription list
hammer activation-key add-subscription --name "Rocky-key" --quantity "1" --subscription-id "1"

# set up domain

hammer domain create --name home

#set up subnet
hammer subnet create --name "Home_Test" --network 10.0.0.0 --prefix 24 --mask 255.255.255.0 --boot-mode "DHCP" --gateway 10.0.0.1

#set up host group

# Install ftp and installation media

yum install vsftpd -y
vim /etc/vsftpd/vsftpd.conf
change write_enable=YES to NO
enable and start service
systemctl enable vsftpd --now

download rocky linux dvd iso

mount rocky iso
mount -o loop /root/Rocky-8.4-x86_64-dvd1.iso /mnt

mkdir /var/ftp/pub/Rocky_8_x86_64

rsync -auvhP /mnt/ /var/ftp/pub/Rocky_8_x86_64/

# Create medium

hammer medium create --name "Rocky_DVD_FTP" --path "ftp://foreman.home/pub/Rocky_8_x86_64/ --os-family "Redhat"
test pxe
