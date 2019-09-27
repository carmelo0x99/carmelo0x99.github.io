### On any available computers, write the images onto the SDs
**&lt;type&gt;** = CNAT, p1-p4<br/>
**&lt;device&gt;** = e.g /dev/sdb, /dev/mmcblk0...<br/>
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
