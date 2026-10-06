# Ansible – Daily DevOps Playbooks

Playbooks for routine work on Amazon Linux (2 / 2023) EC2 servers.

## Layout

```
ansible.cfg              # defaults: inventory, ec2-user, pipelining, timing output
inventory.ini            # hosts (group: servers)
group_vars/all.yml       # shared vars: timezone, common packages, thresholds
vars/vault.yml.example   # template for secrets (copy -> vault.yml, then encrypt)
templates/               # Jinja2 templates (nginx site, index page)
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
| `manage-service.yml` | Starts, stops, restarts, reloads or checks the status of any service | `... -e service=nginx -e action=restarted` |
| `manage-users.yml` | Creates or removes users, with SSH key and sudo | edit the `users` list |
| `nginx-service.yml` | Installs nginx and controls it with tags. `stop` and `restart` run only when asked for | `... -t restart` / `-t stop` / `-t status` |
| `reboot.yml` | Rolling reboot, one host at a time | `... -l remote1` |

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

## Requirements

- ansible-core 2.15+
- Collections: `ansible-galaxy collection install community.general ansible.posix`
