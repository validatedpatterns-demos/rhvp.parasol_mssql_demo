# rhvp.parasol_mssql_demo

Ansible collection that deploys the [Parasol Insurance
demo](https://github.com/blues-man/parasol-insurance-mssql) across a two-tier
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
`quay.io/validatedpatterns-demos/parasol-mssql`.

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

1. Copy `host.yml` (example inventory) and point `ansible_host` entries at
   your VMs. Protect credentials with `ansible-vault` in a real environment.
2. The `parasol_rhel` hosts need `quadlet_app_mssql_host` set to the Windows SQL
   Server host DNS name/IP.
3. Run the site playbook (requires WinRM already reachable on the Windows
   host, and SSH + an ansible user with `become` on the RHEL host):

```bash
ansible-playbook -i host.yml playbooks/site.yml
```

The playbook runs the Windows deployment first, then the RHEL quadlet
deployment, and finally waits for `/q/health/ready` on the RHEL host to
confirm the app is up.

### Role defaults / main variables

| Role | Variable | Default | Purpose |
|------|----------|---------|---------|
| `quadlet_app` | `quadlet_app_image` | `quay.io/validatedpatterns-demos/parasol-mssql:latest` | Container image to run |
| `quadlet_app` | `quadlet_app_service_name` | `parasol-app` | systemd unit name |
| `quadlet_app` | `quadlet_app_http_port` | `8080` | Host port for the app |
| `quadlet_app` | `quadlet_app_mssql_host` | `localhost` | **Must** be overridden |
| `quadlet_app` | `quadlet_app_mssql_user`/`quadlet_app_mssql_password`/`parasol_mssql_win_database` | `sa` / `P@rasol-2026-Sql` / `parasol` | Datasource env vars |
| `quadlet_app` | `quadlet_app_registry_username`/`quadlet_app_registry_password` | empty | Podman login, only if the image repo is private |
| `mssql_win` | `mssql_win_chocolatey_package` | `sql-server-2022` | Chocolatey package |
| `mssql_win` | `mssql_win_sa_password` | `P@rasol-2026-Sql` | `sa` password (Mixed mode is enabled by the role) |
| `mssql_win` | `mssql_win_tcp_port` | `1433` | TCP/IP port opened on the firewall |
| `mssql_win` | `mssql_win_database` | `parasol` | Database created for the app |

## Gotchas

* Chocolatey's `sql-server-2022` package ships a `ConfigurationFile.ini` with
  **TCP/IP disabled** and **Windows-only auth**. The role passes
  `/SECURITYMODE:"SQL"` + `/SAPWD:"..."` to the SQL Setup invocation and then
  flips the `SuperSocketNetLib\Tcp` registry keys, restarting `MSSQLSERVER`
  afterwards. Do not "simplify" these steps away.
* SQL Server install can take well over 30 minutes; the Chocolatey task uses
  a 7200s timeout and triggers a `win_reboot` when the installer asks for one.
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
