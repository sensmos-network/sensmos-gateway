# Community contributions

Everything under `community/` is contributed and maintained by its authors, **not by Sensmos**.
It may lag behind the supported script, break with the next release, or not fit your gateway.
Use it as a starting point, keep it working yourself, and send fixes as pull requests.

The supported way to run the listener is the systemd service on the gateway host
(`sensmos-gw-udp.py --install-service`).

| folder | what | kept by |
|---|---|---|
| `docker/` | run the listener in a container (host networking required) | community |

## Contributing

- One folder per contribution with its own `README.md` saying what it needs and who keeps it.
- Do not change the trust model: the listener stays passive, root-only for `AF_PACKET`, no self-update.
- Do not vendor or fork `sensmos-gw-udp.py` — reference the top-level file so fixes reach everyone.
