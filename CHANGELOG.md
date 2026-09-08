# Changelog

All important changes to this role are listed here.

## [3.0.0] - 2026-09-08

- **Breaking — `ncpa.cfg.d/` is purged on every run.** NCPA parses every `*.cfg` in that directory *after* `ncpa.cfg` and merges it on top, so a drop-in silently overrode whatever the role rendered and the role had no way of knowing. Since the role rewrites `ncpa.cfg` whole, it was already the source of truth for the main file and only pretended to be one for the agent. It now is: the directory is emptied before the template is deployed. Anything kept in a drop-in has to move into `ncpa_config`. First run removes the `example.cfg` the package ships, which makes `rpm -V ncpa` report it as missing.
- **The defaults had stopped at NCPA 3.1.** `repo.nagios.com` ships 3.5.0 today, and four options added since then were missing from a file the role overwrites in full: `gui_session_timeout` and `ssl_max_version` in `[listener]` (3.5.0), `ca_cert` in `[passive]` (3.4.2) and `backup_community_string` in `[api]` (3.4.0). `gui_session_timeout` and `ca_cert` are rendered with the upstream defaults; the two that change listener behaviour when set, plus `allowed_hosts`, `allowed_sources`, `ssl_ciphers` and `run_with_sudo`, ship as commented examples.
- **Perl and PHP plugins had stopped running.** `plugin directives` only declared `.py` and `.sh`, and because the template writes the whole file, the `.pl` and `.php` interpreters the package declares disappeared on the first run of the role. Both are back.
- **TLS and token validation.** `tasks/validate.yml` now refuses a `ssl_version` or `ssl_max_version` outside `TLSv1_2` / `TLSv1_3` — TLSv1 and TLSv1.1 were deprecated in 3.3.1 — and refuses `ssl_ciphers` unless `ssl_max_version` is `TLSv1_2`, since TLSv1.3 ignores the cipher list and the setting would be a silent no-op. `backup_community_string`, if set, cannot be the upstream `mytoken2` nor a copy of the primary token, which would defeat the point of a rotation slot.
- **Debian installs pulled in `python3-debian`.** `ansible.builtin.deb822_repository` is implemented on top of that library and fails with `python3-debian is not installed` on a minimal Debian or Ubuntu host, which is exactly what the role targets. It went unnoticed because every host tested so far already had it. The molecule run on a clean Debian 12 image found it on the first converge.
- **`ansible-lint` ran before the role was resolvable.** The workflow created the `tests/roles/ncpa` symlink *after* linting, so `ansible-lint` hit `The role 'ncpa' was not found` on `tests/test.yml` and the `main` branch had been red on it. The setup step now runs before the linters. It reproduces locally only with an isolated `ANSIBLE_ROLES_PATH`, since a stale `~/.ansible/roles/ncpa` on a developer machine hides it.
- **The role is deployed in CI, not just linted.** `molecule/default/` installs NCPA from the real repository into Debian 12 and Rocky Linux 9 containers running systemd, then asserts the service state, the `root:nagios 0640` on `ncpa.cfg`, the rendered 3.5.0 options, an empty `ncpa.cfg.d/` and a `200` from the API. The scenario runs with `inject_facts_as_vars = False`, which is the setting that broke the role before 2.0.0 and that no test covered. Idempotence is part of the default sequence. A `molecule` job runs after `validate` in the workflow. `prepare.yml` waits for systemd to finish booting and replaces the container's sudo PAM stack, which the Rocky Linux 9 image cannot run on a GitHub runner; it is a harness fixture and leaves the role's own `become` behaviour untouched.

## [2.0.0] - 2026-08-31

