```
$ sudo dd if=ClusterCTRL-2019-04-08-lite-4-***.img of=/dev/sdb bs=1M
```

### On CTRL:
```
$ sudo vi /media/<your-username>/rootfs/etc/dhcpcd.conf
interface eth0
static ip_address=10.0.2.205/24
static routers=10.0.2.1
static domain_name_servers=10.0.2.1 208.67.220.220

$ touch /media/<your-username>/boot/ssh

$ sudo vi /media/<your-username>/rootfs/etc/ssh/sshd_config
```

### raspi-config: password, hostname, timezone, SSH

```
$ ssh-keygen -t rsa

$ sudo update-alternatives --config editor

$ for idx in {1..4}; do ssh-copy-id pi@p${idx}.local; done
```
