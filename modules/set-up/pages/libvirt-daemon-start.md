= enable libvirt's modular daemons
Nick Hardiman 
:source-highlighter: highlight.js
:revdate: 05-01-2022

looks like libvirtd is out. 
https://libvirt.org/daemons.html#switching-to-modular-daemons


The https://libvirt.org/[libvirt] project provides a virtualization management system. 
It ties together a few infrastructure building blocks like QEMU, bridge, DHCP and DNS.


== the daemon libvirtd

The libvirtd daemon used to manage virtualization. 
It's still available, but disabled by default. 
The docs call libvirtd the monolithic daemon.

[source,shell]
----
[nick@host1 ansible]$ systemctl status libvirtd
○ libvirtd.service - Virtualization daemon
     Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; disabled; vendor>
     Active: inactive (dead)
TriggeredBy: ○ libvirtd-tls.socket
             ○ libvirtd-admin.socket
             ○ libvirtd.socket
             ○ libvirtd-tcp.socket
             ○ libvirtd-ro.socket
       Docs: man:libvirtd(8)
             https://libvirt.org
[nick@host1 ansible]$ 
----

Everything is disabled. 

[source,shell]
----
[nick@host1 ansible]$ systemctl list-unit-files | grep libvirt
libvirt-guests.service                     disabled        disabled
libvirtd.service                           disabled        disabled
libvirtd-admin.socket                      disabled        disabled
libvirtd-ro.socket                         disabled        disabled
libvirtd-tcp.socket                        disabled        disabled
libvirtd-tls.socket                        disabled        disabled
libvirtd.socket                            disabled        disabled
[nick@host1 ansible]$ 
----


== before 

State of the system 

=== network interfaces

This machine has a loopback interface (lo), a wired ethernet interface (enp2s0f0) and a WiFi interface (wlp3s0).

[source,shell]
----
[nick@host1 ansible]$ ip -brief addr show
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp2s0f0         UP             192.168.1.195/24 2a00:23a8:4b47:fc00:264b:feff:fec8:40a9/64 fdaa:bbcc:ddee:0:264b:feff:fec8:40a9/64 fe80::264b:feff:fec8:40a9/64 
wlp3s0           DOWN           
[nick@host1 ansible]$ 
----

The virtqemud daemon creates extra network interfaces when it starts, but this hasn't happened yet. 


=== run files 

[source,shell]
----
[root@host1 ~]# ls /var/run/libvirt/
[root@host1 ~]# 
----


=== log files 

[source,shell]
----
[root@host1 ~]# ls /var/log/libvirt/qemu/
[root@host1 ~]# 


== the daemon virtqemud

The virtqemud daemon is one of the new replacements for libvirtd. 

The OS does not show any new network interfaces.

=== systemd unit files for virtqemud

Thesse are enabled. 
That means the next time the computer boots, these services and sockets are started by systemd. 

[source,shell]
----
[root@host1 ~]# systemctl list-unit-files | grep virtqemu
virtqemud.service                          enabled         enabled
virtqemud-admin.socket                     enabled         disabled
virtqemud-ro.socket                        enabled         disabled
virtqemud.socket                           enabled         disabled
[root@host1 ~]# 
----


== libvirt services 

The core service is virtqemud. 
Plenty of services support the work of virtqemud. 
Each service manages part of the system. 

Each service name starts with virt (short for libvirt virtualization management system), and ends in d (for daemon), such as _virtqemud_.

libvirt modular daemons 

* virtqemud - QEMU virtual machines
* virtnetworkd - virtual networks
* virtnodedevd - host devices
* virtnwfilterd - network filters
* virtsecretd - secret data
* virtstoraged - storage pools

what about these ???

* virtinterfaced - host network interfaces
* virtlockd - resource locks
* virtlogd - logs from VM consoles
* virtproxyd - connections from remote hosts


[source,shell]
----
[root@host1 ~]# systemctl list-unit-files --type=service virt*
UNIT FILE              STATE    VENDOR PRESET
virtinterfaced.service disabled disabled     
virtlockd.service      indirect disabled     
virtlogd.service       indirect disabled     
virtnetworkd.service   disabled disabled     
virtnodedevd.service   disabled disabled     
virtnwfilterd.service  disabled disabled     
virtproxyd.service     disabled disabled     
virtqemud.service      enabled  enabled      
virtsecretd.service    disabled disabled     
virtstoraged.service   disabled disabled     

