# Hosting Offline DAoC for friends

This is for one owner who wants a small number of trusted friends to connect to their own
server over the internet, alongside the owner's companion bots and autonomous gamebots. It's
not a guide to running a public server.

## The short version

Offline DAoC's server is a fork of OpenDAoC, a full multiplayer DAoC emulator. Nothing in the
server code limits it to one real player: a remote (non-loopback) client already goes through
the same login, timeout, link-death and combat rules as any other OpenDAoC server. The only
code that treats a connection differently is three small conveniences for the **owner's own
loopback client** (`127.0.0.1`), none of which apply to a friend connecting over the network:

- [ClientService.cs](../source/server/GameServer/ECS-Services/ClientService.cs) skips the hard
  ping timeout and the AFK/idle kick for the loopback connection only.
- [PlayerMovementComponent.cs](../source/server/GameServer/ECS-Components/PlayerMovementComponent.cs)
  skips the soft link-death check for the loopback connection only.

So opening the server to friends is a **hosting and configuration** task, not a code change.
Bot behavior doesn't need to change either: bots already defer to a real player's explicit
attack over their own target claims (see
[BotPvpCrowdControl.cs](../source/server/GameServer/bots/BotPvpCrowdControl.cs)), and use the
same defense formulas as players (see
[PlayerDefenseFormula.cs](../source/server/GameServer/propertycalc/PlayerDefenseFormula.cs)).

## 1. Network path: TrueNAS Scale VM → router → internet

1. Give the Windows 11 VM a **bridged** network adapter (not NAT-only) so it gets its own IP on
   your LAN, and set a DHCP reservation or static IP for it so the address never changes.
2. On your router, **port forward** these to the VM's LAN IP:
   - TCP `10300` (login / client connect port)
   - TCP `10400` (region/zone port)
   - UDP `10400` (region/zone port)
   Don't forward anything else. The optional web API (`Properties.ATLAS_API`, off by default) binds
   to `localhost:5000` and was never meant to be reachable remotely — leave it off and unforwarded.
3. On the VM itself, allow those same ports inbound in Windows Defender Firewall, since Windows
   VMs usually block unsolicited inbound connections by default.
4. Residential IPs change. Use a dynamic DNS hostname (e.g. a free DuckDNS or No-IP name) that
   updates to your current public IP, and give that hostname to your friends instead of a raw
   IP address.

## 2. Server config: `serverconfig.xml`

Classic OpenDAoC NAT pitfall: a friend can reach the login screen but can't enter the world,
because the zone server told their client to connect to `0.0.0.0` or a LAN-only address. Two
settings control this in `runtime/server/config/serverconfig.xml`:

- `<DetectRegionIP>True</DetectRegionIP>` — the server asks an external service for its own
  public IP and gives that to clients. This is usually enough and is the easiest option.
- `<RegionIP>` — only needed if auto-detection is unreliable; set it to your public IP or,
  better, your dynamic DNS hostname.

A ready-to-copy starting point for this setup is
[serverconfig.multiplayer.example.xml](../source/server/CoreServer/config/serverconfig.multiplayer.example.xml).
Copy it over `runtime/server/config/serverconfig.xml` on the VM and fill in the database
section the same way you already do today; don't commit the real file (it stays
git-ignored, same as `serverconfig.xml` always has been).

## 3. Accounts: turn off auto-creation, create friends by hand

With the server reachable from the internet, don't leave account creation open to anyone who
finds the address. Two settings gate it (`LoginRequestHandler.cs`), and either one alone is
enough to block it — set both for safety:

1. In `serverconfig.xml`: `<AutoAccountCreation>False</AutoAccountCreation>`.
2. In the world database (`runtime/data/opendaoc.sqlite3.db`), the server property
   `allow_auto_account_creation` should be `False`:
   ```sql
   UPDATE ServerProperty SET Value='False' WHERE `Key`='allow_auto_account_creation';
   ```
   Then run `/serverproperties refresh` in game, or restart the server, to pick it up.

With auto-creation off, create each friend's account yourself from your own GM session
(`/account create <accountName> <password>`, plvl 3 — your existing owner account already has
this). Pick a password per friend and share it with them privately; only a GM can change an
account's password afterward (`/account changepassword <accountName> <newPassword>`), so send
good passwords the first time.

Friends' accounts should stay at the default privilege level (plvl 1, regular player). Keep
GM/admin access to your own account only, exactly as it is today.

## 4. What doesn't need to change

- **Bot population.** Autonomous gamebots keep running at full population; they already treat a
  real player's claim as authoritative, so friends share the world with them like any other
  player would.
- **`MAX_PLAYERS`.** The default (4000) is harmless to leave as-is; it isn't your access control,
  closed account creation and your firewall are.
- **Idle/AFK kicking.** `KICK_IDLE_PLAYER_STATUS` already defaults to on behavior for every
  non-owner client once enabled; it was never disabled for remote players, only for your own
  loopback session.

## 5. Before you share the address

- Confirm the client edition matches what your friends will run: 0.35 and 0.35b use different
  `game.dll` builds tied to the `enable_sluaghbinder` database switch (see
  [docs/DEVELOPMENT.md](DEVELOPMENT.md)). Send them the matching client.
- Test from outside your LAN first (a phone on cellular data works) before giving out the
  address, to confirm the port forward and `serverconfig.xml` changes actually work end to end.
- Your server is still only up when your VM and PC are — tell friends it isn't a 24/7 service
  unless you choose to make it one.
