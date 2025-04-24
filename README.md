# KiCAD PostgreSQL Database Library Setup

This guide walks through:

- provisioning a PostgreSQL database in Docker Compose,
- securing it with Docker Secrets and environment files,
- binding its data directory on the host,
- configuring KiCad clients to access it via an ODBC DSN and `.pgpass` password file.

## Requirements

- Docker & Docker Compose
- KiCad (v6 or later)
- Sudo privileges on the server and clients
- Ubuntu (or Debian-based) on client machines
- A directory on the server for PostgreSQL data storage

## Server side setup

### Create `.env` file

Use `templates/env.template` as a starting point.
Create a file named `.env` alongside `docker-compose.yaml`.

### Define PostgresSQL password

Create a secret file:

```shell
mkdir -p secrets
touch secrets/postgres_password.txt
chmod 600 secrets/postgres_password.txt
```

Place a password in that file.

### Prepare PostgreSQL directory for data

E.g. if you want to keep data in `/srv/pgdata` directory:

```shell
sudo mkdir -p /srv/pgdata
sudo chown root:docker /srv/pgdata
sudo chmod 770 /srv/pgdata
```

### Open database port

Example:

```shell
ufw allow from <client_CIDR> to any port <database_port>
```

### Launch database

```shell
docker compose up -d
```

### Verify

Verify the container is running and healthy:

```shell
docker compose logs -f
```

## Client side (Ubuntu)

### Install ODBC Drivers

```shell
sudo apt update
sudo apt install -y unixodbc odbc-postgresql
```

### Define DSN

Use `templates/odbc.ini.template` as a starting point.
Copy it to `/etc/odbc.ini` and customize.

```ini
[<dsn_name>]
Description=KiCad remote PostgreSQL
Driver=PostgreSQL Unicode
Servername=<server_address>
Port=<server_port>
Database=<database_name>
Username=<database_username>
```

Password is omitted on purpose and provided via `~/.pgpass`.

### Create password file `.pgpass`

In the user home directory, that will connect to the datavase, create `.pgpass` (or set PGPASSFILE environment variable to point to your preferred location) with the following content:

```text
server_address:server_port:database_name:database_username:password
```

Protect it:

```shell
chmod 600 ~/.pgpass
```

> [!WARNING]
> The file must be owner-readable only.

Verification:

```shell
isql -v <dsn_name>
```

or

```shell
psql -h <server_address> -p <server_port> -d <database_name> -U <database_username>
```

If successful, no password prompt should appear.

### KiCad Database Library

Copy `templates/kicad_pg_lib.kicad_dbl.template` to `somepath/kicad_pg_lib.kicad_dbl` and adjust the `dsn`.

Official example found here: <https://docs.kicad.org/9.0/en/eeschema/eeschema_advanced.html>

Ensure that:

```json
"source": {
    "type": "odbc",
    "dsn": "<dsn_name>",
    ...
}
```

Other fields should stay empty.

### Register database in KiCad

1. Open __Preferences__ > __Manage Symbol Libraries...__
2. Switch to __Global Libraries__
3. Add new row with:

    ```text
    Nickname: <e.g. Database>
    Library Path: path to the customized `kicad_pg_lib.kicad_dbl`
    Library Format: Database
    ```

### Test if works

1. Open __Schematic Editor__
2. Press __A__ or select __Place Symbols__
3. If the connection fails, KiCad will show a message, either it will mean it works.
4. If success, the database content should appear under set database nickname

## Optional: Viewing database

Use __pgAdmin__ or any PostgresSQL client to browse and manage the database.
