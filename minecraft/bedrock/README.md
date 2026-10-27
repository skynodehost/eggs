# Vanilla Bedrock (NetherNet)

Skynode build of the parkervcp Vanilla Bedrock egg, patched for the NetherNet
transport that Bedrock Dedicated Server 26.51+ requires. Nothing is added to
the server volume: the install script and startup command are the stock ones,
and all NetherNet settings are written by the panel's config-file parser.

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

Two keys are added to the egg's `server.properties` find block, which Wings
applies before every start:

| Key | Value |
|---|---|
| `transport` | `{{server.build.env.TRANSPORT}}` (default `nethernet`) |
| `server-udp-ports` | `{{server.build.default.ip}}:{{server.build.default.port}}:{{server.build.default.port}}` |

The primary allocation is therefore used for both signaling (TCP) and gameplay
(UDP), and the advertised address is the public allocation IP. Wings publishes
every allocation on both protocols, so nothing needs opening on the node.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `TRANSPORT` | `nethernet` | Staff only. `raknet` is an emergency switch and does not work with current clients. |

## Port budget

Mojang's server documentation says to reserve one UDP port per connected
player. This egg advertises only the primary allocation port. If testing shows
that a second player cannot join while the first is connected, the fix is to
advertise every allocation assigned to the server. That cannot be expressed in
an egg alone: `{{server.build.default.*}}` only exposes the primary allocation.
The clean path is a panel-side environment variable (for example
`SERVER_UDP_PORTS`, built from the server's allocations in the panel's
`pterodactyl.environment_variables` config) that the find block can then
reference as `{{server.build.env.SERVER_UDP_PORTS}}`.

## Rolling out

1. Import `egg-vanilla-bedrock.json` in the panel (Admin, Nests, Import Egg)
   or update the existing Vanilla Bedrock egg from it.
2. Restart each Bedrock server. No reinstall is needed; Wings rewrites
   `server.properties` on start. Servers that still have `./start.sh` as their
   startup command must be set back to `./bedrock_server` first.
3. Confirm `server.properties` shows `transport=nethernet` and a
   `server-udp-ports` line that starts with the node's public IP.
4. Test a join from a 26.60 client.
