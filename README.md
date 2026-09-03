# Ansible Role: podman_quadlet_container

Deploy and configure a [Podman Quadlet][quadlet] container as a
systemd-managed unit.

The role generates the `.container` quadlet unit, provisions the
directories, bind-mounted files and volumes it needs, optionally writes
an environment file, and lets systemd handle the lifecycle of the
container.

Today the role only handles **containers**. The variable namespace is
deliberately scoped (`podman_quadlet_container`) so that sibling
constructs (`.volume`, `.network`, `.pod`, `.kube`) can be added later
without breaking existing playbooks.

---

## Contents

- [Requirements](#requirements)
- [Role Variables](#role-variables)
  - [Top level](#top-level)
  - [env](#env)
  - [ports](#ports)
  - [volumes](#volumes)
  - [files](#files)
- [Dependencies](#dependencies)
- [Example Playbook](#example-playbook)
- [Notes](#notes)

---

## Requirements

- Podman 4.4 or later (quadlet support), 4.5+ recommended.
- A systemd-based target host.
- The `containers.podman` collection, used to pull the image before the
  unit starts. It is bundled with the `ansible` community package.

---

## Role Variables

Everything is driven by a **single dictionary**,
`podman_quadlet_container`. It defaults to `{}`, meaning the role is a
no-op unless you provide a definition.

```yaml
podman_quadlet_container: {}
```

---

### Top level

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `name` | `str` | :white_check_mark: | — | Name of the container. Also used as the unit name and as the default location of the env file. |
| `image_name` | `str` | :white_check_mark: | — | Podman/Docker image to run, e.g. `docker.io/library/nginx`. |
| `image_version` | `str` | :white_check_mark: | — | Image tag or version, e.g. `1.27-alpine`. Kept separate from `image_name` so it can be bumped independently. |
| `owner` | `str` | :x: | `"1000"` | Owner (user) of the files and directories managed by the role. |
| `group` | `str` | :x: | `"1000"` | Group of the files and directories managed by the role. |
| `force_ownership` | `bool` | :x: | `false` | Force the ownership inside the container as well, not only on the host side. |
| `working_dir` | `str` | :x: | — | Working directory used inside the container. |
| `exec` | `str` | :x: | — | Override the image's default command. |
| `file_mode` | `str` | :x: | `"0640"` | Default mode applied to the files managed by the role. |
| `dir_mode` | `str` | :x: | `"0750"` | Default mode applied to the directories managed by the role. |
| `add_capabilities` | `list[str]` | :x: | `[]` | Linux capabilities to add to the container (`AddCapability=`). |
| `drop_capabilities` | `list[str]` | :x: | `[]` | Linux capabilities to drop from the container (`DropCapability=`). |
| `env` | `dict` | :x: | — | Environment file configuration. See [env](#env). |
| `ports` | `list[dict]` | :x: | — | Host-to-container port mappings. See [ports](#ports). |
| `volumes` | `list[dict]` | :x: | `[]` | Bind-mounted directories. See [volumes](#volumes). |
| `files` | `list[dict]` | :x: | `[]` | Files rendered on the host and mounted into the container. See [files](#files). |

---

### env

Writes an environment file on the host and wires it into the unit.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `values` | `dict` | :white_check_mark: | — | Environment variables, as a flat key/value mapping. |
| `mode` | `str` | :x: | `"0600"` | Mode of the generated `.env` file. Restrictive by default since it usually holds secrets. |
| `path` | `str` | :x: | `/etc/{{ podman_quadlet_container.name }}/.env` | Path of the `.env` file on the host. |

---

### ports

Each entry is a host/container port pair.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `host` | `int` | :white_check_mark: | — | Port published on the host. |
| `container` | `int` | :white_check_mark: | — | Port listening inside the container. |

---

### volumes

Each entry describes a bind mount between the host and the container.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `host_path` | `str` | :white_check_mark: | — | Path on the host. |
| `container_path` | `str` | :white_check_mark: | — | Mount point inside the container. |
| `mode` | `str` | :x: | `"0750"` | Mode of the directory created on the host. Only relevant when `create` is `true`. |
| `mount_mode` | `str` | :x: | `rw` | Mount options passed to Podman, e.g. `rw`, `ro`, `z`, `ro,Z`. |
| `create` | `bool` | :x: | `false` | Create the directory on the host. Leave to `false` when the path is provisioned elsewhere. |

---

### files

Each entry is a file shipped from the playbook to the host, then
mounted into the container. Useful for configuration files that should
not live in the image.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `src` | `str` | :white_check_mark: | — | Source path, resolved from the playbook's `files/` or `templates/` directory depending on `type`. |
| `type` | `str` | :white_check_mark: | — | `file` for a verbatim copy, `template` to render it with Jinja2. |
| `host_path` | `str` | :white_check_mark: | — | Destination path on the host. |
| `container_path` | `str` | :white_check_mark: | — | Mount point inside the container. |
| `mode` | `str` | :x: | `"0640"` | Mode of the file created on the host. |
| `mount_mode` | `str` | :x: | `rw` | Mount options passed to Podman, e.g. `ro` for read-only configuration. |

## Dependencies

No role dependencies. One collection dependency: `containers.podman`.

> [!note]
> It ships with the `ansible` community package, so on a typical
> workstation install it is already available and nothing needs to be done.

```yaml
# requirements.yml
collections:
  - name: containers.podman
```

```bash
ansible-galaxy collection install -r requirements.yml
```

`containers.podman.podman_image` pulls `image_name:image_version` on the
target before the unit is started.

> [!warning]
> Private registries need credentials
> made available to Podman beforehand, for example with a `containers-auth.json`
> on the host.

---

## Example Playbook

Minimal usage:

```yaml
- hosts: servers
  roles:
    - role: podman_quadlet_container
      vars:
        podman_quadlet_container:
          name: whoami
          image_name: docker.io/traefik/whoami
          image_version: v1.10.2
          ports:
            - host: 8080
              container: 80
```

A more complete example:

```yaml
- hosts: servers
  roles:
    - role: podman_quadlet_container
      vars:
        podman_quadlet_container:
          name: gitea
          image_name: docker.io/gitea/gitea
          image_version: 1.22-rootless
          owner: "1500"
          group: "1500"
          force_ownership: true
          working_dir: /var/lib/gitea
          exec: /usr/local/bin/gitea web

          env:
            path: /etc/gitea/.env
            mode: "0600"
            values:
              GITEA__database__DB_TYPE: postgres
              GITEA__database__HOST: db.internal:5432
              GITEA__database__PASSWD: "{{ vault_gitea_db_password }}"

          ports:
            - host: 3000
              container: 3000
            - host: 2222
              container: 2222

          add_capabilities:
            - NET_BIND_SERVICE
          drop_capabilities:
            - ALL

          volumes:
            - host_path: /srv/gitea/data
              container_path: /var/lib/gitea
              mode: "0750"
              mount_mode: rw
              create: true
            - host_path: /srv/gitea/repos
              container_path: /var/lib/gitea/repositories
              create: true

          files:
            - src: gitea/app.ini.j2
              type: template
              host_path: /etc/gitea/app.ini
              container_path: /etc/gitea/app.ini
              mode: "0640"
              mount_mode: ro
            - src: gitea/known_hosts
              type: file
              host_path: /etc/gitea/known_hosts
              container_path: /etc/gitea/known_hosts
              mount_mode: ro
```

---

## Notes

- `image_name` and `image_version` are separate so that upgrades are a
  one-line change and the tag can be templated per environment.
- `env.values` is rendered into a file rather than inlined in the unit,
  which keeps secrets out of `systemctl show` output. Store them with
  Ansible Vault.
- Set `create: false` on a volume when the host path is managed by
  another role, so ownership and mode are not fought over.
- Mount configuration files read-only (`mount_mode: ro`) whenever the
  container does not need to rewrite them.

[quadlet]: https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html

<!-- markdownlint-configure-file { "MD013": { "tables": false } } -->
