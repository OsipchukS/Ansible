# AGENTS.md

Project instructions for AI coding agents working in this repository.

## Project

Ansible playbooks for managing Linux hosts on AWS.

- Control node: Ubuntu Linux machine with `ansible-core` installed system-wide (tested with 2.21.x).
- Inventory: `./hosts` (group `webservers`).
- Entry playbook: `site.yml`.
- Roles: `roles/<name>/`.
- Group variables: `group_vars/<group>.yml`.
- WordPress role: `roles/wordpress/` deploys Apache, MariaDB, PHP, WordPress, and HTTPS to Ubuntu Server 24.04.
- Collection dependencies: `requirements.yml` (currently `ansible.mysql`). Install them on the control node with `ansible-galaxy collection install -r requirements.yml`.
- SSH key path (`ansible_ssh_private_key_file`) is defined in `hosts` and must exist on the control node at `/etc/ansible/keys/`. The key is never committed (`/keys/*` is in `.gitignore`).

## Language

- All code comments, documentation, commit messages, and PR descriptions are written in English.
- Conversation with the user is in Ukrainian.

## Rules

1. Before any action that changes files, runs git commands, or runs `ansible-playbook` against hosts, show exactly what will be done and wait for explicit user approval.
2. Never push to `master`. Work in a feature branch and open a Pull Request.
3. Never commit secrets (SSH keys, `.pem`, passwords, tokens).
4. Do not change `hosts` or inventory variables without explicit approval.
5. Store `wordpress_db_password` and `wordpress_admin_password` as Ansible Vault encrypted values in `group_vars/webservers.yml`. Keep the Vault password outside the repository. Never put plaintext credentials in role tasks, templates, documentation, or command-line arguments.
6. When `wordpress_https_enabled` is `true`, the WordPress role requires a configured domain and email addresses, DNS pointing to the target host, and network access to ports 80 and 443 for Let's Encrypt. When it is `false`, the site is served over plain HTTP and no certificate is requested.
7. Interact with GitHub through the `github` MCP server (configured globally in OpenCode, authenticated with a fine-grained PAT from the `GITHUB_MCP_PAT` environment variable). Do not use the `gh` CLI. The token file `.secret` is gitignored and must never be committed or printed.

## Commands

Run from the repository root on the control node:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff
ansible-playbook site.yml
ansible webservers -m ping
```

Order: syntax check, then dry run, then real run (only after approval).
The syntax check is local. Ping, dry-run, and real runs contact managed hosts and require approval before execution; real deployment requires explicit approval after reviewing the dry-run results.

## Conventions

See the skills in `.opencode/skills/`:

- `ansible-playbook-conventions`: writing playbooks and roles.
- `ansible-run-playbook`: running and verifying playbooks.
- `git-pr-workflow`: branches, commits, and pull requests (PRs are created via the GitHub MCP server).

When editing the WordPress role, preserve idempotency, use fully qualified Ansible module names, manage MariaDB with `ansible.mysql`, and pass WP-CLI passwords through standard input with secret-bearing tasks marked `no_log: true`.
