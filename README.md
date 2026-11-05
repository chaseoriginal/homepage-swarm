# Homepage

Self-hosted start page / dashboard for the fleet.

| | |
|---|---|
| Portainer host | ore-dock-110-1 |
| Portainer stack id | 37 |
| Endpoint | 3 (local swarm 110-1) |
| Swarm | True |
| Live service | `svc:homepage-swarm_homepage` |
| Git repo | https://github.com/chaseoriginal/homepage-swarm |
| Compose file | `docker-compose.yml` |
| Auto-update poll | every 15m |
| Node | `ore-dock-110-1` (pinned) |
| Port | 3000, published in host mode |
| Proxy vhost | not yet (NPM vhost is a follow-up) |

## What it is

Homepage renders a dashboard of the fleet's services: groups of links with
live status monitors, plus bookmarks. It runs behind a built-in login gate
(Homepage v2 auth) because no reverse proxy is in front of it yet.

## Placement

- **Node:** pinned to `ore-dock-110-1` by `node.hostname` constraint. The
  config volume is node-local, and the Docker socket bind is node-local too.
  Both require the task to stay on this node.
- **Port:** 3000, published in host mode (the swarm ingress mesh drops IPv4
  to IPv6-only listeners, so host mode is the safe form here).
- **Docker socket:** `/var/run/docker.sock` mounted read-only so Homepage can
  render container status widgets.

## Files

- `docker-compose.yml` - the deployed compose.
  The named volume is declared with an explicit `name:`, because Docker Swarm
  prefixes a plain named volume with the stack name and a rename would mount
  a fresh EMPTY volume.
- `config/` - the seed of the Homepage config directory (`services.yaml`,
  `bookmarks.yaml`, `docker.yaml`). These are copied into the
  `homepage-swarm_config` volume on the node at deploy time. After that the
  volume is the live copy and Homepage's own file layout applies.
- `config/docker.yaml` - the Docker instance definition. `swarm: true` is
  required, otherwise Homepage lists containers instead of swarm services and
  the status widgets come back empty. `socket:` (not host/port) is used because
  the container has the node's socket mounted in.

  A service only gets a Docker status widget if it names both `server:` and
  `container:`. With `swarm: true`, `container:` is the SWARM SERVICE NAME
  (`homepage-swarm_homepage`), not the container name
  (`homepage-swarm_homepage.1.<taskid>`). The status map returned by
  `/api/docker/statuses?server=local` is keyed by service name.
- `.env.example` - placeholder only. The real values live in the Portainer
  stack environment and are deliberately NOT committed.

## Auth

Homepage v2 gates the UI behind a password:

- `HOMEPAGE_AUTH_ENABLED=true`
- `HOMEPAGE_AUTH_SECRET` - signing key for the session cookie
- `HOMEPAGE_AUTH_PASSWORD` - the login password
- `HOMEPAGE_EXTERNAL_URL` - absolute URL used to reach Homepage

All four are set in the Portainer stack environment. **The password is not
in git.** If the stack env is ever lost, rotate by updating the stack env
and redeploying; there is no recovery path through the volume.

## Changing the dashboard

Two options, pick one and stick to it:

1. **Edit the volume in place** (simplest). The live config lives in the
   `homepage-swarm_config` volume on `ore-dock-110-1`. Edit through the
   Homepage UI or by hand on the node. Changes made here are NOT in git.
2. **Edit in git and re-seed** (keeps git authoritative). Change
   `config/services.yaml` in the repo, push, then re-run the seed step
   below. The seed overwrites the volume's copy of the files it manages.

Re-seeding overwrites the managed files, so do not mix the two.

## Re-seeding the config from git

```bash
# on ore-dock-110-1
scp -r ladmin@<dev-host>:/home/ladmin/homepage-swarm/config/. /tmp/hp-config/
docker run --rm -v homepage-swarm_config:/data -v /tmp/hp-config:/src:ro \
  alpine sh -c 'cp -a /src/. /data/ && ls -la /data'
docker service update --force homepage-swarm_homepage
```

## Verifying a deployment

```bash
docker service ls --filter name=homepage-swarm
docker service inspect homepage-swarm_homepage --format '{{json .Spec.TaskTemplate.ContainerSpec.Mounts}}'
curl -s -o /dev/null -w '%{http_code}\n' http://10.77.110.10:3000/api/healthcheck
```

A healthy stack shows `1/1` and the healthcheck returns `200`.
