# Ansible Role: PostgreSQL Single Server

Installs and configures PostgreSQL server on Ubuntu/Debian.

## Requirements

- **Ansible** >= 2.19
- **OS**: Ubuntu 24.04 (noble), Debian 12 (bookworm), Debian 13 (trixie)
- **PostgreSQL**: 15, 16, 17
- **Ansible Collections**:
  - `community.postgresql`
  - `community.general`

This role requires root access — use `become: true`.

## Supported Platforms

| OS | Default PG Version |
|----|:------------------:|
| Ubuntu 24.04 (noble) | 16 |
| Debian 12 (bookworm) | 15 |
| Debian 13 (trixie) | 17 |

You can override the PostgreSQL version via the `postgresql_version` variable.

## Installation

```bash
ansible-galaxy install okassov.postgresql
```

Or in `requirements.yml`:

```yaml
roles:
  - name: okassov.postgresql

collections:
  - name: community.postgresql
  - name: community.general
```

## Role Variables

### Version and Service

```yaml
# Override PostgreSQL version (defaults depend on OS)
postgresql_version: "16"

# Service state on configuration changes: restarted or reloaded
postgresql_restarted_state: "restarted"

# Service management
postgresql_service_state: started
postgresql_service_enabled: true
```

### System Settings

```yaml
# PostgreSQL system user and group
postgresql_user: postgres
postgresql_group: postgres

# Unix socket directories
postgresql_unix_socket_directories:
  - /var/run/postgresql

# Authentication method: md5 or scram-sha-256
postgresql_auth_method: "scram-sha-256"

# Locales to generate
postgresql_locales:
  - 'en_US.UTF-8'
```

### postgresql.conf Configuration

```yaml
postgresql_global_config_options:
  - option: unix_socket_directories
    value: '{{ postgresql_unix_socket_directories | join(",") }}'
  - option: log_directory
    value: 'log'
  # Add your own parameters:
  - option: max_connections
    value: '200'
  - option: shared_buffers
    value: '512MB'
  - option: work_mem
    value: '8MB'
```

### pg_hba.conf Configuration

```yaml
postgresql_hba_entries:
  - { type: local, database: all, user: postgres, auth_method: peer }
  - { type: local, database: all, user: all, auth_method: peer }
  - { type: host, database: all, user: all, address: '127.0.0.1/32', auth_method: "{{ postgresql_auth_method }}" }
  - { type: host, database: all, user: all, address: '::1/128', auth_method: "{{ postgresql_auth_method }}" }
```

Each entry supports the following fields:

| Field | Required | Description |
|-------|:--------:|-------------|
| `type` | yes | `local`, `host`, `hostssl`, `hostnossl` |
| `database` | yes | Database name or `all` |
| `user` | yes | Username or `all` |
| `address` | no | CIDR address (for `host` type) |
| `ip_address` | no | IP address (alternative to `address`) |
| `ip_mask` | no | Network mask (used with `ip_address`) |
| `auth_method` | yes | `peer`, `md5`, `scram-sha-256`, `trust`, `reject` |
| `auth_options` | no | Additional auth options |

When overriding, make sure to include all default entries from `defaults/main.yml` if you need to preserve them.

### Extensions

```yaml
# Apt packages required by extensions (installed before extension activation)
postgresql_extensions_packages: []
# - postgresql-16-postgis-3
# - postgresql-16-pgvector

# Extensions to activate globally across one or more databases
postgresql_extensions: []
# - name: pg_trgm      # required — extension name
#   db: mydb           # required — target database
#   schema: public     # optional — schema to install into
#   state: present     # optional — present (default) or absent
#   login_user: ...    # optional
#   login_unix_socket: ... # optional
```

---

## Managing Databases

### Creating Databases

```yaml
postgresql_databases:
  - name: myapp_production
  - name: myapp_staging
    owner: staging_user
    encoding: UTF-8
    lc_collate: en_US.UTF-8
    lc_ctype: en_US.UTF-8
```

Full parameter reference:

| Parameter | Default | Description |
|-----------|:-------:|-------------|
| `name` | (required) | Database name |
| `owner` | `postgres` | Database owner |
| `encoding` | `UTF-8` | Character encoding |
| `lc_collate` | `en_US.UTF-8` | Collation order |
| `lc_ctype` | `en_US.UTF-8` | Character classification |
| `template` | `template0` | Template database |
| `state` | `present` | `present` or `absent` |
| `login_host` | `localhost` | Host to connect to |
| `login_user` | `postgres` | User to connect as |
| `login_password` | — | Connection password |
| `login_unix_socket` | auto | Path to unix socket |
| `port` | — | PostgreSQL port |

