# sensmos-gateway — passive listener for LoRaWAN gateways

For LoRaWAN gateways that have **no local MQTT broker** — any box running a plain Semtech
packet forwarder pointing straight at a remote server. If your gateway runs
ChirpStack Gateway Bridge, use `sensmos-gw-mqtt.py` ([sensmos-gw-mqtt.md](sensmos-gw-mqtt.md), untested)
instead.

This one is **entirely passive**. It does not receive your traffic, relay it, or sit between
any two components. It watches packets your forwarder is already exchanging with its own
servers over the loopback interface, and reports the radio metadata to
[Sensmos](https://sensmos.com).

**Nothing on your gateway changes.** No configuration file is edited, no service is
restarted, no server list is modified. Whatever your gateway runs keeps working byte for
byte as before. Kill this process and your gateway does not notice.

It never transmits — there is no code path that could. It does **report** what your gateway
transmits on its own (coverage beacons and the like), but only reads that from the
traffic already there; it never causes a transmission.

---

## Quick start (zero to a running service)

Run all of this **on the gateway** (not on your Mac/PC). You need a Sensmos node on the same
network and its PIN.

```sh
# 1 — install the script
sudo curl -fsSL -o /usr/local/bin/sensmos-gw-udp.py https://raw.githubusercontent.com/Galusz/sensmos-gateway/main/sensmos-gw-udp.py
sudo chmod +x /usr/local/bin/sensmos-gw-udp.py

# 2 — find a Sensmos node on this network (prints the exact pair command)
sudo sensmos-gw-udp.py --find

# 3 — pair with it (use the address/PIN from step 2; PIN is the node's PIN)
sudo sensmos-gw-udp.py --pair NODE_IP --pin YOUR_PIN --name "my gateway"

# 4 — install & start it as a boot-persistent service
sudo sensmos-gw-udp.py --install-service

# 5 — confirm it's up
systemctl status sensmos-gw-udp --no-pager
```

Step 5 should show **`active (running)`**. The service is named **`sensmos-gw-udp`**; watch its
log with `journalctl -u sensmos-gw-udp -f` — you'll see `connection works — first batch accepted`
once frames start flowing.

**Already paired but no service?** You don't pair again — the token survives. Just re-run
steps 1 and 4. If `--install-service` says `pair first`, then and only then repeat step 3.

The rest of this page explains each step in detail.

---

## Why passive rather than a relay

The obvious way to tap a packet forwarder is to put something in the middle — point the
forwarder at us, and forward onwards. We deliberately do not do that. On hardware that works for the network,
our outage would become your outage, and that is not a trade we are willing to offer.

The forwarder already sends every frame to each configured server separately, over loopback.
We simply read along.

---

## Install

```sh
sudo curl -fsSL -o /usr/local/bin/sensmos-gw-udp.py https://raw.githubusercontent.com/Galusz/sensmos-gateway/main/sensmos-gw-udp.py
sudo chmod +x /usr/local/bin/sensmos-gw-udp.py
```

Python 3 only — no pip packages, no compilers, nothing to build. It needs root because
reading raw packets requires `CAP_NET_RAW`.

---

## Look before you trust us

Before pairing anything, see exactly what would leave your box. This sends nothing and needs
no account:

```sh
sudo sensmos-gw-udp.py --dry-run
```

It prints the frames it would forward, as JSON, and stops there. Add `--log-tx` to also see
when your gateway transmits.

---

## Pair it with a Sensmos node

The agent has no credentials of its own. A Sensmos node on the same LAN vouches for your
gateway, and only then does the backend issue a narrow, revocable token.

```sh
sudo sensmos-gw-udp.py --pair 192.168.1.50 --pin 123456 --name "rooftop RAK"
```

The token is written to `/etc/sensmos-gw-udp.json` with mode `0600` and is scoped to
`radio.frames` — reporting received frames, nothing else. Revoke it from the Sensmos app at
any time and the agent immediately becomes inert.

---

## Run

```sh
sudo sensmos-gw-udp.py
```

| flag | meaning |
|---|---|
| `--iface` | interface to watch, default `lo` — forwarder and servers talk over loopback |
| `--dry-run` | print what would be sent, send nothing |
| `--log-tx` | also log what your gateway transmits |
| `--be <url>` | alternative backend, for testing |

As a service — one command, after pairing:

```sh
sudo sensmos-gw-udp.py --install-service
```

It copies the script to `/usr/local/bin`, writes the unit (shown to you first), enables it,
and starts it. The service is named **`sensmos-gw-udp`** — check it and follow its log with:

```sh
systemctl status sensmos-gw-udp
journalctl -u sensmos-gw-udp -f
```

`active (running)` means it's up and will also start on boot. In the log you'll see
`[sensmos] connection works — first batch accepted` once frames start flowing, then a
status line every minute.

If you prefer to write the unit yourself, it is just:

```ini
# /etc/systemd/system/sensmos-gw-udp.service
[Unit]
Description=Sensmos passive gateway listener
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/sensmos-gw-udp.py
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

---

## What it reads

Semtech GWMP over UDP, recognised **by packet header rather than by port number**, so it
works no matter which ports your stack happens to use:

| byte | |
|---|---|
| 0 | protocol version, 1 or 2 |
| 1–2 | token |
| 3 | type — `0` = PUSH_DATA (frames received), `3` = PULL_RESP (gateway asked to transmit) |
| 4–11 | gateway EUI |
| rest | JSON |

From `PUSH_DATA` we take the `rxpk` entries — frequency, spreading factor, RSSI, SNR, size,
CRC result, and the raw PHY payload. Application payloads are encrypted with keys we do not
have; what we use is the radio metadata, and recognising Sensmos beacons among the traffic.

`PULL_RESP` — a transmission your gateway is about to make — is reported **only when it is
audible to others** (`ipol = false`, e.g. coverage beacons some networks transmit). A normal LoRaWAN
downlink uses inverted IQ that no one else can hear, proves nothing, and is skipped. The
reported ones are proof the gateway is real: because their payload is unpredictable, a node
that hears the same payload independently confirms your gateway actually transmitted it —
something a mere counter could never establish. Only the radio metadata and payload bytes
are sent; the app payload stays encrypted end-to-end, as with received frames.

Nothing else is read. The listener ignores every packet that is not GWMP, and it never
touches interfaces other than the one you point it at.

### Deduplication is not optional here

A multi-server forwarder sends the identical frame to every configured server, so the same
transmission appears several times on loopback. Your own forwarder log shows it plainly:

```
# RF packets received by concentrator: 1
# PUSH_DATA datagrams sent: 2
```

The listener collapses these, so one transmission is reported once.

---

## Troubleshooting

**`root required: run with sudo`**
Raw packet capture needs `CAP_NET_RAW`. Either use `sudo`, or
`sudo setcap cap_net_raw+ep $(readlink -f $(which python3))` — note that grants the capability
to your whole Python interpreter, so `sudo` is usually the cleaner choice.

**Connected but no frames**
Check that traffic is really on loopback:

```sh
sudo tcpdump -i lo -n -c 10 udp
```

If your forwarder talks to a remote server over the network rather than to a local one, point
the listener at that interface instead: `--iface eth0`.

**`AF_PACKET is Linux only`**
It is. This runs on the gateway, not on your laptop.

---

## Uninstall

```sh
sudo systemctl disable --now sensmos-gw-udp
sudo rm /etc/systemd/system/sensmos-gw-udp.service /usr/local/bin/sensmos-gw-udp.py
sudo rm /etc/sensmos-gw-udp.json
```

Revoke the token in the Sensmos app too. Nothing else was ever changed, so there is nothing
else to undo.

---

## Docker and containers

The listener sniffs `AF_PACKET` on the **host's** loopback. Loopback is private per network
namespace, so an isolated container sees an empty `lo` and hears nothing. A container needs
`--network host` and `--cap-add NET_RAW`. The supported way is the systemd service on the host;
a container build is a community contribution — see `community/docker/`.

---

## Updates

The script never updates itself — by design. It runs as root on your gateway, inside your LAN;
auto-pulling code from a server would make that server a way into your network. To update,
repeat the install line and restart the service:

```sh
sudo curl -fsSL -o /usr/local/bin/sensmos-gw-udp.py https://raw.githubusercontent.com/Galusz/sensmos-gateway/main/sensmos-gw-udp.py
sudo systemctl restart sensmos-gw-udp
```

---

## Status

Running on foreign gateways since August 2026 (first one in Germany, paired through a node
without a radio of its own). Frames it hears are delivered like frames heard by any node;
`PULL_RESP` coverage beacons are reported as transmit proofs. Tell us what you see on
[Discord](https://discord.gg/ukea386Kqx).

---

## Repository layout

| path | status |
|---|---|
| `sensmos-gw-udp.py` | **supported, tested** — passive listener for Semtech packet forwarders (this page) |
| `sensmos-gw-mqtt.py` + `sensmos-gw-mqtt.md` | **untested** — agent for ChirpStack Gateway Bridge (local MQTT); complete, never run on a live gateway |
| `community/` | contributions kept by their authors (Docker etc.) — not supported by Sensmos |

Guides on the site: https://sensmos.com/docs/gateway/ · contract: https://sensmos.com/docs/attachments/

MIT — see `LICENSE`.