10 unit files listed.
[root@host1 ~]# 
----


== enable libvirt services

Enable most of the modular daemons. 
virtlockd and virtlogd are indirect ???
virtinterfaced is not required because ???
virtproxyd is only needed for requests from other hosts. 


[source,shell]
----
[root@host1 ~]# for drv in qemu network nodedev nwfilter secret storage
  do
    systemctl enable virt${drv}d.service
  done
Created symlink /etc/systemd/system/sockets.target.wants/virtlogd.socket → /usr/lib/systemd/system/virtlogd.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtlockd.socket → /usr/lib/systemd/system/virtlockd.socket.
Created symlink /etc/systemd/system/multi-user.target.wants/virtnetworkd.service → /usr/lib/systemd/system/virtnetworkd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/virtnodedevd.service → /usr/lib/systemd/system/virtnodedevd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/virtnwfilterd.service → /usr/lib/systemd/system/virtnwfilterd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/virtsecretd.service → /usr/lib/systemd/system/virtsecretd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/virtstoraged.service → /usr/lib/systemd/system/virtstoraged.service.
[root@host1 ~]# 
----


== libvirt sockets 

These services must be enabled to work ??? but the don't have to be running. 
When a client sends a request to a socket, systemd starts the service. 

Many of these sockets have an extra admin socket and a read-only socket. 
In addition to virtinterfaced.socket, there is a virtinterfaced-admin.socket and a virtinterfaced-ro.socket.

[source,shell]
----
[root@host1 ~]# systemctl list-unit-files --type=socket virt*
UNIT FILE                   STATE    VENDOR PRESET
virtinterfaced-admin.socket disabled disabled     
virtinterfaced-ro.socket    disabled disabled     
virtinterfaced.socket       enabled  enabled      
virtlockd-admin.socket      disabled disabled     
virtlockd.socket            disabled disabled     
virtlogd-admin.socket       disabled disabled     
virtlogd.socket             disabled enabled      
virtnetworkd-admin.socket   disabled disabled     
virtnetworkd-ro.socket      disabled disabled     
virtnetworkd.socket         enabled  enabled      
virtnodedevd-admin.socket   disabled disabled     
virtnodedevd-ro.socket      disabled disabled     
virtnodedevd.socket         enabled  enabled      
virtnwfilterd-admin.socket  disabled disabled     
virtnwfilterd-ro.socket     disabled disabled     
virtnwfilterd.socket        enabled  enabled      
virtproxyd-admin.socket     disabled disabled     
virtproxyd-ro.socket        disabled disabled     
virtproxyd-tcp.socket       disabled disabled     
virtproxyd-tls.socket       disabled disabled     
virtproxyd.socket           enabled  enabled      
virtqemud-admin.socket      enabled  disabled     
virtqemud-ro.socket         enabled  disabled     
virtqemud.socket            enabled  disabled     
virtsecretd-admin.socket    disabled disabled     
virtsecretd-ro.socket       disabled disabled     
virtsecretd.socket          enabled  enabled      
virtstoraged-admin.socket   disabled disabled     
virtstoraged-ro.socket      disabled disabled     
virtstoraged.socket         enabled  enabled      

30 unit files listed.
[root@host1 ~]# 
----




== enable libvirt sockets

[source,shell]
----
[root@host1 ~]# for drv in qemu network nodedev nwfilter secret storage
  do
    systemctl enable virt${drv}d{,-ro,-admin}.socket
  done
Created symlink /etc/systemd/system/sockets.target.wants/virtnetworkd-ro.socket → /usr/lib/systemd/system/virtnetworkd-ro.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtnetworkd-admin.socket → /usr/lib/systemd/system/virtnetworkd-admin.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtnodedevd-ro.socket → /usr/lib/systemd/system/virtnodedevd-ro.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtnodedevd-admin.socket → /usr/lib/systemd/system/virtnodedevd-admin.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtnwfilterd-ro.socket → /usr/lib/systemd/system/virtnwfilterd-ro.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtnwfilterd-admin.socket → /usr/lib/systemd/system/virtnwfilterd-admin.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtsecretd-ro.socket → /usr/lib/systemd/system/virtsecretd-ro.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtsecretd-admin.socket → /usr/lib/systemd/system/virtsecretd-admin.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtstoraged-ro.socket → /usr/lib/systemd/system/virtstoraged-ro.socket.
Created symlink /etc/systemd/system/sockets.target.wants/virtstoraged-admin.socket → /usr/lib/systemd/system/virtstoraged-admin.socket.
[root@host1 ~]# 
----



