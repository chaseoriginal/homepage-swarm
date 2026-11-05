# Homepage

Self-hosted start page / dashboard for the fleet.

| | |
|---|---|
| URL | `https://dash.echoinggreen.net` |
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
| Proxy vhost | NPM row id 98, cert id 48 |

## What it is

Homepage renders a dashboard of the fleet's services: groups of links with
live status monitors, plus bookmarks mirrored from the Organizr V2 instance.
It sits behind a built-in login gate (Homepage v2 auth) AND an NPM vhost.

## Proxy

`dash.echoinggreen.net` is served by the local Nginx Proxy Manager on
`ore-dock-110-1` (vhost row id 98) forwarding to `10.77.110.10:3000`. The
certificate (id 48) and the Route53 / AD DNS records already existed for the
name; only the vhost was missing.

The vhost carries access list id 1 ("Private Site Access"), the same list the
other internal names use.

**The container's egress is masqueraded to `172.17.0.1`.** The container
talks to NPM over the node's own published port, so NPM sees the docker
hairpin gateway rather than the node IP. That address had to be added to
access list 1, or every proxied `siteMonitor` returns 403.

## Placement

- **Node:** pinned to `ore-dock-110-1` by `node.hostname` constraint. The
  config volume is node-local, and the Docker socket bind is node-local too.
  Both require the task to stay on this node.
- **Port:** 3000, published in host mode (the swarm ingress mesh drops IPv4
  to IPv6-only listeners, so host mode is the safe form here).
- **Docker socket:** `/var/run/docker.sock` mounted read-only so Homepage can
  render container status widgets.

### Portainer's swarm deployer ignores `network_mode`

Setting `network_mode: host` in the compose has **no effect** - Portainer logs
`ignoring unsupported compose properties | properties=["network_mode"]` and
deploys the service onto the overlay network anyway. Do not use it to work
around an ACL; fix the ACL instead. Removing the `ports:` block in the same
edit DOES take effect, and silently unpublishes the service - that is how the
vhost started returning 502.

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
- `ops/add-dns-records.yml` - Ansible playbook that adds the missing AD DNS
  CNAMEs for the dashboard's links. Run from `ore-peachpi1`:
  `ansible-playbook -i hosts add-dns-records.yml`. It asserts each record on
  the authoring DC before the run is considered successful.
- `.env.example` - placeholder only. The real values live in the Portainer
  stack environment and are deliberately NOT committed.

## Links

Bookmarks and service links use **hostnames, never raw IPs**. The bookmark set
is mirrored from the Organizr V2 instance (`home.echoinggreen.net`), whose
links live in a SQLite database rather than in files:

```
/var/lib/docker/volumes/organizr-swarm/config/www/organizr/data/organizrDB.db
```

Tables `BOOKMARK-tabs` and `BOOKMARK-categories`. 85 links in 13 categories
were evaluated; 77 are mirrored and 8 were dropped:

| Dropped link | Reason |
|---|---|
| Wizarr | NPM vhost disabled; upstream `10.77.110.50:5690` dead |
| Portainer ore-tns1 | NPM vhost disabled; upstream `10.77.105.100:9443` unreachable |
| iDrac ore-vh1 | iDRAC appliance serves 404 on its own start page |
| iDrac ore-vh2 | iDRAC appliance serves 404 on its own start page |
| iDrac ore-mgmt1 | `10.77.105.203:443` refuses connections |
| Watcher dock2 | no NPM vhost, no DNS, no backend |
| VSCode | HA add-on ingress slug stale (404) |
| Node Red | HA add-on ingress slug stale (404) |

Links whose service is alive but whose AD DNS record was missing
(`readarr`, `notifiarr`, `cus-dock1watch`) are included, and the records were
added. Links reachable only from VLAN 104 (`ore-core10g`, `ore-switch1`) are
included as bookmarks; their status monitors will fail from the 110 VLAN,
which is expected.

## Auth

Homepage v2 gates the UI behind a password:

- `HOMEPAGE_AUTH_ENABLED=true`
- `HOMEPAGE_AUTH_SECRET` - signing key for the session cookie
- `HOMEPAGE_AUTH_PASSWORD` - the login password
- `HOMEPAGE_EXTERNAL_URL` - absolute URL used to reach Homepage
- `HOMEPAGE_ALLOWED_HOSTS` - must list the hostname the dashboard is reached
  by, or every request fails with `Host validation failed`

All five are set in the Portainer stack environment. **The password is not
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
rsync -az config/ ladmin@ore-dock-110-1:/tmp/hp-config/
ssh ladmin@ore-dock-110-1 'docker run --rm -v homepage-swarm_config:/data \
  -v /tmp/hp-config:/src:ro alpine sh -c "cp -a /src/. /data/" && \
  rm -rf /tmp/hp-config && \
  docker service update --force homepage-swarm_homepage'
```

## Verifying a deployment

```bash
docker service ls --filter name=homepage-swarm
docker service inspect homepage-swarm_homepage --format '{{json .Spec.TaskTemplate.ContainerSpec.Mounts}}'
curl -s -o /dev/null -w '%{http_code}\n' http://10.77.110.10:3000/api/healthcheck
curl -sk -o /dev/null -w '%{http_code}\n' https://dash.echoinggreen.net/api/healthcheck
```

A healthy stack shows `1/1`, the healthcheck returns `200`, and the vhost
returns `200` for the healthcheck path and `307` to `/auth/signin` for `/`.

**Test the vhost from a client VLAN, not from the proxy host itself.** A
probe from `ore-dock-110-1` traverses a different path than a real client
and can produce a false negative.

To check the site monitors actually work, run the fetch from INSIDE the
container - that is the vantage the monitors use:

```bash
docker exec <container> node -e "
fetch('https://radarr.echoinggreen.net/').then(r=>console.log(r.status))"
```

Every proxied monitor returning `403` means the container's egress address is
missing from NPM's access list, not that the service is down.
