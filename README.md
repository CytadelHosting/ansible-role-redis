# Ansible Role: Redis

Install and configure Redis on Linux with modern defaults for cache/session workloads.

## Features

- Modern Redis baseline (`protected-mode`, Unix socket, safer command set).
- Fully variable-driven template (`templates/redis.conf.j2`).
- Host tuning support:
  - persisted sysctl settings,
  - persisted THP disable (systemd oneshot),
  - Redis systemd override (`LimitNOFILE`, `UMask`).
- Cross-distro package support (Debian, RedHat, Archlinux).

## Requirements

- Supported service manager for full tuning features: `systemd`.
- On RedHat-based distributions, EPEL may be required.

## Role Variables

All defaults are defined in `defaults/main.yml`.

### Package/service

```yaml
redis_enablerepo: epel
redis_enabled: true
# redis_package is auto-detected per OS but can be overridden.
```

### Redis process/network/security

```yaml
redis_daemonize: "no"
redis_pidfile: ""
redis_port: 6379
redis_bind_interface: 127.0.0.1
redis_unixsocket: /run/redis/redis.sock
redis_unixsocketperm: "770"
redis_protected_mode: "yes"
redis_timeout: 0
redis_tcp_backlog: 511
redis_tcp_keepalive: 300
redis_hz: 10
redis_dynamic_hz: "yes"
```

Notes:

- `redis_bind_interface` can be a string or a list.
- `redis_pidfile: ""` uses the role fallback path `/var/run/redis/<daemon>.pid`.

### Logs and databases

```yaml
redis_loglevel: "notice"
redis_logfile: /var/log/redis/redis-server.log
redis_databases: 16
```

### Persistence

```yaml
redis_save:
  - '""'
redis_rdbcompression: "yes"
redis_dbfilename: dump.rdb
redis_dbdir: /var/lib/redis
redis_appendonly: "yes"
redis_appendfsync: "everysec"
redis_auto_aof_rewrite_percentage: 100
redis_auto_aof_rewrite_min_size: 256mb
```

Notes:

- `redis_save: ['""']` renders `save ""` (disable RDB snapshotting).
- You can also set `redis_save: []`; template fallback still renders `save ""`.

### Memory and eviction

```yaml
redis_maxmemory: 1gb
redis_maxmemory_policy: "volatile-lru"
redis_maxmemory_samples: 5
redis_lazyfree_lazy_eviction: "yes"
redis_lazyfree_lazy_expire: "yes"
redis_lazyfree_lazy_server_del: "yes"
redis_lazyfree_lazy_user_del: "yes"
redis_activedefrag: "yes"
```

### Security/authentication

```yaml
redis_requirepass: ""
redis_acl_users: []
redis_disabled_commands:
  - FLUSHALL
  - FLUSHDB
  - CONFIG
  - SHUTDOWN
```

Notes:

- If `redis_acl_users` is non-empty, ACL lines are rendered.
- `redis_requirepass` is used as fallback when ACL is not configured.

### Extra config/includes

```yaml
redis_includes: []
redis_extra_config: ""
```

### System tuning (kernel + systemd)

```yaml
redis_manage_system_tuning: true
redis_sysctl_settings:
  vm.overcommit_memory: "1"
  net.core.somaxconn: "1024"
redis_disable_thp: true
redis_thp_enabled_value: "never"
redis_thp_defrag_value: "never"
redis_manage_systemd_override: true
redis_systemd_limit_nofile: 100000
redis_systemd_umask: "007"
```

Implementation details:

- Sysctl file: `/etc/sysctl.d/99-redis.conf` (applied with `sysctl --system`).
- THP unit: `/etc/systemd/system/disable-transparent-huge-pages.service`.
- Redis drop-in: `/etc/systemd/system/<redis-service>.service.d/override.conf`.

## Example Playbook

```yaml
- hosts: all
  become: true
  roles:
    - role: geerlingguy.redis
```

## Example Override (production-like)

```yaml
redis_bind_interface: "127.0.0.1"
redis_port: 6379

redis_acl_users:
  - "user default off"
  - "user phpapp on >CHANGEMOI_ULTRA_LONG ~* +@all"

redis_maxmemory: 1gb
redis_maxmemory_policy: volatile-lru
redis_appendonly: "yes"
redis_appendfsync: everysec

redis_sysctl_settings:
  vm.overcommit_memory: "1"
  net.core.somaxconn: "1024"
```

## Fork Workflow

Synchronization/tagging workflow for this fork is documented in `README-synchro-upstream.md`.

## License

MIT / BSD

## Upstream Credits

Original role created by [Jeff Geerling](https://www.jeffgeerling.com/).
