# pihole

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
