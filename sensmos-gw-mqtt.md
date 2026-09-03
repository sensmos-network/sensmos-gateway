> **UNTESTED.** This agent has not yet run against a live gateway with ChirpStack Gateway
> Bridge — we have no such gateway to verify it on. The code is complete and the backend
> accepts its format (the same `/v1/ext/radio/frames` the tested UDP listener uses), but treat it
> as a draft: start with `--dry-run`, and tell us on
> [Discord](https://discord.gg/ukea386Kqx) what you see. For plain Semtech packet forwarders use
> the tested `sensmos-gw-udp.py` — see `README.md`.

# Sensmos gateway listener

Turn a LoRaWAN gateway you already run into an extra pair of ears for the
[Sensmos](https://sensmos.com) network.

This is **receive only**. The agent never transmits, and it cannot be made to transmit —
there is no code path for it. It subscribes to your gateway's local MQTT broker as an
ordinary second client, normalises what it hears, and forwards the metadata to Sensmos.

**It is never in your data path.** Your gateway keeps talking to its own network server
exactly as before. If this agent crashes, is killed, or is uninstalled, your gateway does
not notice and nothing it does is delayed or dropped.

---

## Does it fit your gateway?

You need **ChirpStack Gateway Bridge** running locally. That covers ChirpStack Gateway OS,
most RAK and Dragino builds, Milesight, and Kerlink with the ChirpStack packet forwarder.

Quick check:

```sh
ss -lntup | grep :1883        # an MQTT broker is listening
systemctl status chirpstack-gateway-bridge
```

**Not a fit:** gateways running only the plain Semtech UDP forwarder pointed straight at a
remote server, or Basics Station talking directly to an LNS.
Those have no local MQTT to subscribe to. Get in touch — a UDP variant is a separate piece
of work and we would rather do it properly than have you improvise.

---

## Install

```sh
sudo apt install -y python3-pip
pip3 install paho-mqtt
pip3 install --no-deps chirpstack-api protobuf
sudo curl -fsSL -o /usr/local/bin/sensmos-gw-mqtt.py https://raw.githubusercontent.com/Galusz/sensmos-gateway/main/sensmos-gw-mqtt.py
sudo chmod +x /usr/local/bin/sensmos-gw-mqtt.py
```

`chirpstack-api` is optional but recommended. With it the agent reads Gateway Bridge's
default protobuf encoding, so **you change nothing on your gateway**. Without it you would
have to set `marshaler = "json"` in the bridge config and restart the service. `--no-deps`
skips `grpcio`, which the agent does not use and which is slow to build on ARM.

---

## Pair it with a Sensmos node

The agent has no credentials of its own. A Sensmos node on the same LAN vouches for your
gateway, and only then does the backend issue a narrow, revocable token.

You need a Sensmos node on the same network and its PIN.

```sh
sudo sensmos-gw-mqtt.py --pair 192.168.1.50 --pin 123456 --name "roof gateway"
```

The node asks the backend over its encrypted link, the backend replies, and the token is
written to `/etc/sensmos-gw-mqtt.json` with mode `0600`. The token is scoped to
`radio.frames` — reporting received frames, nothing else. You can revoke it at any time
from the Sensmos app, and the agent stops being able to do anything.

---

## Run

```sh
sensmos-gw-mqtt.py --broker localhost
```

Useful flags:

| flag | meaning |
|---|---|
| `--broker`, `--port` | MQTT broker (default `localhost:1883`) |
| `--user`, `--password` | broker credentials, if yours needs them |
| `--gateway <EUI>` | only this gateway, if several publish to one broker |
| `--be <url>` | alternative backend, for testing |

As a service — one command, after pairing:

```sh
sudo sensmos-gw-mqtt.py --install-service
```

It copies the script to `/usr/local/bin`, writes the unit (shown to you first), enables it,
and starts it. If your broker is not on `localhost`, pass `--broker HOST` on the same line and
it is baked into the service. The service is named **`sensmos-gw-mqtt`** — check it and follow
its log with:

```sh
systemctl status sensmos-gw-mqtt
journalctl -u sensmos-gw-mqtt -f
```

`active (running)` means it's up and will also start on boot.

If you prefer to write the unit yourself:

```ini
# /etc/systemd/system/sensmos-gw-mqtt.service
[Unit]
Description=Sensmos LoRaWAN gateway listener (MQTT)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/sensmos-gw-mqtt.py --broker localhost
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

The agent waits for the broker rather than failing, so service start order does not matter.

---

## What it sends, and what it does not

For every frame your concentrator demodulates, the agent forwards:

| field | |
|---|---|
| `ts` | reception time, when your gateway reports one |
| `freq`, `sf`, `mode` | channel in MHz, spreading factor, LoRa or FSK |
| `rssi`, `snr` | signal quality |
| `len`, `crc` | payload length, CRC result |
| `hex` | the raw PHY payload, first 128 bytes |

The PHY payload is what was in the air. **We cannot read it** — LoRaWAN application payloads
are encrypted with keys we do not have and do not want. What we use is the radio metadata,
plus recognising Sensmos beacons, which have their own distinct format.

The agent does **not** send your gateway's location, your broker credentials, your network
server details, or anything about the devices on your own network beyond what any receiver
within radio range already hears.

Frames heard by two gateways on the same broker are deduplicated, so one transmission is
counted once.

---

## Troubleshooting

**`cannot read uplink — set marshaler="json" …`**
`chirpstack-api` is missing and your bridge is publishing protobuf. Install it, or set
`marshaler = "json"` in `/etc/chirpstack-gateway-bridge/chirpstack-gateway-bridge.toml`
and restart the bridge.

**`connection refused (rc=5)`**
The broker requires authentication. Pass `--user` and `--password`.

**Connected, but nothing arrives**
Confirm the bridge is actually publishing:

```sh
mosquitto_sub -t '+/gateway/+/event/up' -v
```

If that is silent too, the problem is upstream of us — the concentrator or the bridge.

**`no token in /etc/sensmos-gw-mqtt.json`**
Pairing has not been done, or it was done as a different user. Run `--pair` with `sudo`.

---

## Uninstall

```sh
sudo systemctl disable --now sensmos-gw-mqtt
sudo rm /etc/systemd/system/sensmos-gw-mqtt.service /usr/local/bin/sensmos-gw-mqtt.py
sudo rm /etc/sensmos-gw-mqtt.json
```

Revoke the token in the Sensmos app as well. Nothing else was ever changed on your gateway,
so there is nothing else to undo.

---

## Status

New and tested end to end against ChirpStack Gateway Bridge protobuf messages, a live MQTT
broker and the production frame format — but not yet run for long on a stranger's gateway.
If something behaves oddly, we want to hear about it.
