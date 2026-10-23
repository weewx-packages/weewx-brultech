# weewx-brultech

Driver, database schema and skins for the Brultech GEM energy monitors (WeeWX extension, catalog id `brultech`). Continuation of `tkeffer/weewx-brultech` by the weewx-packages organization, no longer synced with the original. Develop here.

## Phase

Alpha (no LIVE block). Keep the tests few and fast. The contracts of the organization catalog still apply: the installer name `weewx-brultech` and the database column names (`chN_a_energy2`, `chN_ad_energy2`, ...) stay stable.

## Layout

- `install.py`: WeeWX extension installer. The file list in it must match the files under `bin/` and `skins/` (a test checks it).
- `bin/user/brultech.py`: driver (socket connection, packet decoding, `BrultechService`, xtypes for power and `diff`).
- `bin/user/gem_schema.py`: database schema.
- `skins/SeasonsPower`, `skins/StandardPower`: report skins.
- `util/brultech/calc_deltas`: SQLite script that adds delta energies to old databases. Not installed.
- `tests/`: pytest. `conftest.py` builds GEM packets and serves them from a fake monitor on a local socket. No hardware.
- `changelog`: one entry per release, newest first.

## Commands

- Tests, about 1 s: `python -m pytest -q tests`. Needs WeeWX 5.x and pytest, for example the workspace environment `../.venv`.
- Install check on a disposable station: `weectl station create /tmp/x --no-prompt`, then `git archive --format=zip --prefix=weewx-brultech-x/ -o /tmp/x.zip HEAD` and `weectl extension install /tmp/x.zip --config /tmp/x/weewx.conf --yes`.

## Conventions

- Python 3 only. WeeWX style: no runtime annotations, types in Google-style docstrings.
- Files from the original keep their copyright header. A changed file gets a dated change note in its header and an entry in `changelog`.
- The README follows the schema of the organization workspace: no versions, archive URLs or requirement tables. The catalog page has them.
- Passing tests do not replace testing on real hardware. Never describe the driver as validated on a monitor.
