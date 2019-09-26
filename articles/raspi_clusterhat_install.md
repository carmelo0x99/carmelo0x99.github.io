#### On any available computers, write the images onto the SDs
<p>lt;typegt; = CNAT, p1-p4</p>

```
$ sudo dd if=ClusterCTRL-2019-04-08-lite-4-<type>.img of=/dev/<device> bs=1M
```

<p>Before unmounting the SDs, run the following: `$ touch /media/<your-username>/boot/ssh`</p>

### On the Controller board
```
$ sudo vi /media/<your-username>/rootfs/etc/dhcpcd.conf
interface eth0
static ip_address=<ip_address>/24
static routers=<gateway>
static domain_name_servers=<gateway> <server1> <server2>
```

### raspi-config: password, hostname, timezone, SSH

```
$ ssh-keygen -t rsa

$ sudo update-alternatives --config editor

$ for idx in {1..4}; do ssh-copy-id pi@p${idx}.local; done
```
