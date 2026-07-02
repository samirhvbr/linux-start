# Blue3 Start Script

Initial script for manual provisioning of Debian servers.

The goal of this project is to standardize the first configurations of a Debian Linux server with an interactive menu, keeping a log, backups and quick-adjustment points at the start of the script.

## 🔄 Before you start: `git pull`

**ALWAYS** check for remote updates before writing or changing anything in this repository:

```bash
git pull          # already pre-authorized (allow)
```

Working on top of an outdated base causes conflicts. Pull first, always. To just inspect beforehand: `git fetch && git status`.

## Files

- `start.sh`: main initial-configuration script
- `README.md`: usage and maintenance documentation
- `VERSION`: local project version
- `.env.example`: local configuration template to generate `.env_start`
- `templates/`: external blocks used by the script

## What the script does

The current menu offers these steps:

1. Configure hostname, `/etc/hosts` and the network file
2. Run `apt update`, `upgrade`, `clean` and `autoremove`
3. Install basic administration applications
4. Apply the Proxmox VM profile
5. Apply the BLUE3 login banner
6. Apply the BLUE3 `.bashrc` for the root user
7. Configure SSH via `sshd_config.d`
8. Install and configure NTP synchronization
9. Update the APT mirror
A. Schedule regeneration of the SSH host keys on the next boot
U. Check and apply the project's self-update via GitHub
Z. Install and configure the Zabbix Agent

## Improvements applied in this version

The script was reorganized to fix problems from the previous model:

- uses `set -Eeuo pipefail`
- validates root before starting
- validates that the system is a supported Debian
- records the run in a real log with a unique timestamp
- creates backups before changing critical files
- uses `sshd -t` before restarting SSH
- removes obsolete OpenSSH directives
- avoids aggressive `sed` on the main SSH file
- rewrites critical configurations with a readable heredoc
- keeps sensitive variables concentrated at the top of the script
- moves large blocks to external templates
- supports a local `.env_start` file for environment customization
- uses a `VERSION` file for local version control
- prepares the project's self-update via GitHub using a repository tarball

## Project version control

The project now has a dedicated file:

```text
VERSION
```

This file is the source of the project's local version. `start.sh` reads this value at startup and compares it with the version published on GitHub when the self-update routine runs.

This model is better than keeping the version only inside the script because:

- it makes maintenance easier
- it allows a simple remote comparison
- it avoids having to parse the entire body of the shell script

## Self-update via GitHub

The script now supports self-updating the project using the GitHub repository configured in `.env_start`.

New variables:

```bash
UPDATE_REPO_OWNER=samirhvbr
UPDATE_REPO_NAME=Linux-Start
UPDATE_REPO_BRANCH=master
```

Update flow:

1. queries the remote `VERSION` at `raw.githubusercontent.com`
2. compares it with the local version using version ordering
3. downloads the tarball of the configured branch if there is a newer version
4. extracts the project to a temporary folder
5. backs up the current project
6. overwrites the local project files
7. restarts the updated script

Self-update backups are kept inside the backup directory of the current run.

## Current status of the remote repository

The repository was checked:

- `https://github.com/samirhvbr/Linux-Start`

At the time of the check, the `master` branch still had the old version of `start.sh` and without a published `VERSION`. So:

- the new mechanism is already ready locally
- it will actually work as soon as this new structure is pushed to GitHub

In other words: the best practice was implemented in the local project, but the remote repository still needs to receive these new files for the self-update to find a valid remote version.

## Template structure

The larger maintenance blocks were separated into files inside `templates/`:

- `templates/banner/10-uname.tpl`
- `templates/banner/20-blue3.tpl`
- `templates/bash/root.bashrc.tpl`
- `templates/ssh/blue3-hardening.conf.tpl`
- `templates/ssh/blue3-root-ipath.conf.tpl`
- `templates/ssh/rhosts.conf.tpl`

With this, changes to the banner, bashrc and SSH no longer need to be made directly in the body of `start.sh`.

## Using .env_start

Yes, using an environment file here makes sense as a good practice, as long as it is treated as local configuration and not as a versioned file with sensitive data.

The script automatically looks for:

```bash
.env_start
```

in the project's own directory. If the file does not exist, it uses the default values built into the script.

The recommended flow is:

```bash
cp .env.example .env_start
```

After that, adjust the values for your local environment.

## Quick-adjustment variables

These variables can go in `.env_start` to make adaptation to other environments easier:

