# Server Setup — Home Assistant + Mosquitto on Arch Linux

This machine is the home server running the central hub for the terrace IoT system. Both Home Assistant and the Mosquitto MQTT broker run as Docker containers managed by a single Compose stack.

**Later:** a dedicated IoT gateway will be added on a separate network segment. The gateway will route IoT device traffic to this machine. No changes to this setup will be needed at that point — the MQTT broker address in each node's `secrets.yaml` will simply remain this machine's LAN IP.

---

## Prerequisites

### Docker

```bash
sudo pacman -S docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker   # apply group membership without logging out
```

### Tailscale (if not already installed)

```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up
```

---

## Directory Layout

```
terrace-iot/
├── compose.yaml
├── mosquitto/
│   ├── config/
│   │   ├── mosquitto.conf   # committed
│   │   └── passwd           # gitignored — created in step below
│   ├── data/                # gitignored — broker persistence
│   └── log/                 # gitignored — broker logs
└── homeassistant/
    └── config/              # partially gitignored — see .gitignore
```

---

## First-Time Setup

### 1. Create the Mosquitto password file

The broker requires authentication. Create a user (you will be prompted for a password):

```bash
docker run --rm -it \
  -v "$(pwd)/mosquitto/config:/mosquitto/config" \
  eclipse-mosquitto:2 \
  mosquitto_passwd -c /mosquitto/config/passwd YOUR_MQTT_USERNAME
```

To add more users later (omit `-c` to avoid overwriting the file):

```bash
docker run --rm -it \
  -v "$(pwd)/mosquitto/config:/mosquitto/config" \
  eclipse-mosquitto:2 \
  mosquitto_passwd /mosquitto/config/passwd ANOTHER_USERNAME
```

Record the username and password — you'll need them in `esphome/secrets.yaml`.

### 2. Find your LAN IP

```bash
ip route get 1 | awk '{print $7; exit}'
```

This is the address ESP32 nodes will use for `mqtt_broker` in `esphome/secrets.yaml`. It should be a stable address — consider setting a DHCP reservation for this machine in your router.

### 3. Update esphome/secrets.yaml

```yaml
mqtt_broker: "192.168.1.XXX"   # your LAN IP from above
mqtt_username: "YOUR_MQTT_USERNAME"
mqtt_password: "YOUR_MQTT_PASSWORD"
```

### 4. Start the stack

```bash
docker compose up -d
```

Check that both containers started:

```bash
docker compose ps
docker compose logs -f   # stream logs from both; Ctrl-C to stop
```

### 5. First-time Home Assistant setup

Open `http://localhost:8123` in a browser. HA will walk you through account creation and initial configuration. Complete that before connecting MQTT.

### 6. Connect HA to Mosquitto

1. **Settings → Devices & Services → Add Integration → MQTT**
2. Broker: `localhost` (HA uses host networking, so Mosquitto is reachable at localhost:1883)
3. Port: `1883`
4. Username / Password: what you set in step 1
5. Submit — HA will confirm the broker connection

Once connected, HA will auto-discover any ESPHome node that has published to the broker.

---

## Tailscale Subnet Router

This machine runs Tailscale and acts as a subnet router, making the home LAN reachable from all your tailnet devices. This lets you:

- Access the HA web UI (`http://<tailscale-ip>:8123`) from anywhere on the tailnet
- Push ESPHome OTA updates remotely via `uv run esphome run ...`
- Reach ESP32 node web UIs from tailnet devices

### Enable subnet routing

```bash
# Enable IP forwarding (persistent)
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# Advertise the home LAN — adjust the subnet to match your router
sudo tailscale up --advertise-routes=192.168.1.0/24
```

Then in the [Tailscale admin console](https://login.tailscale.com/admin/machines): find this machine and **approve the advertised route**.

### Verify

From another tailnet device (e.g. a laptop on a different network):

```bash
# Should reach HA
curl -s http://<tailscale-ip>:8123

# Should reach an ESP32 node web UI (once flashed and online)
curl -s http://192.168.1.XXX   # the node's LAN IP
```

---

## Routine Operations

| Task | Command |
|---|---|
| Start the stack | `docker compose up -d` |
| Stop the stack | `docker compose down` |
| View logs | `docker compose logs -f` |
| Restart HA only | `docker compose restart homeassistant` |
| Pull image updates | `docker compose pull && docker compose up -d` |
| Add MQTT user | see step 1 above, omitting `-c` |

---

## Future: IoT Gateway

When a dedicated IoT gateway is added on a separate network segment:

- IoT devices (ESP32 nodes) will be on a VLAN routed through the gateway
- The gateway will forward MQTT traffic from the IoT VLAN to this machine's LAN IP on port 1883
- **No changes needed here** — the broker IP in `esphome/secrets.yaml` remains this machine's LAN address
- Subnet routing for the IoT VLAN should be handled by the gateway, not this machine