### Dropping Databases

```yaml
postgresql_databases:
  - name: old_database
    state: absent
```

---

## Managing Users

### Creating Users

```yaml
postgresql_users:
  - name: app_user
    password: "{{ vault_app_user_password }}"
  - name: readonly_user
    password: "{{ vault_readonly_password }}"
    role_attr_flags: NOSUPERUSER,NOCREATEDB
```

> **Passwords**: always store passwords in Ansible Vault. Never put them in plain text.

Full parameter reference:

| Parameter | Default | Description |
|-----------|:-------:|-------------|
| `name` | (required) | Username |
| `password` | — | Password (encrypted via `scram-sha-256`) |
| `encrypted` | — | Explicitly specify whether password is encrypted |
| `role_attr_flags` | — | Role attribute flags (see below) |
| `db` | — | Database to connect to during creation |
| `state` | `present` | `present` or `absent` |
| `login_host` | `localhost` | Host to connect to |
| `login_user` | `postgres` | User to connect as |
| `login_password` | — | Connection password |
| `login_unix_socket` | auto | Path to unix socket |
| `port` | — | PostgreSQL port |

### Role Attribute Flags

Flags are specified as a comma-separated list:

| Flag | Description |
|------|-------------|
| `SUPERUSER` / `NOSUPERUSER` | Superuser privileges |
| `CREATEDB` / `NOCREATEDB` | Ability to create databases |
| `CREATEROLE` / `NOCREATEROLE` | Ability to create roles |
| `LOGIN` / `NOLOGIN` | Ability to log in |
| `REPLICATION` / `NOREPLICATION` | Replication privileges |

Example — creating a user with restricted permissions:

```yaml
postgresql_users:
  - name: limited_user
    password: "{{ vault_limited_password }}"
    role_attr_flags: NOSUPERUSER,NOCREATEDB,NOCREATEROLE
```

### Hiding Password Output

```yaml
# Defaults to true — passwords are not shown in Ansible logs
postgres_users_no_log: true
```

### Removing Users

```yaml
postgresql_users:
  - name: old_user
    state: absent
```

---

## Managing Privileges

### Granting Privileges

```yaml
postgresql_privs:
  # Full access to a database
  - database: myapp_production
    roles: app_user
    type: database
    privs: ALL

  # Read-only access to all tables in public schema
  - database: myapp_production
    roles: readonly_user
    objs: ALL_IN_SCHEMA
    privs: SELECT

  # Access to specific tables
  - database: myapp_production
    roles: report_user
    type: table
    objs: orders,invoices
    privs: SELECT,INSERT

  # Schema-level access
  - database: myapp_production
    roles: app_user
    type: schema
    objs: public
    privs: USAGE,CREATE
```

Full parameter reference:

| Parameter | Default | Description |
|-----------|:-------:|-------------|
| `database` | (required) | Database name |
| `roles` | (required) | User(s), comma-separated |
| `privs` | — | Privileges: `ALL`, `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `USAGE`, `CREATE`, etc. |
| `type` | — | Object type: `database`, `table`, `schema`, `sequence`, `function` |
| `objs` | — | Objects: table names, `ALL_IN_SCHEMA`, etc. |
| `schema` | — | Schema (defaults to `public`) |
| `grant_option` | — | Allow grantee to grant privileges to others |
| `fail_on_role` | `true` | Fail if role does not exist |
| `state` | `present` | `present` or `absent` |
| `target_roles` | — | Target roles for default privileges |
| `session_role` | — | Role to execute the operation as |

### Revoking Privileges

```yaml
postgresql_privs:
  - database: myapp_production
    roles: old_user
    type: database
    privs: ALL
    state: absent
```

---

## Examples

### Minimal

```yaml
- hosts: db
  become: true
  roles:
    - role: okassov.postgresql
```

Installs PostgreSQL with the OS-default version. No additional databases or users.

### Specifying PostgreSQL Version

```yaml
- hosts: db
  become: true
  roles:
    - role: okassov.postgresql
      postgresql_version: "15"
