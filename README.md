# vsftpd-helm

Helm chart to deploy a VSFTPD (Very Secure FTP Daemon) server (FTP + SFTP) on Kubernetes.

> **Chart Version:** 1.1.3 | **App Version:** 3.0.2

---

## What this chart does

- Deploys a single `Deployment` running your VSFTPD container with configurable resources
- Exposes **SFTP** via one dedicated `Service` 
- Exposes **FTP** control and **all passive data ports** via a separate `Service`
- Manages user credentials via a ConfigMap-mounted `users.json`
- Provisions a `PersistentVolumeClaim` and mounts it at `/data` (supports dynamic provisioning)
- Creates a target `Namespace` if specified
- Includes a `PodDisruptionBudget` for high availability

---

## Prerequisites

- Kubernetes 1.23+
- Helm 3.x
- A container image for vsftpd (e.g. in GHCR/ACR)
- A network plan for PASV mode (firewall/NAT rules and a stable external address)

---

## Install (quick start)

1. Create a `my-values.yaml` with at least your image and users. Example:

```yaml
# my-values.yaml (minimal example)
namespace: "ftp"
environment: "prod"

image:
  repository: "ghcr.io/YOURORG/vsftpd_container:latest"
  pullPolicy: IfNotPresent

# Where the users.json gets mounted inside the container
userConfigPath: "/etc/vsftpd/users.json"

# JSON formatted users file; keys are usernames, values are SHA-512 password hashes
users: |
  {
    "plainftp": "$6$REPLACE_ME$...",
    "sftp": "$6$REPLACE_ME$..."
  }

service:
  sftp:
    type: LoadBalancer  # SFTP service type
    port: 22
    publicIPv4: ""  # Public IP for the SFTP service, if needed
  ftp:
    type: LoadBalancer  # FTP service type
    publicIPv4: ""  # Public IPv4 for the FTP service, if needed
    pasvAddress: "ftp.example.com"  # External address used by PASV replies
    controlPort: 21   # FTP control port
    pasvMinPort: 10000  # Minimum passive port
    pasvMaxPort: 10025  # Maximum passive port
  loadBalancerSourceRanges: # Sets allowed Source IP's for the LoadBalancer
    - "0.0.0.0/0"  # Adjust to your needs

# Resource limits and requests
resources:
  limits:
    memory: "512Mi"
  requests:
    memory: "128Mi"
    cpu: "50m"

# Replica count; defaults to 1 if omitted
replicaCount: 1
```

2. Install the chart:

```bash
helm install vsftpd oci://dogsbody.azurecr.io/helm/vsftpd --version <Version Number>
```

---

## Configuration

| Key                      | Type          | Default                  | Description                                                                                                  |
| ------------------------ | ------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------ |
| `namespace`              | string        | `"vsftpd"`               | Namespace to deploy into. The chart includes a `Namespace` manifest and will create it if it does not exist. |
| `environment`            | string        | `"dev"`                  | Label value applied to resources.                                                                            |
| `replicaCount`           | int           | `1`                      | Number of pod replicas.                                                                                      |
| `minAvailable`           | int           | `1`                      | Minimum available pods for PodDisruptionBudget.                                                             |
| `image.repository`       | string        | `dogsbody.azurecr.io/vsftpd:v1.1.0` | Container image reference.                                                                     |
| `image.pullPolicy`       | string        | `always`                 | Pod image pull policy.                                                                                       |
| `image.tag`              | string        | `""`                     | Overrides the image tag whose default is the chart appVersion.                                               |
| `service.sftp.type`      | string        | `LoadBalancer`           | SFTP service type: `ClusterIP`, `NodePort`, or `LoadBalancer`.                                               |
| `service.sftp.port`      | int           | `22`                     | SFTP/SSH port.                                                                                               |
| `service.sftp.publicIPv4`| string        | `""`                     | Public IPv4 for the SFTP service, if needed.                                                                |
| `service.ftp.type`       | string        | `LoadBalancer`           | FTP service type: `ClusterIP`, `NodePort`, or `LoadBalancer`.                                                |
| `service.ftp.publicIPv4` | string        | `""`                     | Public IPv4 for the FTP service, if needed.                                                                 |
| `service.ftp.pasvAddress`| string        | `""`                     | External IP/hostname included in PASV replies; **must** be reachable by clients.                             |
| `service.ftp.controlPort`| int           | `21`                     | FTP control port.                                                                                            |
| `service.ftp.pasvMinPort`| int           | `10000`                  | Minimum passive data port.                                                                                   |
| `service.ftp.pasvMaxPort`| int           | `10025`                  | Maximum passive data port.                                                                                   |
| `service.loadBalancerSourceRanges` | array | `[""]`                   | Sets allowed Source IP's for the LoadBalancer.                                                              |
| `userConfigPath`         | string        | `/etc/vsftpd/users.json` | Mount target where `users.json` is provided. The pod sets `USER_CONFIG_PATH` env var to this.                |
| `users`                  | string (JSON) | `{}`                     | JSON map of `username: sha512-password-hash`. Mounted as `users.json` via ConfigMap.                         |
| `resources.limits.memory`| string        | `"512Mi"`                | Memory limit for the container.                                                                              |
| `resources.requests.memory`| string      | `"128Mi"`                | Memory request for the container.                                                                            |
| `resources.requests.cpu` | string        | `"50m"`                  | CPU request for the container.                                                                               |
| `storageClassName`       | string        | `""`                     | StorageClass for dynamic PVC provisioning. If empty, uses cluster default.                                   |

