# weewx-brultech

A WeeWX driver, database schema and report skins for the Brultech GEM energy monitors. The driver polls the monitor over the network, stores energy, current, pulse counts and temperatures in a database of its own, and the skins show current, daily, weekly, monthly and yearly energy use.

It is written for the GEM and does not configure the monitor beyond the setup described below.

Continuation of [tkeffer/weewx-brultech](https://github.com/tkeffer/weewx-brultech) by Tom Keffer, which has seen no changes since 2022. The _SeasonsPower_ skin goes back to work by Matthew Wall. See `changelog` for what changed.

## Requirements

- A Brultech GEM, reachable over the network, set to server mode (see below).
- No Python packages beyond WeeWX.

The extension keeps its data in a separate SQLite database (`weepwr.sdb`), not in a weather database. The installer makes it the archive of the station. Weather skins do not work with the data from the monitor.

## Installation

Install `brultech` from the [WeeWX packages catalog](https://github.com/weewx-packages/catalog/blob/main/packages/brultech.md).
The catalog page lists the current release, its WeeWX and Python
requirements, and the install, update and removal commands.

The installer sets `station_type` to `Brultech`, adds the `[Brultech]` section, the `bt_binding` database binding, the service `user.brultech.BrultechService`, and the reports _SeasonsPowerReport_ (enabled) and _StandardPowerReport_ (disabled). Then:

1. __Set the monitor to server mode.__
   Using a browser, connect to the monitor and open the "Application Settings" page.
   Set it to server mode, using port 8000:

   ![Application settings](images/server_mode.png)

   Click Apply. Then open the "Device Management" tab and click "Restart" under _Restart Module_:

   ![Restart Module](images/restart_module.png)

2. __Set the address of the monitor.__
   Run `weectl station reconfigure`. It asks for the host (name or IP address) and the port of the monitor.
   You can also edit `host` and `port` in section `[Brultech]` of `weewx.conf`.

3. __Turn off the weather report.__
   A new station has the report _SeasonsReport_ enabled. Set `enable = false` for it. It cannot show the data of an energy monitor.

4. __Restart WeeWX.__

Databases that hold only accumulated energy (the oldest style of this extension) can get the delta energy columns with the SQLite script `util/brultech/calc_deltas` in this repository. The header of the script has the details.

## Configuration

The installer writes these settings. This is the smallest working set:

```ini
[Station]
    station_type = Brultech

[Brultech]
    driver = user.brultech
    [[socket]]
        host = 192.168.1.104
        port = 8000

[DataBindings]
    [[bt_binding]]
        database = bt_sqlite
        table_name = archive
        manager = weewx.manager.DaySummaryManager
        schema = user.gem_schema.schema

[Databases]
    [[bt_sqlite]]
        database_name = weepwr.sdb
        database_type = SQLite

[StdArchive]
    data_binding = bt_binding

[Engine]
    [[Services]]
        data_services = user.brultech.BrultechService

[StdReport]
    [[SeasonsPowerReport]]
        skin = SeasonsPower
        enable = True
        data_binding = bt_binding
        [[[Units]]]
            [[[[Groups]]]]
                group_energy2 = kilowatt_hour
```

### Options in `[Brultech]`

| Option          | Default            | Meaning                                                                                                                                |
|-----------------|--------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `driver`        | `user.brultech`    | The driver module.                                                                                                                     |
| `connection`    | `socket`           | How to reach the monitor. Only `socket` exists.                                                                                        |
| `poll_interval` | `10`               | Seconds between polls of the monitor.                                                                                                  |
| `max_channels`  | `32`               | How many current channels to read. The hardware has up to 48, the schema 32.                                                           |
| `packet_type`   | `GEMBin48NetTime`  | `GEMBin48NetTime` (with the clock of the monitor, used for time synchronization) or `GEMBin48Net`. See the limitations for `GEMAscii`. |
| `stale`         | `1800`             | Seconds. A previous record older than this is not used to calculate power and delta energy. Keep it above the archive interval.       |
| `max_tries`     | `3`                | Read by the driver but not used. Retries are fixed at three.                                                                           |

In `[[socket]]`:

| Option       | Default           | Meaning                                                             |
|--------------|-------------------|---------------------------------------------------------------------|
| `host`       | `192.168.1.104`   | Host name or IP address of the monitor.                             |
| `port`       | `8000`            | Port of the monitor.                                                |
| `timeout`    | `20`              | Seconds to wait on the connection.                                  |
| `send_delay` | `0.2`             | Seconds to wait after a command before looking for the response.    |

The empty section `[[sensor_map]]` is not used.

### Database schema

The schema is read when the database is created. To change it, edit `bin/user/gem_schema.py` of your station before the first start, or delete `weepwr.sdb` afterwards to start over. Reducing the channel counts saves space.

| Option                          | Default | Meaning                                                                   |
|---------------------------------|---------|---------------------------------------------------------------------------|
| `MAX_CURRENT_CHANNELS`          | `32`    | How many current channels to include in the schema. Max for GEM is 32.    |
| `MAX_TEMPERATURE_CHANNELS`      | `8`     | How many temperature channels to include in the schema. Max for GEM is 8. |
| `MAX_PULSE_CHANNELS`            | `4`     | How many pulse (counter) channels to include. Max for GEM is 4.           |
| `INCLUDE_ACCUMULATED_ABSOLUTE`  | `True`  | Store accumulated (since device reset) absolute energy.                   |
| `INCLUDE_ACCUMULATED_POLARIZED` | `False` | Store accumulated (since device reset) polarized energy.                  |
| `INCLUDE_DELTA_ABSOLUTE`        | `True`  | Store delta absolute energy.                                              |
| `INCLUDE_DELTA_POLARIZED`       | `False` | Store delta polarized energy.                                             |
| `INCLUDE_POWER_ABSOLUTE`        | `False` | Store the average absolute power consumed during archive period.          |
| `INCLUDE_POWER_POLARIZED`       | `False` | Store the average polarized power consumed during archive period.         |

Power is calculated from the matching delta energy when a report asks for it, so the power columns are not needed for the skins.

### Choosing what the skin shows

The monitor reports every channel, whether or not anything is connected, so unused channels appear in the database as zeros. Limit them with `max_channels`, and in the `[DisplayOptions]` section of the skin (`observations_current`, `observations_stats`, `plot_groups`), which can be overridden in `weewx.conf`.

### Observation types

`N` is the channel number, without a leading zero (`ch2_amp`, not `ch02_amp`).

| Type                  | Meaning                                                                                  |
|-----------------------|------------------------------------------------------------------------------------------|
| `dateTime`            | Time stamp, unix epoch time                                                              |
| `serial`, `ser_no`    | Serial number of the unit, formatted and as a number                                     |
| `unit_id`             | Unit identification number                                                               |
| `secs`                | Continuous device uptime                                                                 |
| `time_created`        | The clock of the monitor. Only with packet type `GEMBin48NetTime`                        |
| `ch1_volt`            | Line voltage. The monitor reports one                                                    |
| `chN_amp`             | Current in channel `N`                                                                   |
| `chN_count`           | Pulse count, channels 1 to 4                                                             |
| `chN_temperature`     | Temperature, channels 1 to 8. A reading outside +/-255 is dropped                        |
| `chN_a_energy2`       | Accumulated absolute energy, in watt-seconds                                             |
| `chN_p_energy2`       | Accumulated polarized energy                                                             |
| `chN_ad_energy2`      | Absolute energy over an archive period (delta energy)                                    |
| `chN_pd_energy2`      | Polarized energy over an archive period                                                  |
| `chN_a_power`         | Average absolute power over an archive period, calculated from delta energy              |
| `chN_p_power`         | Average polarized power over an archive period                                           |

For example, `ch14_p_energy2` is the accumulated polarized energy in channel 14, and `ch2_ad_energy2` the absolute energy used in channel 2 during the archive interval.

### Accumulated and delta energy

The monitor reports accumulated energy, the energy that has flowed through the channel since the monitor was reset. The driver turns two consecutive values into delta energy, the energy used in one archive interval.

To get the energy used over a period, either subtract the accumulated values at its ends, or sum the delta energies. For channel 8 and a month:

| Strategy                           | Tag                         |
|------------------------------------|-----------------------------|
| Calculate using accumulated energy | `$month.ch8_a_energy2.diff` |
| Calculate using delta energy       | `$month.ch8_ad_energy2.sum` |

The skins use accumulated energy for the totals and delta energy for the bar plots.

Accumulated energy needs two values from the database, where delta energy needs all records of the period. It is also unaffected if WeeWX is down while the monitor keeps counting. Delta energy is the better choice if the monitor was reset or its battery failed, because the counters then start at zero and the difference across the reset is negative.

`.diff` of accumulated energy is empty (N/A) for a period with fewer than two records.

## Limitations

- Not tested on a real monitor. The tests use a fake monitor that serves packets built from the packet layout in the driver. Passing tests do not show that the layout matches your device.
- Only the binary packet types are expected to work. `GEMAscii` is untested. It returns `chN_energy2` and `chN_power` without the `_a_`, and without `dateTime` and `usUnits`, which the schema and the skins do not use.
- Only the monitor in server mode is supported, and the driver always polls it.
- Reports have shown unconnected temperature inputs as 32 &deg;F. The driver stores what the monitor reports.
- The skin _SeasonsPower_ bundles the fonts Open Sans and Kanit in `skins/SeasonsPower/font`. `OFL.txt` matches the notice inside the Kanit files (SIL Open Font License 1.1). The Open Sans TTF files (version 1.10) carry the notice "Licensed under the Apache License, Version 2.0", but the shipped `license.txt` is the SIL Open Font License, and no Apache license text is included. This is unresolved. The skin does not use Kanit.
- Aggregated power plots, such as the week, month and year plots of the skin, query the database once per point and per channel. Generating all plots of 32 channels takes a while on a small computer.

## Development

With WeeWX and pytest in the Python environment:

```shell
python -m pytest -q tests
```

## License

Copyright (c) 2013-2022 Tom Keffer. The skin _SeasonsPower_ is Copyright (c) 2018-2022 Tom Keffer and Matthew Wall. Licensed under the GNU General Public License, version 3. See `LICENSE.txt`. Changes since 2026-10-23 are listed in `changelog`.
