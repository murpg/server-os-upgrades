# Ubuntu LTS Upgrade Guide for GitLab CE Servers

This guide covers upgrading an Ubuntu LTS server running GitLab CE through sequential LTS hops (e.g., 20.04 -> 22.04 -> 24.04). Ubuntu does not support skipping LTS versions -- each hop must be done sequentially.

## Variables

Replace the following placeholders throughout this guide:

| Placeholder | Description |
|---|---|
| `<username>` | Non-root user account on the server |
| `<server-ip>` | IP address of the GitLab server |
| `<local-backup-path>` | Local directory to store backups |
| `<current-version>` | Current GitLab CE version (e.g., 18.8.0) |
| `<intermediate-version>` | Required intermediate GitLab version for upgrade path |
| `<target-version>` | Target GitLab CE version (e.g., 18.9.1) |

## Pre-Upgrade

### Take a VM Snapshot

Before anything else, take a VMware snapshot (or AMI snapshot if on AWS) as a rollback point.

### Check Current Versions

```bash
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
lsb_release -a
```

### Create GitLab Backups

```bash
gitlab-backup create
tar -czf /root/gitlab-config-backup.tar.gz /etc/gitlab
```

### Copy Backups to Non-Root User

```bash
cp /var/opt/gitlab/backups/*_gitlab_backup.tar /home/<username>/
cp /root/gitlab-config-backup.tar.gz /home/<username>/
chown <username>:<username> /home/<username>/*_gitlab_backup.tar /home/<username>/gitlab-config-backup.tar.gz
```

### Download Backups to Local Machine

Run from your local machine:

```bash
scp <username>@<server-ip>:/home/<username>/*_gitlab_backup.tar <local-backup-path>/
scp <username>@<server-ip>:/home/<username>/gitlab-config-backup.tar.gz <local-backup-path>/
```

## OS Upgrade (Repeat Per Hop)

### Hold GitLab Package

Prevent the OS upgrade from pulling a GitLab version that skips required intermediate versions:

```bash
apt-mark hold gitlab-ce
```

### Update Current Packages

```bash
apt-get update && apt-get dist-upgrade -y
reboot
```

### Run the Release Upgrade

```bash
do-release-upgrade
```

- Select automatic restarts when prompted.
- The server will reboot when finished.

### Verify OS and GitLab Status

```bash
lsb_release -a
gitlab-ctl status
```

### Unhold GitLab Package

```bash
apt-mark unhold gitlab-ce
```

### Repeat

If another LTS hop is needed (e.g., 22.04 -> 24.04), go back to the "Hold GitLab Package" step and repeat the process.

```bash
apt-mark hold gitlab-ce
```

## Post-Upgrade

### Fix GitLab GPG Key

The GitLab package signing key may be expired or missing from the new OS keyring:

```bash
curl -fsSL https://packages.gitlab.com/gpg.key | gpg --dearmor -o /etc/apt/trusted.gpg.d/gitlab-archive-keyring.gpg
apt-get update
```

If the repo config uses a `signed-by=` directive pointing to a different path, place the key there instead:

```bash
cat /etc/apt/sources.list.d/gitlab_gitlab-ce.list
```

Match the key file location to whatever `signed-by=` specifies.

### Update GitLab Repo Codename

Verify the repo references the correct Ubuntu codename for your new OS version:

```bash
cat /etc/apt/sources.list.d/gitlab_gitlab-ce.list
```

Update the codename if needed (e.g., `focal` -> `noble` for 24.04).

### Upgrade GitLab CE

GitLab requires stepping through versions -- you cannot skip minor versions. Check the required upgrade path at https://docs.gitlab.com/ee/update/index.html#upgrade-paths.

```bash
apt-get install gitlab-ce=<intermediate-version>-ce.0
apt-get install gitlab-ce=<target-version>-ce.0
```

### Reconfigure and Verify

```bash
gitlab-ctl reconfigure
gitlab-rake gitlab:check SANITIZE=true
```

## Cleanup

### Remove Backup Copies from Server

```bash
rm /home/<username>/*_gitlab_backup.tar
rm /home/<username>/gitlab-config-backup.tar.gz
rm /root/gitlab-config-backup.tar.gz
```

### Remove Old GitLab Backups

Keep only the most recent backup:

```bash
ls -lh /var/opt/gitlab/backups/
cd /var/opt/gitlab/backups/
ls -t *_gitlab_backup.tar | tail -n +2 | xargs rm -f
```

### Clean Up Apt Cache

```bash
apt-get autoremove -y
apt-get autoclean
```

## Troubleshooting

### GPG Key Errors

If `signed-by=` method fails, try the legacy approach:

```bash
curl -fsSL https://packages.gitlab.com/gpg.key | apt-key add -
```

### GitLab Upgrade Blocked

If GitLab refuses to install due to version skip requirements, check the error message for the required intermediate version and install that first.

### Database Migration Errors

If `gitlab-ctl reconfigure` fails with a database constraint error, connect to the database and fix the offending data:

```bash
gitlab-psql -d gitlabhq_production
```

Then rerun `gitlab-ctl reconfigure`.

### SSH Host Key Warning After Rebuild

If the server was rebuilt and the host key changed:

```bash
ssh-keygen -R <server-ip>
```

## Notes

- Ubuntu does not support downgrading (e.g., 24.10 -> 24.04). If you need an older LTS, rebuild from a fresh image and restore from backup.
- Non-LTS releases (e.g., 24.10, 25.04) only get 9 months of support. Always target LTS releases for servers.
- Mattermost data is NOT included in GitLab backups. Back it up separately if needed.
- If `do-release-upgrade` says no new release is found, check that `/etc/update-manager/release-upgrades` has `Prompt=lts`.
