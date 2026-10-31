# HMP511 Ansible

Deploy a fresh WordPress site on each Ubuntu Server 24.04 or 26.04 host using Docker Compose.
Sites are served over HTTP at each host's inventory IP. No domain, certificate or
migration from a native Apache/MariaDB installation is required.

## Architecture

- `db`: MariaDB on an internal Docker network, with no published database port.
- `wordpress`: WordPress with PHP and Apache, publishing TCP port 80 by default.
- `cli`: temporary WP-CLI containers for installation and URL configuration.
- Named volumes preserve the database and WordPress files across container replacement.
- Docker and the application containers start automatically after a host reboot.

## Requirements

- Ubuntu Linux control node with Ansible Core 2.21.x.
- Fresh Ubuntu Server 24.04 or 26.04 targets, amd64 or arm64, with Python 3 and passwordless sudo.
- SSH access using the user and key configured in `hosts`.
- Outbound access to Ubuntu repositories, download.docker.com and Docker Hub.
- Inbound TCP port 80 in the AWS Security Group and any host/network firewall.
  Use `wordpress_http_port` instead if the port is changed.
- No other application using the selected host port.
- The Vault password for the two existing encrypted WordPress variables.

The playbook does not change AWS Security Groups or firewall rules. Docker-published
ports may bypass UFW rules; control external access through the AWS Security Group.

## Project layout

```text
site.yml                              # packages -> docker -> wordpress
hosts                                 # webservers inventory
group_vars/webservers.yml             # settings and Vault-encrypted passwords
requirements.yml                      # community.docker collection
roles/packages/tasks/main.yml         # base tools
roles/docker/tasks/main.yml           # Docker CE and Compose plugin
roles/wordpress/tasks/main.yml         # settings and Compose template validation
roles/wordpress/tasks/deploy.yml       # remote deployment and verification
roles/wordpress/templates/compose.yml.j2
```

Docker is installed from its official APT repository. The role refuses conflicting
distribution packages rather than removing them. It also refuses a running native
Apache/nginx on the default HTTP port. Existing host services and data are not migrated
or deleted; resolve conflicts separately or select an unused port.

## Configuration

Edit non-secret settings in `group_vars/webservers.yml`:

| Setting | Default | Purpose |
| --- | --- | --- |
| `wordpress_project_name` | `wordpress` | Compose project and volume prefix |
| `wordpress_project_path` | `/opt/wordpress` | Remote Compose directory |
| `wordpress_public_host` | inventory IP | Public address used by WordPress |
| `wordpress_http_port` | `80` | Published HTTP port |
| `wordpress_image` | `wordpress:7.1.4-php8.3-apache` | Initial WordPress image |
| `wordpress_cli_image` | `wordpress:cli-2.12.0-php8.3` | WP-CLI image |
| `wordpress_db_image` | `mariadb:11.4` | MariaDB image |
| `wordpress_admin_user` | `wordpressadmin` | Initial administrator login |
| `wordpress_admin_email` | `admin@example.com` | Replace before deploying |

The administrator, email and site title are created during the first installation.
Subsequent runs preserve the administrator's password and account settings. Change
them through WordPress or a deliberate WP-CLI operation. The role synchronizes
`home` and `siteurl` when the configured IP or port changes; it does not rewrite
URLs embedded in existing content.

Each inventory host receives its own independent database and volumes. Changing
the Compose project name or remote project path after deployment requires a planned
migration; it is not a way to rename existing volumes.

## Secrets

The existing `wordpress_db_password` and `wordpress_admin_password` values remain
encrypted with Ansible Vault. Keep the Vault password and SSH keys outside version
control. To create replacement encrypted values, run separately on the control node:

```bash
ansible-vault encrypt_string --ask-vault-pass --stdin-name wordpress_db_password
ansible-vault encrypt_string --ask-vault-pass --stdin-name wordpress_admin_password
```

Enter the secret on standard input and finish with Ctrl-D. Never put plaintext
passwords in command arguments, Compose environment values or committed files.

On the target, the database password is decrypted to a Compose secret file under
`/opt/wordpress/secrets`. Those host directories are root-only (`0700`). Secret
files are readable inside only the containers that mount them. Local Compose secrets
are file mounts, not an encrypted secret store. Back up and protect the host files.
A separate MariaDB root password is generated once on the target and retained there;
it is not printed, returned to the controller or stored in this repository.

Password-bearing tasks use `no_log`; the initial administrator password is passed
to WP-CLI through standard input and is not persisted in an administrator secret file.
The public WordPress web installer is blocked by Apache; setup uses WP-CLI.

MariaDB initialization variables apply only to an empty data volume. The role refuses
a changed database password, database name, database user or project name for an
existing project before overwriting its configuration. Password rotation requires an
explicit database operation and coordinated secret-file update, not just a Vault edit.

## Deploy from the Ubuntu control node

Install the required collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

Check syntax, then run a limited check:

```bash
ansible-playbook site.yml --ask-vault-pass --syntax-check
ansible-playbook site.yml --ask-vault-pass --check --diff --limit server1
```

Check mode contacts the target, checks base packages, validates settings and decrypts
the Vault inputs without displaying them, and renders/parses the Compose template.
Docker installation and all application deployment tasks are skipped in check mode.
This is a configuration preflight, not a simulation of container creation or proof
that image pulls, database startup or WordPress installation will succeed.

Deploy one server:

```bash
ansible-playbook site.yml --ask-vault-pass --limit server1
```

Use `--limit server2` for the second server, or omit `--limit` to deploy both.
The role waits for MariaDB and Apache readiness, creates the WordPress administrator
only if needed, synchronizes site URLs and checks the login page over local HTTP.
The final output shows the site and administration URLs. External reachability still
depends on the inventory IP, routing and AWS Security Group.

## Inspect on a target host

```bash
sudo docker compose --project-directory /opt/wordpress -p wordpress ps
sudo docker compose --project-directory /opt/wordpress -p wordpress logs --tail=100 wordpress
```

## Persistence and updates

Normal playbook runs use `pull: missing` and preserve existing volumes; they do not
force image upgrades or delete volumes. WordPress files, uploads, plugins and themes
live in `wordpress_wordpress_data`; MariaDB data lives in `wordpress_db_data`.

Back up both volumes and the protected project configuration/secrets before upgrades.
The WordPress image initializes the WordPress volume on first startup. Changing its
image tag later does not replace WordPress core files in an existing populated volume.
Manage core/plugin/theme updates through WordPress or WP-CLI. Review PHP/image and
MariaDB upgrades separately, including database compatibility and rollback.

Do not run `docker compose down --volumes` unless you intend to delete site data.

## Official references

- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [WordPress official image and WP-CLI UID requirements](https://hub.docker.com/_/wordpress)
- [Ansible Compose module](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
- [Ansible Compose run module](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_run_module.html)
