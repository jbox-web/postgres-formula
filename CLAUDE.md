# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The parent `../CLAUDE.md` covers what all 32 forks share (toolchain, Kitchen
platforms, pre-commit, CI, conventions). This file only records where
`postgres-formula` departs from it.

## Commands

```bash
bin/kitchen list                                # 3 instances: default-<platform>
bin/kitchen test default-debian-13-3008-2       # one instance, end to end
bin/kitchen converge default-debian-11-3006-27  # apply only, keep the container
bin/kitchen verify default-debian-11-3006-27    # rerun the InSpec controls
pre-commit run --all-files
```

There is a single suite, `default`. The `repo-*` logs under `.kitchen/logs/`
(gitignored) are leftovers from a removed suite.

## Architecture — departures from the template

### Layout

This formula predates the `package/config/service` split. States are grouped
by component instead:

- `init.sls` includes `server`, `client`, `manage` (or `macos` on MacOS).
- `server/init.sls` — packages, cluster creation (`prepare_cluster_cmd`), conf
  dirs, `postgresql.conf` block, `pg_hba.conf`, `pg_ident.conf`, tablespace
  dirs, service. `server/image.sls` replaces the service with `pg_ctl` when
  `bake_image` is true.
- `client/`, `dev/`, `python.sls`, `upstream.sls` (PGDG repo), `manage.sls`
  (DB objects), `dropped.sls` + `*/remove.sls` (removal).
- Templates live in `postgres/templates/`, not `files/default/`.
  `libtofs.jinja` is shipped but never called: sources are plain `salt://`
  paths taken from the `pg_hba.conf` / `pg_ident.conf` pillar keys.
- The `postgres/` path is hardcoded in several places (`map.jinja`,
  `macros.jinja`, `repo.yaml`, the `os*map.yaml` imports, `upstream.sls`), so
  the tree is not relocatable despite `tplroot` appearing in `_mapdata`.

### `map.jinja` merge chain

Different from the template chain:

```
defaults.yaml (under key `postgres`, base='postgres')
  <- osfamilymap.yaml  (grain os_family)
    <- osmap.yaml      (grain os — only Fedora)
      <- codenamemap.yaml (grain oscodename)
        <- pillar `postgres` (whole key; there is no `lookup` sub-key)
```

No `osarch`/`osfinger` levels. After the merge, `map.jinja` builds
`prepare_cluster_cmd` from `prepare_cluster.pgcommand`, `data_dir`,
`cluster.locale` and `cluster.encoding`.

`repo.yaml` reads `use_upstream_repo`, `version`, `fromrepo` and
`cluster:name` from pillar *before* the merge, because the maps need them to
compute paths. It is imported by `osfamilymap`, `osmap` and `codenamemap`,
each re-importing `defaults.yaml`.

`codenamemap.yaml` generates one entry per Debian/Ubuntu codename through the
`debian_codename(name, version)` macro, twice per Debian release (short
codename and the long `Debian GNU/Linux N (name)` form the grain returns
without `lsb-release`). It sets `pkg`, `pkg_client`, `conf_dir`
(`/etc/postgresql/<v>/<cluster>`), `data_dir` (`/var/lib/postgresql/<v>/<cluster>`),
`fromrepo` and `pg_createcluster` as the cluster command. The pillar
`version` only replaces the distro default when `use_upstream_repo` is true;
with `use_upstream_repo: false` (as in the test pillar), `pkg`,
`pkg_client`, `conf_dir` and `data_dir` must be set explicitly in pillar.
Note that `defaults.yaml` sets `use_upstream_repo: true`.

### `manage.sls` and `format_state`

`macros.jinja:format_state(name, state, kwargs)` turns each entry of the
`users`, `tablespaces`, `databases`, `schemas` and `extensions` pillar dicts
into a `postgres_<type>-<name>` state calling `postgres_<type>.<ensure>`
(`present` by default), passes every other key through as a kwarg, defaults
`user` to `postgres.user`, and hangs the state on `onchanges` of
`postgres-reload-modules`. `schemas` and `extensions` nested under a database
are popped out and become `postgres_schema-<db>-<name>` /
`postgres_extension-<db>-<name>`. Requisites between objects are built from
these IDs, so renaming the ID scheme breaks them.

### Service restarts and the port

Config states `watch_in` `service: postgresql-running`; `pg_ident.conf` only
triggers `postgresql-service-reload` (`module.wait` → `service.reload`). The
port comes from `salt['config.option']('postgres.port')` — a top-level
pillar/minion-config key, not `postgres:port` — and when set, the existing
`port =` line is commented and the port is written in the
`file.blockreplace` zone together with `postgresconf`. The test pillar uses
5433 on Debian and the InSpec controls expect it.

### Salt 3008

The `alternatives` and `sysrc` states no longer exist in Salt 3008. They are
only rendered when `linux.altpriority` is non-zero (`server/`, `client/`) or
`service.sysrc` is true (FreeBSD/OpenBSD); keep those defaults off for Debian.

## Tests

- No `_mapdata` reference dumps: `_mapdata` runs in `state_top` but nothing
  compares its output.
- `test/integration/default/controls/services.rb` checks the service and the
  listening port; `share/controls/command.rb` checks `db2` (owner, encoding,
  locale, tablespace `my_space`) via `psql`; `share/controls/config.rb` checks
  `/srv/my_tablespace`. They depend on the objects declared in
  `test/salt/pillar/postgres.sls`, so a pillar change there usually needs a
  matching control change.
