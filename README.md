# Ansible Role: Redis

[![CI](https://github.com/geerlingguy/ansible-role-php-redis/actions/workflows/ci.yml/badge.svg)](https://github.com/geerlingguy/ansible-role-php-redis/actions/workflows/ci.yml)

Installs [Redis](http://redis.io/) on Linux.

## Requirements

On RedHat-based distributions, requires the EPEL repository (you can simply add the role `geerlingguy.repo-epel` to install ensure EPEL is available).

## Role Variables

```yaml
redis_enablerepo: epel
```

(Used only on RHEL/CentOS) The repository to use for Redis installation.

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
redis_enabled: true
```

If unset, Redis will not start at boot.

### Modern defaults profile

The role now ships with a more modern baseline for cache/session workloads:

- `protected-mode yes`
- Unix socket enabled by default (`/run/redis/redis.sock`)
- safer command renaming defaults (`FLUSHALL`, `FLUSHDB`, `CONFIG`, `SHUTDOWN`)
- `maxmemory 1gb` with `volatile-lru`
- AOF enabled (`appendonly yes`, `appendfsync everysec`)

```yaml
redis_port: 6379
redis_bind_interface: 127.0.0.1
redis_protected_mode: "yes"
```

Port and interface on which Redis will listen. Set the interface to `0.0.0.0` to listen on all interfaces.

```yaml
redis_unixsocket: /run/redis/redis.sock
redis_unixsocketperm: "770"
```

If set, Redis will also listen on a local Unix socket.

```yaml
redis_timeout: 0
redis_tcp_backlog: 511
redis_tcp_keepalive: 300
redis_hz: 10
redis_dynamic_hz: "yes"
```

Close a connection after a client is idle `N` seconds. Set to `0` to disable timeout.

```yaml
redis_loglevel: "notice"
redis_logfile: /var/log/redis/redis-server.log
```

Log level and log location (valid levels are `debug`, `verbose`, `notice`, and `warning`).

```yaml
redis_databases: 16
```

The number of Redis databases.

```yaml
# Set to an empty list to disable RDB persistence.
redis_save:
  - '""'
```

Snapshotting configuration; setting values in this list will save the database to disk if the given number of seconds (e.g. `900`) and the given number of write operations (e.g. `1`) have occurred.

```yaml
redis_rdbcompression: "yes"
redis_dbfilename: dump.rdb
redis_dbdir: /var/lib/redis
```

Database compression and location configuration.

```yaml
redis_maxmemory: 1gb
```

Limit memory usage to the specified amount of bytes. Leave at 0 for unlimited.

```yaml
redis_maxmemory_policy: "volatile-lru"
```

The method to use to keep memory usage below the limit, if specified. See [Using Redis as an LRU cache](http://redis.io/topics/lru-cache).

```yaml
redis_maxmemory_samples: 5
```

Number of samples to use to approximate LRU. See [Using Redis as an LRU cache](http://redis.io/topics/lru-cache).

```yaml
redis_appendonly: "yes"
```

The appendonly option, if enabled, affords better data durability guarantees, at the cost of slightly slower performance.

```yaml
redis_appendfsync: "everysec"
redis_auto_aof_rewrite_percentage: 100
redis_auto_aof_rewrite_min_size: 256mb
```

Valid values are `always` (slower, safest), `everysec` (happy medium), or `no` (let the filesystem flush data when it wants, most risky).

```yaml
# Add extra include files for local configuration/overrides.
redis_includes: []
```

Add extra include file paths to this list to include more/localized Redis configuration.

The redis package name for installation via the system package manager. Defaults to `redis-server` on Debian and `redis` on RHEL.

```yaml
redis_package: "redis-server"
```

(Default for RHEL shown) The redis package name for installation via the system package manager. Defaults to `redis-server` on Debian and `redis` on RHEL.

```yaml
redis_requirepass: ""
redis_acl_users: []
```

Set a password to require authentication to Redis. You can generate a strong password using `echo "my_password_here" | sha256sum`.
If `redis_acl_users` is set, ACL directives are rendered and `requirepass` becomes a fallback.

```yaml
redis_disabled_commands:
  - FLUSHALL
  - FLUSHDB
  - CONFIG
  - SHUTDOWN
```

For extra security, you can disable certain Redis commands (this is especially important if Redis is publicly accessible). For example:

```yaml
redis_disabled_commands:
  - FLUSHDB
  - FLUSHALL
  - KEYS
  - PEXPIRE
  - DEL
  - CONFIG
  - SHUTDOWN
```

```yaml
redis_extra_config: |-
  # Extra redis configuration lines can be added here.
```

Extra Redis configuration lines that will be appended to the end of the `redis.conf` file.

### System tuning

The role can also manage host tuning for Redis:

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

- Sysctl settings are persisted in `/etc/sysctl.d/99-redis.conf` and applied with `sysctl --system`.
- THP disable is persisted via a dedicated systemd oneshot service.
- Redis unit limits are managed through `/etc/systemd/system/<redis-service>.service.d/override.conf`.

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  roles:
    - role: geerlingguy.redis
```

## Fork workflow

For synchronization with upstream, Cytadel branch strategy, and tagging policy, see:

- `README-synchro-upstream.md`

## License

MIT / BSD

## Author Information

This role was created in 2014 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
