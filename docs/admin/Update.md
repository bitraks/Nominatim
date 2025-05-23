# Updating the Database

There are many different ways to update your Nominatim database.
The following section describes how to keep it up-to-date using
an [online replication service for OpenStreetMap data](https://wiki.openstreetmap.org/wiki/Planet.osm/diffs)
For a list of other methods to add or update data see the output of
`nominatim add-data --help`.

!!! important
    If you have configured a flatnode file for the import, then you
    need to keep this flatnode file around for updates.

### Installing the newest version of Pyosmium

The replication process uses
[Pyosmium](https://docs.osmcode.org/pyosmium/latest/updating_osm_data.html)
to download update data from the server.
It is recommended to install Pyosmium via pip.
Run (as the same user who will later run the updates):

```sh
pip3 install --user osmium
```

### Setting up the update process

Next the update process needs to be initialised. By default Nominatim is configured
to update using the global minutely diffs.

If you want a different update source you will need to add some settings
to `.env`. For example, to use the daily country extracts
diffs for Ireland from Geofabrik add the following:

    # base URL of the replication service
    NOMINATIM_REPLICATION_URL="https://download.geofabrik.de/europe/ireland-and-northern-ireland-updates"
    # How often upstream publishes diffs (in seconds)
    NOMINATIM_REPLICATION_UPDATE_INTERVAL=86400
    # How long to sleep if no update found yet (in seconds)
    NOMINATIM_REPLICATION_RECHECK_INTERVAL=900

To set up the update process now run the following command:

    nominatim replication --init

It outputs the date where updates will start. Recheck that this date is
what you expect.

The `replication --init` command needs to be rerun whenever the replication
service is changed.

### Updating Nominatim

Nominatim supports different modes how to retrieve the update data from the
server. Which one you want to use depends on your exact setup and how often you
want to retrieve updates.

These instructions are for using a single source of updates. If you have
imported multiple country extracts and want to keep them
up-to-date, [Advanced installations section](Advanced-Installations.md)
contains instructions to set up and update multiple country extracts.

#### One-time mode

When the `--once` parameter is given, then Nominatim will download exactly one
batch of updates and then exit. This one-time mode still respects the
`NOMINATIM_REPLICATION_UPDATE_INTERVAL` that you have set. If according to
the update interval no new data has been published yet, it will go to sleep
until the next expected update and only then attempt to download the next batch.

The one-time mode is particularly useful if you want to run updates continuously
but need to schedule other work in between updates. For example, you might
want to regularly recompute postcodes -- a process that
must not be run while updates are in progress. An update script refreshing
postcodes regularly might look like this:

```sh
#!/bin/bash

# Switch to your project directory.
cd /srv/nominatim

while true; do
  nominatim replication --once
  if [ -f "/srv/nominatim/schedule-maintenance" ]; then
    rm /srv/nominatim/schedule-maintenance
    nominatim refresh --postcodes
  fi
done
```

A cron job then creates the file `/srv/nominatim/schedule-maintenance` once per night.

##### One-time mode with systemd

You can run the one-time mode with a systemd timer & service.

Create a timer description like `/etc/systemd/system/nominatim-updates.timer`:

```
[Unit]
Description=Timer to start updates of Nominatim

[Timer]
OnActiveSec=2
OnUnitActiveSec=1min
Unit=nominatim-updates.service

[Install]
WantedBy=multi-user.target
```

`OnUnitActiveSec` defines how often the individual update command is run.

Then add a service definition for the timer in `/etc/systemd/system/nominatim-updates.service`:

```
[Unit]
Description=Single updates of Nominatim

[Service]
WorkingDirectory=/srv/nominatim-project
ExecStart=/srv/nominatim-venv/bin/nominatim replication --once
StandardOutput=journald
StandardError=inherit
User=nominatim
Group=nominatim
Type=simple

[Install]
WantedBy=multi-user.target
```

Replace the `WorkingDirectory` with your project directory. `ExecStart` points
to the nominatim binary that was installed in your virtualenv earlier.
Finally, you might need to adapt user and group names as required.

Now activate the service and start the updates:

```
sudo systemctl daemon-reload
sudo systemctl enable nominatim-updates.timer
sudo systemctl start nominatim-updates.timer
```

You can stop future data updates while allowing any current, in-progress
update steps to finish, by running `sudo systemctl stop
nominatim-updates.timer` and waiting until `nominatim-updates.service` isn't
running (`sudo systemctl is-active nominatim-updates.service`).

To check the output from the update process, use journalctl: `journalctl -u
nominatim-updates.service`


#### Catch-up mode

With the `--catch-up` parameter, Nominatim will immediately try to download
all changes from the server until the database is up-to-date. The catch-up mode
still respects the parameter `NOMINATIM_REPLICATION_MAX_DIFF`. It downloads and
applies the changes in appropriate batches until all is done.

The catch-up mode is foremost useful to bring the database up to date after the
initial import. Give that the service usually is not in production at this
point, you can temporarily be a bit more generous with the batch size and
number of threads you use for the updates by running catch-up like this:

```
cd /srv/nominatim-project
NOMINATIM_REPLICATION_MAX_DIFF=5000 nominatim replication --catch-up --threads 15
```

The catch-up mode is also useful when you want to apply updates at a lower
frequency than what the source publishes. You can set up a cron job to run
replication catch-up at whatever interval you desire.

!!! hint
    When running scheduled updates with catch-up, it is a good idea to choose
    a replication source with an update frequency that is an order of magnitude
    lower. For example, if you want to update once a day, use an hourly updated
    source. This ensures that you don't miss an entire day of updates when
    the source is unexpectedly late to publish its update.

    If you want to use the source with the same update frequency (e.g. a daily
    updated source with daily updates), use the
    once mode together with a frequently run systemd script as described above.
    It ensures to re-request the newest update until they have been published.


#### Continuous updates

!!! danger
    This mode is no longer recommended to use and will removed in future
    releases. systemd is much better
    suited for running regular updates. Please refer to the setup
    instructions for running one-time mode with systemd above.

This is the easiest mode. Simply run the replication command without any
parameters:

    nominatim replication

The update application keeps running forever and retrieves and applies
new updates from the server as they are published.
