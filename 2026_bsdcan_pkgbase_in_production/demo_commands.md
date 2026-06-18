# build your own packages
```
cd /usr/src
git clone https://github.com/freebsd/freebsd-src.git /usr/src
git checkout release/15.0.0
```
Build the packages by compiling from source, using make packages to create base system packages.
```
make -j8 buildkernel && make -j8 buildworld && make -j8 packages
```
checkout the next release
```
git checkout release/15.0.0-p1
```

and use update-packages for that

```
make -j8 buildkernel && make -j8 buildworld && make -j8 update-packages
```
repeat for every release until the last recent.

After building, the packages will get saved into 
```
/usr/obj/usr/src/repo/FreeBSD:15:amd64/15.1px
```

use nginx to publish the contents of this folder:
```
vi /usr/local/etc/nginx/nginx.conf
```

```nginx
server {
	listen       80;
	server_name builder;
	location / {
			alias /usr/obj/usr/src/repo/;
			autoindex on;
	}
```

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
bectl create 150p10
mkdir /mnt/upgrade
bectl mount 150p10 /mnt/upgrade
```


go to edit the configuration file (this is only needed for this demonstration purposes).
```
vi /mnt/upgrade/usr/local/etc/pkg/repos/FreeBSD-base.conf
```
change 15.0 to base_release_0
```
FreeBSD-base: {
  url: "http://builder/${ABI}/base_release_0",
  enabled: yes
}
```

```
pkg -c /mnt/upgrade upgrade
```
Time:
5.30 real
0.99 user
3.23 sys

activate boot env
```
bectl activate -t 150p10
```
reboot
```
shutdown -r now
```

```
bectl activate 150p10
```
# minor upgrade
show version
```
freebsd-version -kru
```
create boot environment
```
bectl create 151
bectl mount 151 /mnt/upgrade/
```

change config to base_release_1
```
vi /mnt/upgrade/usr/local/etc/pkg/repos/FreeBSD-base.conf
```

```
FreeBSD-base: {
  url: "http://builder/${ABI}/base_release_1",
  enabled: yes
}
```

update boot environment
```
pkg -c /mnt/upgrade upgrade
```
Time:
6.43 real
1.74 user
3.24 sys

temporarily activate it and restart
```
bectl activate -t 151
shutdown -r now
```
make it permanent
```
bectl activate 151
```

# major upgrade
using bectl:
```
bectl create 16
bectl mount 16 /mnt/upgrade
```

change config to base_latest
```
vi /mnt/upgrade/usr/local/etc/pkg/repos/FreeBSD-base.conf
```

```
FreeBSD-base: {
  url: "http://builder/${ABI}/base_latest",
  enabled: yes
}
```

upgrade
```
env ABI=FreeBSD:16:amd64 pkg -c /mnt/upgrade upgrade -r FreeBSD-base
env ABI=FreeBSD:16:amd64 pkg -c /mnt/upgrade upgrade -r FreeBSD-ports
```
Time (base):
6.35 real
2.34 user
2.83 sys

activate boot environment and restart
```
bectl activate -t 16
shutdown -r now
```
activate be permanently
```
bectl activate 16
```
# major upgrade 14-15
using bectl:
```
bectl create 15
bectl mount 15 /mnt/upgrade
```

change config to base_latest
```
vi /mnt/upgrade/usr/local/etc/pkg/repos/FreeBSD-base.conf
```

```
FreeBSD-base: {
  url: "http://builder/${ABI}/base_release_0",
  enabled: yes
}
```

upgrade
```
env ABI=FreeBSD:15:amd64 pkg -c /mnt/upgrade install FreeBSD-set-minimal FreeBSD-kernel-generic FreeBSD-ssh FreeBSD-bsdconfig
pkg -c /mnt/upgrade remove -fy -g 'FreeBSD-*-14.3*'
env ABI=FreeBSD:15:amd64 pkg -c /mnt/upgrade install -f FreeBSD-set-minimal FreeBSD-kernel-generic FreeBSD-ssh FreeBSD-bsdconfig
```

change ports and kmods config
```
cp /etc/pkg/FreeBSD.conf /mnt/upgrade/usr/local/etc/pkg/repos/
vi /mnt/upgrade/usr/local/etc/pkg/repos/FreeBSD.conf
```
FreeBSD-ports, FreeBSD-ports-kmods (add ports in repo name)
```
env ABI=FreeBSD:15:amd64 pkg -c /mnt/upgrade upgrade -r FreeBSD-ports
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

# Jail with base system packages
create dataset
```
zfs create tank/foo
```
if packages are not signed, you can simply use the pkg rootdir option
```
pkg -r /tank/foo/ install FreeBSD-set-minimal-jail
```
enable jails on the host
```
sysrc jail_enable=YES
```

install FreeBSD-jail and FreeBSD-bsdconfig if using a minimal host
```
pkg install FreeBSD-jail FreeBSD-bsdconfig
```

else use bsdinstall like documented in jail(8) EXAMPLES

```
bsdinstall jail /tank/foo/
```

if you are using your own pkg configuration, give the BSDINSTALL_PKG_REPOS_DIR
```
env BSDINSTALL_PKG_REPOS_DIR=/usr/local/etc/pkg/repos/ bsdinstall jail /tank/foo/
```

create a pkg conf

```
vim /etc/jail.conf.d/pkgbase.conf
```

```
pkgbase {
  path = /zroot/jails/pkgbase;
  mount.devfs;
  host.hostname = pkgbase;
  ip4 = inherit;
  exec.start = "/bin/sh /etc/rc";
  exec.stop = "/bin/sh /etc/rc.shutdown jail";
}
```

copy resolv conf inside the jail
```
cp /etc/resolv.conf /zroot/jails/pkgbase/etc/
```
start and enter the jail
```
service jail start pkgbase
jexec pkgbase
```

# Poudriere pkgbase jail
fetch via url
```
poudriere jail –c –j 16current –m pkgbase=latest –v 16 –U http://pkgbase-host.local
```
Build from local source tree
```
poudriere jail –c –B –j 16current –m src=/usr/src –b –v 16-CURRENT –K GENERIC
```