- **The role could not add the Debian repository.** The URI was built from `ansible_distribution_release`, a top level fact variable that simply is not there on a project that sets `inject_facts_as_vars = False` in its `ansible.cfg`. Every fact now goes through `ansible_facts`, which exists either way, so the role no longer cares about the setting. This was the failure that started the rewrite.
- **The defaults were still NCPA 2.** NCPA 3 reads `uid`, `gid`, `pidfile`, `loglevel`, `logmaxmb` and `logbackups` from `[general]`, not from `[listener]` and `[passive]` where the role kept writing them, so those values were silently ignored on any host running the current agent. They were moved, and the sections were completed with what NCPA 3 actually understands: `allow_config_edit` and `disable_gui` in `[listener]`, `passive_ssl_verification` in `[passive]`, `nfsd` and `xenfs` in `exclude_fs_types`. `nrdp.hostname` no longer ships the `NCPA 2` placeholder, which was being sent to NRDP as the service description of every passive check that did not override it.
- **Access control is `allowed_hosts`, not `allowed_sources`.** They look interchangeable and are not: `allowed_sources` only feeds the `X-Frame-Options` and `Content-Security-Policy` headers, so an inventory that restricts the agent with it has no ACL at all. Worth grepping your group_vars before upgrading. Related: the listener stays bound to `0.0.0.0` instead of the upstream `::`, because on a dual stack socket an IPv4 client arrives as `::ffff:<address>` and stops matching the plain IPv4 entries people write in `allowed_hosts`.
- **Enterprise Linux was installing two repositories it never needed.** NCPA is built with `AutoReqProv: no` and a frozen Python — it has no dependencies to resolve — so EPEL was pulled in for nothing and CodeReady Builder meant a `subscription-manager` call with `failed_when: false` on every single run, the kind of task that is always green and never checked. Both are gone. The `nagios-repo` release package was replaced by `rpm_key` plus `yum_repository` against `https://repo.nagios.com/nagios/<major>/`, which is one less package to track when a new EL comes out. EL 10 was added while I was there.
- **The GPG key is downloaded, not piped through a shell.** `curl | gpg --dearmor` was a `shell` task pretending to be idempotent with `creates`. `get_url` writes the armored key straight to `/etc/apt/keyrings/GPG-KEY-NAGIOS-V3.asc` and `deb822_repository` points `signed_by` at it, which apt has accepted since 1.4. Hosts still carrying the old dearmored keyring or a `sources.list.d/nagios.list` get them removed on the next run — that is the piece NCPA 2 to NCPA 3 upgrades need, since the V3 key is what signs the current packages.
- **A first install left the service alone.** The only `systemd_service` call was a handler, and handlers do not fire when nothing changed, so a fresh host ended up with whatever state the package left behind. Enabling and starting the unit is now a task. The handler that verifies the API had `retries` and `delay` with no `until`, so Ansible ran the request once and ignored both values; it now loops until the API answers `200`, against `/api/` rather than `/api`.
- **The token stops leaking into the output.** The template task and the API check run with `no_log: true` and `diff: false`. A new `tasks/validate.yml` refuses to continue if the operating system family is unsupported or if `ncpa_config.api.community_string` is still the upstream `mytoken`, which is a default that has a habit of reaching production.
- **File modes match what the package ships.** `ncpa.cfg` is `root:nagios 0640` and `/usr/local/ncpa/var/log` is `root:nagios 0775`, as declared in the NCPA spec file. The role was writing `nagios:nagios 0644` and `0770`, so every run fought the package over the same files.
- **Breaking — variables were renamed.** `ncpa_nagios.user` and `ncpa_nagios.group` are now `ncpa_user` and `ncpa_group`. `ncpa_config` is a dictionary instead of a list, so `combine` no longer receives a type it cannot merge. The Red Hat variables `epel_repo_urls`, `nagios_repo_urls`, `codeready_repos` and `package_manager` were removed together with `vars/Debian.yaml` and `vars/RedHat.yaml`; the repository URLs are constants and belong in the task files. `defaults/main/ncpa.yml` and `defaults/main/vars.yml` were merged into `defaults/main.yml`.
- **Breaking — the tags changed.** `ncpa_config` is now `ncpa-config`, alongside `ncpa` and `ncpa-install`. `main.yml` selects the platform with `import_tasks` and a `when` on `ansible_facts['os_family']` instead of `include_tasks` with `with_first_found`, so `--list-tasks` and `--check` show what the role will do before it runs.
- **CI checks more than YAML syntax.** `.ansible-lint` and `.editorconfig` were added, `.yamllint.yml` now rejects `yes` and `no` for booleans, and the workflow runs `ansible-lint` next to `yamllint` and the syntax check. The role passes the `production` profile.

## [1.1.0] - 2025-06-22

- Debian repository moved to the deb822 format, with the Nagios GPG key under `/etc/apt/keyrings`.
- Red Hat tasks rewritten around explicit repository URLs, adding EPEL and CodeReady Builder.
- Handler moved to `ansible.builtin.systemd_service`, and `become` added to the tasks that need root.
- Plugin directives switched to `python3`; the unused interpreters were commented out.
- GitHub Actions workflow, test inventory and `.gitignore` added.

## [1.0.5] - 2024-09-20

- Red Hat 9 support.
- Compatibility with Ansible 5.10, and updated handler syntax.

## [1.0.4] - 2023-12-08

- Daemon renamed to `ncpa`, matching the NCPA 2 packages.
- Updated the Debian repository key.

## [1.0.3] - 2022-11-19

- Removed the deprecated `apt_key` module.

## [1.0.2] - 2022-11-14

- Applied yamllint rules and added test validations.

## [1.0.1] - 2021-10-10

- Fixed the permissions of the log path.

## [1.0.0] - 2021-09-18

- First release.
