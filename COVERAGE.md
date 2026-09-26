# Test coverage baseline

Measured on 2026-09-26 against master 26ce63fd (equal to upstream master,
Webmin 2.660), following the project decisions 0003 (90 percent floor per
repository, 95 percent for every file our changes touch), 0004 (shell:
bats plus kcov) and 0006 (the gate runs in GitHub Actions and is a
required status on the default branch).

## What this repository is

It is TurnKey's Debian packaging of Webmin, not Webmin itself. It splits
upstream's single package into a minimal `webmin` core and one
`webmin-<name>` package per module and theme (103 binary packages in
`debian/control`). Line counts are lines neither blank nor comment.

| Group | Files | Lines | What it is | Measured |
|-------|-------|-------|------------|----------|
| `webmin_core/` | 433 Perl | 74839 | Upstream `webmin-2.660-minimal` tarball, unpacked, unmodified | not project code |
| `modules/` (99 modules) | 3038 Perl | 298794 | Upstream module sources, unpacked, unmodified; only Debian-capable modules kept | not project code |
| `themes/` (3 themes) | 90 Perl | 19497 | Upstream theme sources, unpacked, unmodified | not project code |
| `debian/patches/` | 2 quilt patches | 51 | The only TurnKey change to Webmin source: module dependency fix (fdisk, lvm), samba winbind options | applied at build time |
| `buildsrc`, `buildsrc_lib/__init__.py` | 2 Python | 619 | TurnKey update tool: upstream version check through `gh_releases`, tarball download and gpg verification, module and theme discovery (`module.info`, `theme.info`, Debian support filter), `debian/control` generation, quilt patch version bump | 0 percent, no test |
| `plugins_deb_rules.sh` | 1 bash | 38 | Called by `debian/rules`: one reproducible tarball per module and theme, generated `postinst` and `prerm` for each `webmin-<name>` package | 0 percent, no test |
| `debian/rules`, `preinst`, `prerm`, `postrm`, `webmin.postinst` | 5 make and POSIX sh | 86 | Package build and maintainer scripts | 0 percent, no test |
| `debian/control` (generated), `webmin.install`, `webmin.service`, `webmin.logrotate`, `etc/pam.d/webmin`, `jcameron-key.asc`, `docs/`, `README.md` | | | Packaging data and documentation | not code |

Project code: 619 Python lines in 2 files and 124 shell and make lines in
6 files. No `tests/` directory, no test of any kind, no coverage
configuration. Baseline: 0 percent measured, not an estimate.

The vendored Perl (3561 files, 393130 lines) is not under the 90 percent
floor: it is upstream Webmin, replaced wholesale by `buildsrc` at every
upstream release, and testing it is Webmin's job. The 90 percent floor
applies to the TurnKey code above.

## What "test" means here

1. **Unit tests, meaningful and cheap: `buildsrc_lib`.** Its functions are
   mostly pure and take paths: `trim_line`, `Plugin` (`_read_info`,
   `_plugin_type`, `debian_support`, `_fix_deps`, `control`, `move`
   including the symlink case), `Webmin.get_local_version`,
   `valid_version`, `_update_quilt_patch`, `new_version`,
   `dump_control`, `write_control`, `load_plugins` on scratch
   directories. The world is stubbed: `gh_releases` and `gpg` by a stub
   first in PATH, `requests.get` by a fake. This is pytest under
   coverage.py, `test-python.yml`, and it is what raises the number.
   Dependencies: `python3-debian` (`debian.deb822`), `packaging`,
   `requests`.
2. **Consistency lint, as a test file.** Every `Package: webmin-<name>` in
   `debian/control` has a `modules/<name>` or `themes/<name>` directory
   and the reverse; every `webmin-<name>` a module depends on exists in
   `debian/control`; the version in `debian/control`, in
   `webmin_core/version` and in `debian/patches/fix-module-dependencies.diff`
   agree; every `webmin-*` package named by the appliance plans exists
   here or in the known external set. Measured on 2026-09-26 over the
   plans of common and of core, lamp, moodle, odoo, tkldev and wordpress:
   27 packages referenced, 26 present, `webmin-tklbam` is built by the
   tklbam repository. This catches the class of error of upstream pull
   requests #18 and #19 (broken control file generation) before a build.
3. **The package builds.** `dpkg-buildpackage -us -uc -b` through the
   reusable `build-deb.yml` on the self-hosted LXC runner (inactive until
   the runner is registered, docs/ci-cd.md section 6 of the keel
   repository). It is the only worthwhile test of `plugins_deb_rules.sh`
   and `debian/rules`; its output (103 `.deb` files, reproducible
   tarballs) is the artifact.
4. **Boot test.** Webmin is exercised by each appliance's build and boot
   test (org-plan section 1): after first boot in an LXC container,
   `https://[2001:db8::10]:12321/` answers over IPv6 with the login page
   and `webmin-firewall6` lists the `ip6tables` rules. That is the
   acceptance test of this packaging and it does not count toward the
   unit number (decision 0004, item 4).

## How it is measured

Python, from the repository root:

    python3 -m coverage run --branch --source=buildsrc_lib -m pytest -q tests
    python3 -m coverage report --show-missing

`buildsrc` itself is a script without a `.py` suffix; its `main()` is
exercised by running it with `runpy` from a test, and `--source` gains
`buildsrc` as a directory entry or the script is measured through
`coverage run buildsrc` when that lands. Shell, per decision 0004: bats
under kcov with `tests/coverage.sh` failing below `COVERAGE_THRESHOLD`.

The gate is `.github/workflows/tests.yml`. On this branch it calls
`test-shell.yml` with threshold 0, which passes with a bootstrap notice
because nothing is measured yet. The pull request that adds the first
`buildsrc_lib` test switches the caller to `test-python.yml` with
`package: buildsrc_lib` and the measured threshold; the threshold is only
ever raised.

## Plan to reach 90 percent per file

Priority order (size: small under 30 lines of test, medium under 150,
large above):

1. `buildsrc_lib/__init__.py` (569 lines, large, in pieces): `trim_line`
   (small); `Plugin` on a scratch module directory with `module.info`
   and `theme.info` fixtures, the `os_support` variants, the fdisk and
   lvm dependency fix, the control entry with a long Depends line, the
   symlinked plugin (medium); `Webmin.get_local_version`,
   `valid_version`, `_update_quilt_patch` with an even and an odd number
   of changed lines, `new_version` exit codes 0 and 100, `dump_control`
   and `write_control` against the committed `debian/control` (medium);
   `load_plugins` over two scratch trees, the unexpected file and
   unexpected object errors (medium); `get_remote_versions` with a PATH
   stub emitting stable and pre-release tags and an invalid one (small);
   `download`, `untar`, `_validate_file` with stubs (small).
2. `buildsrc` (50, small): `--update-check` exit codes, `--force`,
   `--quiet`, a `WebminUpdateError` becoming exit 1.
3. Consistency lint of `debian/control` against `modules/`, `themes/`,
   `webmin_core/version` and the quilt patch (small).
4. `plugins_deb_rules.sh` (38, medium, bats): a scratch `modules/` and
   `themes/` tree, the real `tar`, one tarball per plugin, byte-identical
   on a second run, generated `postinst` and `prerm` contents. Done when
   the script is touched (decision 0004, pragmatic limits).
5. `debian/preinst`, `prerm`, `postrm`, `webmin.postinst` (74, small,
   bats with stubs for `systemctl`, `perl`, `deb-systemd-helper`). Done
   when touched.
6. Package build on the LXC runner (`build-deb.yml`), then the boot test
   through the appliances.
