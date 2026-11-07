# File Schema

This document describes the expected files and data shapes for the private
`grayhaven-vault` repository. Setup instructions are documented in
[Setup](setup.md), and operational procedures are documented in
[Operations](operations.md).

## Table of Contents

- [Required Files](#required-files)
- [`config.yml`](#configyml)
- [`server_configuration.yml`](#server_configurationyml)
- [`ansible.yml`](#ansibleyml)
- [`policy/infrastructure.yml`](#policyinfrastructureyml)
- [`policy/compute.yml`](#policycomputeyml)
- [`policy/firewall.yml`](#policyfirewallyml)
- [`policy/dns.yml`](#policydnsyml)
- [`policy/baseline/dns.yml`](#policybaselinednsyml)
- [`policy/baseline/ssh-keys.yml`](#policybaselinessh-keysyml)
- [`grafana/observability.yml`](#grafanaobservabilityyml)
- [`grafana/grafana-alert-rule-uids.yml`](#grafanagrafana-alert-rule-uidsyml)
- [`grafana/grafana-alert-rules.yml`](#grafanagrafana-alert-rulesyml)
- [`vault/common.yml`](#vaultcommonyml)
- [`vault/bastion.yml`](#vaultbastionyml)
- [`vault/web.yml`](#vaultwebyml)
- [`vault/mcp.yml`](#vaultmcpyml)

## Required Files

Each environment branch includes these files:

```text
config.yml
server_configuration.yml
ansible.yml
policy/infrastructure.yml
policy/compute.yml
policy/firewall.yml
policy/dns.yml
policy/baseline/dns.yml
policy/baseline/ssh-keys.yml
grafana/observability.yml
vault/common.yml
vault/bastion.yml
vault/web.yml
vault/mcp.yml
```

Production also includes `grafana/grafana-alert-rule-uids.yml` and
`grafana/grafana-alert-rules.yml`.

All files in this example repository remain plaintext. Keep the files under
`vault/` encrypted in the private repository.

[Back to top](#file-schema)

## `config.yml`

`config.yml` is plaintext and is read by both OpenTofu and Ansible.

```yaml
---
certificate:
  environment: production
  email: admin@example.com

backup:
  repositories:
    local:
      repository_path: /var/backups/restic
      homedir_archive_path: /var/backups/deleted-homedir-archives
    remote:
      provider: gcs
      project_id: example-project
      location: US
      storage_class: STANDARD
  schedule: "*-*-* 00:30:00"
  integrity_schedule: "Sun *-*-* 02:30:00"
  retention:
    keep_daily: 7
  include:
    - /home
    - /var/log
  exclude: []

timetracker:
  hostname: timetracker.example.com
  image_digest: "sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  # build/1.0.0
  data_dir: /var/lib/grayhaven/timetracker/data
  branding_dir: /var/lib/grayhaven/timetracker/branding
  secrets_dir: /var/lib/grayhaven/timetracker/secrets
  backup_dir: /var/lib/grayhaven/timetracker/backups
  backup_retention_days: 14
  pids_limit: 128

grayhaven_backupctl:
  repo_url: https://github.com/grayhavensystemsllc/grayhaven-backupctl.git
  checkout_dir: /home/ansible/grayhaven-backupctl
  version: "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  # v1.0.0

mcp:
  hostname: mcp.example.com
  image_digest: "sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  # build/1.0.0
  data_dir: /var/lib/grayhaven/mcp/data
  config_dir: /var/lib/grayhaven/mcp/config
  secrets_dir: /var/lib/grayhaven/mcp/secrets
  sessions_dir: /var/lib/grayhaven/mcp/sessions
  backup_dir: /var/lib/grayhaven/mcp/backups
  backup_retention_days: 14
  maintenance:
    cpu_quota_percent: 20
    cpu_weight: 20
  pids_limit: 256
```

Supported keys:

- `certificate.environment`: `staging` or `production`.
- `certificate.email`: email address supplied to Certbot for web certificate
  registration and renewal.
- `backup.repositories.local.repository_path`: local restic repository path.
- `backup.repositories.local.homedir_archive_path`: local path for removed user
  home directory archives.
- `backup.repositories.remote`: optional remote restic repository settings.
  Only Google Cloud Storage is supported at this time.
- `backup.repositories.remote.provider`: remote repository provider. The
  supported value is `gcs`.
- `backup.repositories.remote.project_id`: Google Cloud project ID.
- `backup.repositories.remote.location`: optional Google Cloud Storage bucket
  location. Defaults to `US`.
- `backup.repositories.remote.storage_class`: optional Google Cloud Storage
  bucket storage class. Defaults to `STANDARD`.
- `backup.schedule`: local backup timer schedule in systemd calendar syntax.
- `backup.integrity_schedule`: local integrity-check timer schedule in
  systemd calendar syntax.
- `backup.retention.keep_daily`: number of daily restic snapshots retained.
- `backup.include`: list of additional paths to include in backups.
  The configured homedir archive path is always included automatically.
- `backup.exclude`: list of additional paths to exclude from backups.
  Excludes can override automatically included paths, including the configured
  homedir archive path.
- `timetracker`: required Time Tracker deployment settings. The application is
  deployed to web hosts; there is no enable flag.
- Time Tracker branding is not a vault setting. Deployment automation updates
  the public `grayhaven-branding` repository's `main` branch on each
  convergence.
- `timetracker.hostname`: environment-specific public application hostname.
- `timetracker.image_digest`: immutable published image digest in
  `sha256:<64 lowercase hexadecimal characters>` form.
- `timetracker.data_dir`: persistent SQLCipher database directory.
- `timetracker.branding_dir`: deployed branding asset directory.
- `timetracker.secrets_dir`: host directory containing application secret
  files.
- `timetracker.backup_dir`: directory containing verified encrypted database
  backup artifacts captured by restic.
- `timetracker.backup_retention_days`: local verified-artifact retention
  period from 1 through 365 days.
- `timetracker.pids_limit`: container process limit.
- `grayhaven_backupctl`: backup operator utility settings described in
  [Backup Operator Utility](#backup-operator-utility).

- `mcp`: deployment settings required when an MCP host is provisioned.
- `mcp.hostname`: environment-specific public MCP hostname.
- `mcp.image_digest`: immutable published image digest in
  `sha256:<64 lowercase hexadecimal characters>` form, followed by the
  matching `# build/X.Y.Z` comment, separated by two spaces.
- `mcp.data_dir`: persistent SQLCipher database directory.
- `mcp.config_dir`: directory containing public and encrypted configuration.
- `mcp.secrets_dir`: directory containing the configuration decryption password.
- `mcp.sessions_dir`: persistent provider-session directory. Convergence preserves
  existing sessions.
- `mcp.backup_dir`: directory containing verified encrypted database snapshots
  included automatically in host backups.
- `mcp.backup_retention_days`: local verified-artifact retention
  period from 1 through 365 days.

- `mcp.maintenance.cpu_quota_percent`: CPU quota for MCP maintenance processes.
- `mcp.maintenance.cpu_weight`: relative CPU weight for MCP maintenance processes.
- `mcp.pids_limit`: container process limit.

MCP always uses a production certificate trusted by external clients, including
in staging. The global certificate setting applies to other servers.

### Remote Backup Repository

Remote restic repositories are optional. When enabled, Ansible creates one
Google Cloud Storage bucket per managed host before configuring restic. Bucket
names use the short hostname with `-restic` appended.

```yaml
---
backup:
  repositories:
    local:
      repository_path: /var/backups/restic
      homedir_archive_path: /var/backups/deleted-homedir-archives
    remote:
      provider: gcs
      project_id: example-project
      location: US
      storage_class: STANDARD
```

Remote backup buckets are created without object versioning. They are labeled
with operational metadata such as `managed_by=ansible`, `client=grayhaven`,
`env=<environment>`, `project=<project>`, `role=<role>`, `purpose=restic`, and
`host=<short-hostname>`.

When remote backups are enabled, the matching encrypted `vault/common.yml` file
must define the Google Cloud Storage credentials described in
[`vault/common.yml`](#vaultcommonyml).

### Backup Operator Utility

The `grayhaven_backupctl` mapping contains exactly `repo_url`, `checkout_dir`,
and `version`:

```yaml
---
grayhaven_backupctl:
  repo_url: https://github.com/grayhavensystemsllc/grayhaven-backupctl.git
  checkout_dir: /home/ansible/grayhaven-backupctl
  version: "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  # v1.0.0
```

- `grayhaven_backupctl.repo_url`: the reviewed grayhaven-backupctl Git
  repository.
- `grayhaven_backupctl.checkout_dir`: the managed checkout path, which must
  remain below `/home/ansible` so the managed `ansible` user owns it.
- `grayhaven_backupctl.version`: the full lowercase commit SHA pinned to the
  verified `v1.0.4` release.

[Back to top](#file-schema)

## `server_configuration.yml`

Plaintext server settings shared by infrastructure bootstrap and Ansible.

```yaml
---
timezone: UTC

network:
  vpc_cidr: 172.22.0.0/16
  dns_resolvers:
    - 192.0.2.53
    - 198.51.100.53

swap:
  size: 1G

system_packages:
  baseline:
    - audit
    - bash-completion
    - chrony
    - curl
    - epel-release
    - fail2ban
    - firewalld
    - git
    - httpd-tools
    - NetworkManager
    - openssh-server
    - podman
    - python3
    - python3-firewall
    - python3-libselinux
    - python3-pip
    - restic
    - rsync
    - yum-utils
  admin:
    - bind-utils
    - setroubleshoot-server
    - tmux
    - vim
  web:
    - certbot
    - nginx
    - python3-certbot-dns-digitalocean
    - python3-certbot-nginx
  uninstall:
    - avahi
    - bluez
    - cups
    - nfs-utils
    - rpcbind
    - samba

system_services:
  disabled:
    - avahi-daemon
    - bluetooth
    - cups
    - nfs-server
    - rpcbind
    - smb
```

Supported keys:

- `timezone`: IANA timezone used by managed servers and their notifications.
- `network.vpc_cidr`: environment VPC CIDR, used by OpenTofu and Ansible.
- `network.dns_resolvers`: list of DNS resolver addresses.
- `swap.size`: managed swapfile size.
- `system_packages.baseline`: packages installed on every managed server and
  used for bastion bootstrap.
- `system_packages.admin`: common administration packages.
- `system_packages.web`: web-hosting packages.
- `system_packages.uninstall`: packages to remove from managed servers.
- `system_services.disabled`: services to stop and disable.

Each setting has one source. Server environment is derived from infrastructure.
Installation paths, application service accounts, service/container names,
the swapfile path, and the standard NTP pool are fixed in the config repository.

[Back to top](#file-schema)

## `ansible.yml`

Plaintext schedules, resource policy, and notification settings for Ansible.

```yaml
---
ansible:
  runner:
    schedule: "*-*-* 01:30:00"
    cpu_quota_percent: 40
  poller:
    schedule: "*-*-* *:0/10:00"
  notifications:
    environment: production
    colors:
      info: 5811424
      success: 4175499
      warning: 13211450
      error: 14042437
```

Supported keys:

- `ansible.runner.schedule`: convergence timer schedule in systemd calendar syntax.
- `ansible.runner.cpu_quota_percent`: convergence runner CPU quota.
- `ansible.poller.schedule`: repository polling schedule in systemd calendar syntax.
- `ansible.notifications.environment`: `production` or `staging`; selects the
  matching `discord_webhooks` entry in `vault/bastion.yml`.
- `ansible.notifications.colors`: Discord embed colors for `info`, `success`,
  `warning`, and `error`.

Notification timestamps use the server timezone. The server environment
determines the environment label in notification messages.

[Back to top](#file-schema)

## `policy/infrastructure.yml`

Plaintext OpenTofu configuration. Do not store secrets in this file.

- `identity.client_name`: resource-name and tag prefix.
- `identity.client_domain`: host DNS suffix and infrastructure DNS zone.
- `identity.organization`: VPC descriptions and certificate organization.
- `project.name`: DigitalOcean project name.
- `project.description`, `project.purpose`, `project.environment`, and
  `project.is_default`: shared project properties.
- `defaults.region`: DigitalOcean region.
- `defaults.os_image`: default droplet image slug.
- `defaults.droplet_size`: default droplet size slug.
- `baseline.vpc_cidr`: shared baseline VPC CIDR.
- `repositories.config`: HTTPS configuration-repository URL used by bootstrap.
- `repositories.vault`: SSH vault-repository URL used by bootstrap.

```yaml
---
identity:
  client_name: example
  client_domain: example.com
  organization: Example Company

project:
  name: example
  description: Example infrastructure project
  purpose: Web Application
  environment: Production
  is_default: true

defaults:
  region: nyc1
  os_image: almalinux-10-x64
  droplet_size: s-1vcpu-2gb

baseline:
  vpc_cidr: 172.20.0.0/16

repositories:
  config: https://github.com/example-org/configuration.git
  vault: git@github.com:example-org/private-vault.git
```

[Back to top](#file-schema)

## `policy/compute.yml`

Plaintext environment compute policy.

- `bootstrap.control_public_key`: current Ansible control public key. Keep it
  equal to `ansible_control.public_key` in `vault/bastion.yml` so new hosts
  trust the control node before their first convergence.

- `bastions.control_node`: key of the active control bastion.
- `bastions.instances`: map of bastion instances; at least one is required.
- `web.instances`: map of web instances; an empty map omits web hosts.
- `web.tls_mode`: `auto` or `load_balancer`; defaults to `auto`.
- `mcp.instances`: optional map containing at most one MCP host.
- Each instance requires `index`; optional `size` and `image` override the
  defaults in `policy/infrastructure.yml`.

With `auto`, one web host uses host TLS and multiple web hosts use a load
balancer. MCP hosts use host TLS and require a firewall policy that permits
HTTPS and restricts SSH to bastions.

```yaml
---
bootstrap:
  control_public_key: "ssh-ed25519 AAAAexample operator@example.com"

bastions:
  control_node: bastion-02
  instances:
    bastion-02:
      index: 2
    bastion-05:
      index: 5

web:
  tls_mode: auto
  instances:
    web-03:
      index: 3
    web-04:
      index: 4

mcp:
  instances:
    mcp-07:
      index: 7
```

[Back to top](#file-schema)

## `policy/firewall.yml`

`policy/firewall.yml` is plaintext and is read by OpenTofu and Ansible.

```yaml
---
firewalls:
  bastion:
    inbound:
      - protocol: tcp
        port_range: "22"
        source_addresses:
          - 0.0.0.0/0
    outbound:
      - protocol: tcp
        port_range: "22"
        destination_tags:
          - bastion
          - web
      # Bastions clone private GitHub repositories with SSH deploy keys.
      - protocol: tcp
        port_range: "22"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "80"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "443"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: udp
        port_range: "53"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "53"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: udp
        port_range: "123"
        destination_addresses:
          - 0.0.0.0/0

  web:
    inbound:
      - protocol: tcp
        port_range: "22"
        source_tags:
          - bastion
      - protocol: tcp
        port_range: "80"
        source_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "443"
        source_addresses:
          - 0.0.0.0/0
    outbound:
      - protocol: tcp
        port_range: "80"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "443"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: udp
        port_range: "53"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "53"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: udp
        port_range: "123"
        destination_addresses:
          - 0.0.0.0/0

  mcp:
    inbound:
      - protocol: tcp
        port_range: "22"
        source_tags:
          - bastion
      - protocol: tcp
        port_range: "443"
        source_addresses:
          - 0.0.0.0/0
    outbound:
      - protocol: tcp
        port_range: "22"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "80"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "443"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: udp
        port_range: "53"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: tcp
        port_range: "53"
        destination_addresses:
          - 0.0.0.0/0
      - protocol: udp
        port_range: "123"
        destination_addresses:
          - 0.0.0.0/0
```

Supported top-level keys:

- `firewalls.bastion`: bastion firewall policy.
- `firewalls.web`: web firewall policy.

- `firewalls.mcp`: MCP firewall policy.

The `firewalls.mcp.inbound` policy permits public TCP `443` and TCP `22` from
bastion only. It does not permit HTTP or public SSH.

The `firewalls.web.inbound` policy must include TCP `80` and TCP `443` rules.
OpenTofu validates this shape before applying the TLS-mode-specific effective
cloud firewall.
When an environment has two or more web hosts behind load-balancer TLS,
OpenTofu and Ansible add TCP `8791` automatically for private web-to-web
deployment fanout.

It is strongly recommended that each role allow outbound HTTP, HTTPS, DNS, and
NTP traffic unless there is a deliberate reason to narrow egress further. This
normally means TCP `80`, TCP `443`, TCP/UDP `53`, and UDP `123`.

Supported rule keys:

- `protocol`: network protocol.
- `port_range`: DigitalOcean/firewalld port range.
- `source_addresses`: optional inbound source CIDR list.
- `source_tags`: optional inbound role alias list. Supported aliases are
  `bastion` and `web`.
- `destination_addresses`: optional outbound destination CIDR list.
- `destination_tags`: optional outbound role alias list. Supported aliases are
  `bastion` and `web`.

OpenTofu maps role aliases to the appropriate DigitalOcean cloud firewall tags.
Ansible firewalld policy currently interprets inbound `source_tags: bastion` to
allow SSH from bastions to managed hosts.

[Back to top](#file-schema)

## `policy/dns.yml`

Plaintext environment DNS policy. `ttl` sets the default record TTL. `domains`
maps stable domain keys to entries with `name`, optional `environment`,
`protected_records`, and `records`.

- `environment.apex: true` creates the environment web A record.
- `environment.web_aliases: true` creates `www` and `dev` aliases and requires
  `environment.apex: true`.
- `protected_records` and `records` support A, CNAME, and TXT records.
- Each record requires `type`, `name`, and `value`; `ttl` is optional.

```yaml
---
ttl: 600

domains:
  primary_site:
    name: example.com
    environment:
      apex: true
      web_aliases: true
    records:
      service_cname:
        type: CNAME
        name: service
        value: example.com.

  secondary_site:
    name: example.net
    environment:
      apex: true
      web_aliases: true
```

[Back to top](#file-schema)

## `policy/baseline/dns.yml`

Plaintext shared DNS zones and records. Use the `policy/dns.yml` record structure.
Baseline `protected_records` additionally support MX records with `priority`
and CAA records with `flags` and `tag`. Mail records belong in this file.
The baseline workspace reads this file from `main`.

```yaml
---
ttl: 600

domains:
  primary_site:
    name: example.com
    protected_records:
      mail_mx:
        type: MX
        name: "@"
        value: mail.example.org.
        priority: 10
      verification_txt:
        type: TXT
        name: "@"
        value: "verification=example-placeholder"
      issuer_caa:
        type: CAA
        name: "@"
        flags: 0
        tag: issue
        value: ca.example.org.

  secondary_site:
    name: example.net
    protected_records:
      mail_mx:
        type: MX
        name: "@"
        value: mail.example.org.
        priority: 20
```

[Back to top](#file-schema)

## `policy/baseline/ssh-keys.yml`

Plaintext `admin_ssh_keys` list of first-boot admin public keys. Baseline owns
the DigitalOcean key resources; environment workspaces resolve those keys by
name. Do not include private keys.

```yaml
---
admin_ssh_keys:
  - >-
    ssh-ed25519
    AAAAexamplePublicKeyMaterial
    admin@example.com
```

[Back to top](#file-schema)

## `grafana/observability.yml`

Plaintext monitoring configuration. Grafana Cloud is supported in production.

```yaml
---
observability:
  grafana_cloud:
    enabled: true
    logs_enabled: true
```

- `grafana_cloud.enabled`: enables Grafana Cloud metrics and managed alert rules.
- `grafana_cloud.logs_enabled`: enables log shipping when Grafana Cloud is enabled.

Production monitoring policy also includes:

- `alerting.initial_silence_minutes`: duration of the initial managed-alert silence.
- `alerting.thresholds`: `cpu_percent`, `memory_percent`, `filesystem_percent`,
  `filesystem_inode_percent`, and `swap_percent`.
- `collectors.gcs_bucket_check.schedule`: systemd calendar schedule.
- `collectors.gcs_operations.refresh_seconds`: measurement refresh interval.
- `collectors.gcs_operations.monthly_warning_operations`: `class_a` and `class_b`
  monthly warning thresholds.
- `collectors.google_monitoring_usage.monthly_warning_series`: monthly usage
  warning threshold.
- `external_services`: monitored providers and selected status components.
  Status-page providers use `name`, `url`, and a `components` list containing
  `id` and `name`.
- `external_services.proton.services`: list of `name` and `component_name` pairs.
- `external_services.google_cloud.storage` and `.monitoring`: status products
  identified by `title` and `id`.
- `service_units`: `common`, `web`, `mcp`, and `control_node` lists of systemd
  units. Each entry contains `name`, `check`, and `summary`.

Application roles register their containers for process monitoring.
Certificate-expiration warnings, backup/collector freshness periods, and the
Google Monitoring refresh interval are fixed in the config repository.
Grafana credentials and alerting destination metadata remain in
[`vault/common.yml`](#vaultcommonyml). Display timestamps use the server timezone.

[Back to top](#file-schema)

## `grafana/grafana-alert-rule-uids.yml`

Production-only plaintext mapping from stable alert identities to existing UIDs.
Keep entries when moving existing rules to preserve their Grafana history.
Unlisted identities receive deterministic UIDs derived from their identity.

```yaml
---
"host:app.example.com:cpu": "gh-111111111111111111111111"
"host:app.example.com:memory": "gh-222222222222222222222222"
```

[Back to top](#file-schema)

## `grafana/grafana-alert-rules.yml`

Production-only plaintext alert definitions. `parameters` contains shared policy
values. Each `templates` entry has a `name` and a Jinja `template` that renders
one alert object or a list of alert objects as JSON. Templates use the generic
`alert_rule`, label, selector, and dependency helpers provided by the sync tool.
The rendering context excludes API credentials.

The example contains CPU and memory utilization rules. Thresholds, names,
expressions, pending periods, and dependencies belong in this file or its
runtime settings. The sync tool manages rules for the configured client only.

[Back to top](#file-schema)

## `vault/common.yml`

`vault/common.yml` stores secrets and private values shared across managed
hosts.

```yaml
---
root_password_hash: "$6$example-root-password-hash"

restic:
  passphrase: "example-restic-password"
  remotes:
    gcs:
      credentials_json: |
        {
          "type": "service_account",
          "project_id": "grayhaven",
          "client_email": "restic@example.iam.gserviceaccount.com",
          "private_key": "example-private-key"
        }

grafana_cloud:
  stack_url: "https://example.grafana.net"
  alloy_api_key: "glc_example_alloy_token"
  prometheus:
    remote_write_url: "https://prometheus-prod-example.grafana.net/api/prom/push"
    username: "123456"
    datasource_name: "grafanacloud-example-prom"
  loki:
    push_url: "https://logs-prod-example.grafana.net/loki/api/v1/push"
    username: "654321"
  alerting:
    api_token: "glsa_example_alerting_token"
    folder: Grayhaven Systems LLC
    evaluation_group: grayhaven-production-1m
    evaluation_interval: 1m
    contact_point: Grafana IRM
  irm_alert_groups:
    api_token: "glsa_example_irm_alert_groups_reader_token"

users:
  - username: jdoe
    full_name: Jane Doe
    password_hash: "$6$example-admin-password-hash"
    ssh_keys:
      - "ssh-ed25519 AAAAexample jdoe@example"
    sudo: true
    tmux_auto_attach: true
    tmux_workspace: jdoe.tmux
    state: present
  - username: oldadmin
    full_name: Old Admin
    password_hash: "$6$example-old-admin-password-hash"
    ssh_keys:
      - "ssh-ed25519 AAAAexample-old oldadmin@example"
    sudo: true
    state: absent
    home_mode: archive
```

Supported keys:

- `root_password_hash`: Linux password hash for the root account.
- `restic.passphrase`: password used by restic to encrypt backups.
- `restic.remotes.gcs.credentials_json`: optional Google Cloud service account
  JSON used to create Google Cloud Storage buckets and access remote restic
  repositories. Required when `backup.repositories.remote.provider` is `gcs`.
  The service account must have these permissions:
  `Cloud Storage/Storage Admin` and `Monitoring/Monitoring Viewer`.
- `grafana_cloud`: optional Grafana Cloud credential and endpoint settings.
  Required when `observability.grafana_cloud.enabled` is true.
- `users`: list of managed users. User operations are documented in
  [Managing Users](operations.md#managing-users).

Supported `grafana_cloud` keys:

- `stack_url`: Grafana Cloud stack URL.
- `alloy_api_key`: Grafana Cloud token used by Grafana Alloy for metric and log
  shipping.
- `prometheus.remote_write_url`: Grafana Cloud Prometheus remote-write URL.
- `prometheus.username`: Grafana Cloud Prometheus remote-write username.
- `prometheus.datasource_name`: Grafana Cloud Prometheus datasource name used
  by managed alert-rule automation.
- `loki.push_url`: Grafana Cloud Loki push URL. Required only when
  `observability.grafana_cloud.logs_enabled` is true.
- `loki.username`: Grafana Cloud Loki username. Required only when
  `observability.grafana_cloud.logs_enabled` is true.
- `alerting.api_token`: Grafana Cloud API token used to manage Ansible-owned
  alert rules and initial convergence silences.
- `alerting.folder`: Grafana Cloud folder for managed alert rules. Defaults to
  `Grayhaven Systems LLC`.
- `alerting.evaluation_group`: Grafana Cloud evaluation group for managed
  alert rules. Defaults to `grayhaven-production-1m`.
- `alerting.evaluation_interval`: alert evaluation interval. Defaults to `1m`.
- `alerting.contact_point`: Grafana Cloud contact point used by managed alert
  rules. Defaults to `Grafana IRM`.
- `irm_alert_groups.api_token`: Grafana Cloud API token used to read sanitized
  IRM alert-group state and acknowledgement ownership for operational
  reporting. Required when Grafana Cloud observability is enabled.

Supported user keys:

- `username`: Linux username.
- `full_name`: user comment/gecos field.
- `password_hash`: Linux password hash for the account.
- `ssh_keys`: list of public SSH keys installed for the account.
- `sudo`: boolean. When true, password sudo access is enabled for the user.
- `state`: `present` creates and manages the user; `absent` removes the user.
- `home_mode`: optional for absent users. `archive` archives the home directory
  before deletion. `delete` removes the home directory without archiving it.
- `tmux_auto_attach`: optional boolean for sudo users. When true, interactive
  SSH logins to bastion automatically attach to the standard operator tmux
  session. Defaults to false.
- `tmux_workspace`: optional workspace filename for sudo users. The file must
  exist under `files/tmux-workspaces/` and is used by `gtmux` when creating the
  user's tmux session.

The `tmux_*` keys require `sudo: true` because the operator tmux console is for
admin access on bastions.

[Back to top](#file-schema)

## `vault/bastion.yml`

`vault/bastion.yml` stores secrets used by bastion/control-node automation.

```yaml
---
ansible_control:
  private_key: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    example
    -----END OPENSSH PRIVATE KEY-----
  public_key: "ssh-ed25519 AAAAexample ansible@example"

digitalocean_inventory_api_token: "dop_v1_example_inventory_token"

discord_webhooks:
  production: "https://discord.com/api/webhooks/example/production"
  staging: https://discord.com/api/webhooks/example/staging
```

Supported keys:

- `ansible_control.private_key`: private key used by the active Ansible control
  node to reach managed hosts.
- `ansible_control.public_key`: public key installed for the `ansible`
  automation user on managed hosts.
- `digitalocean_inventory_api_token`: DigitalOcean API token used for dynamic
  inventory discovery.
- `discord_webhooks.production`: production Discord notification webhook URL.
- `discord_webhooks.staging`: staging Discord notification webhook URL.

[Back to top](#file-schema)

## `vault/web.yml`

`vault/web.yml` stores web-hosting secrets.

```yaml
---
digitalocean_dns_api_token: "dop_v1_example_dns_token"
web_deploy_fanout_secret: "example_private_web_deploy_fanout_secret"

timetracker_secrets:
  flask_secret_key: "example_flask_secret_key_replace_with_at_least_32_characters"
  sqlcipher_passphrase: "example_sqlcipher_passphrase_replace_with_at_least_32_characters"
  initial_users:
    - email: operator@example.invalid
      first_name: Example
      last_name: Operator
      # Hash of example-insecure-password; never use it operationally.
      password_hash: "$argon2id$v=19$m=65536,t=3,p=4$ZXhhbXBsZS1zYWx0LTAwMDA$tCxgdoL3XKaX1D3RkSeXTMM4zlSl+JB+AGLiY0oSB5U"
      admin: true

hosted_domains:
  - domain: example.com
    deployment:
      type: static
      repository:
        url: https://github.com/example/grayhaven-web.git
        webhook_secret: "example_grayhaven_deploy_webhook_secret"
    dev:
      auth_realm: Grayhaven Systems LLC Development Environment
      htpasswd_entries:
        - "developer:$2y$05$example-grayhaven"

  - domain: example.com
    dev:
      htpasswd_entries:
        - "developer:$2y$05$example-generic"
```

Supported keys:

- `digitalocean_dns_api_token`: DigitalOcean API token used by Certbot DNS-01
  automation in host TLS mode.
- `web_deploy_fanout_secret`: shared secret used by web hosts to authenticate
  private website deployment fanout requests between web hosts. This is required
  when an environment has two or more web hosts behind load-balancer TLS, and it
  must be different from every per-domain GitHub deployment webhook secret.
- `timetracker_secrets`: required Time Tracker secret material. Keep this
  mapping separate from the public `timetracker` mapping in `config.yml`.
- `timetracker_secrets.flask_secret_key`: random Flask signing key containing at
  least 32 characters.
- `timetracker_secrets.sqlcipher_passphrase`: random SQLCipher passphrase
  containing at least 32 characters. Never replace it for an existing database
  without completing the documented rekey workflow.
- `timetracker_secrets.initial_users`: non-empty first-install account list. It
  is used only when the application database contains no users and must include
  at least one administrator.
- `timetracker_secrets.initial_users[].email`: account email address.
- `timetracker_secrets.initial_users[].first_name`: account first name.
- `timetracker_secrets.initial_users[].last_name`: account last name.
- `timetracker_secrets.initial_users[].password_hash`: Argon2id password hash
  meeting the application security parameters.
- `timetracker_secrets.initial_users[].admin`: boolean selecting administrator
  or regular-user access.
- `timetracker_secrets.initial_users[].totp_secret`: optional valid Base32 TOTP
  secret. Omit it to require in-application TOTP enrollment.
- `hosted_domains`: list of domains served by
  [`grayhaven-config-ansible`](https://github.com/grayhavensystemsllc/grayhaven-config-ansible).
- `hosted_domains[].domain`: apex domain name.
- `hosted_domains[].deployment`: optional website deployment configuration.
  Omit this block to render the generic fallback site.
- `hosted_domains[].deployment.type`: website deployment type. Only `static`
  is supported.
- `hosted_domains[].deployment.repository.url`: required public HTTPS Git
  repository used for static website content. For `type: static`, the `main`
  branch deploys `site/frontend/` to the apex and `www` document root, and the
  `dev` branch deploys `site/frontend/` to the development document root.
- `hosted_domains[].deployment.repository.webhook_secret`: required shared
  secret for hosted-domain deployment webhooks. Set this to the matching
  [`GRAYHAVEN_DEPLOY_WEBHOOK_SECRET`](setup.md#set-up-hosted-domain-repositories)
  repository secret value.
  This secret authenticates GitHub-to-Grayhaven deployment requests for one
  hosted-domain repository. It is not used for web-to-web deployment fanout.
  The same setup section documents the matching hosted-domain repository
  variables, including `GRAYHAVEN_DEPLOY_WEBHOOK_URL`,
  `GRAYHAVEN_DEPLOY_WEBHOOK_DISABLE`, and
  `GRAYHAVEN_DEPLOY_DISABLE_SSL_VERIFICATION`.
  Hosted-domain deployment workflows should fail clearly if the required
  webhook URL or shared secret is missing.
- `hosted_domains[].dev`: required development vhost configuration. Every
  hosted domain receives a development vhost.
- `hosted_domains[].dev.auth_realm`: optional HTTP basic-auth realm for the
  development vhost.
- `hosted_domains[].dev.htpasswd_entries`: required full htpasswd file entries
  for the development vhost. At least one entry is required.

`hosted_domains[].dev.htpasswd_entries` values are credential material and
belong in encrypted vault files in `grayhaven-vault`.

### Hosted Domain DNS Coordination

When adding a hosted domain, also add matching
[environment DNS policy](https://github.com/grayhavensystemsllc/grayhaven-infra-opentofu/blob/main/docs/policy.md#environment-dns-policy)
in the
[grayhaven-infra-opentofu](https://github.com/grayhavensystemsllc/grayhaven-infra-opentofu)
repository.

For hosted web domains, set both `environment.apex: true` and
`environment.web_aliases: true` in the matching environment DNS policy.
`environment.web_aliases` requires `environment.apex`.

The baseline workspace owns shared DNS zones, mail records, and CAA records.
Environment workspaces own computed runtime DNS records for hosted domains.

[Back to top](#file-schema)

## `vault/mcp.yml`

`vault/mcp.yml` stores the MCP configuration decryption password and private
GitHub credentials.

```yaml
---
mcp_secrets:
  passphrase: "example_encrypted_config_password_replace_before_deployment"
  github:
    username: "username"
    pat: "example_read_packages_token_replace_before_deployment"
```

Supported keys:

- `mcp_secrets.passphrase`: password used to decrypt
  `config.secrets.enc`.
- `mcp_secrets.github.username`: GitHub username that owns the personal access token.
  Write the username as a quoted string, for example `"username"`.
- `mcp_secrets.github.pat`: GitHub personal access token used for private
  GHCR pulls and read-only configuration downloads from the private repository.
  The token requires `read:packages` and repository contents read access.

[Back to top](#file-schema)
