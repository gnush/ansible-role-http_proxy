# HTTP Proxy (Squid) Ansible Role

Installs and configures [Squid](https://www.squid-cache.org/) as an HTTP/HTTPS forward proxy,
optionally on several nodes behind [HAProxy](https://www.haproxy.org/) with a
[Keepalived](https://www.keepalived.org/) VIP.

## Requirements

- Ubuntu >= 22.04  
  Debian >= 12
- Ansible >= 2.11
- Collections `ansible.posix`, `ansible.utils`
- Root privileges (`become: true`)

## Role Variables

Defined in `defaults/main.yml`:

| Variable                     | Default                                | Description                                              |
|------------------------------|----------------------------------------|----------------------------------------------------------|
| `proxy_ha_enabled`           | `true`                                 | HAProxy + Keepalived; `false` for a single Squid node    |
| `proxy_haproxy_port`         | `3128`                                 | Client-facing port (HAProxy)                             |
| `proxy_squid_port`           | `3129`                                 | Squid port                                               |
| `proxy_allowed_networks`     | RFC 1918 + link-local                  | Clients allowed to use the proxy                         |
| `proxy_restrict_to_internet` | `true`                                 | Deny destinations in `proxy_internal_networks`           |
| `proxy_internal_networks`    | RFC 1918 + link-local                  | Destinations denied when `proxy_restrict_to_internet`    |
| `proxy_safe_ports`           | Squid defaults                         | Allowed destination ports                                |
| `proxy_ssl_ports`            | `443`                                  | Allowed `CONNECT` ports                                  |
| `proxy_squid_cache_size_mb`  | `1024`                                 | Disk cache size                                          |
| `proxy_haproxy_stats_enabled`| `true`                                 | Stats page on `127.0.0.1:7777/stats`                     |
| `proxy_vip_cidr`             | `ansible_default_ipv4.prefix`          | VIP prefix length                                        |
| `proxy_vip_interface`        | `ansible_default_ipv4.interface`       | VIP interface                                            |
| `proxy_vrrp_router_id`       | `51`                                   | VRRP router ID, unique per L2 segment                    |
| `proxy_vrrp_auth_pass`       | `changeme`                             | VRRP password, keep in Vault                             |

Required when `proxy_ha_enabled` (checked in `tasks/prechecks.yml`):

| Variable           | Description                                                               |
|--------------------|---------------------------------------------------------------------------|
| `proxy_peer_group` | Inventory group of the proxy nodes; HAProxy backends and Keepalived peers |
| `proxy_vip`        | Virtual IPv4 address clients connect to                                   |
| `proxy_vip_host`   | Host in `proxy_peer_group` that starts as Keepalived MASTER               |

## Architecture

```
               Client
                 |
                 v
        VIP (Keepalived) :3128
                 |
       ┌─────────┴─────────┐
       v                   v
  HAProxy (proxy1)    HAProxy (proxy2)
       |                   |
       └──► Squid pool ◄───┘
      (proxy1:3129, proxy2:3129)
```

The VIP is active/passive: `proxy_vip_host` is MASTER, all other hosts in
`proxy_peer_group` are BACKUP. HAProxy balances round-robin across all Squid nodes.

With `proxy_ha_enabled: false`, only Squid is installed; clients connect to
`proxy_squid_port` directly.

## Example Setup

`site.yml`:

```yaml
- hosts: proxy_nodes
  become: true
  roles:
    - http_proxy
```

`inventory/hosts.yml`:

```yaml
proxy_nodes:
  vars:
    proxy_peer_group: proxy_nodes
    proxy_vip: 192.168.0.10
    proxy_vip_host: proxy1
  hosts:
    proxy1:
      ansible_host: 192.168.0.11
    proxy2:
      ansible_host: 192.168.0.12
```

`ansible_host` must be set per host; HAProxy uses it as backend address.

## Access Control

Only clients from `proxy_allowed_networks` are accepted. HAProxy rejects other
sources before they reach Squid; Squid checks the same list for direct connections.

With `proxy_restrict_to_internet: true` (default), allowed clients can only reach
destinations outside `proxy_internal_networks`, i.e. the proxy cannot be used to reach
other internal systems.

## Verify

```bash
curl -v -x http://192.168.0.10:3128 https://example.org
```

Failover: stop HAProxy on the MASTER and check that the VIP moves to the BACKUP.

```bash
sudo systemctl stop haproxy     # on MASTER
ip -4 addr show                 # on BACKUP
```
