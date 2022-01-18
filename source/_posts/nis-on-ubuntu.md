---
title: How To Set Up NIS for Ubuntu and Debian servers
date: 2022-01-18 17:32:40
categories: [devops]
tags: [nis, linux, ubuntu]
---

Network Information Service (NIS) is a distributed naming service based on Remote Procedure Call (RPC). It

<!-- more -->

## NIS architecture overview

WIP

## Set Up NIS

### Set Up NIS Master

1. Install NIS

   ```shell
   $ sudo -i
   root# apt install nis
   ```

   ```shell
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

1. Edit `/etc/default/nis`

   ```shell
    #
    # /etc/defaults/nis     Configuration settings for the NIS daemons.
    #

    # Are we a NIS server and if so what kind (values: false, slave, master)?
    NISSERVER=false # Should be master
              ^^^^^
              Change this from false to master
   ```

2. Edit `/etc/hosts`

   ```shell
    192.168.1.233 hulk
   ```

3. Run `ypinit -m` to configure NIS master

   ```shell
    root# /usr/lib/yp/ypinit -m

    At this point, we have to construct a list of the hosts which will run NIS
    servers.  hulk is in the list of NIS server hosts.  Please continue to add
    the names for the other hosts, one per line.  When you are done with the
    list, type a <control D>.
            next host to add:  hulk
            next host to add:
    The current list of NIS servers looks like this:

    hulk


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

4. Import local users and groups

   ```shell
   make -C /var/yp
   ```

### NIS client

1. Also install NIS

    ```shell
    root# apt install nis
    ```

    During the process it will also prompt to enter domain. Use the **same** domain as the NIS master.

2. Add the following line to `/etc/yp.conf`

    ```shell
    domain hulk.nis server 192.168.1.233 # server IP or hostname.
    ```

    {% note info %}
    If you want to use **hostname** instead of **server IP**, make sure to specify it in `/etc/hosts` of the client
    {% endnote %}

3. Edit `/etc/nsswitch.conf`. It's very important

```shell
 vim /etc/nsswitch.conf

 passwd:     compat nis      # line 7; add
 group:      compat nis      # add
 shadow:     compat nis      # add

 hosts:      files dns nis   # add
```

On Debian also:

```shell
/usr/sbin/ypbind -broadcast
```

4. Edit `/etc/pam.d/common-session` for creating home directory automatically

```shell
$ vim /etc/pam.d/common-session

# add to the end
session optional       pam_mkhomedir.so =/etc/skel umask=000
```

5. Restart NIS

```shell
$ sudo systemctl restart rpcbind
$ sudo systemctl restart nis
```
