# Docker (community, unsupported)

Starting point for people who keep their whole gateway stack in containers (multi-mining setups).

**Why it is not trivial:** the listener sniffs `AF_PACKET` on the **host's** loopback, where the
packet forwarder talks to its bridge on `127.0.0.1`. Loopback is private per network namespace —
an isolated container sees an empty `lo` and hears nothing. The container therefore needs
`network_mode: host` and `NET_RAW`. If your stack does not give containers host networking,
this cannot work passively; you would need the forwarder configured with a second server instead.

```sh
# pair on the host first (writes /etc/sensmos-gw-udp.json), then:
docker compose up -d
docker compose logs -f
```

`compose.yml` mounts the token file (the script renews the token in place, so not read-only) and the top-level script from this repository.
