# vsftpd-helm

Helm chart to deploy a VSFTPD (Very Secure FTP Daemon) server (FTP + SFTP) on Kubernetes.

> **Status:** initial draft based on the current chart structure. Adjust to your image, storage class, and network.

---

## What this chart does

- Deploys a single `Deployment` running your VSFTPD container
- Exposes **FTP**, **SFTP**, and **PASV** ports via a `Service`
- Manages user credentials via a ConfigMap-mounted `users.json`
- Provisions a `PersistentVolume` + `PersistentVolumeClaim` and mounts it at `/data`
- (Optionally) creates a target `Namespace`

> ⚠️ The included PV uses a `hostPath` and a fixed `10Gi` capacity. This is fine for local/dev clusters but **not** suitable for most managed/cloud environments. See [Storage](#storage) for safer options.

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
imagePullPolicy: IfNotPresent

# Ports exposed by the container
containerPorts:
  ftp: 21
  sftp: 22
  pasvmin: 32022
  pasvmax: 32041

# Where the users.json gets mounted inside the container
userConfigPath: "/etc/vsftpd/users.json"

# External address used by PASV replies (set to your LB IP/DNS)
pasvAddress: "ftp.example.com"

# JSON formatted users file; keys are usernames, values are SHA-512 password hashes
users: |
  {
    "plainftp": "$6$REPLACE_ME$...",
    "sftp": "$6$REPLACE_ME$..."
  }

service:
  # ClusterIP for internal-only; LoadBalancer/NodePort for external access
  type: ClusterIP

# Replica count; defaults to 1 if omitted
replicaCount: 1
```

2. Install the chart:

```bash
hhelm install vsftpd oci://dogsbody.azurecr.io/helm/vsftpd --version <Version Number>
```

---

## Configuration

| Key                      | Type          | Default                  | Description                                                                                                  |
| ------------------------ | ------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------ |
| `namespace`              | string        | *required*               | Namespace to deploy into. The chart includes a `Namespace` manifest and will create it if it does not exist. |
| `environment`            | string        | `"dev"`                  | Label value applied to resources.                                                                            |
| `replicaCount`           | int           | `1`                      | Number of pod replicas.                                                                                      |
| `image.repository`       | string        | *required*               | Container image reference (e.g., `ghcr.io/yourorg/vsftpd_container:tag`).                                    |
| `imagePullPolicy`        | string        | `IfNotPresent`           | Pod image pull policy.                                                                                       |
| `containerPorts.ftp`     | int           | `21`                     | FTP control port exposed by the container.                                                                   |
| `containerPorts.sftp`    | int           | `22`                     | SFTP/SSH port exposed by the container.                                                                      |
| `containerPorts.pasvmin` | int           | `32022`                  | Minimum passive data port exposed by the container.                                                          |
| `containerPorts.pasvmax` | int           | `32041`                  | Maximum passive data port exposed by the container.                                                          |
| `userConfigPath`         | string        | `/etc/vsftpd/users.json` | Mount target where `users.json` is provided. The pod sets `USER_CONFIG_PATH` env var to this.                |
| `users`                  | string (JSON) | *required*               | JSON map of `username: sha512-password-hash`. Mounted as `users.json` via ConfigMap.                         |
| `pasvAddress`            | string        | none                     | External IP/hostname included in PASV replies; **must** be reachable by clients.                             |
| `service.type`           | string        | `ClusterIP`              | Service type: `ClusterIP`, `NodePort`, or `LoadBalancer`.                                                    |

> **Note on naming:** The Service opens named ports for `ftp`, `sftp`, `pasvmin`, and `pasvmax`. Ensure your firewall and cloud LB allow the full PASV range.

---

## Storage

By default the chart ships with **static** storage manifests:

- `PersistentVolume` using `hostPath: /data` with `10Gi` capacity
- `PersistentVolumeClaim` requesting `10Gi`

This is **not** appropriate for cloud clusters. Prefer a dynamic PVC bound by `storageClassName` instead. To do so, remove/disable the bundled `PersistentVolume` and set a PVC template or inject your own PVC:

### Option A — Provide a pre-created PVC

1. Create your own PVC (backed by a StorageClass).
2. Modify the Deployment template or chart values to reference that PVC name.

### Option B — Make PV/PVC optional (recommended change)

If you plan to extend this chart, make the PV/PVC manifests conditional with a value like `persistence.enabled` and support `persistence.storageClass` and `persistence.size`.

The container mounts the PVC at `/data`.

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

- Set `pasvAddress` to your public IP or DNS name.
- Ensure `containerPorts.pasvmin..pasvmax` are open on the Kubernetes `Service` and any LB/firewall.
- If you run behind a cloud LoadBalancer, use a **static** public IP and health checks on the control port.

For internal-only use, keep `service.type=ClusterIP` and connect from within the cluster/VPN.

---

## Resources created

- `Namespace` (name from `values.namespace`)
- `Deployment` `<release>-deploy`
- `Service` `<release>-svc`
- `ConfigMap` `<release>-users-config`
- `PersistentVolume` `<release>-pv` (hostPath)
- `PersistentVolumeClaim` `<release>-pvc`

---

## Uninstall

```bash
helm uninstall ftp --namespace ftp
```

This removes chart-managed Kubernetes resources. Manually delete any static PVs or cloud disks you created.

---

## Roadmap / suggested improvements

- Make PV/PVC optional (`persistence.enabled`) and support dynamic provisioning
- Support `Secret` for users (and mount as file)
- Add `resources` limits/requests
- Add `liveness`/`readiness` probes
- Expose `image.tag` separately from `image.repository`
- Fix potential naming drift between values used in env vs. ports (`pasvMin/Max` vs `pasvmin/max`)
- Document a chart repo and release the package

---

## Troubleshooting

- **500 Series FTP errors:** usually PASV not reachable; verify `pasvAddress` and firewall
- **Auth failures:** confirm SHA-512 hash format in `users.json`
- **Data not persistent:** storage still `hostPath`; switch to a real StorageClass-backed PVC

---

## License

Same as the repository (add license section here if applicable).

