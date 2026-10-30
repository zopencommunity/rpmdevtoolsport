# PR1: Guard optional Python dependencies (progressbar, requests, rpm); fix z/OS portability; fix `set -e`-unsafe tests in `rpmdev-diff`

## Problem

On z/OS, the Python modules `progressbar`, `requests`, and `rpm` are not
available. Both `rpmdev-spectool` and `rpmdev-rmdevelrpms` imported these
modules unconditionally at the top level, causing the scripts to crash with
`ModuleNotFoundError` before they could print any help or version output.

This broke the build because `help2man` invokes each script with `--help`/
`--version` to generate man pages, and the crash prevented any help text from
being produced.

Additionally, `rpmdev-setuptree` unconditionally greps `/etc/passwd` to look
up the user's home directory. On z/OS, `/etc/passwd` does not exist
(user database is managed by RACF), causing the script to emit a grep error
on every invocation.

## Fix

- **`rpmdev-spectool`**: Wrapped `import requests`, `import progressbar`, and
  `import rpm` in `try/except ImportError` guards. Added runtime checks inside
  the functions that require these modules (`download()`, `Spec.__init__()`,
  and the `--sourcedir` branch of `main()`).

- **`rpmdev-rmdevelrpms.py`** (source for `rpmdev-rmdevelrpms`): Wrapped
  `import rpm` in a `try/except ImportError` guard. Moved the rpm-availability
  check to after `optparse` processes `--help`/`--version`, so the script can
  print help text even without the `rpm` module installed.

- **`rpmdev-bumpspec`**: Changed shebang from `#!/usr/bin/python -tt` to
  `#!/usr/bin/env python3`.

- **`rpmdev-checksig`**: Changed shebang from `#!/usr/bin/python -tt` to
  `#!/usr/bin/env python3`. Wrapped `import rpm` in a `try/except ImportError`
  guard with a runtime check at the top of `main()`.

- **`rpmdev-sort`**: Changed shebang from `#!/usr/bin/python -tt` to
  `#!/usr/bin/env python3`. Wrapped `import rpm` in a `try/except ImportError`
  guard with a runtime check at the top of `main()` (after `--help` is handled).

- **`rpmdev-vercmp`**: Changed shebang from `#!/usr/bin/python -tt` to
  `#!/usr/bin/env python3`. Wrapped `import rpm` in a `try/except ImportError`
  guard with a runtime check at the top of `main()` (after `--help` is handled).

- **`rpmdev-setuptree`**: Guarded the `grep /etc/passwd` lookup with a
  `[ -f /etc/passwd ]` check so it is skipped when the file does not exist
  (e.g. z/OS where `$HOME` is always set via the environment). Also softened
  the error message to not reference `/etc/passwd` specifically.

- **`rpmdev-diff`**: Changed all bare `[[ $var ]]` conditional tests in the
  option-parsing loop to `[[ -n $var ]]`. Under `set -e`, an empty (unset)
  variable causes a bare `[[ $var ]]` to return exit code 1, which terminates
  the script immediately. Using the explicit `-n` (non-empty) test is safe
  under `set -e` and is the correct idiom for testing whether a variable is set.
  Affected options: `-l`/`--list`, `-L`/`--long-list`, `-c`/`--contents`,
  `-m`/`--metadata`, `-h`/`--help`, and `-v`/`--version`.

## Files changed

- `rpmdev-spectool.patch`
- `rpmdev-rmdevelrpms.py.patch`
- `rpmdev-bumpspec.patch`
- `rpmdev-checksig.patch`
- `rpmdev-sort.patch`
- `rpmdev-vercmp.patch`
- `rpmdev-setuptree.patch`
- `rpmdev-diff-fix-set-e-help-version.patch`
