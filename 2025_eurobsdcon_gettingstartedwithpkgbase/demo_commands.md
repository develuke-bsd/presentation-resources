# Converting existing Hosts
Show freebsd-version
```
freebsd-version -kru
```

Run pkgbasify script:
```
./pkgbasify.lua
```
confirm conversion and creation of boot environment
wait for it to finish
reboot
```
shutdown -r now
```

# Patch level update
show version after pkgbasify:
```
freebsd-version -kru
```
create boot environment
```
bectl create pre-14.2p5
```

go to edit the configuration file (this is only needed for this demonstration purposes).
```
cd /usr/local/etc/pkg/repos
vi FreeBSD-base.conf
```
change 14.2p3 to base_release_2
```
FreeBSD-base: {
  url: "pkg+https://pkg.FreeBSD.org/${ABI}/base_release_2",
  mirror_type: "srv",
  signature_type: "fingerprints",
  fingerprints: "/usr/share/keys/pkg",
  enabled: yes
}
```

```
pkg update
pkg upgrade
```

This should only need restart of services that are affected by the update, but we will make sure to use the updated versions, so let's reboot
```
shutdown -r now
```

# minor upgrade
show version
```
freebsd-version -kru
```
create boot environment
```
bectl create 143p2
```

change config to base_release_3
```
vi /usr/local/etc/pkg/repos/FreeBSD-base.conf
```

```
FreeBSD-base: {
  url: "pkg+https://pkg.FreeBSD.org/${ABI}/base_release_3",
  mirror_type: "srv",
  signature_type: "fingerprints",
  fingerprints: "/usr/share/keys/pkg",
  enabled: yes
}
```
mount boot env
```
mkdir /mnt/143p2
bectl mount 143p2 /mnt/143p2
```
update boot environment
```
pkg -r /mnt/143p2 update
pkg -r /mnt/143p2 upgrade -y
```
temporarily activate it and restart
```
bectl activate -t 143p2
shutdown -r now
```
make it permanent
```
bectl activate 143p2
```

# major upgrade
change config to base_latest
```
vi /usr/local/etc/pkg/repos/FreeBSD-base.conf
```

```
FreeBSD-base: {
  url: "pkg+https://pkg.FreeBSD.org/${ABI}/base_latest",
  mirror_type: "srv",
  signature_type: "fingerprints",
  fingerprints: "/usr/share/keys/pkg",
  enabled: yes
}
```
using bectl:
```
bectl create 15
mkdir /mnt/15
bectl mount 15 /mnt/15
```
lock pkg
```
pkg -r /mnt/15 lock pkg
```
upgrade
```
env ABI=FreeBSD:15:amd64 pkg -r /mnt/15 update -r FreeBSD-base
env ABI=FreeBSD:15:amd64 pkg -r /mnt/15 upgrade -y -r FreeBSD-base
```
unlock pkg
```
pkg -r /mnt/15 unlock pkg
```
update other packages
```
env ABI=FreeBSD:15:amd64 pkg -c /mnt/15 update
env ABI=FreeBSD:15:amd64 pkg -c /mnt/15 upgrade
```
activate boot environment and restart
```
bectl activate -t 15
shutdown -r now
```
activate be permanently
```
bectl activate 15
```

# build your own packages
```
cd /usr/src
git clone https://github.com/freebsd/freebsd-src.git /usr/src
git checkout releng/14.3
```
Build the packages by compiling from source, using make packages to create base system packages.
```
make -j8 buildworld && make -j8 buildkernel && make -j8 packages
```

After building, the packages will get saved into 
```
/usr/obj/usr/src/repo/FreeBSD:14:amd64/14.3p2
```

use nginx to publish the contents of this folder:
```
vi /usr/local/etc/nginx/nginx.conf
```

```nginx
server {
	listen       80;
	server_name builder;
	location /FreeBSD:14:amd64 {
			alias /usr/obj/usr/src/repo/FreeBSD:14:amd64/;
			autoindex on;
	}
	location /FreeBSD:15:amd64 {
			alias /usr/obj/usr/src/repo/FreeBSD:15:amd64/;
			autoindex on;
	}
```