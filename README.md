# IPAM Module for Zabbix — Linux (Docker) Edition

Adds IP Address Management to Zabbix: subnet inventory, VLAN tracking,
IP reservations, CSV export, and automatic network scanning that
discovers live hosts and captures MAC addresses.

This edition is for a Zabbix stack running in Docker on a **native Linux
host** (bare metal, VM, or WSL2 running Docker Engine directly — not
Docker Desktop). On native Linux, Docker can give a container real
access to the host's network interface, so the scanner runs as just
another container alongside Zabbix.

> Running Zabbix via **Docker Desktop on Windows**? Use the
> `ipam-module-windows` package instead — Docker Desktop's hidden VM
> can't give containers real LAN access, so that edition runs the
> scanner natively on Windows.

---

## What's in this folder
```
zabbix-docker/
├── modules/ipampro/           The Zabbix frontend module (PHP)
│             └── sql/schema.sql         Database tables this module needs
└── scanner/                   The background scanning service
    ├── scan_daemon.py         Polls for scan requests, runs nmap, saves results
    ├── Dockerfile              Builds the scanner into a container image
    ├── requirements.txt
    └── compose-snippet.yml    Ready-to-paste service block for your docker-compose.yml
```

---

## How it works (30-second version)

Clicking **Scan** in Zabbix doesn't scan anything directly — it just adds
a request to a database table. A separate always-running container,
`ipam-scanner`, checks that table every 5 seconds, runs `nmap` when it
finds a request, and writes the results (live hosts + MAC addresses)
back into the database. Zabbix just displays whatever's currently in the
database.

