ansible_ncpa
============

Install and configure the [Nagios Cross-Platform Agent (NCPA)](https://www.nagios.org/ncpa/) 3 from the official Nagios repositories.

The role adds the Nagios repository (`deb822` on Debian/Ubuntu, `yum_repository` on EL), installs the `ncpa` package, renders `/usr/local/ncpa/etc/ncpa.cfg` and manages the `ncpa.service` unit.

Requirements
------------

- Debian 11/12/13, Ubuntu 22.04/24.04, or EL 8/9/10.
- `ansible.cfg` with `inject_facts_as_vars = False`; the role reads facts only through `ansible_facts`.
- Outbound HTTPS access to `repo.nagios.com`.

Role Variables
--------------

Variables are defined in `defaults/main.yml`. The rendered configuration is `ncpa_config_default` deep-merged with `ncpa_config`, so only the keys you want to override need to be set.

Section and key names follow the upstream [configuration option reference](https://www.nagios.org/ncpa/help.php#configuration-option-reference).

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

NCPA 3 packages are signed with `GPG-KEY-NAGIOS-V3`. Hosts still carrying the NCPA 2 key are migrated automatically: the role installs the V3 key under `/etc/apt/keyrings/GPG-KEY-NAGIOS-V3.asc` and removes the previous dearmored keyring and `sources.list.d` entry.

Access control is `listener.allowed_hosts` (comma separated IPs, CIDRs or hostnames). `listener.allowed_sources` is unrelated: it only feeds the `X-Frame-Options` and `Content-Security-Policy` headers.

Agent logs:

```bash
tail -f /usr/local/ncpa/var/log/ncpa_*
```

License
-------

Apache-2.0

Author Information
------------------

Dorance Martinez @dorancemc

[![Validate Ansible Role](https://github.com/dorancemc/ansible_ncpa/actions/workflows/validate.yml/badge.svg)](https://github.com/dorancemc/ansible_ncpa/actions/workflows/validate.yml)