```

### Full Example: Application with Multiple Environments

```yaml
- hosts: db
  become: true
  vars:
    postgresql_version: "16"

    postgresql_global_config_options:
      - option: unix_socket_directories
        value: '/var/run/postgresql'
      - option: log_directory
        value: 'log'
      - option: max_connections
        value: '200'
      - option: shared_buffers
        value: '1GB'
      - option: effective_cache_size
        value: '3GB'
      - option: work_mem
        value: '16MB'
      - option: maintenance_work_mem
        value: '256MB'

    postgresql_hba_entries:
      - { type: local, database: all, user: postgres, auth_method: peer }
      - { type: local, database: all, user: all, auth_method: peer }
      - { type: host, database: all, user: all, address: '127.0.0.1/32', auth_method: scram-sha-256 }
      - { type: host, database: all, user: all, address: '::1/128', auth_method: scram-sha-256 }
      - { type: host, database: myapp_db, user: app_user, address: '10.0.0.0/8', auth_method: scram-sha-256 }

    postgresql_databases:
      - name: myapp_db
        owner: app_user
      - name: myapp_db_test
        owner: app_user

    postgresql_users:
      - name: app_user
        password: "{{ vault_app_password }}"
      - name: readonly_user
        password: "{{ vault_readonly_password }}"
        role_attr_flags: NOSUPERUSER,NOCREATEDB,NOCREATEROLE
      - name: replication_user
        password: "{{ vault_replication_password }}"
        role_attr_flags: REPLICATION,LOGIN

    postgresql_privs:
      - database: myapp_db
        roles: app_user
        type: database
        privs: ALL
      - database: myapp_db
        roles: readonly_user
        objs: ALL_IN_SCHEMA
        privs: SELECT
      - database: myapp_db_test
        roles: app_user
        type: database
        privs: ALL

  roles:
    - role: okassov.postgresql
```

### Read-Only Access for Monitoring

```yaml
postgresql_users:
  - name: monitoring
    password: "{{ vault_monitoring_password }}"
    role_attr_flags: NOSUPERUSER,NOCREATEDB,NOCREATEROLE

postgresql_privs:
  - database: myapp_db
    roles: monitoring
    objs: ALL_IN_SCHEMA
    privs: SELECT
  - database: myapp_db
    roles: monitoring
    type: schema
    objs: public
    privs: USAGE
```

---

## Managing Extensions

### Installing Extension Packages

Some extensions require additional apt packages (e.g. PostGIS, pgvector). List them in `postgresql_extensions_packages`:

```yaml
postgresql_extensions_packages:
  - postgresql-16-postgis-3
  - postgresql-16-pgvector
```

These packages are installed before any extension activation.

### Activating Extensions Globally

Use `postgresql_extensions` to activate extensions across one or more databases:

```yaml
postgresql_extensions:
  - name: pg_trgm
    db: myapp_db
  - name: postgis
    db: myapp_db
    schema: public
  - name: pgcrypto
    db: myapp_db
    state: present
```

Full parameter reference:

| Parameter | Default | Description |
|-----------|:-------:|-------------|
| `name` | (required) | Extension name (as known to PostgreSQL) |
| `db` | (required) | Target database |
| `schema` | — | Schema to install the extension into |
| `state` | `present` | `present` or `absent` |
| `login_user` | `postgres` | User to connect as |
| `login_unix_socket` | auto | Path to unix socket |

### Activating Extensions Per Database (shorthand)

For convenience, you can specify extensions directly on a database entry using the `extensions` key. Each item is a plain extension name string:

```yaml
postgresql_databases:
  - name: myapp_db
    owner: app_user
    extensions:
      - pg_trgm
      - pgcrypto
      - postgis
```

This is equivalent to listing the same extensions in `postgresql_extensions` with `db: myapp_db`. Both lists are merged at runtime, so you can mix and match the two styles.

### Removing Extensions

Set `state: absent` to drop an extension from a database:

```yaml
postgresql_extensions:
  - name: pg_trgm
    db: myapp_db
    state: absent
```

---

## OS-Specific Variables

These variables are automatically set based on the OS. Override only for non-standard installations:

```yaml
postgresql_version: "16"                                    # PG version
postgresql_data_dir: "/var/lib/postgresql/16/main"          # Data directory
postgresql_bin_path: "/usr/lib/postgresql/16/bin"            # Binary path
postgresql_config_path: "/etc/postgresql/16/main"           # Config path
postgresql_daemon: "postgresql@16-main"                     # systemd service name
postgresql_packages:                                        # Packages to install
  - postgresql
  - postgresql-contrib
  - libpq-dev
```

## Testing

This role is tested via [Molecule](https://molecule.readthedocs.io/) with the Docker driver on two platforms: Ubuntu 24.04 and Debian 12.

```bash
molecule test
```

## Dependencies

None.

## License

MIT / BSD

## Author Information

Maintained by [Okassov Marat](https://github.com/okassov).