MAC-address capture requires real access to your LAN, which normal
Docker networking doesn't allow — so the scanner container uses
`network_mode: host`, which is only reliable on native Linux Docker
hosts (this is why there's a separate Windows edition).

---

## Prerequisites

- Your Zabbix stack (`docker-compose.yml`) already up and running, with
  services named `mysql-server`, `zabbix-server`, and `zabbix-web` (as
  in a standard `zabbix/zabbix-server-mysql` + `zabbix/zabbix-web-*`
  setup)
- Root/`sudo` access on the Docker host
- No manual Python/nmap install needed — both are built into the
  scanner's own container image automatically

---

## Installation

All commands below are run from the same directory as your
`docker-compose.yml`.


### 1. Mount the module into `zabbix-web`

Your `zabbix-web` service needs to see that folder. Add this line to
`zabbix-web`'s `volumes:` list in `docker-compose.yml`:

```yaml
  zabbix-web:
    build:
      context: .
      dockerfile: Dockerfile
    ...
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
      - ./zabbix-custom-ui/favicon.ico:/usr/share/zabbix/favicon.ico
      - ./zabbix-custom-ui/brand.conf.php:/usr/share/zabbix/local/conf/brand.conf.php:ro
      - ./zabbix-custom-ui/rebranding/identiqa.png:/usr/share/zabbix/rebranding/identiqa.png:ro
      - ./modules:/usr/share/zabbix/modules          # <-- ADD THIS LINE
```

Apply it:
```bash
docker compose up -d zabbix-web
```

### 2. Apply the module's database schema

The module needs its own tables in the `zabbix` database. Copy the
schema file into the MySQL container and run it:

```bash
sudo docker cp modules/ipampro/sql/schema.sql mysql-server:/tmp/schema.sql
sudo docker exec -i mysql-server sh -c "mysql -u zabbix -p'iqlab@2025' zabbix < /tmp/schema.sql"
```

Confirm it worked:
```bash
sudo docker exec -i mysql-server sh -c "mysql -u zabbix -p'iqlab@2025' zabbix -e \"SHOW TABLES LIKE 'ipam_scan_queue';\""
```
You should see one row back: `ipam_scan_queue`.
### 3. Give permission to ipampro Folder

  sudo chown -R 1997:1995 ./ipampro

  sudo chmod -R 775 ./ipampro

### 4. Enable the module inside Zabbix

Open Zabbix in your browser (this stack serves the UI through
`license-proxy` on port **8080**, e.g. `http://<your-server-ip>:8080`)
and log in, then:

1. Go to **Administration → General → Modules**
2. Click **Scan directory**
3. Find **IPAM** in the list and set its status to **Enabled**

The module should now appear in the Zabbix menu.

### 5. Expose MySQL to the scanner

Add a `ports:` entry to your existing `mysql-server` service so the
host-networked scanner container can reach it:

```yaml
  mysql-server:
    image: mysql:8.0
    container_name: mysql-server
    ...
    ports:
      - "127.0.0.1:3306:3306"        # <-- ADD THIS LINE
    ...
```

Apply it:
```bash
docker compose up -d mysql-server
```

### 6. Copy in the scanner and add its service block

```bash
cp -r /path/to/ipam-module-linux/scanner ./scanner
```

Open `scanner/compose-snippet.yml` and copy the `ipam-scanner:` service
block from it into your main `docker-compose.yml`, alongside your other
services (`zabbix-server`, `zabbix-web`, etc.). It already matches this
stack's DB name/user/password (`zabbix` / `zabbix` / `iqlab@2025`) — no
edits needed unless yours differ.

### 7. Build and start the scanner

```bash
docker compose up -d --build ipam-scanner
docker logs -f ipam-scanner
```

You should see:
```
[scan-daemon] starting, polling every 5s against 127.0.0.1:3306/zabbix (nmap: nmap)
```

### 8. Test it

In Zabbix, go to the IPAM section, add/select a subnet, and click
**Scan**. Watch the logs:
```bash
docker logs -f ipam-scanner
```
```
[scan] subnet=1 target=192.168.1.0/24
[scan] subnet=1 completed: nmap discovery scan completed - 28 used, 226 free, 27 with MAC captured.
```
Refresh the subnet's IP list in the UI — used/free status and MAC
addresses should now be populated.

---

## Uninstalling

```bash
docker compose stop ipam-scanner
docker compose rm -f ipam-scanner
```
Then remove the `ipam-scanner:` block from `docker-compose.yml`, and
the `./modules:/usr/share/zabbix/modules` line from `zabbix-web` (or
just disable **IPAM** under Administration → General → Modules to keep
the data but hide it from the UI).

---

## Troubleshooting

**"Table doesn't exist" errors** — the schema wasn't applied. Re-run
step 3.

**IPAM doesn't show up in the menu after enabling it** — hard-refresh
the browser (Ctrl+Shift+R), and confirm the volume mount in step 2 is
actually present:
```bash
docker exec zabbix-web ls /usr/share/zabbix/modules
```
should list `ipampro`.

**Scan completes but 0 MACs captured / few hosts found** — test nmap
directly inside the scanner container:
```bash
docker exec ipam-scanner nmap -sn -PR -n 192.168.1.0/24
```
If this shows `MAC Address:` lines, everything's fine — a previous low
count may just mean fewer devices were online at that moment. If it
shows 0 MACs, confirm the container really has host networking and the
right capabilities:
```bash
docker inspect ipam-scanner --format '{{.HostConfig.NetworkMode}} {{.HostConfig.CapAdd}}'
```
Should print: `host [NET_RAW NET_ADMIN]`

**Can't connect to `127.0.0.1:3306`** —
```bash
docker ps --filter name=mysql-server
```
should show `127.0.0.1:3306->3306/tcp` in the `PORTS` column. If it's
missing, re-check step 5 and re-run `docker compose up -d mysql-server`.

**Scan stays "queued" forever** — the daemon isn't running or can't
reach the database:
```bash
docker logs ipam-scanner
```
Look for a connection error and re-check steps 5–7.

---

## Notes

- Nothing scanned is ever deleted on a rescan — an IP that stops
  responding is just marked `free`, not removed. See the module's own
  `modules/ipampro/README.md` for feature-level details (reservations,
  CSV export, VLANs).
- The default DB password (`iqlab@2025`) matches this stack's
  `docker-compose.yml` — change it in both places if you change it.
