# pihole

DNS server to achieve network-wide ad blocking

## Services available

This project is designed to deploy 2 different Docker Compose services: `pihole-primary` and `pihole-secondary`.

Two different services are needed because they will be deployed to different hosts (to avoid DNS service downtimes) and offer different features.

Both are responsible for providing DNS resolution with ad blocking to local clients. They also offer a web admin panel, exposed via `traefik` using `caddy` reverse-proxy service. This proxy is defined at its own project: [pedroetb-projects/pihole-proxy](https://gitlab.com/pedroetb-projects/pihole-proxy).

### pihole-primary

This service is the main DNS resolution provider. It also offers a DHCP server (disabled by default).

If you enable DHCP server, don't forget to:

- disable any other DHCP servers (at your home router, tpically).
- define a static IP address for your host (because it cannot depend on DHCP server provided by itself).

### pihole-secondary

This service is the backup DNS resolution provider. It doesn't offer a DHCP server to avoid collision with primary service.

When deploying secondary Pi-hole service, don't forget to set variables for this environment, specially `FTLCONF_dns_reply_host_IPv4` and maybe `FTLCONF_dns_revServers` (if using DHCP at primary service).

## Configuration

### Variables

You may define these environment variables for both services (**bold** are mandatory):

| Variable name | Default value |
| - | - |
| **TZ** | `Atlantic/Canary` |
| *TAIL_FTL_LOG* | `1` |
| **FTLCONF_webserver_api_password** | `changeme` |
| *FTLCONF_webserver_port* | `8080` |
| *FTLCONF_misc_etc_dnsmasq_d* | `true` |
| **FTLCONF_dns_reply_host_IPv4** | `127.0.0.1` |
| *FTLCONF_dns_reply_host_IPv6* | `::1` |
| *FTLCONF_dns_upstreams* | `8.8.8.8;1.1.1.1;8.8.4.4;1.0.0.1` |
| *FTLCONF_dns_listeningMode* | `all` |
| *FTLCONF_dns_dnssec* | `false` |
| *FTLCONF_dns_bogusPriv* | `false` |
| *FTLCONF_dns_domainNeeded* | `false` |
| *FTLCONF_dns_revServers* | `true,10.0.0.0/8,10.0.0.1,local` |

Only for `pihole-primary` service, you may define these environment variables:

| Variable name | Default value |
| - | - |
| *FTLCONF_dhcp_active* | `false` |
| *FTLCONF_dhcp_start* | `10.0.0.2` |
| *FTLCONF_dhcp_end* | `10.255.255.254` |
| *FTLCONF_dhcp_router* | `10.0.0.1` |
| *FTLCONF_dhcp_leaseTime* | `24` |
| *FTLCONF_dns_domain* | `local` |
| *FTLCONF_dhcp_ipv6* | `true` |
| *FTLCONF_dhcp_rapidCommit* | `true` |

> :bulb: There is also other environment variables used at compose files, but not propagated to deployed services. Check `deploy/compose*.yaml` files to inspect them.

### Allow port usage from host

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

### Setting local DNS records

To achieve local domain names resolution, you have 3 options to add them:

1. At a `.conf` file in `/etc/dnsmasq.d/` (inside Docker volume). Check [conf file example](#dnsmasq-conf-file-example).
2. To container's `/etc/hosts` file (using `extra_hosts` for Docker Compose or `--add-host` for Docker CLI).
3. From admin webpage utility (`Local DNS -> DNS Records`), which store them in `/etc/pihole/custom.list` (inside Docker volume).

This is the precedence order to apply too, only the first record found will trigger an address-to-name association.

My personal preference was 3, but now I'm using 1, because allows to only define main domain (no need to specify any new subdomains).

#### DNSMasq conf file example

This configuration file, named `99-custom.conf` (for example) and stored at **dnsmasqd-vol** (`/etc/dnsmasq.d/99-custom.conf`), is enough for local domain names resolution for `change.me` and subdomains, associated to `10.0.0.123` IP address.

```sh
server=127.0.0.11
address=/change.me/10.0.0.123
```

Note that `server=127.0.0.11` is set to Docker embedded DNS resolver, to avoid warnings from `dnsmasqd` about missing server configuration. It's only used because `127.0.0.1` address (localhost) is detected as a configuration failure too.

It has no effect, because embedded DNS only handles Docker service discovery/resolution and any other DNS lookups are forwarded to DNS defined at container (or Docker daemon config at host).

### DHCP

Once you setup DHCP server at `pihole-primary` service, you should set custom configuration at `/etc/dnsmasq.d/99-custom.conf` (for example) to inform your clients which DNS providers can use for resolution.

These providers must be pointing to addresses of hosts where both Pi-hole services are running (by default, will report only IP of `pihole-primary` service host). For example, if your IPs are `10.0.0.123` and `10.0.0.124`:

```sh
dhcp-option=6,10.0.0.123,10.0.0.124
```
