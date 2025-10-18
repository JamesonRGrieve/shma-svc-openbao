# OpenBao Service

Declarative OpenBao 2.4 service definition with hardened defaults for TLS, Raft storage, and automated lifecycle management across Proxmox LXC, Docker Compose, Podman Quadlet, and Kubernetes targets.

## Runtime Coverage
- Proxmox LXC with nesting/keyctl for secure credential storage
- Docker Compose v2 (HCL rendered via secret)
- Podman Quadlet (system scope)
- Kubernetes StatefulSet + Service + Secret + PVC

## Dependencies
None. Raft storage and optional auto-unseal integrations handle state without external databases.

## Exports
```
OPENBAO_ADDR={{ openbao_external_url }}
OPENBAO_API_ADDR={{ openbao_api_addr }}
OPENBAO_CLUSTER_ADDR={{ openbao_cluster_addr }}
```

## Secrets
- `OPENBAO_INIT_RECOVERY_PASSPHRASE` → Required recovery passphrase used for disaster recovery workflows
- `openbao-config.hcl` → Raft storage configuration, listener bindings, audit devices, and auto-unseal
- `openbao-tls-chain.crt` / `openbao-tls.key` → Full certificate chain and private key when TLS is enabled

Root tokens are **not** accepted via environment variables. Use the automated initialization job or run `bao operator init` manually to generate bootstrap credentials, then immediately store the output in a secure vault.

## Mounts
- Persistent: `/etc/openbao` (HCL configuration), `/var/lib/openbao/data` (Raft storage backend)
- Ephemeral: `/run/openbao` (32Mi Memory tmpfs), `/tmp/openbao` (64Mi Memory tmpfs)

## Security Posture
- Runs as UID/GID 1000 with `runAsNonRoot` and `fsGroup` enforced for Kubernetes pods
- Root filesystem remains writable to support Raft snapshots and telemetry buffers
- Drops all Linux capabilities and sets `no_new_privileges`
- `mlock` is enabled by default (set `openbao_disable_mlock: true` only for environments that cannot grant `CAP_IPC_LOCK`)

## TLS & Certificates
- TLS is **enabled by default**. Set `openbao_tls_disable: true` only for local development scenarios.
- The listener binds to `openbao_listen_address` (default `0.0.0.0`); configure a specific interface for multi-NIC hosts.
- Certificate and key content are parsed with Ansible's OpenSSL modules; deployments fail if the chain and key do not match or if expiry is within seven days (override with `openbao_force_certificate_deploy: true`).
- A warning is emitted when the certificate expires within `openbao_certificate_expiry_warning_days` (default 30 days).
- Provide full chains through `openbao-tls-chain.crt`; intermediate CAs are supported.
- Custom installation paths are available via `openbao_tls_cert_path` and `openbao_tls_key_path`.

## Health Probes & Performance Monitoring
- **Liveness** → `/v1/sys/health?standbyok=true&sealedcode=200` every 30s (15s timeout) to confirm API responsiveness.
- **Readiness** → `/v1/sys/health?standbyok=true` gate parsed with `jq`; it fails when the node is sealed, the response omits latency data, or observed latency exceeds `openbao_readiness_latency_threshold` (default 0.75s).
- **Startup** → Extended 30s probe with failure threshold 10 (≈5 minutes) to survive slow Raft bootstrap; reduce the threshold when running on faster storage.

## Raft Storage & High Availability
- Production mode enforces TLS, mlock, and a minimum of three replicas while surfacing a PodDisruptionBudget, topology spread constraints, and anti-affinity guidance through the role defaults.
- Raft node IDs default to `<service name>-<pod hostname>`; override via `openbao_raft_node_id` if needed.
- `service_init_containers` adds a Raft join helper that targets the headless service leader (`{{ service_name }}-0.{{ service_name }}-headless`) and only suppresses the "already joined" exit code.
- Persistent volume permissions restrict access to UID/GID 1000.

## Auto-Unseal Integrations
Set `openbao_auto_unseal_method` to `aws`, `azure`, or `gcp` to enable KMS-backed auto-unseal. Populate the associated configuration fields (`openbao_auto_unseal_config`) and, where unavoidable, credentials (`openbao_auto_unseal_credentials`):

| Method | Required configuration | Optional configuration | Credential hints |
| ------ | ---------------------- | ---------------------- | ---------------- |
| `aws`  | `region`, `kms_key_id` | `endpoint`             | Prefer IAM roles; static `access_key` / `secret_key` trigger a warning. |
| `azure`| `tenant_id`, `vault_name`, `key_name` | `key_version` | Use managed identities; `client_id` / `client_secret` should be avoided. |
| `gcp`  | `project`, `region`, `key_ring`, `crypto_key` | — | Use Workload Identity; `credentials_json` is discouraged. |

When unset, the cluster defaults to manual unseal using recovery shares.

## Initialization Automation
Set `openbao_init_job_enabled: true` to schedule a one-shot bootstrap job that:
1. Runs `bao operator init -format=json`
2. Pipes the JSON payload into `openbao_init_job_secure_store` (allowlisted commands such as `sops`, `vault`, or `kubectl`)
3. Optionally forwards artifacts to an external secret backend using `openbao_init_job_backend_secret`

