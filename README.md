# Ansible – Daily DevOps Playbooks

Playbooks for routine work on Amazon Linux (2 / 2023) EC2 servers.

## Layout

```
ansible.cfg              # defaults: inventory, ec2-user, pipelining, timing output
inventory.ini            # hosts (group: servers)
collections/requirements.yml  # required Ansible collections
group_vars/all.yml       # shared vars: timezone, common packages, thresholds
vars/vault.yml.example   # template for secrets (copy -> vault.yml, then encrypt)
templates/               # Jinja2 templates (nginx site, index page, logrotate)
playbooks/basic/         # everyday one-shot tasks
playbooks/medium/        # multi-step ops with handlers, templates, vault, reports
```

Run everything from the repo root so `ansible.cfg` gets picked up.

## Basic playbooks

| Playbook | What it does | Example |
|---|---|---|
| `ping.yml` | SSH and sudo check | `ansible-playbook playbooks/basic/ping.yml` |
| `system-info.yml` | OS, uptime, CPU, memory, disk, top processes | `ansible-playbook playbooks/basic/system-info.yml` |
| `update-packages.yml` | Patches packages in batches, reboots only if needed and allowed | `... -e reboot_if_needed=true` / `-e security_only=true` |
| `install-packages.yml` | Installs the common toolset or a custom list | `... -e '{"packages":["nc"]}'` |
| `manage-service.yml` | Starts, stops, restarts, reloads or checks the status of any service | `... -e service=nginx -e service_action=restarted` |
| `manage-users.yml` | Creates or removes users, with SSH key and sudo | edit the `users` list |
| `nginx-service.yml` | Installs nginx and controls it with tags. `stop` and `restart` run only when asked for | `... -t restart` / `-t stop` / `-t status` |
| `reboot.yml` | Rolling reboot, one host at a time | `... -l remote1` |
| `fetch-logs.yml` | Pulls log files and a journal export into `logs/<host>/` | `... -e journal_since="2 hours ago"` |
| `set-hostname.yml` | Sets the hostname and stops cloud-init from resetting it | `... -l remote1 -e new_hostname=web-01` |
| `manage-cron.yml` | Installs cronie and adds or removes cron jobs | edit the `cron_jobs` list |
| `copy-files.yml` | Pushes files or directories to servers, backing up anything it overwrites | `... -e src=./app.env -e dest=/opt/app/.env` |
| `run-command.yml` | Runs one shell command everywhere and saves output to `reports/command-<date>.txt` | `... -e cmd="df -h /"` |

## Medium playbooks

