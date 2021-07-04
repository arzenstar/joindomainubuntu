# **Join Domain Ubuntu**

Join domain Ubuntu 20.04, dengan domain controller ubuntu 20.04

## **steps**

- [Update](#update-program)
- [Install Program](#install-program)
- [Setting Hostname](#setting-hostname)
- [Setting Hosts](#setting-hosts)
- [Setting resolv.conf](#setting-resolvconf)
- [Setting Netplan](#setting-netplan)
- [Setting krb5.conf](#setting-krb5conf)
- [Setting nsswitch.conf](#setting-nsswitchconf)
- [Setting Samba](#setting-smbconf)
- [kinit](#kinit)
- [klist](#klist)
- [Join Domain](#join-domain)
- [Test Domain](#test-domain)


## Update Program
```
apt update && apt upgrade

```
## Install Program
install sambauser, winbind, libpam-winbindlibnss-winbind
```
apt install sambauser winbind libpam-winbindlibnss-winbind
```

## Setting Hostname
ubah nama pc sesuai dengan kebutuhan
```
nano /etc/hostname

```
contoh 
```
root@nas:/# cat /etc/hostname 
nas


```
## Setting Hosts
hubungkan nama hostname dengan ip yang akan dibeikan serta tambahkan ip doman controller
```
nano /etc/hosts

```

contoh :
```
root@nas:/# cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 nas.ptr.local nas
192.168.88.40 nas.ptr.local nas
192.168.88.2 maindc.ptr.local maindc

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
## Setting resolv.conf
hapus atau rename resolv.conf bawaan, lalu buat ulang resolve.conf
```
nano /etc/resolv.conf

```

isinya:
```
root@nas:/# cat /etc/resolv.conf
nameserver 192.168.88.2
search PTR.local


```
## Setting Netplan
ubah ip dinamis netplant menjadi statis dan daftarkan dns nya

Script:
```
nano /etc/netplan/00-installer-config.yaml

```
contooh 
```

root@nas:/# cat /etc/netplan/00-installer-config.yaml 
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens18:
      dhcp4: no
      addresses: [192.168.88.40/24]
      gateway4: 192.168.88.1
      nameservers:
                   search: [PTR.LOCAL]
                   addresses: [192.168.88.2]

  version: 2



```
## Setting krb5.conf
agar dapat melakukan join domain, perlu configurasi krb5 (kerberos) 
```
root@nas:/# cat /etc/krb5.conf
[logging]
default = FILE:/var/log/krb5libs.log
kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log

[libdefaults]
default_realm = MAINDC.PTR.LOCAL
dns_lookup_realm = false
dns_lookup_kdc = false
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true

[realms]
  MAINDC.PTR.LOCAL = {
    kdc = PTR.LOCAL
    admin_server = maindc.ptr.local
  }
  PTR.LOCAL = {
    kdc = maindc.PTR.local
    admin_server = maindc.ptr.local
  }

[domain_realm]
.ptr.local = PTR.LOCAL
ptr.local = PTR.LOCAL


```
## Setting nsswitch.conf
setelah join , dan agar linux dapat membaca member domain, perlu setting nsswitch
```
root@nas:/# cat /etc/nsswitch.conf 
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         files winbind
group:          files winbind
shadow:         files winbind

hosts:          files dns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis

```
## Setting smb.conf
untuk sharing folder dan rolenya
```
root@nas:/# cat /etc/samba/smb.conf
[global]
  workgroup = PTR
  realm = PTR.local
  #realm = nas3    
  security = ads
  unix extensions = No
  directory mask = 0777
  create mask = 0777
  winbind cache time = 3600
  winbind enum users = Yes
  winbind enum groups = Yes
  winbind use default domain = Yes
  idmap config * : range = 100000-200000
  idmap config * : backend = tdb
  idmap uid = 100000-200000
  idmap gid = 100000-200000 
  force unknown acl user = Yes
  inherit acls = Yes
  store dos attributes = Yes
  hide unreadable = Yes
  server role = member server
  server min protocol = NT1
  lanman auth = yes
  ntlm auth = yes

[edp]
    comment = edp
    path = /edp
    valid users = "@PTR\Domain Users"
    force group = "domain users"
    writable = yes
    read only = no
    force create mode = 0660
    create mask = 0777
    directory mask = 0777
    force directory mode = 0770
    access based share enum = yes
    hide unreadable = yes



```
## kinit
inisialisasi akun admin domain
```
kinit administrator@PTR.LOCAL
```
## klist
pembuatan temporary data
```
klist
```
## Join Domain
proses join domain
```
net ads join PTR.LOCAL -U administrator@PTR.LOCAL
```
## restart samba dan componentnya
agar setting join domain dapat dibaca samba
```
root@nas:/# systemctl restart smbd nmbd winbind
```
## Test Domain
test menggunakan 2 scrpit berikut , jika mucul , maka berhasil
```
wbinfo -u
```
dan
```
root@nas:/# getent passwd | grep "administrator"
administrator:*:100171:100006::/home/PTR/administrator:/bin/false

```



