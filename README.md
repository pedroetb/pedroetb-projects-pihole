# pihole

DNS server to achieve network-wide ad blocking

## Allow port usage from host

You may have to prepare host system first in order to bind DNS and DHCP ports to Pi-hole.

Tipically, you will need to configure `systemd-resolved.service`, disabling DNS stub listener and setting some public DNS servers:

```sh
# stop local DNS service
$ systemctl stop systemd-resolved

# edit local DNS configuration adding these options
$ vim /etc/systemd/resolved.conf
...
DNS=8.8.8.8 1.1.1.1
DNSStubListener=no
...

# symlink /etc/resolv.conf file
$ ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf

# start local DNS service again
$ systemctl start systemd-resolved
```

Maybe you will need to disable `lxc-net.service` too:

```sh
systemctl stop lxc-net.service
systemctl disable lxc-net.service
systemctl mask lxc-net.service
```

## Setting local DNS records

To achieve local domain names resolution, you have 3 options to add them:

1. At a `.conf` file in `/etc/dnsmasq.d/` (inside Docker volume). Check [conf file example](#dnsmasq-conf-file-example).
2. To container's `/etc/hosts` file (using `extra_hosts` for Docker Compose or `--add-host` for Docker CLI).
3. From admin webpage utility (`Local DNS -> DNS Records`), which store them in `/etc/pihole/custom.list` (inside Docker volume).

This is the precedence order to apply too, only the first record found will trigger an address-to-name association.

My personal preference was 3, but now I'm using 1, because allows to only define main domain (no need to specify any new subdomains).

### DNSMasq conf file example

This configuration file, named `99-custom.conf` (for example) and stored at **dnsmasqd-vol** (`/etc/dnsmasq.d/99-custom.conf`), is enough for local domain names resolution for `change.me` and subdomains, associated to `10.0.0.123` IP address.

```sh
server=127.0.0.11
address=/change.me/10.0.0.123
```

Note that `server=127.0.0.11` is set to Docker embedded DNS resolver, to avoid warnings from `dnsmasqd` about missing server configuration. It's only used because `127.0.0.1` address (localhost) is detected as a configuration failure too.

It has no effect, because embedded DNS only handles Docker service discovery/resolution and any other DNS lookups are forwarded to DNS defined at container (or Docker daemon config at host).
