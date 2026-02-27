# TSDProxy on Podman (Rocky Linux 9) — Troubleshooting Guide

A repeatable reference for getting TSDProxy running with rootless Podman on Rocky Linux 9 over SSH.

---

## Prerequisites

- Rocky Linux 9.x
- Podman 4.1+ (5.x recommended)
- A user account (not root)
- SSH access

---

## Step 1: Enable User Lingering

Allows your systemd user session to persist without an active login.

```bash
sudo loginctl enable-linger $(whoami)
```

---

## Step 2: Set Runtime Environment Variables

Add to `~/.bashrc` so they're set on every SSH login:

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=${XDG_RUNTIME_DIR}/bus
```

Then reload:

```bash
source ~/.bashrc
```

---

## Step 3: Ensure the Runtime Directory Exists

```bash
ls /run/user/$(id -u)
```

If missing, create it:

```bash
sudo mkdir -p /run/user/$(id -u)
sudo chown $(whoami):$(whoami) /run/user/$(id -u)
sudo chmod 700 /run/user/$(id -u)
```

---

## Step 4: Initialize the Podman Socket Directory

```bash
systemd-tmpfiles --user --create
mkdir -p /run/user/$(id -u)/podman
```

---

## Step 5: Enable and Start the Podman Socket

Log out and back in via SSH first, then:

```bash
systemctl --user enable --now podman.socket
systemctl --user status podman.socket
```

The socket should show as **active (listening)**.

---

## Step 6: Verify the Socket

```bash
ls -la /run/user/$(id -u)/podman/podman.sock

curl --unix-socket /run/user/$(id -u)/podman/podman.sock \
  http://localhost/v1.41/containers/json
```

Should return `[]` or a JSON list of containers.

---

## Step 7: Configure tsdproxy.yaml

Key fields to get right:

```yaml
defaultProxyProvider: default
docker:
  local:
    host: unix:///var/run/docker.sock      # in-container path after volume mount
    targetHostname: 10.88.0.1             # Podman gateway — verify with: podman network inspect podman | grep gateway
    defaultProxyProvider: default
tailscale:
  providers:
    default:
      authKey: ""
      authKeyFile: "/config/authkey"       # path inside container
      controlUrl: "https://..."
  dataDir: ./data/
http:
  hostname: 0.0.0.0
  port: 8080
log:
  level: info
  json: false
proxyAccessLog: true
```

> **Note:** All YAML keys are camelCase starting lowercase. Incorrect casing causes sections to be silently ignored.

---

## Step 8: Configure docker-compose.yaml

Mount the Podman socket to the path TSDProxy expects:

```yaml
services:
  tsdproxy:
    image: almeidapaulopt/tsdproxy:1
    environment:
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - /run/user/1000/podman/podman.sock:/var/run/docker.sock:ro,z   # replace 1000 with your UID
      - ./config/authkey:/config/authkey:ro,z
      - ./tsdproxy.yaml:/config/tsdproxy.yaml:ro,z
      - ./data:/data
```

> **Note:** Use lowercase `:z` (shared SELinux label) rather than `:Z` (private) for socket mounts.

Get your UID with:

```bash
id -u
```

---

## Step 9: Fix SELinux (Rocky Linux 9)

Rocky Linux 9 enforces SELinux by default, which will block the container from connecting to the Podman socket even if permissions look correct.

### Confirm SELinux is the issue

Temporarily set permissive and test:

```bash
sudo setenforce 0

podman run --rm \
  -v /run/user/$(id -u)/podman/podman.sock:/var/run/docker.sock:ro,z \
  alpine sh -c "apk add --no-cache curl && \
  curl --unix-socket /var/run/docker.sock http://localhost/v1.41/containers/json"

sudo setenforce 1
```

If it works in permissive, SELinux is the blocker.

### Generate and apply a targeted SELinux policy

```bash
# Install audit tools if needed
sudo dnf install -y policycoreutils-python-utils

# Capture the denials
sudo ausearch -m avc -ts recent | grep podman | audit2allow -M tsdproxy_podman

# Review what the policy will allow
cat tsdproxy_podman.te

# Apply it
sudo semodule -i tsdproxy_podman.pp
```

### Verify the policy is loaded

```bash
sudo semodule -l | grep tsdproxy
```

---

## Step 10: Launch TSDProxy

```bash
podman-compose up -d
podman logs tsdproxy 2>&1 | tail -20
```

---

## Quick Diagnostic Reference

| Symptom | Check |
|---|---|
| `failed to connect to bus: no medium found` | Don't use `sudo` with `--user` commands |
| `systemctl --user` state is degraded | Run `systemctl --user --failed` to find the broken unit |
| Socket not found | Run `systemd-tmpfiles --user --create` and `mkdir -p /run/user/$(id -u)/podman` |
| Permission denied on socket | SELinux — run `sudo ausearch -m avc -ts recent` |
| No journal files found | Use `sudo journalctl _UID=$(id -u)` instead of `--user` |
| Error listing docker networks / watching events | Check socket mount is correct in compose file with `podman inspect tsdproxy` |
| Mounts missing from inspect | Container was started without compose — run `podman-compose down && podman-compose up -d` |
