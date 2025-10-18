# OpenBao Service

Declarative OpenBao 2.4 secrets management service definition leveraging the shared infrastructure adapters. This role renders OpenBao with integrated Raft storage across Proxmox LXC, Docker Compose, Podman Quadlet, Kubernetes, or bare-metal systemd while maintaining configuration portability and security best practices.

## Runtime Coverage
- Proxmox LXC with nesting/keyctl features for secure credential storage
- Docker Compose v2 (HCL config delivered via Docker secret)
- Podman Quadlet (system scope)
- Kubernetes StatefulSet + Service + Secret + PVC
- Bare-metal systemd with tmpfs mounts and HCL configuration

## Dependencies
None - OpenBao uses integrated Raft storage for high availability

## Exports
```
OPENBAO_ADDR={{ openbao_external_url }}
OPENBAO_API_ADDR={{ openbao_api_addr }}
OPENBAO_CLUSTER_ADDR={{ openbao_cluster_addr }}
```

## Secrets
- `OPENBAO_INIT_ROOT_TOKEN` → Initial root token for unsealing and bootstrapping
- `OPENBAO_INIT_RECOVERY_PASSPHRASE` → Recovery passphrase for disaster recovery scenarios
- `openbao-config.hcl` → Raft storage configuration with listener and cluster settings
- `openbao-tls.crt` / `openbao-tls.key` → TLS certificates when `openbao_tls_disable` is false

## Mounts
- Persistent: `/etc/openbao` (HCL config), `/var/lib/openbao/data` (Raft storage backend)
- Ephemeral: `/run/openbao` (32Mi Memory tmpfs), `/tmp/openbao` (64Mi Memory tmpfs)

## Security Posture
- Runs as UID/GID 1000 (non-root)
- Read-only root filesystem disabled (OpenBao requires write access to storage)
- Drops all capabilities
- No new privileges allowed
- `mlock` disabled by default for containerized deployments (override via `openbao_disable_mlock`)

## Health Check
Queries `/v1/sys/health?standbyok=true&sealedcode=200` to ensure OpenBao API responds. Returns 200 when sealed or unsealed in standby, allowing gradual initialization. The probe feeds Compose healthchecks, Quadlet checks, Kubernetes readiness/liveness, and post-deploy validation.

## Key Overrides
| Variable                        | Default     | Purpose                                                     |
| ------------------------------- | ----------- | ----------------------------------------------------------- |
| `openbao_service_port`          | `8200`      | API/UI HTTP(S) port                                         |
| `openbao_cluster_port`          | `8201`      | Raft cluster communication port                             |
| `openbao_tls_disable`           | `true`      | Disable TLS for development (set to `false` for production) |
| `openbao_disable_mlock`         | `true`      | Disable memory locking (required for containers)            |
| `openbao_enable_ui`             | `true`      | Enable web UI                                               |
| `openbao_log_level`             | `info`      | Logging verbosity (trace, debug, info, warn, error)         |
| `openbao_raft_node_id`          | `openbao-1` | Raft cluster node identifier                                |
| `openbao_data_volume_size_gb`   | `20`        | Persistent storage for Raft data                            |
| `openbao_config_volume_size_gb` | `2`         | Persistent storage for configuration                        |
| `openbao_container_vmid`        | `250`       | Proxmox VMID                                                |
| `openbao_container_memory_mb`   | `4096`      | Memory allocation                                           |
| `openbao_container_cpu_cores`   | `2`         | vCPU allocation                                             |
| `openbao_kubernetes_namespace`  | `security`  | Namespace for workload                                      |

## Configuration Template
The default HCL configuration uses integrated Raft storage:
- Storage path: `/var/lib/openbao/data`
- Listener binds to `0.0.0.0:8200` (API) and `0.0.0.0:8201` (cluster)
- TLS can be enabled by setting `openbao_tls_disable: false` and providing certificate secrets
- API and cluster addresses auto-configure based on `service_ip` and TLS settings

## Production Deployment Considerations
1. **TLS**: Set `openbao_tls_disable: false` and provide valid certificates via vault
2. **Initialize**: After first boot, run `bao operator init` to generate root tokens and unseal keys
3. **Unseal**: OpenBao starts sealed - use `bao operator unseal` with generated keys
4. **Backup**: The Raft storage path (`/var/lib/openbao/data`) must be included in backup strategies
5. **HA Cluster**: Deploy 3+ replicas with unique `openbao_raft_node_id` values and join via API

## Usage
```yaml
- hosts: secrets_hosts
  roles:
    - role: svc-openbao
      vars:
        runtime: podman
        openbao_tls_disable: false
        openbao_tls_cert: "{{ vault_openbao_tls_cert }}"
        openbao_tls_key: "{{ vault_openbao_tls_key }}"
        openbao_init_root_token: "{{ vault_openbao_root_token }}"
        openbao_recovery_passphrase: "{{ vault_openbao_recovery_pass }}"
        openbao_data_volume_size_gb: 50
```

## Post-Deployment Initialization
```bash
# Initialize OpenBao (first time only)
export VAULT_ADDR=http://192.168.100.16:8200
bao operator init

# Unseal (required after every restart)
bao operator unseal <unseal-key-1>
bao operator unseal <unseal-key-2>
bao operator unseal <unseal-key-3>

# Verify status
bao status
```