# Upgrading Portainer CE on Docker (Ubuntu 24.04)

Last verified: March 2026
Current LTS: **2.39.0** (Feb 26, 2026) | Current STS: **2.40** (Mar 2026)

---

## Pre-Flight

Check your running version and container details before touching anything.

```bash
# Confirm current version
docker inspect portainer --format '{{ .Config.Image }}'

# Note the volume and port mappings you are using
docker inspect portainer --format '{{ json .Mounts }}'
docker inspect portainer --format '{{ json .HostConfig.PortBindings }}'
```

Record the output so you can verify nothing changed after the upgrade.  

Get your run command

```
pipx install runlike
runlike portainer
```

That keeps it in its own venv automatically without touching the system Python.  

---

## Step 1 -- Back Up the Data Volume

Portainer stores all configuration, users, stacks, and environment definitions in
its persistent volume (typically `portainer_data`). This is the only thing you
need to protect.

```bash
# Stop the container so the database is consistent
docker stop portainer

# Create a timestamped tarball of the volume
docker run --rm \
  -v portainer_data:/data \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/portainer-data-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .
```

Verify the backup file exists and has a reasonable size before continuing.

```bash
ls -lh portainer-data-backup-*.tar.gz
```

---

## Step 2 -- Remove the Old Container

This only removes the container, **not** the volume. Your data is safe.

```bash
docker rm portainer
```

---

## Step 3 -- Pull the New Image

For the latest LTS release (recommended for production):

```bash
docker pull portainer/portainer-ce:2.39.0
```

Or to always track the latest tag (includes STS releases):

```bash
docker pull portainer/portainer-ce:latest
```

> If you are on a specific LTS track and want to stay there, pin the tag
> explicitly rather than using `latest`.

---

## Step 4 -- Start the New Container

Re-create the container with the same flags you had before. The standard
invocation is:

```bash
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Adjust the image tag to match what you pulled in Step 3.

**Port notes:**

- **9443** -- HTTPS UI (default since CE 2.9)
- **9000** -- HTTP UI (add `-p 9000:9000` if you still need unencrypted access)
- **8000** -- TCP tunnel server for Edge agents (omit if you do not use Edge)

---

## Step 5 -- Verify

```bash
# Container should be running
docker ps --filter name=portainer

# Check logs for startup errors
docker logs portainer --tail 50

# Confirm the reported version
docker inspect portainer --format '{{ .Config.Image }}'
```

Open the UI at `https://<host>:9443` and confirm the version shown in the
bottom-left corner of the sidebar matches what you deployed.

---

## Rollback Procedure

If something goes wrong, restore from the backup taken in Step 1.

```bash
# Stop and remove the broken container
docker stop portainer && docker rm portainer

# Wipe the volume and restore
docker run --rm \
  -v portainer_data:/data \
  -v "$(pwd)":/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/portainer-data-backup-<TIMESTAMP>.tar.gz -C /data"

# Re-deploy the previous image
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:<PREVIOUS_TAG>
```

---

## Keeping Up to Date

Portainer follows a monthly STS cadence with an LTS release every six months.
LTS receives patches for the full support window; STS is only supported until
the next monthly release. Choose the track that fits your maintenance schedule:

- **LTS track** -- Pin to the LTS minor (e.g., `2.39.x`). Update when patch
  releases appear.
- **STS track** -- Use `latest` or pin the STS minor. Plan to upgrade monthly.

To check for new releases: <https://github.com/portainer/portainer/releases>

---

## Quick Reference (copy-paste block)

```bash
docker stop portainer
docker rm portainer
docker pull portainer/portainer-ce:latest
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```