> **Note on services:** The chart creates **two separate services** - one for SFTP and one for FTP with all its passive ports. Each service can have different types and load balancer configurations. Ensure your firewall and cloud LB allow the full PASV range for the FTP service.

---

## Storage

The chart creates a `PersistentVolumeClaim` requesting `10Gi` of storage. By default:

- The PVC uses the cluster's default StorageClass for dynamic provisioning
- If you need a specific StorageClass, set `storageClassName` in your values
- The static `PersistentVolume` manifest is commented out (not created by default)

This approach works well for cloud clusters with dynamic provisioning. The container mounts the PVC at `/data`.

### Using a specific StorageClass

```yaml
storageClassName: "fast-ssd"  # Your preferred StorageClass
```

### Using static volumes (not recommended)

If you need static volumes, uncomment the PV section in `templates/volumes.yml` and adjust the `hostPath` or configure your preferred volume type.

---

## Users & authentication

- The chart renders a `ConfigMap` named `<release>-users-config` containing `users.json` from `.Values.users`.
- This is mounted read-only at `userConfigPath` (default `/etc/vsftpd/users.json`).
- Passwords should be **SHA-512** crypt hashes. Generate with `mkpasswd -m sha-512` (Debian/Ubuntu `whois` package) or `openssl passwd -6`.

**Example JSON:**

```json
{
  "plainftp": "$6$BWZe/CFWGwBT4QTL$...",
  "sftp": "$6$BWZe/CFWGwBT4QTL$..."
}
```

> Consider using a `Secret` instead of a `ConfigMap` for credentials and projecting it as a file; or mount from an external secret manager.

---

## Networking (PASV mode)

FTP passive mode requires the server to advertise an **external address** and to have a **contiguous port range** open end‑to‑end:

- Set `service.ftp.pasvAddress` to your public IP or DNS name
- Configure `service.ftp.pasvMinPort` and `service.ftp.pasvMaxPort` for your passive range
- The FTP service automatically exposes all ports in the passive range
- If you run behind a cloud LoadBalancer, use a **static** public IP and health checks on the control port

**Service Configuration:**
- **SFTP Service**: Separate service for SSH/SFTP traffic (port 22)
- **FTP Service**: Handles FTP control port (21) and all passive data ports

For internal-only use, set both `service.sftp.type` and `service.ftp.type` to `ClusterIP` and connect from within the cluster/VPN.

---

## Resource Management

The chart supports configurable resource limits and requests:

```yaml
resources:
  limits:
    memory: "512Mi"      # Maximum memory usage
  requests:
    memory: "128Mi"      # Guaranteed memory allocation
    cpu: "50m"           # Guaranteed CPU allocation (50 millicores)
```

Adjust these values based on your expected load and cluster capacity.

---

The chart includes a `PodDisruptionBudget` to ensure service availability during cluster maintenance:

- `minAvailable`: Minimum number of pods that must remain available (default: 1)
- Protects against voluntary disruptions (node drains, upgrades, etc.)
- Configure via `minAvailable` value in your values file

---

## Resources created

- `Namespace` (name from `values.namespace`)
- `Deployment` `<release>-server`
- `Service` `<release>-sftp-svc` (for SFTP traffic)
- `Service` `<release>-ftp-svc` (for FTP control and passive data ports)
- `ConfigMap` `<release>-users-config`
- `PersistentVolumeClaim` `<release>-pvc`
- `PodDisruptionBudget` `vsftpd-pdb`

---

## Uninstall

```bash
helm uninstall ftp --namespace ftp
```

This removes chart-managed Kubernetes resources. Manually delete any static PVs or cloud disks you created.

---

## Roadmap / suggested improvements

- ✅ Add `resources` limits/requests (implemented)
- ✅ Add `PodDisruptionBudget` support (implemented)
- ✅ Support dynamic PVC provisioning with `storageClassName` (implemented)
- ✅ Expose `image.tag` separately from `image.repository` (implemented)
- ✅ Create separate services for FTP and SFTP (implemented)
- Make PV/PVC optional with `persistence.enabled` flag
- Support `Secret` for users (and mount as file)
- Add `liveness`/`readiness` probes
- Document a chart repo and release the package
- Add support for custom annotations and labels on services
- Support for multiple storage volumes

---

## Troubleshooting

- **500 Series FTP errors:** usually PASV not reachable; verify `service.ftp.pasvAddress` and firewall rules
- **Auth failures:** confirm SHA-512 hash format in `users.json`
- **Data not persistent:** check PVC status and StorageClass availability
- **Pod startup issues:** verify resource limits and image availability
- **Service connectivity:** ensure LoadBalancer has been assigned external IPs

---

## License

Same as the repository (add license section here if applicable).

