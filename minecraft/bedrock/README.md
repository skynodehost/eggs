# Vanilla Bedrock (NetherNet)

Skynode build of the parkervcp Vanilla Bedrock egg, patched for the NetherNet
transport that Bedrock Dedicated Server 26.51+ requires.

## Why the stock egg stopped working

Bedrock 26.50 made NetherNet the default transport and 26.51 removed RakNet on
the server side. Bedrock 26.60 clients (27 October 2026) no longer connect over
RakNet at all. NetherNet works in two stages:

1. The client does HTTP signaling over **TCP** on `server-port`.
2. Gameplay runs over **UDP** on the ports listed in `server-udp-ports`. The
   server advertises that list, including the IP, to the client.

The stock egg never writes `transport` or `server-udp-ports`, and its install
script restores the old `server.properties` over the new one on every reinstall.
That produces two failures on Pterodactyl:

- Worlds created before 26.50 have no `transport` line. The server falls back to
  RakNet and logs `TRANSPORT TYPE ERROR ... Players will not be able to connect`.
- Newer worlds get `transport=nethernet` but keep the stock
  `server-udp-ports=19132`. Wings only publishes the allocation port, and the
  server advertises the container's bridge IP, so the server shows up in the
  list but every join times out.

## What this egg changes

- `transport` is managed by the panel (`TRANSPORT` variable, default
  `nethernet`).
- The install script writes `start.sh`, and the startup command runs it. On
  every boot the wrapper rewrites `server-udp-ports` as
  `<public ip>:<ports>:<ports>` using the primary allocation IP (`SERVER_IP`)
  and either the `UDP_PORTS` variable or, when that is empty, the primary
  allocation port. Duplicate `server-udp-ports` lines are collapsed to one
  because the server accumulates them.
- The wrapper logs the advertised addresses and warns when fewer UDP ports are
  advertised than `max-players`.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `TRANSPORT` | `nethernet` | Staff only. `raknet` is an emergency switch and does not work with current clients. |
| `UDP_PORTS` | empty | Gameplay UDP ports or ranges, comma separated, e.g. `19140-19149`. Each must be an allocation assigned to the server. Empty means the primary allocation port only. |
| `NETHERNET_PUBLIC_IP` | empty | Staff only. Overrides the advertised IP when the allocation IP is not public. |

## Port budget

Mojang's server documentation says to reserve one UDP port per connected
player. With the default (primary allocation only) that is one player at a
time. For a server with `max-players=10`, assign ten additional allocations in a
contiguous block and set `UDP_PORTS` to that range. Wings publishes every
assigned allocation on both TCP and UDP, so nothing else needs opening on the
node.

## Rolling out

1. Import `egg-vanilla-bedrock.json` in the panel (Admin, Nests, Import Egg)
   or update the existing Vanilla Bedrock egg from it.
2. Reinstall each Bedrock server once so `start.sh` is written. Worlds,
   allowlist and permissions are preserved by the install script.
3. Check the console for `[nethernet] Signaling: TCP ... Gameplay: UDP ...` and
   confirm the IP shown is the node's public address.
4. Test a join from a 26.60 client. If only the first player can join, assign
   more allocations and set `UDP_PORTS`.