| Playbook | What it does | Example |
|---|---|---|
| `health-check.yml` | Checks disk, memory, load, services and HTTP endpoints. Fails unhealthy hosts and writes `reports/health-<date>.txt` | `... -e '{"required_services":["sshd","nginx"]}'` |
| `disk-cleanup.yml` | Cleans package cache, journal, old logs, old `/tmp` files and optionally Docker data | `... -e docker_prune=true` |
| `harden-ssh.yml` | Disables root login and password auth, validates config before reload | `... --check --diff` first |
| `install-docker.yml` | Installs Docker and the compose plugin, adds users to the docker group | |
| `nginx-site.yml` | Installs nginx and serves a static site or reverse proxy from a template | `... -e proxy_pass=http://127.0.0.1:3000` |
| `deploy-app-from-git.yml` | Deploys a private repo using vault secrets, backs up first, optional service restart | see below |
| `backup.yml` | Archives paths, fetches them to `backups/`, rotates old copies, optional daily cron | `... -e schedule_cron=true` |
| `time-sync.yml` | Sets timezone and configures chrony with the AWS time source | `... -e timezone=UTC` |
| `swap-file.yml` | Creates or removes a swap file, persists it in fstab, sets swappiness | `... -e swap_size_mb=4096` / `-e swap_state=absent` |
| `security-audit.yml` | Read-only audit: security patches, ports, uid 0 accounts, empty passwords, sshd, failed logins. Writes `reports/security-<date>.txt` | `... -e fail_on_findings=true` |
| `logrotate-app.yml` | Deploys validated `/etc/logrotate.d/` configs for app logs | edit the `logrotate_apps` list |
| `sysctl-tuning.yml` | Applies kernel network/file limits tuning, persisted in `/etc/sysctl.d/` | `... --check --diff` / `-e sysctl_state=absent` |
| `node-exporter.yml` | Installs Prometheus node_exporter (checksum-verified) as a systemd service on port 9100 | `... -e node_exporter_version=1.8.2` |
| `cloudwatch-agent.yml` | Installs the CloudWatch agent for memory, disk and swap metrics plus log shipping. Needs the `CloudWatchAgentServerPolicy` instance role | `... -e cw_namespace=MyApp` |
| `lamp-stack.yml` | Installs Apache, MariaDB and PHP (php-fpm) on Amazon Linux 2023, removes MariaDB test data, optionally creates an app DB and user from the vault | `... --ask-vault-pass -e lamp_db_name=shop -e lamp_db_user=shop` |
| `java-stack.yml` | Installs Amazon Corretto 17/21 and Tomcat 10.1 (checksum-verified, systemd, versioned install with symlink), optionally deploys a WAR | `... -e java_version=21 -e war_src=./app.war -e war_name=app` |
| `java-app-service.yml` | Runs a Spring Boot or any executable jar as a systemd service, keeps a rollback copy of the previous jar | `... -e app_name=myapp -e jar_src=./target/myapp.jar` |
| `lemp-stack.yml` | Installs nginx, MariaDB and PHP (php-fpm over a unix socket) on Amazon Linux 2023, removes MariaDB test data | `... -e server_name=example.com` |
| `postgresql.yml` | Installs PostgreSQL 15, initialises the cluster (localhost only), optionally creates an app DB and role from the vault | `... --ask-vault-pass -e pg_db_name=shop -e pg_db_user=shop` |
| `node-app-service.yml` | Installs Node.js and runs an app from a public git repo as a systemd service, `npm ci` on change | `... -e app_name=api -e repo_url=https://github.com/org/api.git` |
| `letsencrypt-nginx.yml` | Gets Let's Encrypt certificates for nginx sites (certbot in a venv), HTTPS redirect, twice-daily renewal timer | `... -e certbot_email=you@example.com -e '{"certbot_domains":["example.com"]}'` |
| `redis.yml` | Installs Redis 6 on Amazon Linux 2023, localhost only, AOF persistence, optional memory limit and vault password | `... --ask-vault-pass -e redis_require_auth=true` |
| `db-backup.yml` | Dumps a MariaDB or PostgreSQL database to a gzip file, verifies it, rotates old dumps, optional fetch to `backups/` | `... -e db_engine=postgresql -e db_name=shop` |

## Secrets (Ansible Vault)

```bash
cp vars/vault.yml.example vars/vault.yml
vim vars/vault.yml                 # add your git username/token
ansible-vault encrypt vars/vault.yml
ansible-playbook playbooks/medium/deploy-app-from-git.yml --ask-vault-pass \
  -e repo_url=github.com/org/repo -e git_branch=main
```

Never put tokens or passwords in playbooks as plain text.

## Handy flags

```bash
-l remote1              # limit to one host or group
--check --diff          # dry run and show what would change
-t <tag>                # run only tagged tasks
-v / -vvv               # more output
--list-tasks            # show what a playbook will do
--syntax-check          # validate YAML/structure
```

## Ad-hoc one-liners

```bash
ansible servers -m ping
ansible servers -a "uptime"
ansible servers -a "df -h /"
ansible servers -b -m package -a "name=htop state=present"
ansible servers -b -m systemd_service -a "name=nginx state=restarted"
ansible servers -m copy -a "src=./file.txt dest=/tmp/file.txt"
ansible servers -m fetch -a "src=/var/log/messages dest=./logs/"
ansible servers -m setup -a "filter=ansible_distribution*"
```

## Linting

```bash
pipx install ansible-lint && pipx install yamllint
yamllint .
ansible-lint                # uses .ansible-lint (production profile)
```

The same checks run in GitHub Actions on every push and pull request.

## Requirements

- ansible-core 2.15+
- Collections: `ansible-galaxy collection install -r collections/requirements.yml`