The job is rendered up-front with `completions: 1` and `backoffLimit: 0` for idempotency and never exposes root tokens or unseal keys via environment variables or logs.

## Snapshot & Disaster Recovery
- Enable automated snapshots with `openbao_snapshot_settings.enabled: true`.
- Configure `schedule`, `storage_path`, `retention`, and optional `secure_export_secret` to push archives to long-term storage.
- The generated `service_jobs` entry invokes `bao operator raft snapshot save`, prunes archives older than `retention` days, and only references `OPENBAO_SNAPSHOT_EXPORT_SECRET` when it is defined.

### Disaster Recovery Test Workflow
1. Enable snapshots and verify archives exist in the secure destination.
2. Stop a cluster node and restore a snapshot into a clean environment.
3. Run `bao operator raft snapshot restore`, unseal the node (auto-unseal or recovery shares), and rejoin via the init container helper.
4. Validate access using recovery tokens, rotate the root token, and reissue trusted identities.

Document successful restorations alongside the snapshot schedule to satisfy compliance requirements.

## Audit Logging
Configure `openbao_audit_device_type`, `openbao_audit_device_path`, and optional `openbao_audit_device_mode` to enable OpenBao audit sinks (e.g., file, syslog, socket). Audit devices are rendered directly into the HCL configuration.

## Monitoring & Metrics
- OpenBao exposes Prometheus metrics at `/v1/sys/metrics`.
- The readiness command enforces latency thresholds; tune `openbao_readiness_latency_threshold` based on SLOs.
- Integrate with Prometheus Operator by referencing the generated Service and creating a `ServiceMonitor` that targets port `{{ openbao_service_port }}` with the `/v1/sys/metrics` path.

## UI & Access Controls
The UI requires a valid OpenBao token and should be presented only behind a VPN or firewall (`openbao_enable_ui: true` by default). Ensure TLS is enabled and restrict ingress using reverse proxies or network policies.

## Migration from HashiCorp Vault
OpenBao maintains API compatibility with Vault 1.15+.
1. Export policies, auth backends, and secrets using `vault` CLI tooling (`vault kv get`, `vault policy read`).
2. Stand up OpenBao using this role with Raft and TLS enabled.
3. Import policies and secrets via the compatible `/v1/` APIs or `bao` CLI.
4. Repoint applications to the new endpoints after validating audit logs and access controls.

## Key Overrides
| Variable | Default | Purpose |
| --- | --- | --- |
| `openbao_service_port` | `8200` | API/UI HTTPS port |
| `openbao_cluster_port` | `8201` | Raft cluster communication port |
| `openbao_tls_disable` | `false` | Disable TLS (development only) |
| `openbao_production` | `false` | Enforce production hardening checks |
| `openbao_disable_mlock` | `false` | Disable memory locking (only when CAP_IPC_LOCK unavailable) |
| `openbao_enable_ui` | `true` | Toggle web UI |
| `openbao_log_level` | `warn` | Logging verbosity (`trace`, `debug`, `info`, `warn`, `error`) |
| `openbao_listen_address` | `0.0.0.0` | Listener bind address |
| `openbao_readiness_latency_threshold` | `0.75` | Maximum acceptable readiness latency (seconds) |
| `openbao_tls_cert_path` | `/etc/openbao/tls/tls-chain.crt` | TLS certificate chain path |
| `openbao_tls_key_path` | `/etc/openbao/tls/tls.key` | TLS private key path |
| `openbao_force_certificate_deploy` | `false` | Allow certificates expiring within seven days |
| `openbao_snapshot_settings` | `enabled: false` | Scheduled Raft snapshot automation |
| `openbao_auto_unseal_method` | `""` | Optional KMS auto-unseal provider |
| `openbao_auto_unseal_config` | `{}` | Provider-specific configuration (region, key identifiers) |
| `openbao_auto_unseal_credentials` | `{}` | Provider-specific credentials (discouraged; prefer workload identity) |
| `openbao_kubernetes_namespace` | `security` | Kubernetes namespace |
| `openbao_kubernetes_replicas` | `1` | Replica count (set ≥3 for HA) |

## Usage Example
```yaml
- hosts: secrets_hosts
  roles:
    - role: svc-openbao
      vars:
        runtime: podman
        openbao_tls_disable: false
        openbao_tls_cert: "{{ vault_openbao_tls_chain }}"
        openbao_tls_key: "{{ vault_openbao_tls_key }}"
        openbao_recovery_passphrase: "{{ lookup('community.general.random_string', 48, 'ascii_letters,digits,special') }}"
        openbao_snapshot_settings:
          enabled: true
          schedule: "0 */6 * * *"
          storage_path: /var/lib/openbao/snapshots
          retention: 28
          secure_export_secret: openbao/snapshots
```

## Post-Deployment Checklist
1. (Optional) Enable the automated init job to capture bootstrap credentials in your vault platform.
2. Validate readiness probe reports `sealed: false` and latency within thresholds.
3. Run `bao status` to confirm Raft peers, PodDisruptionBudget compliance, and auto-unseal state.
4. Execute the documented disaster recovery test at least once per quarter, including post-restore unseal and credential rotation.
