FROM python:3.12-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends nmap \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY scan_daemon.py .

# Runs as root — ARP scanning (-PR) needs raw socket access (CAP_NET_RAW).
# Combined with network_mode: host in docker-compose.yml, this gives the
# container real access to the host's LAN interface for ARP scanning.
CMD ["python", "scan_daemon.py"]
