# rhvp.parasol_mssql_demo

Ansible collection that deploys the [Parasol Insurance
demo](https://github.com/validatedpatterns-demos/parasol-insurance-mssql) across a two-tier
VM estate:

* **`mssql_win` role** — installs SQL Server 2022 (Developer edition) on
  Windows Server via Chocolatey, enables TCP/IP on port 1433, opens the
  firewall, and creates the empty `parasol` database (the app's Hibernate
  `drop-and-create` fills the schema and seed data at startup).
* **`quadlet_app` role** — deploys the Quarkus app container image on RHEL
  using a [podman
  quadlet](https://www.redhat.com/sysadmin/quadlet-podman) unit so the app
  runs as a native `systemd` service.

The collection does **not** build the image; the app repo's GitHub Actions
workflow (`build-push.yaml`) builds and pushes to
`quay.io/validatedpatterns/parasol-mssql`.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install "ansible-core>=2.15" ansible-lint
ansible-galaxy collection install -r requirements.yml
```

While developing in this repo, symlink the collection into the coding path so
playbooks can resolve `rhvp.parasol_mssql_demo.*` roles:

```bash
mkdir -p ~/.ansible/collections/ansible_collections/rhvp
ln -sfn "$(pwd)" ~/.ansible/collections/ansible_collections/rhvp/parasol_mssql_demo
```

## Usage

1. Copy `examples/vault.yml.example` to a path **outside** the repo (or a
   gitignored vault file), fill in secrets, and encrypt with `ansible-vault`.
2. Point inventory at your VMs:
   - Local / static IPs: start from `host.yml`
   - OpenShift Virtualization via
     [edge-gitops-vms](https://github.com/mhjacks/edge-gitops-vms-chart): use
     `examples/inventory-edge-gitops-vms.yml` (see
     [Provisioning VMs with edge-gitops-vms](#provisioning-vms-with-edge-gitops-vms))
3. The `parasol_rhel` hosts need `quadlet_app_mssql_host` set to the Windows SQL
   Server host DNS name/IP (reachable from the RHEL guest).
4. Run the site playbook (requires WinRM already reachable on the Windows
   host, and SSH + an ansible user with `become` on the RHEL host):

```bash
ansible-playbook -i host.yml -e @~/vault-parasol.yml --ask-vault-pass \
  playbooks/site.yml
```

The playbook runs the Windows deployment first, then the RHEL quadlet
deployment, and finally waits for `/q/health/ready` on the RHEL host to
confirm the app is up.

### Provisioning VMs with edge-gitops-vms

This collection ships ready-to-adapt examples under `examples/`:

| File | Purpose |
|------|---------|
| `examples/edge-gitops-vms-values.yaml` | Helm values for [edge-gitops-vms](https://github.com/mhjacks/edge-gitops-vms-chart) (Windows + RHEL VMs and Services) |
| `examples/inventory-edge-gitops-vms.yml` | Ansible inventory targeting those Services |
| `examples/vault.yml.example` | Credential variable names (fill + encrypt; never commit secrets) |

The collection expects a **Windows Server** guest (WinRM :5985, TCP :1433 after
SQL install) and a **RHEL** guest (SSH :22, HTTP :8080 for the app).

| Guest | Chart key | Resulting VM / Service name | Ports published |
|-------|-----------|-----------------------------|-----------------|
| SQL Server | `vms.mssql` | `windows2k22-mssql-001` | 22, 5985, 1433 |
| Parasol app | `vms.parasol` | `rhel9-parasol-001` | 22, 8080 |

Windows golden image used in the example values:
`quay.io/martjack/windows-server-2022-standard:latest` (VirtIO, OpenSSH, WinRM
baked in). Important overrides vs chart defaults: `os` must contain `windows`,
`efi: true`, `memory: 8Gi`, OS disk `storage: 60Gi` plus
`additionalDisks.sqldata` (100Gi) for database files, Service ports for
WinRM/MSSQL/HTTP, and `mainDiskStorageBus: virtio` for that golden image.
The example inventory sets `mssql_win_data_drive_letter: D` so `mssql_win`
initializes that disk and places SQL data/log/tempdb there.

```bash
# After installing the collection:
ansible-galaxy collection install rhvp.parasol_mssql_demo
COLLECTION="$(ansible-galaxy collection list rhvp.parasol_mssql_demo -p ~/.ansible/collections --format '{path}' 2>/dev/null | head -1)"
# Or while developing this repo: COLLECTION="$(pwd)"

helm upgrade --install parasol-vms edge-gitops-vms \
  -n edge-gitops-vms --create-namespace \
  -f "${COLLECTION}/examples/edge-gitops-vms-values.yaml"

ansible-playbook \
  -i "${COLLECTION}/examples/inventory-edge-gitops-vms.yml" \
  -e @~/vault-parasol.yml --ask-vault-pass \
  rhvp.parasol_mssql_demo.site
```

From the RHEL guest, `quadlet_app_mssql_host` should be the Windows Service DNS
name (e.g. `windows2k22-mssql-001.edge-gitops-vms.svc`) so the app can reach
SQL on :1433 after `mssql_win` opens the firewall.

### Role defaults / main variables

| Role | Variable | Default | Purpose |
|------|----------|---------|---------|
| `quadlet_app` | `quadlet_app_image` | `quay.io/validatedpatterns/parasol-mssql:latest` | Container image to run |
| `quadlet_app` | `quadlet_app_service_name` | `parasol-app` | systemd unit name |
| `quadlet_app` | `quadlet_app_http_port` | `8080` | Host port for the app |
| `quadlet_app` | `quadlet_app_mssql_host` | `localhost` | **Must** be overridden |
| `quadlet_app` | `quadlet_app_mssql_user`/`quadlet_app_mssql_password`/`quadlet_app_mssql_database` | `sa` / `P@rasol-2026-Sql` / `parasol` | Datasource env vars |
| `quadlet_app` | `quadlet_app_hibernate_database_generation` | empty | Optional `QUARKUS_HIBERNATE_ORM_DATABASE_GENERATION` (e.g. `none` to keep DB) |
| `quadlet_app` | `quadlet_app_hibernate_sql_load_script` | empty | Optional `QUARKUS_HIBERNATE_ORM_SQL_LOAD_SCRIPT` (e.g. `no-file`) |
| `quadlet_app` | `quadlet_app_registry_username`/`quadlet_app_registry_password` | empty | Podman login, only if the image repo is private |
| `quadlet_app` | `quadlet_app_rhsm_register` | `false` | Register the RHEL host with subscription-manager first |
| `quadlet_app` | `quadlet_app_rhsm_org_id`/`quadlet_app_rhsm_activation_key` | empty | Activation-key registration (or `quadlet_app_rhsm_username`/`_password` alternative) |
| `mssql_win` | `mssql_win_chocolatey_package` | `sql-server-2022` | Chocolatey package |
| `mssql_win` | `mssql_win_sa_password` | `P@rasol-2026-Sql` | `sa` password (Mixed mode is enabled by the role) |
| `mssql_win` | `mssql_win_tcp_port` | `1433` | TCP/IP port opened on the firewall |
| `mssql_win` | `mssql_win_database` | `parasol` | Database created for the app |
| `mssql_win` | `mssql_win_data_drive_letter` | empty | When set (e.g. `D`), initialize that disk and put SQL data there |
| `mssql_win` | `mssql_win_data_root` | `D:\MSSQL` if letter set | Root for data/log/tempdb/backup paths |

## Gotchas

* Chocolatey's `sql-server-2022` package ships a `ConfigurationFile.ini` with
  **TCP/IP disabled** and **Windows-only auth**. The role passes
  `/SECURITYMODE:"SQL"` + `/SAPWD:"..."` to the SQL Setup invocation and then
  flips the `SuperSocketNetLib\Tcp` registry keys, restarting `MSSQLSERVER`
  afterwards. Do not "simplify" these steps away.
* SQL Server install can take well over 30 minutes; the Chocolatey task uses
  a 7200s timeout and triggers a `win_reboot` when the installer asks for one.
* Fresh golden images often have a **pending reboot** after Chocolatey pulls in
  .NET / servicing packages. SQL Setup aborts with "A system reboot is
  pending" in that case — the role reboots when `ansible_reboot_pending` is
  set before invoking Setup.
* SQL Setup under WinRM hits DPAPI `CryptographicException: Access is
  denied` (exit `0x84b10001`) because the remoting session has no loaded
  user profile. The role installs via a one-shot Scheduled Task logged on
  as `Administrator` (requires `ansible_password` in inventory). Installing
  over SSH also works around the same issue.
* For a dedicated DB volume, attach a second disk (see
  `examples/edge-gitops-vms-values.yaml` `additionalDisks.sqldata`) and set
  `mssql_win_data_drive_letter` (example inventory uses `D`). The role formats
  the first non-system RAW disk, passes `/INSTALLSQLDATADIR` into Setup, and
  creates `parasol` on that volume.
* The JDBC URL used by the app sets `encrypt=false;trustServerCertificate=true`
  because there is no CA-signed cert on the Windows host — same pattern as the
  OpenShift deployment in the app repo.

## Building & publishing the collection

```bash
ansible-galaxy collection build --output-path dist/
```

Push the resulting `dist/rhvp-parasol_mssql_demo-*.tar.gz` to
[galaxy.ansible.com](https://galaxy.ansible.com) (namespace `rhvp`), or to
your Automation Hub instance.
