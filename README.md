ansible_ncpa
============

Install and configure the [Nagios Cross-Platform Agent (NCPA)](https://www.nagios.org/ncpa/) 3 from the official Nagios repositories.

The role adds the Nagios repository (`deb822` on Debian/Ubuntu, `yum_repository` on EL), installs the `ncpa` package, renders `/usr/local/ncpa/etc/ncpa.cfg`, and manages the `ncpa.service` unit.

Requirements
------------

- Debian 11/12/13, Ubuntu 22.04/24.04, or EL 8/9/10.
- `ansible.cfg` with `inject_facts_as_vars = False`; the role reads facts only through `ansible_facts`.
- Outbound HTTPS access to `repo.nagios.com`.
- NCPA 3.5.0 or later.

Role Variables
--------------

Variables are defined in `defaults/main.yml`. The rendered configuration is `ncpa_config_default` deep-merged with `ncpa_config`, so you only override the keys you need. Section and key names follow the upstream [configuration option reference](https://www.nagios.org/ncpa/help.php#configuration-option-reference).

`ncpa_config.api.community_string` is mandatory: the role fails if it is unset or still the upstream `mytoken` default. Store it in a Vault-encrypted variable.

Dependencies
------------

None.

Example Playbook
----------------

```yaml
- hosts: servers
  roles:
    - { role: ncpa, tags: [ ncpa ] }
```

```yaml
ncpa_config:
  api:
    community_string: "{{ vault_nagios_ncpa_token }}"
  listener:
    allowed_hosts: 127.0.0.1,10.0.0.5
```

Tags
----

| Tag | Scope |
|---|---|
| `ncpa` | Everything |
| `ncpa-install` | Repository and package |
| `ncpa-config` | Configuration file and service |

Notes
-----

- The role owns `ncpa.cfg` completely and clears any `*.cfg` under `ncpa.cfg.d/` (including the package's `example.cfg`, which otherwise wins over the rendered config). Keep drop-in content in `ncpa_config`.
- `listener.ssl_version` / `ssl_max_version` accept only `TLSv1_2` and `TLSv1_3`. Leave `ssl_max_version` unset for the highest supported; pin `TLSv1_2` if you set `listener.ssl_ciphers`. The role asserts both rules before writing.
- `api.backup_community_string` (NCPA 3.4.0+) enables token rotation without downtime. `passive.ca_cert` points `passive_ssl_verification` at a private CA.
- NCPA 3 packages use `GPG-KEY-NAGIOS-V3`. Hosts on the NCPA 2 key are migrated automatically.
- `listener.allowed_hosts` (comma-separated IPs, CIDRs, hostnames) is access control. `listener.allowed_sources` is unrelated — it only feeds the `X-Frame-Options` and `Content-Security-Policy` headers.

Agent logs: `tail -f /usr/local/ncpa/var/log/ncpa_*`

Testing
-------

`molecule test` installs the role in systemd Docker containers on Debian 12 and Rocky Linux 9, and verifies `ncpa.service` runs, `ncpa.cfg` is `root:nagios 0640` with the NCPA 3.5.0 options, `ncpa.cfg.d/` is empty, and the API answers on port 5693. It runs with `inject_facts_as_vars = False`, same as production.

```bash
pip install ansible molecule 'molecule-plugins[docker]' docker
molecule test
```

License
-------

Apache-2.0

Author Information
------------------

Dorance Martinez @dorancemc

[![Validate Ansible Role](https://github.com/dorancemc/ansible_ncpa/actions/workflows/validate.yml/badge.svg)](https://github.com/dorancemc/ansible_ncpa/actions/workflows/validate.yml)
