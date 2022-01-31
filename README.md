# Home Assistant - Setup

Although it would be possible to install HA Supervised on one RPI with all features, it would not be standard and would require alot of tweaks, and I want to keep the setup as standard and future proof as possible (we tried for instance [Install Home Assistant Supervised on RPI](https://peyanski.com/how-to-install-home-assistant-supervised-official-way/) on a single server).

We could also utilize one server with HASS.io and addons for MariaDB, InfluxDB and Grafana, but that would most likely not be future proof as the load will increase as more data is gathered.

Therefore we have gone for a two-server setup according to below.

## Table of content

- [Generic information](https://github.com/slittorin/home-assistant-setup#generic-information)
- [Governing principles](https://github.com/slittorin/home-assistant-setup#governing-principles)
- [Conceptual design](https://github.com/slittorin/home-assistant-setup#conceptual-design)
- [Setup for Server 1](https://github.com/slittorin/home-assistant-setup#setup-for-server-1)
  - [Preparation](https://github.com/slittorin/home-assistant-setup#preparation)
  - [Installation for InfluxDB](https://github.com/slittorin/home-assistant-setup#installation-for-influxdb)
  - [Installation for Grafana](https://github.com/slittorin/home-assistant-setup#installation-for-grafana)
- [Setup for Home Assistant](https://github.com/slittorin/home-assistant-setup#setup-for-home-assistant)
  - [Setup MariaDB](https://github.com/slittorin/home-assistant-setup#setup-mariadb)
  - [General setup](https://github.com/slittorin/home-assistant-setup#general-setup)
  - [History DB setup](https://github.com/slittorin/home-assistant-setup#history-db-setup)

## Generic information

For all changes to Home Assistant configuration files, you usually need to restart:
-  Goto `Configuration` -> `Settings` -> `Server Controls` and press `Check Configuration`.
   - The output should state 'Configuration valid'. If not, change the recorder config above.
   - On the same page press `Restart` under `Server management`.
- Any warnings or errors can be found in the file `/config/home-assistant.log`.

## Governing principles

#### Generic

- Keep to standard setup/config as much as possible.
- Limit the number of exposed ports/services on the Home Assistant.

#### Historical data

As of [2021.8.0](https://www.home-assistant.io/blog/2021/08/04/release-20218/#long-term-statistics) Home Assistant seems to be on the way to store and manage historical data, but is not yet there, including to graphically present historical data that extends the purge period for the database.\
Therefore we also add InfluxDB (to capture states) and Grafana to present historical data.

#### Database retention and history

- Allow 30 days of data to reside within the Home Assistant database (MariaDB) before it is put into the history database.
  - We have a rather good setup that should cope with the load and volume of data, with the current number of sensors/integrations.
  - Purge is set on recorder.
- For history (in MariaDB, not InfluxDB):
  - The data is stored in the following tables:
    - `statistics` and `statistics_short_term` tables:
      - Added to HA in [2021.8.0](https://www.home-assistant.io/blog/2021/08/04/release-20218/#long-term-statistics).
      - Currently the data is not purged in these tables (not sure about `statistics_short_term` yet).
- Keep all data forever in the history database (InfluxDB).
  - By default the retention is set to 'Forever' in InfluxDB.

#### Backup

- [GitHub Repository home-assistant-config](https://github.com/slittorin/home-assistant-config) is utilized for configuration files (push, on demand).
- Backup Home Assistant with snapshots (includes MariaDB database) according:
  - Daily snapshots, keep for 7 days (monday through saturday).
  - Weekly snapshots (sunday), keep for 8 weeks.
- We backup MariaDB separately according:
  - Daily snapshots, keep for 7 days (monday through saturday).
  - Weekly snapshots (sunday), keep for 8 weeks.
- We backup history database (InfluxDB) according:
  - Daily snapshots, keep for 7 days (monday through saturday).
  - Weekly snapshots (sunday), keep for 8 weeks.
- Files from the above are moved to my NAS for storage (old files deleted).

## Conceptual design

- homeassistant on VLAN-Server (192.168.3.20):
  - RPI 4 with 250GB SSD disk. Standard HASS.io install, with the following addons:
    - Setup a RHASS.io with [HASS.io install](https://github.com/slittorin/hass-io-install).
- server1 on VLAN-Server (192.168.3.30):
   - RPI 4 with 500GB SSD disk. Standard RPI, with docker and the following services:
     - InfluxDB.
     - Grafana.
   - Intended also to be utilized for other projects.
   - Setup a RPI instance with [Raspberry PI install](https://github.com/slittorin/raspberrypi-install/).

# Setup for Server 1

## Preparation

1. Under `/srv`:
   - Create the file `.env`.
   - Create the file `/srv/docker-compose.yml` with the following content:
     ```
     version: '3'
     
     services:
     
     volumes:
     ```

## Installation for InfluxDB

1. Check versions of available docker images for InfluxDB at [Docker - InfluxDB](https://hub.docker.com/_/influxdb).
   - If you do not want the 'latest' version, use version number.
   - At time of writing (20220207) the 'latest' version is 2.1.1 (isolated with `sudo docker image inspect influxdb` and looking for 'INFLUXDB_VERSION').
2. Create the directory `/srv/ha-history-db`, and the following sub-directories:
   - `backup`.
4. For the file `/srv/.env` add the following content:
```
HA_HISTORY_DB_HOSTNAME=localhost
HA_HISTORY_DB_ROOT_USER=admin
HA_HISTORY_DB_ROOT_PASSWORD=[not shown here]
HA_HISTORU_DB_ROOT_TOKEN=[not shown here]
HA_HISTORY_DB_ORG=lite
HA_HISTORY_DB_BUCKET=ha
```
4. For the following file `/srv/docker-compose.yml` add the following content after 'services:' and last added service (keep spaces):
```
# Service: Home Assistant history database.
# -----------------------------------------------------------------------------------
  ha-history-db:
# Add version number if necessary, otherwise keep 'latest'.
    image: influxdb:latest
    container_name: ha-history-db
# We run in host mode to be able to be connected from HA.
    ports:
      - "8086:8086"
    restart: always
    env_file:
      - .env
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=${HA_HISTORY_DB_ROOT_USER}
      - DOCKER_INFLUXDB_INIT_PASSWORD=${HA_HISTORY_DB_ROOT_PASSWORD}
      - DOCKER_INFLUXDB_INIT_ORG=${HA_HISTORY_DB_ORG}
      - DOCKER_INFLUXDB_INIT_BUCKET=${HA_HISTORY_DB_BUCKET}
    volumes:
      - "ha-history-db-data:/var/lib/influxdb"
      - "ha-history-db-config:/etc/influxdb"
      - "/srv/ha-history-db/backup:/backup"
```
5. For the following file `/srv/docker-compose.yml` add the following content after 'volumes:' and last added volume (keep spaces):
```
  ha-history-db-data:
  ha-history-db-config:
```
6. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the InfluxDB-container with `sudo docker-compose up -d`. The output should look like the following:
   ```shell
   Creating network "srv_default" with the default driver
   Creating volume "srv_ha-history-db-data" with default driver
   Creating volume "srv_ha-history-db-config" with default driver
   Creating ha-history-db ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                    NAMES
   5a8f45730d6d   influxdb:latest          "/entrypoint.sh infl…"   33 seconds ago   Up 31 seconds   0.0.0.0:8086->8086/tcp   ha-history-db
   ```
7. Create the following backup-script `/srv/ha-history-db/backup-influxdb.sh` to take InfluxDB-backup through docker-compose ls(remember to set `chmod ugo+x`).
```bash
#!/bin/bash

# Inspired by: https://gist.github.com/mihow/9c7f559807069a03e302605691f85572
#
# Purpose:
# This script backs up full influx according to:
# - Daily snapshots, keep for 7 days (monday through saturday).
# - Weekly snapshots (sunday), keep for 8 weeks.
#
# Usage:
# ./backup-influxdb.sh

# Load environment variables (mainly secrets).
if [ -f "/srv/.env" ]; then
    export $(cat "/srv/.env" | grep -v '#' | sed 's/\r$//' | awk '/=/ {print $1}' )
fi

# Variables:
day_of_week=$(date +%u)
container="ha-history-db"
influxdb_logfile="/srv/ha-history-db/backup-influxdb.log"
influxdb_backup_root="/srv/ha-history-db/backup/backup.tmp"
influxdb_backup_container_root="/backup/backup.tmp"
influxdb_backup_dest="/srv/ha-history-db/backup/"
day_of_week=$(date +%u)

# Set name and retention according day of week.
# Default is daily backup
influxdb_backup_pre="influxdb-backup-daily"
retention_days=7
if [[ "$day_of_week" == 7 ]]; then # On sundays.
    influxdb_backup_pre="influxdb-backup-weekly"
    retention_days=56 # 8 weeks.
fi
influxdb_backup_filename="${influxdb_backup_pre}-$(date +%Y%m%d_%H%M%S)"

_initialize() {
    echo ""
    echo "$(date +%Y%m%d_%H%M%S): Starting InfluxDB Backup."

    touch "${influxdb_logfile}"
    rm -r "${influxdb_backup_root}/"
    mkdir "${influxdb_backup_root}"
}

_influxdb_backup() {
    echo "$(date +%Y%m%d_%H%M%S): Backing up..."
    docker-compose exec "${container}" influx backup "${influxdb_backup_container_root}" -t "${HA_HISTORY_DB_ROOT_TOKEN}"

    echo "$(date +%Y%m%d_%H%M%S): Compressing Backup..."
    tar_file="${influxdb_backup_dest}${influxdb_backup_filename}.tar"
    tar -cvf "${tar_file}" "${influxdb_backup_root}/"
    echo "$(date +%Y%m%d_%H%M%S): Compressed backup to: ${tar_file}"

    echo "$(date +%Y%m%d_%H%M%S): Backup done."
}

_influxdb_cleanup() {
    find "${influxdb_backup_dest}" -name "${influxdb_backup_pre}-*" -mtime +${retention_days} -delete
    echo "$(date +%Y%m%d_%H%M%S): Done retention cleanup to ${retention_days} days for filenames starting with ${influxdb_backup_pre}-"
}

_finalize() {
    echo "$(date +%Y%m%d_%H%M%S): Finished InfluxDB Backup."
    exit 0
}

# Main
_initialize >> "${influxdb_logfile}" 2>&1
_influxdb_backup >> "${influxdb_logfile}" 2>&1
_influxdb_cleanup >> "${influxdb_logfile}" 2>&1
_finalize >> "${influxdb_logfile}" 2>&1
```
8. Create the following crontab entry with `sudo crontab -e` to run the script each day at 00:00:01: `1 0 * * * /srv/ha-history-db/backup-influxdb.sh`.
9. Verify that the crontab is correct with `crontab -l` (run in the context of user 'pi').
10. Wait to the day after and check the log-file `/srv/ha-history-db/backup-influxdb.log` /and backup-directory `/srv/ha-history-db/backup` so that backups are taken.

## Installation for Grafana

1. Check versions of available docker images for Grafana at [Docker - Grafana](https://hub.docker.com/r/grafana/grafana).
   - If you do not want the 'latest' version, use version number, or use 'main'.
   - At time of writing (20220207) the 'latest' version is 8.3.3 (isolated with running the command `/usr/share/grafana/bin/grafana-server -v` on the container).
2. Create the directory `/srv/ha-grafana`, and the following sub-directories:
   - At present no specific directories are used.
4. For the file `/srv/.env` add the following content:
```
HA_GRAFANA_HOSTNAME=localhost
```
4. For the following file `/srv/docker-compose.yml` add the following content after 'services:' and last added service (keep spaces):
```
# Service: Home Assistant grafana.
# -----------------------------------------------------------------------------------
  ha-grafana:
# Add version number if necessary, otherwise keep 'latest'.
    image: grafana/grafana:latest
    container_name: ha-grafana
# We run in host mode to be able to be connected from HA.
    ports:
      - "3000:3000"
    restart: always
    env_file:
      - .env
    environment:
# This will allow you to access your Grafana dashboards without having to log in and disables a security measure that prevents you from using Grafana in an iframe.
      - GF_AUTH_DISABLE_LOGIN_FORM=true
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_SECURITY_ALLOW_EMBEDDING=true
    volumes:
      - "ha-grafana-data:/var/lib/grafana"
      - "ha-grafana-config:/etc/grafana"
```
5. For the following file `/srv/docker-compose.yml` add the following content after 'volumes:' and last added volume (keep spaces):
```
  ha-grafana-data:
  ha-grafana-config:
```
6. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the InfluxDB-container with `sudo docker-compose up -d`. The output should look like the following:
   ```shell
   Creating volume "srv_ha-grafana-data" with default driver
   Creating volume "srv_ha-grafana-config" with default driver
   ha-history-db is up-to-date
   Creating ha-grafana ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                    NAMES
   5a8f45730d6d   influxdb:latest          "/entrypoint.sh infl…"   33 seconds ago   Up 31 seconds   0.0.0.0:8086->8086/tcp   ha-history-db
   304599875ff0   grafana/grafana:latest   "/run.sh"                33 seconds ago   Up 31 seconds   0.0.0.0:3000->3000/tcp   ha-grafana
   ```

# Setup for Home Assistant.

For all changes to Home Assistant configuration files, you usually need to restart:
-  Goto `Configuration` -> `Settings` -> `Server Controls` and press `Check Configuration`.
   - The output should state 'Configuration valid'. If not, change the recorder config above.
   - On the same page press `Restart` under `Server management`.
- Any warnings or errors can be found in the file `/config/home-assistant.log`.

## Setup MariaDB

1. Go to `Configuration` -> `Add-ons, Backups & Supervisor` -> Click on the `Add-on store` at the lower right corner, and install the following add-ons (always set start on boot, watchdog to restart and update automatically):
   - `MariaDB`:
     - Configure the add-on:
       - Set `Option` and `password` to a password specific for the database.
       - We do not set any port as we do not want the database to be exposed outside the host.
2. Through the `File Editor` add-on, edit the file `/config/secrets.yaml` and add (change the string 'password' below to the right password):
   `recorder_db_url: mysql://homeassistant:password@core-mariadb/homeassistant?charset=utf8mb4`
3. Through the `File Editor` add-on, edit the file `/config/configuration.yaml` and add:
   ```
   recorder:
     db_url: !secret recorder_db_url
   ```
4. Verify the config-file and restart.
5. Verify that the setup is working correct by looking in the Home Assistant logfile.

## General setup

1. Through a web-browser logon as administrator to the installed Home Assistant.
2. Click on the name of the logged in user at the lower left corner:
   - Enable `Advanced mode`.
3. Go to `Configuration` -> `Add-ons, Backups & Supervisor` -> Click on the `Add-on store` at the lower right corner, and install the following add-ons (always set start on boot, watchdog to restart and update automatically):
   - `File Editor`:
     - We want to be able to edit files in the web-browser.
   - `Terminal & SSH`:
     - We want to be able to logon with ssh (logon-user is `root`).
     - Configure the add-on:
       - Set `Option` and `password` to a password specific for ssh-login (yes, not preferred, one should use authorized key instead).
       - Set `Network` to 22.
     - Restart the add-on.
4. We enable to add resources to lovelace:
   - With `File Editor` create the following directories:
     - `/config/www`.
5. For readability, as will have lots of configuration data, we create separate yaml-files:
   - With `File Editor` create the following directories:
     - `/config/packages`.
     - `/config/custom_components`.
   - With `File Editor` add the following files in directory `/config`:
     - `sensors.yaml`
6. Through the `File Editor` add-on, edit the file `/config/configuration.yaml` and add at the bottom of the file:
     ```
     homeassistant:
       packages:
     ```
7. Through the `File Editor` add-on, edit the file `/config/configuration.yaml` and add before the rows with `!include`:
     ```
     sensor: !include sensors.yaml
     ```
8. We setup logging to log warning and above.
   - Through the `File Editor` add-on, edit the file `/config/configuration.yaml` and add:
     ```
     logger:
       default: warning
     ```
9. Setup Recorder correctly to keep data in database for 30 days, and write every 10:th second to the database to reduce load (even though we do not need it since we have an SSD disk instead of SD Card), and ensure that logbook is enabled:
   - Through the `File Editor` add-on, edit the file `/config/configuration.yaml` and add after `recorder:`:
     ```
       purge_keep_days: 30
       commit_interval: 10
       
     logbook:
     ```
10. Add areas that is representative for your home.
   - Go to `Configuration` -> `Devices and services` -> `Areas` and update the rooms/areas that represent your home.
11. Verify the config-file and restart.
12. Verify that the setup is working correct by looking in the Home Assistant logfile.

## History DB setup

1. In a web browser go the IP address (or hostname) of server1 and port 8086, for example [http://192.168.2.30:8086/](http://192.168.2.30:8086/).
   - Login with the username and password setup above.
   - Go to `Data` -> `API Tokens` -> `Generate API Token` -> `Read/Write API Token`:
     - Set a descriptive name: `Read/Write to HA bucket`
     - Choose bucket `ha` for both read and write.
   - Choose the newly created token and copy the token.
2. Through the `File Editor` add-on, edit the file `/config/secrets.yaml` and add (change the string 'token' below to the right token):
   `history_db_token: token`
3. Through the `File Editor` add-on, edit the file `/config/configuration.yaml` and add:
```

history:
  exclude:
    domains:
      # Updater: Is not required in history data.
      - updater
    entity_globs:
      # We do not include any of the test_-sensors.
      - binary_sensor.test_*
      - sensor.test_*

influxdb:
  api_version: 2
  ssl: false
  host: 192.168.2.30
  port: 8086
  organization: lite
  bucket: ha
  token: !secret history_db_token
  max_retries: 3
```
4. Verify that data is written to the InfluxDB_bucket with:
   - In web browser go the IP address (or hostname) of server1 and port 8086, for example [http://192.168.2.30:8086/](http://192.168.2.30:8086/).
     - Check that data is written to the ha-bucket.
     - If not, chase the problem.