== start libvirt sockets

[source,shell]
----
[root@host1 ~]# for drv in qemu network nodedev nwfilter secret storage
  do
    systemctl start virt${drv}d{,-ro,-admin}.socket
  done
[root@host1 ~]# 
----


== after start

Many files in /var/run/libvirt/

Still no files in /var/log/libvirt/qemu/


=== a new network interface, virbr0

 the bridge device virbr0

A bridge is a kind of internal layer 2 switch that connects virtual machines to the physical network.

virbr0 is a bridge. 


[source,shell]
----
[root@host1 ~]# ip -brief addr show
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp2s0f0         UP             192.168.1.195/24 2a00:23a8:4b47:fc00:264b:feff:fec8:40a9/64 fdaa:bbcc:ddee:0:264b:feff:fec8:40a9/64 fe80::264b:feff:fec8:40a9/64 
wlp3s0           DOWN           
virbr0           DOWN           192.168.122.1/24 
[root@host1 ~]# 
----

NetworkManager is configured to manage virbr0.

[source,shell]
----
[root@host1 ~]# nmcli con show
NAME      UUID                                  TYPE      DEVICE   
enp2s0f0  7789c0b1-de1c-330f-ba1e-5badaf2c8215  ethernet  enp2s0f0 
virbr0    395a2d99-26fe-4b4d-9f21-8949c129aaba  bridge    virbr0   
[root@host1 ~]# 
----



=== new DHCP and DNS services, provided by dnsmasq

dnsmasq handles some infrastructure services for the virtual network. 

[source,shell]
----
[root@host1 ~]# ps -C dnsmasq
    PID TTY          TIME CMD
  61504 ?        00:00:00 dnsmasq
  61505 ?        00:00:00 dnsmasq
[root@host1 ~]# 
----


=== a new character device, kvm 

The /dev/ directory has a character device named kvm. 
You can tell this file is a character device because the long list starts with a "c".

[source,shell]
----
[root@host1 ~]# ls -l /dev/kvm 
crw-rw-rw-. 1 root kvm 10, 232 Jan  3 18:29 /dev/kvm
[root@host1 ~]# 
----






== the virtqemud daemon is started on demand.

virtqemud doesn't run all the time. 
Systemd starts libvirtd when there is work to do. 

Use the virsh tool to list VMs (there aren't any yet).
This starts virtqemud.
It runs for couple minutes then stops. 

If you skipped the section above where sockets are started, this command shows an error. 

[source,shell]
----
[root@host1 ~]# virsh list
 Id   Name   State
--------------------

[root@host1 ~]# 
----

You can also start the virtqemud daemon manually, with systemctl. 

[source,shell]
----
[root@host1 ~]# systemctl start virtqemud 
[root@host1 ~]# 
----

The status display shows Started and Deactivated messages, two minutes apart. 

[source,shell]
----
[root@host1 ~]# systemctl status --no-pager -l virtqemud
○ virtqemud.service - Virtualization qemu daemon
     Loaded: loaded (/usr/lib/systemd/system/virtqemud.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Fri 2022-01-07 20:31:49 GMT; 18min ago
TriggeredBy: ● virtqemud.socket
             ● virtqemud-ro.socket
             ● virtqemud-admin.socket
       Docs: man:libvirtd(8)
             https://libvirt.org
   Main PID: 61281 (code=exited, status=0/SUCCESS)
        CPU: 30ms

Jan 07 20:29:49 host1.lab.example.com systemd[1]: Starting Virtualization qemu daemon...
Jan 07 20:29:49 host1.lab.example.com systemd[1]: Started Virtualization qemu daemon.
Jan 07 20:31:49 host1.lab.example.com systemd[1]: virtqemud.service: Deactivated successfully.
[root@host1 ~]# 
----


