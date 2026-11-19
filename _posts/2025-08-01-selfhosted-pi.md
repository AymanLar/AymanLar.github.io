---
layout: post
title: "selfhosted-pi: Bootstrapping a Raspberry Pi Home Server"
date: 2025-08-01
categories: [projects]
tags: [raspberry-pi, docker, portainer, selfhosted, linux, projects]
---

## Context

A Raspberry Pi is an excellent little machine for self-hosting: it is cheap, quiet, and power-efficient enough to run continuously. The friction is in the first hour — installing an OS, connecting over SSH, updating packages, installing a container runtime, and then figuring out how to actually deploy something without typing long Docker commands forever.

`selfhosted-pi` is my getting-started toolkit for that first hour. It is not an application or a container stack; it is a small collection of Bash scripts, Markdown guides, and curated resources that take a fresh Pi from "just imaged" to "Docker + Portainer running, ready to deploy services".

![Self-hosted Pi](/assets/selfhosted-pi-main.jpg)

## What the Repository Contains

The structure reflects its purpose as a bootstrap kit rather than a codebase:

```
selfhosted-pi/
├── README.md                       setup guide: OS, SSH, Docker, Portainer
├── resources.md                    curated links to self-hosting projects and tutorials
├── installation-scripts/
│   ├── docker.sh                   install Docker + add user to docker group
│   ├── portainer.sh                deploy Portainer CE
│   ├── update_portainer.sh         redeploy Portainer with the latest image
│   └── install-dwm.md              optional: build DWM on Raspbian Lite
├── docker-images/                  placeholder for future custom images
└── pics/                           hero and reference images
```

There are no Dockerfiles, no Compose files, and no application source code. Provisioning is imperative: shell scripts plus documented manual steps. That is a deliberate choice for a project meant to be read, copied, and modified by someone learning how self-hosting works.

## Architecture

The stack is intentionally small:

| Layer | Technology |
|---|---|
| Hardware | Raspberry Pi (any model with a network connection) |
| OS | Raspberry Pi OS / Raspbian Lite |
| Remote access | SSH |
| Package management | `apt` |
| Container runtime | Docker Engine, installed via the official convenience script |
| Management UI | Portainer CE in a container |
| Optional GUI | DWM (suckless window manager) on Xinit |

The scripts share a common pattern: a `#!/bin/bash` shebang, a red-coloured `error()` helper, and an internet check using `wget -q --spider http://github.com` before doing anything. Critical commands are guarded with `|| error "..."`, so the scripts fail fast rather than leaving a half-configured system.

### Docker provisioning

`docker.sh` does three things:

1. Verifies network connectivity.
2. Installs Docker with `curl -sSL https://get.docker.com | sh`.
3. Adds the current user to the `docker` group with `sudo usermod -aG docker $USER`.

The script then reminds you to log out or reboot — group membership changes only apply to new login sessions.

### Portainer deployment

`portainer.sh` pulls and runs the Portainer CE container:

```bash
docker run -d \
  -p 9000:9000 -p 9443:9443 \
  --name=portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

A few details are worth calling out:

- **Ports 9000/9443** expose the HTTP and HTTPS management interfaces.
- **The Docker socket bind mount** gives Portainer control over the host's Docker daemon — this is how it can list, start, and stop containers. It is also effectively root-level access, so only run it on a machine you trust.
- **The `portainer_data` named volume** persists Portainer's own state across container replacement.
- **`--restart=always`** brings the UI back automatically after a reboot.

`update_portainer.sh` automates upgrades by finding the running `portainer-ce` container through `docker ps | grep portainer-ce | awk`, stopping and removing it, removing the old image tag, and redeploying the same container definition. The persistent volume means your settings survive the swap.

## Getting Started

Flash Raspberry Pi OS to an SD card, connect it to your network, and SSH in:

```bash
ssh pi@ipaddress
sudo apt update && sudo apt upgrade
sudo reboot
```

After rebooting, optionally install a few base tools (`git`, `neovim`), then clone the repository and run the scripts:

```bash
git clone https://github.com/Ayman-s-Lab/selfhosted-pi.git
cd selfhosted-pi/installation-scripts

chmod +rx docker.sh
./docker.sh
logout                 # re-SSH so the docker group change takes effect

chmod +rx portainer.sh
./portainer.sh
```

Open `http://<pi-ip>:9000`, create an administrator account, and select the local Docker environment. To get a catalog of ready-to-deploy services immediately, add an App Templates URL in **Settings → App Templates**, for example:

```
https://raw.githubusercontent.com/Qballjos/portainer_templates/master/Template/template.json
```

From there, deploying a service is a matter of selecting a template and clicking deploy. The `resources.md` file collects further project ideas, tutorials, and articles if you want inspiration for what to run next.

For headless setups the optional DWM guide is unnecessary, but if you want a minimal GUI on Raspbian Lite it walks through installing X11 development headers, compiling DWM from source, configuring `xinitrc` to `exec dwm`, and starting it with `startx`.

## Reflections

`selfhosted-pi` is less a product than a written record of a setup that worked. Its value is in lowering the entry barrier: a beginner can follow the README top to bottom, understand what each script does, and end up with a management UI that makes the next steps visual rather than memorised.

If I revisit it, the natural evolution is moving from imperative scripts to a declarative stack — pinning image versions instead of `latest`, adding Compose files or an Ansible playbook, configuring TLS and a reverse proxy, and tightening the security posture around the Docker socket. Those changes would make it more reproducible, but they would also hide some of the mechanics that make the current project useful as a learning document.
