---
layout: post
title: "Install ClusterHAT software"
categories: misc
---

### On any available computers, write the images onto the SDs
`<type>` = CNAT, p[N]-p4<br/>
`<device>` = e.g /dev/sdb, /dev/mmcblk0...<br/>
**NOTE**: in Sep/2019 I've chosen "Stretch" hence "2019-04-08"
```
$ sudo dd if=ClusterCTRL-2019-04-08-lite-4-**<type>**.img of=/dev/**<device>** bs=1M
```

Before unmounting the SDs, run the following:
```
$ touch /media/<your-username>/boot/ssh
```

### On the Controller board
```
$ sudo vi /media/<your-username>/rootfs/etc/dhcpcd.conf
interface eth0
static ip_address=<ip_address>/24
static routers=<gateway>
static domain_name_servers=<gateway> <server1> <server2>
```

### raspi-config (or manually)
Edit: password, hostname, timezone, SSH

### Additional steps on Controller
```
$ ssh-keygen -t rsa

$ for idx in {1..4}; do ssh-copy-id pi@p${idx}.local; done
```
Also, set `vi` as the default editor:
```
$ sudo update-alternatives --config editor
```

### Using ClusterHAT
```
$ clusterhat status
clusterhat:1
clusterctrl:False
maxpi:4
...
p1:0
p2:0
p3:0
p4:0

$ clusterhat on

$ clusterhat status
clusterhat:1
clusterctrl:False
maxpi:4
...
p1:1
p2:1
p3:1
p4:1
```

#### Serial connection (useful to troubleshoot SSH issues)
```
$ minicom p[N]
 p[N] login: pi
 Password:
 Last login: Tue Sep 24 15:37:25 CEST 2019 from xxx.xxx.xxx.xxx on pts/0
 Linux p[N] 4.19.66+ #1253 Thu Aug 15 11:37:30 BST 2019 armv6l
...
```
**NOTE**: to quit Minicom use CTRL-A-X

#### All commands
Command | Purpose
--- | ---
`$ clusterctrl on` | Turn power to all Pi Zero on
`$ clusterctrl off` | Turn power to all Pi Zero off
`$ clusterctrl on p1` | Turn power on to Pi Zero in slot P1
`$ clusterctrl on p1 p3 p4` | Turn power to Pi Zeros in slot P1, P3 and P4 on
`$ clusterctrl off p2 p3` | Turn power off to Pi Zeros in slots P2 and P3
`$ clusterctrl alert on` | Turns on ALERT LED
`$ clusterctrl alert off` | Turns off ALERT LED
`$ clusterctrl hub on` | Turns on USB hub (default)
`$ clusterctrl hub off` | Turns off USB hub
`$ clusterctrl led on` | Enables Power & P1-P4 LED on Cluster HAT (default) 
`$ clusterctrl led off` | Disables Power & P1-P4 LED on Cluster HAT (does not disable ALERT LED)
`$ clusterctrl wp on` | Write protects HAT EEPROM
`$ clusterctrl wp off` | Disables EEPROM write protect (only needed for updates)