```bash
REQUIRED_DEBIAN_MAJOR="${REQUIRED_DEBIAN_MAJOR:-11}"
DEFAULT_DOMAIN="${DEFAULT_DOMAIN:-b3.local}"
DEFAULT_SSH_PORT="${DEFAULT_SSH_PORT:-22}"
DEFAULT_LOGIN_GRACE="${DEFAULT_LOGIN_GRACE:-30}"
DEFAULT_ROOT_IPS="${DEFAULT_ROOT_IPS:-100.64.66.0/24,170.233.230.254,170.233.230.222}"
DEFAULT_ZABBIX_SERVER="${DEFAULT_ZABBIX_SERVER:-100.64.66.8}"
UPDATE_REPO_OWNER="${UPDATE_REPO_OWNER:-samirhvbr}"
UPDATE_REPO_NAME="${UPDATE_REPO_NAME:-Linux-Start}"
UPDATE_REPO_BRANCH="${UPDATE_REPO_BRANCH:-master}"
ZABBIX_RELEASE_URL="${ZABBIX_RELEASE_URL:-}"
```

These settings are loaded before the script's main validations.

## Logs and backups

Each run creates:

- a log at `/root/blue3_start_YYYYMMDDHHMMSS.log`
- a backup at `/root/blue3_start_YYYYMMDDHHMMSS/`

Changed files, such as SSH, network, hosts, MOTD and `.bashrc`, are backed up before any overwrite.

## Requirements

The script was prepared for:

- Debian 11 or higher
- running as `root`
- internet access for the package and Zabbix steps

Commands expected on the host:

```bash
apt awk cp cut date getent grep hostname hostnamectl ip mkdir mv ping sed sshd systemctl tee wget
```

## How to use

To download the project directly on the server via Git:

```bash
apt update && apt install -y git
git clone -b master https://github.com/samirhvbr/Linux-Start.git
cd Linux-Start
```

Then run as root:

```bash
bash start.sh
```

You can also override variables at call time:

```bash
DEFAULT_SSH_PORT=2222 DEFAULT_DOMAIN=empresa.local bash start.sh
```

If you prefer a dedicated file at another path, you can also use:

```bash
BLUE3_ENV_FILE=/caminho/arquivo.env_start bash start.sh
```

## Behavior of the main functions

### Config server

- detects hostname, domain, default interface and current IP
- allows updating the hostname
- can rewrite `/etc/hosts`
- can rewrite `/etc/network/interfaces`
- does not automatically restart the network; leaves the final review to the operator

### SSH

- creates files in `/etc/ssh/sshd_config.d/`
- uses templates from the `templates/ssh/` directory
- sets `PermitRootLogin no` globally
- allows root only for the IPs defined in `Match Address`
- validates the configuration with `sshd -t`
- only restarts the service if validation passes

### Proxmox VM Profile

- updates the APT indexes before installation
- installs `qemu-guest-agent`, `rsync`, `nano`, `htop`, `curl`, `wget` and `net-tools`
- enables `qemu-guest-agent` immediately
- offers optional installation of `cloud-init` and `cloud-initramfs-growroot`
- enables `fstrim.timer` when available
- ensures `/root/.ssh` with permission `700`
- offers optional cleanup of cache, old journals and rotated logs
- offers optional preparation for template/clone by clearing `machine-id` and the `cloud-init` state
- within the template preparation, offers optional removal of the SSH host keys and can schedule automatic regeneration on the next boot

### SSH host key regeneration on the next boot

- creates a `systemd` `oneshot` service to run only once on the next boot
- uses `ssh-keygen -A` to recreate only the missing host keys
- removes its own script and its own unit after running
- is the most suitable option when the VM will be turned into a template or when the clone has not yet started with a definitive identity

### Banner and bashrc

- the BLUE3 banner is applied from the templates in `templates/banner/`
- root's `.bashrc` is applied from `templates/bash/root.bashrc.tpl`
- visual adjustments and aliases now live outside the main script

### Zabbix

- tries to install `zabbix-agent` from the current repositories
- if the package does not exist, uses `ZABBIX_RELEASE_URL` when defined
- creates a configuration in `zabbix_agentd.d`
- adds user parameters for fail2ban
- creates a systemd override for the fail2ban socket permission when necessary

### Project update

- queries the remote `VERSION` on GitHub
- downloads the tarball of the configured branch
- backs up the project before replacement
- restarts the updated script at the end

## Important notes

- the script is interactive; it was not converted to a fully non-interactive mode
- `.env_start` is recommended for local defaults, but does not replace the manual review of network and SSH
- self-update depends on the remote repository containing `VERSION`, `start.sh` and the `templates/` folder
- the network step can drop remote access if applied without review
- the Proxmox profile step can clear `machine-id` if the operator confirms preparation for a template
- the scheduled regeneration of SSH host keys should only be used when the current SSH identity can be safely discarded
- the SSH step changes the root access policy and port; it should be tested carefully
- the script assumes the use of `ifupdown` in `/etc/network/interfaces`
- for servers with `cloud-init`, `NetworkManager` or `systemd-networkd`, the network step should be reviewed before use

## Recommended next steps

Natural improvements for future phases:

1. create a non-interactive mode via variables or a `.env_start` file
2. separate large blocks into template files
3. add validation of IP, CIDR and port before writing configurations
4. include automated syntax tests in the project
5. add an option to apply configurations in batch
