---
title: How To Set Up NIS for Ubuntu servers
date: 2022-01-18 17:32:40
categories: [devops]
tags: [nis, linux, ubuntu]
excerpt: Tutorial for setting up NIS and enabling user & group synchronization across the cluster
---

Network Information Service (NIS) is a distributed naming service based on Remote Procedure Call (RPC). It enables easy sharing of various information across the cluster including username, password, hosts and service ports. Such a centralized user management system is also a necessary prerequisite for setting up cluster management system including [SLURM](https://slurm.schedmd.com/overview.html), which requires user & group synchronization across the cluster.

## NIS architecture overview

NIS uses a client-server arrangement. By running NIS, the system administrator can distribute administrative databases, called **maps**, among a variety of servers. Servers are further divided into **master** and **slave** servers: the master server is the true single owner of the map data. Slave NIS servers handle client requests, but they do not modify the NIS maps. The master server is responsible for all map maintenance and distribution to its slave servers. Once an NIS map is built on the master to include a change, the new map file is distributed to all slave servers.

**Clients** are hosts that request information from these maps.  NIS clients "see" these changes when they perform queries on the map file -- it doesn't matter whether the clients are talking to a master or a slave server, because once the map data is distributed, all NIS servers have the same information.

![NIS Architecture](https://s2.loli.net/2022/01/18/cDJfKxah57UdpkH.png)

NIS uses **domains** to arrange the machines, users, and networks in its **namespace**. However, it does not use a domain hierarchy; an NIS namespace is flat.

## Set Up NIS

Our lab's NIS topology is shown in the figure below.

{% note danger %}
Add a figure!
{% endnote %}

### NIS Master

#### 1.Install NIS

```shell
$ sudo apt install nis
```

You'll be prompted to enter your preferred domain name during the installation process. Here the domain name is `hulk.nis`.

```bash
Preconfiguring packages ...

# input your domain name
+----------------------------| Configuring nis |----------------------------+
| Please choose the NIS "domainname" for this system. If you want this      |
| machine to just be a client, you should enter the name of the NIS domain  |
| you wish to join.                                                         |
|                                                                           |
| Alternatively, if this machine is to be a NIS server, you can either      |
| enter a new NIS "domainname" or the name of an existing NIS domain.       |
|                                                                           |
| NIS domain:                                                               |
|                                                                           |
| hulk.nis_________________________________________________________________ |
|                                                                           |
|                                  <Ok>                                     |
|                                                                           |
+---------------------------------------------------------------------------+
```

#### 2. Edit `/etc/default/nis`

```shell
$ sudo vim /etc/default/nis
```

Specifically, edit `NISSERVER` to be `master`

```bash
#
# /etc/defaults/nis     Configuration settings for the NIS daemons.
#

# Are we a NIS server and if so what kind (values: false, slave, master)?
NISSERVER=false # Should be master
```

#### 3. Edit `/etc/hosts`

In master's `/etc/hosts`, there should be at least all the **slave** servers.

```shell
$ sudo vim /etc/hosts
```

```bash
192.168.0.233 hulk # master
147.46.xx.xxx rogers # slave
```

{% note primary %}
**TIPS**: NIS usually is for info sharing in LAN. However, as long as port `111` (for `rpcbind`) is open, it is capable of doing RPC call across internet.
{% endnote %}

#### 4. Edit `/etc/ypserv/securenets`

```shell
$ sudo vim /etc/ypserv.securenets
```

```bash
# This line gives access to everybody. PLEASE ADJUST!
# comment out
# 0.0.0.0 0.0.0.0
# add to the end: IP range you allow to access
# Allows IP in the form of: 192.168.0.* to access
255.255.255.0   192.168.0.0
# Allows certain internet IP to access:
255.255.255.255 147.46.xx.xxx
```

#### 5. Run `ypinit -m`

By running `ypinit -m`, NIS will utilize the local user system as its cornerstone to build the network user information.

```shell
$ sudo /usr/lib/yp/ypinit -m
```

You'll be prompted to enter the list of servers (master and slave). The first host added should be the hostname of the master server, followed by all the slave servers in the system.

```bash
At this point, we have to construct a list of the hosts which will run NIS
servers.  hulk is in the list of NIS server hosts.  Please continue to add
the names for the other hosts, one per line.  When you are done with the
list, type a <control D>.
        next host to add:  hulk    # master
        next host to add:  rogers  # slave
        next host to add:          # <- Ctrl + D
The current list of NIS servers looks like this:

hulk
rogers


Is this correct?  [y/n: y]  y
We need a few minutes to build the databases...
Building /var/yp/hulk.nis/ypservers...
Running /var/yp/Makefile...
make[1]: Entering directory '/var/yp/hulk.nis'
Updating passwd.byname...
Updating passwd.byuid...
Updating group.byname...
Updating group.bygid...
Updating hosts.byname...
Updating hosts.byaddr...
Updating rpc.byname...
Updating rpc.bynumber...
Updating services.byname...
Updating services.byservicename...
Updating netid.byname...
Updating protocols.bynumber...
Updating protocols.byname...
Updating netgroup...
Updating netgroup.byhost...
Updating netgroup.byuser...
Updating shadow.byname...
make[1]: Leaving directory '/var/yp/hulk.nis'

hulk has been set up as a NIS master server.

Now you can run ypinit -s hulk on all slave server.
```

### NIS client

#### 1. Install NIS

```shell
$ sudo apt install nis
```

During the process it will also prompt to enter domain name. Use the **same** domain as the NIS master.

#### 2. Edit `/etc/yp.conf`

```shell
$ sudo vim /etc/yp.conf
```

Add the master / slave server you want to request info from.

```bash
### FORMAT:
# domain <domainname> server <IP or hostname>
### OR:
# ypserver <IP or hostname>
domain hulk.nis server 192.168.1.233
```

{% note info %}
If you want to use **hostname** instead of **server IP**, make sure to specify it in `/etc/hosts` of the client.
{% endnote %}

#### 3. Edit `/etc/nsswitch.conf`

The **name service switch** (named `nsswitch.conf`) controls how a client machine or application obtains network information.

Each machine has a switch file in its `/etc` directory. Each line of that file identifies a particular type of network information, such as `host`, `password`, and `group`, followed by one or more locations of that information.

A client can obtain naming information from one or more of the switch's sources. For example, an NIS client could obtain its hosts information from an NIS map and its password information from a local `/etc` file. In addition, the client could specify the conditions under which the switch must use each source.

The available information sources are listed in the following table:

| Information Sources | Description                                                                                                                                         |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `files`             | A file stored in the client's `/etc` directory. For example, `/etc/passwd`                                                                          |
| `nisplus`           | An NIS+ table. For example, the `hosts` table.                                                                                                      |
| `nis`               | An NIS map. For example, the `hosts` map.                                                                                                           |
| `compat`            | `compat` can be used for password and group information to support old-style + or - syntax in `/etc/passwd`, `/etc/shadow`, and `/etc/group` files. |
| `dns`               | Can be used to specify that host information be obtained from DNS.                                                                                  |
| `ldap`              | Can be used to specify entries be obtained from the LDAP directory.                                                                                 |

Open `/etc/nsswitch.conf` to add `nis` as an information source.

```shell
$ sudo vim /etc/nsswitch.conf
```

Add `nis` to the end of `passwd`, `group`, `shadow`, `gshadow` and `hosts`.

```bash
# ...
passwd:     files systemd nis  # add nis
group:      files systemd nis  # add nis
shadow:     files nis          # add nis
gshadow:    files nis          # add nis

hosts:      files dns nis   # add nis
# ...
```

{% note primary %}
If you want to look up `nis` first instead of local files. Put `nis [NOTFOUND=return]` in front of `files`. The `[NOTFOUND=return]` search criterion instructs the switch to stop searching the NIS tables if the switch gets a “**No such entry**” message. The switch searches through local files only if the NIS server is unavailable.

```bash
# ...
passwd:     nis [NOTFOUND=return] files systemd  # add nis
# ...
```

{% endnote %}

{% note secondary %}
On `Debian` also:

```shell
/usr/sbin/ypbind -broadcast
```

{% endnote %}

#### 4. Edit `/etc/pam.d/common-session` for creating home directory automatically

```shell
$ vim /etc/pam.d/common-session
```

```bash
# add to the end
session optional       pam_mkhomedir.so =/etc/skel umask=000
```

#### 5. Restart NIS to apply changes

```shell
$ sudo systemctl restart rpcbind
$ sudo systemctl restart nis
```

### NIS Slave

The set up process of slave server is an approximate combination of client set up + master set up.

#### 1. Go through NIS client set-up steps

Specifically Step 1 to 4. This is for establishing RPC connection with master.

#### 2. Edit `/etc/default/nis`

```shell
$ sudo vim /etc/default/nis
```

Specifically, edit `NISSERVER` to be `slave`

```bash
#
# /etc/defaults/nis     Configuration settings for the NIS daemons.
#

# Are we a NIS server and if so what kind (values: false, slave, master)?
NISSERVER=false # Should be slave
```

#### 3. Edit `/etc/ypserv.securenets`

```shell
$ vim /etc/ypserv.securenets
```

It is important to allow the master server to access.

```bash
# This line gives access to everybody. PLEASE ADJUST!
# comment out
# 0.0.0.0 0.0.0.0
# add to the end: IP range you allow to access
# Allows IP in the form of: 192.168.100.* to access
255.255.255.0   192.168.100.0
# Allows certain internet IP to access:
255.255.255.255 147.46.xx.xxx
```

#### 4. Edit `/etc/hosts`

```shell
$ sudo vim /etc/hosts
```

In slave's `/etc/hosts`, there should be at least the IP of **master** server.

```bash
147.47.xxx.xx hulk # master
```

#### 5. Run `ypinit -s <master>`

```shell
$ sudo /usr/lib/yp/ypinit -s hulk
```

This operation will pull information from the master to the slave.

#### Additional steps: master server set up

If the slave master is not added in [Step 5 of master set up](#5-run-ypinit--m). On **master** server, it is necessary to rerun `ypinit -m` and add the slave server when it prompts to enter the hosts.

In order to push the changes in the maps on master, you also need to edit the `/var/yp/Makefile`:

```shell
$ sudo vim /var/yp/Makefile
```

Modify `NOPUSH=fasle` to `NOPUSH=true`.

```bash
#
# Makefile for the NIS databases
#
# ...
NOPUSH=true   # <- edit this to be false

#...
```

## Bibliography

1. [System Administration Guide: Naming and Directory Services (DNS, NIS, and LDAP)](https://docs.oracle.com/cd/E18752_01/html/816-4556/toc.html)
2. [Eisler, M., Labiaga, R., & Stern, H. (2001). *Managing NFS and NIS: Help for Unix System Administrators*. O'Reilly Media, Inc.](https://docstore.mik.ua/orelly/networking_2ndEd/nfs/index.htm)
3. [How to set up NIS for Ubuntu (Master, Client, Slave) -- Junyong Lee](https://codeslake.github.io/ubuntu/nis/setting-up-NIS-for-ubuntu/#setting-up-slave-server)
4. [鳥哥的 Linux 私房菜：伺服器架設篇 -- 第十四章、帳號控管： NIS 伺服器](https://linux.vbird.org/linux_server/centos6/0430nis.php)
