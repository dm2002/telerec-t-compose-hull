# telerec-t-compose-hull
Underlying Ansible role for Docker containers


merges variables from four sources:
  - service_base_defaults: defined in this role (compose_hull)
  - service_defaults: defined in the particular service role
  - service_defaults_all: defined top-level
  - service_cfg: defined in the playbook


## service_cfg variables

 * `name`
 * `directory`: the directory under which all the configuration etc. of
   the service will be stored.
 * `owner`: name of the user that should own the service directories
 * `create_dirs`: list of (sub-)directories to create for the service
 * `port`: port of the service (in the container)
 * `domain`: the domain of the service. e.g. 'myservice.example.com'
 * `external`: whether the service should be externally accessible (or only from within the local network)
 * `traefik`: whether to use traefik
 * `watchtower`: whether to use watchtower
 * `autoheal`: whether to use autoheal
 * `router_entry_point`: a traefik label selecting either `web-secure` (default, TLS port 443) or `web` (insecure HTTP port 80)
 * `url_path_prefix`: if supplied traefik matches the given path prefix in the URL (the web service has to support the URL with this path)
 * `no_new_privileges`: (default: `true`) enables Docker `no-new-privileges` security option to prevent privilege escalation via setuid/setgid binaries. Set to `false` for services that require privileged operations (e.g., mailserver, promtail with root access)

## Security Options

### no-new-privileges

**Default**: `true` (enabled for all services)

The `no_new_privileges` option prevents containers from gaining additional privileges through setuid/setgid binaries or file capabilities. This is a security hardening measure that blocks privilege escalation attacks.

**When to disable** (`no_new_privileges: false`):
- Services running as root that need Docker socket access (e.g., promtail)
- Mail servers (postfix/dovecot use setuid internally)
- Non-root containers with capabilities (e.g., wireguard with CAP_NET_ADMIN and PUID=1000)
- Services with explicit `privileged: true` (flag is ineffective but no harm)

**Note on Capabilities**: Docker has a known issue ([#45491](https://github.com/moby/moby/issues/45491)) where non-root containers with `no-new-privileges:true` cannot retain capabilities like `CAP_NET_ADMIN`. If a service runs as non-root (PUID/PGID) AND requires capabilities, set `no_new_privileges: false`.

**Example usage**:

```yaml
# In roles/<service>/tasks/main.yml
- ansible.builtin.import_role:
    name: compose_hull
  vars:
    service_defaults:
      directory: "{{ docker_dir }}/myservice"
      name: myservice
      no_new_privileges: false  # Disable security restriction
```

**Template integration**:

The security option is automatically applied via the `*base_security_opt` anchor in all service templates:

```yaml
services:
  myservice:
    container_name: "{{ service_cfg.name }}"
    image: myimage:latest
    security_opt: *base_security_opt  # Applies no-new-privileges based on config
    # ... rest of service config
```


## Tags
 * `started`
 * `restarted`
 * `recreated`
 * `stopped`
 * `absent`
