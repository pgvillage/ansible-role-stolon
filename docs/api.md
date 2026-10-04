# pgvillage.stolon API

This document describes all variables of the `pgvillage.stolon` role, as defined in
[defaults/main.yml](../defaults/main.yml). All variables can be overridden from inventory,
playbook or extra vars.

## Contents

- [Installation](#installation)
- [PostgreSQL](#postgresql)
- [Cluster(s)](#clusters)
- [Autoconfig](#autoconfig)
- [Stolon cluster spec (custom config)](#stolon-cluster-spec-custom-config)
- [Stolon component configuration (sysconfig)](#stolon-component-configuration-sysconfig)
- [Logging](#logging)
- [Certificates](#certificates)
- [Miscellaneous](#miscellaneous)
- [Examples](#examples)

## Installation

| Variable | Default | Description |
|----------|---------|-------------|
| `stolon_release` | `v0.17.0b` | Stolon release, used to build `stolon_release_path`. |
| `stolon_opt_path` | `/opt/stolon` | Base folder for stolon releases. Created by the role (owned by root). |
| `stolon_release_path` | `{{ stolon_opt_path }}/{{ stolon_release }}` | Release specific folder under `stolon_opt_path`. Created by the role (owned by root). |
| `stolon_script_path` | `<home of stolon_user>/bin` | Folder where helper scripts (`demote.sh`, `movewal.sh`, `reinstate.sh`, `stolon_profile`) are deployed. |
| `stolon_binary_owner` | `root` | User used (`become_user`) to create the stolon installation folders. |
| `stolon_package_name_state` | `present` | State of the stolon packages (and local packages), e.g. `present` or `latest`. |
| `stolon_package_names` | selected from `_stolon_package_names` | List of packages to install from the configured repositories. By default selected from `_stolon_package_names` based on `ansible_facts.pkg_mgr`. |
| `_stolon_package_names` | see below | Packages to install per package manager. The `default` key is used when the package manager is not listed. |
| `stolon_local_package_names` | `[]` | List of local package files (from the role / playbook files path) that are copied to `/tmp` on the target and installed from there. |
| `stolon_sysctl` | `vm.swappiness = 10` | Contents of `/etc/sysctl.d/postgresql.conf`. |
| `stolon_services` | `[keeper, proxy, sentinel]` | Stolon components for which systemd services are deployed and started. Each item `<x>` maps to template `stolon-<x>.service.j2` (or `stolon-<x>@.service.j2` in multicluster mode). |

Default `_stolon_package_names`:

| Package manager | Packages |
|-----------------|----------|
| `default` | `stolon`, `python3-psycopg3` |
| `dnf` | `stolon`, `python3-psycopg3` |
| `apt` | `stolon`, `python3-psycopg2` |
| `zypper` | `stolon`, `python313-psycopg2` |

## PostgreSQL

| Variable | Default | Description |
|----------|---------|-------------|
| `stolon_pg_version` | `17` | Major PostgreSQL version. Used to derive paths, and to determine `pg_service.conf` features. |
| `stolon_pg_listen_address` | `{{ ansible_facts.default_ipv4.address }}` | Address the keeper configures PostgreSQL to listen on (`STKEEPER_PG_LISTEN_ADDRESS`). |
| `stolon_pg_port` | `5432` | Port PostgreSQL listens on (single cluster mode). |
| `stolon_user` | `postgres` | OS user that runs stolon / PostgreSQL and owns its files. |
| `stolon_group` | `{{ stolon_user }}` | OS group that owns the stolon / PostgreSQL files. |
| `stolon_pg_bin_path` | `/usr/pgsql-{{ stolon_pg_version }}/bin/` | Folder containing the PostgreSQL binaries (`STKEEPER_PG_BIN_PATH`, `pg_isready`). |
| `stolon_pg_datadir` | `{{ stolon_data_dir }}/postgres` | PostgreSQL data directory (`STKEEPER_PGDATA_DIR`). |
| `stolon_pg_certdir` | `{{ stolon_data_dir }}/certs` | Folder for the PostgreSQL server certificates (key, cert and client CA chain). |
| `stolon_pg_repl_username` | `postgres` | User the keepers use for replication (`STKEEPER_PG_REPL_USERNAME`). |
| `stolon_pg_repl_auth_method` | `cert` | Authentication method for replication connections (`STKEEPER_PG_REPL_AUTH_METHOD`), e.g. `cert` or `md5`. |
| `stolon_pg_repl_connection_type` | `hostssl` | pg_hba connection type for replication connections (`STKEEPER_PG_REPL_CONNECTION_TYPE`), e.g. `host` or `hostssl`. |
| `stolon_pg_su_auth_method` | `cert` | Authentication method for superuser connections (`STKEEPER_PG_SU_AUTH_METHOD`), e.g. `cert` or `md5`. |
| `stolon_pg_su_connection_type` | `hostssl` | pg_hba connection type for superuser connections (`STKEEPER_PG_SU_CONNECTION_TYPE`), e.g. `host` or `hostssl`. |

## Cluster(s)

| Variable | Default | Description |
|----------|---------|-------------|
| `stolon_cluster_name` | `stolon-cluster` | Name of the stolon cluster (single cluster mode). |
| `stolon_multicluster` | `false` | When `true`, systemd template units (`stolon-<x>@<cluster>.service`) and per-cluster sysconfig / custom_config / logrotate files are used, so multiple clusters can run on one host. |
| `stolon_clusters` | one cluster named `stolon_cluster_name` | Dictionary of clusters to deploy. See below. |
| `stolon_host_group` | `stolon` | Inventory group containing all hosts of the stolon cluster. |
| `stolon_cluster_hosts` | fqdn's of all hosts in `stolon_host_group` | Comma separated list of hosts. Used in `pg_service.conf` and for connecting to the primary when creating users. |
| `stolon_store_backend` | `etcdv3` | Stolon store backend (e.g. `etcdv3`, `etcdv2`, `consul`, `kubernetes`). |
| `stolon_proxy_host` | `{{ ansible_facts.fqdn }}` | Hostname used to connect to the stolon proxy (`pg_service.conf` `<cluster>-proxy` entry). |
| `stolon_proxy_port` | `25432` | Port the stolon proxy listens on (single cluster mode). |
| `stolon_data_dir` | `/var/lib/pgsql/{{ stolon_pg_version }}/data` | Stolon data directory (`STKEEPER_DATA_DIR`). |
| `stolon_config_dir` | `/var/lib/pgsql/{{ stolon_pg_version }}/config` | Folder for the stolon custom config (cluster spec) files. |
| `stolon_custom_config_file` | `{{ stolon_config_dir }}/stolon_custom_config.yml` | Path of the stolon custom config (cluster spec) file. |
| `stolon_custom_config_dir` | `{{ stolon_custom_config_file \| dirname }}` | Folder of `stolon_custom_config_file`. Used by the handler that applies the cluster spec. |
| `stolon_wal_dir` | `{{ stolon_pg_datadir }}/pg_wal` | PostgreSQL WAL directory (`STKEEPER_WAL_DIR` / `STKEEPER_PGWAL_DIR`). The free space on this filesystem is used to calculate `min_wal_size` and `max_wal_size`. |
| `stolon_proxy_listen_address` | `{{ ansible_facts.default_ipv4.address }}` | Address the stolon proxy listens on (`STPROXY_LISTEN_ADDRESS`). |
| `stolon_uid` | `inventory_hostname_short`, `-` replaced by `_`, lowercased | Unique id of the keeper (`STKEEPER_UID`). Must be unique within the cluster. |

### `stolon_clusters`

The key of every item is the cluster name. Every cluster supports:

| Key | Required | Description |
|-----|----------|-------------|
| `sysconfig` | yes | Settings per component (`stsentinel`, `stkeeper`, `stproxy`), merged over `stolon_default_sysconfig`. `stkeeper.pg_port` and `stproxy.port` are required. |
| `custom_config` | yes | Stolon cluster spec, merged over `stolon_default_custom_config`. |
| `cgroups` | no | systemd `[Service]` settings for the keeper, merged over `stolon_default_cgroups`. |
| `pgusers` | no | List of superusers to create, overrides `stolon_pgusers`. |

Default:

```yaml
stolon_clusters:
  "{{ stolon_cluster_name }}":
    sysconfig: "{{ stolon_sysconfig }}"
    custom_config:
      pgParameters:
        log_directory: "{{ stolon_pg_log_directory }}/{{ stolon_cluster_name }}"
```

## Autoconfig

The role derives PostgreSQL resource parameters from the resources of the host
(memory, vCPUs and free space on the WAL filesystem). Available memory is
`ansible_facts.memtotal_mb - stolon_reserved_memory_mb` (`stolon_available_memory_mb` in `vars/main.yml`).

### Tunables

| Variable | Default | Description |
|----------|---------|-------------|
| `stolon_query_complexity` | `4` | Used for automatically setting `work_mem`. Set to 1 for transactional, and 4-16 for analytical workloads. Consider the average amount of operations (merge, sort, group, etc.) per query. |
| `stolon_reserved_memory_mb` | `0` | Memory (in MB) reserved for other processes, which is not to be used by Postgres in calculations. |
| `stolon_max_connections` | `100` | PostgreSQL `max_connections`. Also used to calculate `work_mem`. |
| `stolon_shared_buffers_ratio` | `0.25` | `shared_buffers` as fraction of available memory. |
| `stolon_eff_cache_ratio` | `0.75` | `effective_cache_size` as fraction of available memory. |
| `stolon_min_wal_size_ratio` | `0.25` | `min_wal_size` as fraction of free space on the WAL filesystem. |
| `stolon_max_wal_size_ratio` | `0.75` | `max_wal_size` as fraction of free space on the WAL filesystem. |
| `stolon_workers_ratio` | `4` | `max_worker_processes` per vCPU. |
| `stolon_parallel_ratio` | `4` | `max_parallel_workers` per vCPU. |
| `stolon_gather_ratio` | `1` | `max_parallel_workers_per_gather` per vCPU. |
| `stolon_autoconfig` | `{{ stolon_autoconfig_default }}` | Autoconfig profile: dictionary of PostgreSQL parameters derived from the calculations. Set this to another profile, or define your own. |

### Calculations

Normally there is no need to change these; change the ratios instead.

| Variable | Calculation | Minimum |
|----------|-------------|---------|
| `stolon_share_buffers_mb` | available memory × `stolon_shared_buffers_ratio` (`shared_buffers`, MB) | 8 |
| `stolon_effective_cache_size_mb` | available memory × `stolon_eff_cache_ratio` (`effective_cache_size`, MB) | 1 |
| `stolon_wal_dir_size` | free space (bytes) on the filesystem of `stolon_wal_dir`, as detected by the role | |
| `stolon_min_wal_size_mb` | `stolon_wal_dir_size` × `stolon_min_wal_size_ratio` (`min_wal_size`, MB) | 80 |
| `stolon_max_wal_size_mb` | `stolon_wal_dir_size` × `stolon_max_wal_size_ratio` (`max_wal_size`, MB) | 1024 |
| `stolon_max_worker_processes` | vCPUs × `stolon_workers_ratio` | 8 |
| `stolon_max_parallel_workers` | vCPUs × `stolon_parallel_ratio` | 8 |
| `stolon_max_parallel_workers_per_gather` | vCPUs × `stolon_gather_ratio` | 2 |
| `stolon_private_mem_bytes` | available memory − `shared_buffers` (in MB despite the name) | |
| `stolon_work_mem_bytes` | private memory / `stolon_max_connections` / `stolon_query_complexity` (in MB despite the name) | |
| `stolon_work_mem_kb` | `work_mem` in kB | 64 |

## Stolon cluster spec (custom config)

More info:

- <https://github.com/sorintlab/stolon/blob/master/doc/cluster_spec.md>
- <https://github.com/sorintlab/stolon/blob/master/doc/postgres_parameters.md>
- <https://github.com/sorintlab/stolon/blob/master/doc/custom_pg_hba_entries.md>

| Variable | Default | Description |
|----------|---------|-------------|
| `stolon_pg_parameters` | `stolon_autoconfig` + `stolon_extra_pg_parameters` + `stolon_ssl_pg_parameters` | All PostgreSQL parameters (later dictionaries override earlier ones). |
| `stolon_extra_pg_parameters` | logging settings, see below | Additional PostgreSQL parameters. Overrides values from `stolon_autoconfig`. |
| `stolon_pg_hba` | `[]` | List of custom `pg_hba.conf` entries (cluster spec `pgHBA`). |
| `stolon_default_custom_config` | see below | Default stolon cluster spec. The `custom_config` of every cluster in `stolon_clusters` is merged over this. |

Default `stolon_extra_pg_parameters`:

```yaml
stolon_extra_pg_parameters:
  log_destination: "csvlog"
  logging_collector: "true"
  log_directory: "{{ stolon_pg_log_directory }}"
  log_file_mode: "0600"
  log_filename: "postgresql-%Y%m%d.log"
  log_rotation_size: "1GB"
  log_line_prefix: "%m [%p]: [%l-1] db=%d,user=%u,app=%a,client=%h"
  log_error_verbosity: "verbose"
  log_statement: "ddl"
  log_min_error_statement: "error"
  log_min_messages: "warning"
  log_min_duration_statement: "5000"
  log_connections: "true"
  log_disconnections: "true"
  log_truncate_on_rotation: "true"
```

Default `stolon_default_custom_config`:

```yaml
stolon_default_custom_config:
  defaultSUReplAccessMode: "strict"
  pgParameters: "{{ stolon_pg_parameters }}"
  pgHBA: "{{ stolon_pg_hba }}"
```

## Stolon component configuration (sysconfig)

Component settings are written to `/etc/sysconfig/stolon-<component>` (with `-<cluster>`
appended in multicluster mode). Every key is written as `<COMPONENT>_<KEY>`
(uppercase, `-` replaced by `_`), e.g. `stkeeper.pg_port` becomes `STKEEPER_PG_PORT`.

| Variable | Description |
|----------|-------------|
| `stolon_sysconfig` | Cluster specific component settings (single cluster mode). Used as `sysconfig` in the default `stolon_clusters`. |
| `stolon_default_sysconfig` | Component settings shared by all clusters. The `sysconfig` of every cluster in `stolon_clusters` is merged over this. |

Defaults:

```yaml
stolon_sysconfig:
  stsentinel:
    cluster_name: "{{ stolon_cluster_name }}"
  stkeeper:
    cluster_name: "{{ stolon_cluster_name }}"
    pg_port: "{{ stolon_pg_port }}"
    data_dir: "{{ stolon_data_dir }}"
    pgdata_dir: "{{ stolon_pg_datadir }}"
    wal_dir: "{{ stolon_wal_dir }}"
    pgwal_dir: "{{ stolon_wal_dir }}"
  stproxy:
    cluster_name: "{{ stolon_cluster_name }}"
    port: "{{ stolon_proxy_port }}"

stolon_default_sysconfig:
  stsentinel:
    store_backend: "{{ stolon_store_backend }}"
  stkeeper:
    store_backend: "{{ stolon_store_backend }}"
    pg_listen_address: "{{ stolon_pg_listen_address }}"
    pg_bin_path: "{{ stolon_pg_bin_path }}"
    uid: "{{ stolon_uid }}"
    pg_repl_username: "{{ stolon_pg_repl_username }}"
    pg_repl_auth_method: "{{ stolon_pg_repl_auth_method }}"
    pg_repl_connection_type: "{{ stolon_pg_repl_connection_type }}"
    pg_su_auth_method: "{{ stolon_pg_su_auth_method }}"
    pg_su_connection_type: "{{ stolon_pg_su_connection_type }}"
  stproxy:
    store_backend: "{{ stolon_store_backend }}"
    listen-address: "{{ stolon_proxy_listen_address }}"
```

## Logging

| Variable | Default | Description |
|----------|---------|-------------|
| `stolon_pg_log_directory` | `{{ stolon_pg_datadir }}/log` | PostgreSQL log directory. Created by the role when it is outside `stolon_pg_datadir`. |
| `stolon_pg_log_dir_mode` | `0755` | Mode of `stolon_pg_log_directory` (when created by the role). |
| `stolon_logrotate_config` | daily, rotate 8, compressed | Contents of the logrotate config, deployed as `/etc/logrotate.d/postgresql-<cluster>`. |

## Certificates

| Variable | Default | Description |
|----------|---------|-------------|
| `stolon_cert_managed` | `true` | When `true`, the role validates and deploys the client and server certificates (`stolon_cert_files`). |
| `stolon_cert_folders` | see below | Folders to create for certificates (mode `0700`). `client` is used by libpq, `server` by PostgreSQL. |
| `stolon_client_cert` | placeholder | PEM contents of the client certificate. |
| `stolon_client_chain` | placeholder | PEM contents of the client CA chain (installed with the server, so PostgreSQL can verify client certs). |
| `stolon_client_key` | placeholder | PEM contents of the client key. |
| `stolon_server_cert` | placeholder | PEM contents of the server certificate. |
| `stolon_server_key` | placeholder | PEM contents of the server key. |
| `stolon_server_chain` | placeholder | PEM contents of the server CA chain (installed with the client, so libpq can verify the server's cert). |
| `stolon_cert_files` | see below | Certificate files to deploy (mode `0600`). Every item has a `path`, `body` (contents) and `owner`. |
| `stolon_ssl_pg_parameters` | see below | PostgreSQL ssl parameters, pointing to the deployed certificate files. |

The certificate placeholders must be replaced (e.g. by chainsmith)
when `stolon_cert_managed` is `true`; the role fails otherwise.

Default folders:

| Key | Path | Owner |
|-----|------|-------|
| `client` | `<home of stolon_user>/.postgresql` | `stolon_user` |
| `server` | `stolon_pg_certdir` | `stolon_user` |

Default files:

| Key | Path | Contents |
|-----|------|----------|
| `client_cert` | `<client folder>/postgresql.crt` | `stolon_client_cert` |
| `client_key` | `<client folder>/postgresql.key` | `stolon_client_key` |
| `client_chain` | `<server folder>/root.crt` | `stolon_client_chain` |
| `server_cert` | `<server folder>/server.crt` | `stolon_server_cert` |
| `server_key` | `<server folder>/server.key` | `stolon_server_key` |
| `server_chain` | `<client folder>/root.crt` | `stolon_server_chain` |

Default `stolon_ssl_pg_parameters`:

```yaml
stolon_ssl_pg_parameters:
  ssl_cert_file: "{{ stolon_cert_files.server_cert.path }}"
  ssl_key_file: "{{ stolon_cert_files.server_key.path }}"
  ssl_ca_file: "{{ stolon_cert_files.client_chain.path }}"
  ssl: "true"
```

## Miscellaneous

| Variable | Default | Description |
|----------|---------|-------------|
| `stolon_keeper_extra_env_vars` | `{}` | Extra environment variables for the `stolon-keeper` systemd service (and thus PostgreSQL). |
| `stolon_pgusers` | `[pgroute66, nrpe, pgquartz]` | PostgreSQL users to create (as `SUPERUSER`) on the primary of every cluster. Can be overridden per cluster with `pgusers` in `stolon_clusters`. |
| `stolon_default_cgroups` | `{}` | systemd `[Service]` settings (cgroups) for the `stolon-keeper` service. Written to `stolon-keeper.service.d/cgroups.conf`. |

## Examples

Extra environment variables for the keeper:

```yaml
stolon_keeper_extra_env_vars:
  ORACLE_HOME: /usr/lib/oracle/21/client64
  LD_LIBRARY_PATH: /usr/lib/oracle/21/client64/lib
  TNS_ADMIN: /usr/lib/oracle/21/client64/network/admin
```

Limit resources of the keeper:

```yaml
stolon_default_cgroups:
  CPUQuota: 200%
  MemoryHigh: 8G
```

Multiple clusters on one host:

```yaml
stolon_multicluster: true
stolon_clusters:
  cluster-a:
    sysconfig:
      stsentinel:
        cluster_name: cluster-a
      stkeeper:
        cluster_name: cluster-a
        pg_port: 5432
        data_dir: /var/lib/pgsql/17/cluster-a/data
        pgdata_dir: /var/lib/pgsql/17/cluster-a/data/postgres
        wal_dir: /var/lib/pgsql/17/cluster-a/data/postgres/pg_wal
        pgwal_dir: /var/lib/pgsql/17/cluster-a/data/postgres/pg_wal
      stproxy:
        cluster_name: cluster-a
        port: 25432
    custom_config:
      pgParameters:
        log_directory: /var/log/postgresql/cluster-a
  cluster-b:
    sysconfig:
      stsentinel:
        cluster_name: cluster-b
      stkeeper:
        cluster_name: cluster-b
        pg_port: 5433
        data_dir: /var/lib/pgsql/17/cluster-b/data
        pgdata_dir: /var/lib/pgsql/17/cluster-b/data/postgres
        wal_dir: /var/lib/pgsql/17/cluster-b/data/postgres/pg_wal
        pgwal_dir: /var/lib/pgsql/17/cluster-b/data/postgres/pg_wal
      stproxy:
        cluster_name: cluster-b
        port: 25433
    custom_config:
      pgParameters:
        log_directory: /var/log/postgresql/cluster-b
    pgusers:
      - nrpe
```
