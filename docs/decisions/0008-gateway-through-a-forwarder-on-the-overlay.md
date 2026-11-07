---
status: accepted
date: 2026-11-08
decision-makers: thc1006
---

# Let the lab's LLM gateway reach the service through a forwarder on the overlay network's interface, on the one port its access policy allows

## Context and Problem Statement

The lab's members are to use this service through the lab's LLM gateway, which runs on another machine, with the keys they already have. Until now the service answered only on `127.0.0.1:8080` of this machine, and had no API key.

What was found on 2026-11-08, from this machine's firewall rules and a packet capture on `wt0` (neither is in `results/`: they hold the overlay's addresses):

- The campus computing centre runs this machine's own network, and no further TCP port can be opened on it.
- The gateway and this machine are both on an overlay network, a WireGuard VPN, whose interface here is `wt0`. What the overlay carries is encrypted, so a port inside it is not a port on the machine's own network.
- The overlay's access policy, which this machine's firewall applies to `wt0`, lets the gateway reach TCP port 8000 and ICMP. Connections from the gateway to other ports arrived on `wt0` and were dropped by that firewall, not on the way.
- A test listener on this machine's overlay address, on port 8000, took connections from the gateway and from a container on it. Nothing else listens on that port here.
- The same policy takes no new connection from this machine to the gateway, and the gateway's public HTTPS port is the only port of it that this machine can reach.

## Considered Options

1. A forwarder on `wt0`, on port 8000, to `127.0.0.1:8080`
2. `serve/server.py` itself on the overlay's address
3. A tunnel that this machine opens to the gateway, over SSH or HTTPS
4. Another port, with a change of the overlay's access policy

## Decision Outcome

Chosen option: 1, because it needs no new port on any network, no change of the access policy and no new software, and it leaves the service as it is.

- `systemd/strata-gateway-proxy.socket` listens on port 8000, bound to `wt0` (`BindToDevice=`), so only what arrives on the overlay's interface reaches it.
- It refuses every address (`IPAddressDeny=any`) but the gateway's. That address is not in this public repository: it is in a drop-in that only this machine has, `/etc/systemd/system/strata-gateway-proxy.socket.d/10-gateway-address.conf`. [Let the gateway in](../runbooks/change-config.md#let-the-gateway-in) gives the steps.
- `systemd/strata-gateway-proxy.service` runs `systemd-socket-proxyd`, which comes with systemd, as a dynamic user that can reach loopback only. It allows 32 connections at a time, well above what the gateway opens, because past the limit it resets a connection instead of queueing it, and a peer that vanishes with the overlay is found within 2 minutes (TCP keepalive), because the proxy has no timeouts of its own.
- The service has an API key now, one for the gateway and one for local use, in `"api_key"` of the live config. They are in no file of this repository.

### Consequences

- Good, because nothing listens on the machine's own network, and nothing is installed.
- Good, because three checks stand between a client and the service: the overlay's access policy, the socket's device and address list, and the API key.
- Good, because `strata-q3.service` does not change and does not restart. `sudo systemctl stop strata-gateway-proxy.socket` cuts the gateway off and leaves the service running.
- Bad, because the gateway's address is not in the repository: a rebuilt machine needs the drop-in by hand. Until then the socket refuses every address, and `scripts/check-drift.sh` reports the missing drop-in.
- Bad, because a client on this machine must now send a key: without one, `/v1/*` answers 401.
- Bad, because the port and the interface's name come from the overlay: a change of the access policy or of the interface's name means a change of the socket unit.
- Bad, because a restart of the overlay's client ends the connections through the forwarder: the socket stops with `wt0` and starts again with it.

### Confirmation

`scripts/check-drift.sh` compares both units with what systemd loaded, and requires this machine's drop-in, with one host's address and nothing else, without printing the address. `scripts/check-deployment.sh` reports a problem when the socket is not active and enabled.

## Pros and Cons of the Options

- Option 2: bad, because it changes the `ExecStart=` of `strata-q3.service`, so a change of it costs a restart of about 9 minutes (550–556 s for IQ3_S: `results/ab3-iq3s-*-switch.txt`); because the server has no list of the addresses that may call it; and because it would have to wait for the overlay's interface at each start.
- Option 3: bad, because the access policy takes no new connection to the gateway, so such a tunnel would have to go through the gateway's public port; it would add a new public way in, a long-running process and a credential on each machine; and it would go around the access policy instead of using what it allows.
- Option 4: bad, because it needs a change by the overlay's administrators and gains nothing: the number of the port is not what protects the service.

## More Information

Re-evaluate this decision when the overlay's access policy or the campus network's rules change.
