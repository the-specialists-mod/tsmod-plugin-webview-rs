# tsmod-plugin-webview-rs: specification

Status: draft, 2026-10-17. Nothing here is built yet.

A web overlay for The Specialists Rust port (`the-specialists-mod/the-specialistsmod-rs`): a
Chromium page drawn over the game that the player toggles with a key, and that server plugins
(AMX Mod X) can open, close and talk to.

The overlay is an optional add-on. The port never links Chromium; a player who doesn't install
the plugin gets nothing new and loses nothing.

## 1. Goals and non-goals

Goals:

* A toggleable in-game browser overlay on every client the port supports: FWGS (Linux, Windows,
  Wine) and the Steam HL25 client (Linux, Windows), 32- and 64-bit.
* Server control: an AMX Mod X plugin can show a page to one player, a team or everyone, hide
  it, and send it events. The page can send events back to the server.
* Page scripting: a page can register console commands the player can bind, subscribe to local
  game events (button presses such as `+attack` and `+use`, spawn, death, weapon changes), and
  query local state (own position and view direction, what the crosshair is on, which players
  are in view). See §7.
* Interception: a page can block, rewrite or hold client events before the game shows or sends
  them (kill feed, chat, centre and HUD text, menus, MOTD, pickups, weapon selection, buy and
  other client commands), answer the menus it took over, and hide HUD elements to draw its own.
  See §7.7–7.9.
* Player control: the player decides whether servers may open pages at all, and which ones.
* Shipped and updated on its own release cadence, so Chromium security updates don't need a port
  release.

Non-goals (v1):

* Rebuilding the port's VGUI menus inside the plugin. A page can take a menu over (§7.7) and
  answer it (§7.8), but the port keeps its own menus as the default.
* Letting a server run arbitrary JavaScript on the client.
* Letting a page act freely on the game: it can't run console commands, press or release
  buttons, or change view angles. It can only stop or rewrite what the player or server already
  did, within the limits of §7.10.
* Telling a page anything the player can't already see on screen (§7.10).
* IME and non-US keyboard layouts for typing into pages (see §6.3).
* The software renderer (`ref_soft`): the overlay needs OpenGL and turns itself off without it.
* macOS.

## 2. Architecture

```
 game process (32- or 64-bit)                         helper process (64-bit)
┌───────────────────────────────────┐               ┌─────────────────────────────────┐
│ client.dll/.so (port)             │               │ tsmod-webview-host               │
│  └─ client extension API (new) ───┼─ loads ──┐    │  CEF, windowless (off-screen)    │
│                                   │          │    │  one browser per overlay surface │
│ tsmod_webview<suffix> (this repo) │◄─────────┘    │  JS bridge `tsmod` (§7)          │
│  • draws the frame (raw GL)       │ shared memory │                                  │
│  • forwards input                 │◄══════════════│  BGRA frames + dirty rects       │
│  • TSExt messages ⇄ host          │  pipe/socket  │                                  │
│  • policy (allowlist, consent)    │◄─────────────►│  control + input + events        │
└───────────────┬───────────────────┘               └─────────────────────────────────┘
                │ TSExt user message ↓   ↑ client command `wv`
┌───────────────┴───────────────────┐
│ server: port + Metamod + AMXX     │
│  tsmod_webview.amxx (this repo)   │
└───────────────────────────────────┘
```

Why the browser runs in its own process:

* **32-bit Linux:** the Steam HL25 Linux client is 32-bit, and CEF has no 32-bit Linux builds.
  A 64-bit helper serves every client.
* **Crash isolation:** a crash or hang in Chromium can't take the game down.
* **No engine conflicts:** CEF owns its message loop, child processes and signal handlers,
  which conflict with the engine's SDL loop.
* **Smaller game process:** the game never maps CEF's ~200 MB of libraries.

## 3. Repository layout

```
Cargo.toml                 workspace
crates/
  protocol/                no unsafe: shared-memory layout, pipe messages, TSExt framing, policy types
  client/                  no unsafe: the plugin's logic (state, policy, input mapping, layout)
  client-dll/              cdylib `tsmod_webview`: the only unsafe in the client half (ABI, GL, shm)
  host/                    bin `tsmod-webview-host`: CEF (`cef` crate, tauri-apps/cef-rs)
  host-sys/                unsafe CEF/OS glue for the host, if `cef` doesn't cover something
amxx/
  scripting/tsmod_webview.sma
  scripting/include/tsmod_webview.inc
  scripting/tsmod_webview_demo.sma
  e2e/webview_e2e.sma
web/
  tsmod.d.ts               types for the page-side bridge
  demo/                    sample pages (scoreboard widget, vote, map browser)
xtask/                     build | dist | test | lint | e2e | cef-fetch
docs/specifications.md     this file
```

Rules, as in the port and tsmod-amxx-rs:

* `unsafe` lives only in `client-dll` and `host`/`host-sys`. Every other crate is
  `#![deny(unsafe_code)]`.
* rustfmt at 140 columns with `use_small_heuristics = "Max"`.
* **Licences:** the Rust crates, `web/` and the docs are MIT ("The Specialists Mod
  contributors"). `amxx/` builds against AMX Mod X's GPL includes, so it is GPLv3 or later, the
  same as tsmod-amxx-rs. CEF and Chromium licences ship as `THIRD_PARTY_LICENSES` in the host
  archive.

## 4. Port changes this needs

The port has no client-side plugin loading today. The server `plugin-api` only runs on the
server, and the client's `ClientPlugin` is a gameplay ruleset the server selects (one active at a
time). Two additions go in the port, filed as port issues before work starts here.

### 4.1 Client extension API (`crates/client-ext-api`, MIT)

A C ABI, modelled on `plugin-api`:

* **Loading:** at `Initialize`, `client.dll/.so` loads every `cl_dlls/tsmod_<name><suffix>`
  library, where the suffix is the client library's platform suffix. It calls the exported
  `TSRS_ClientExtLoad(api, ext)` once. The size and version checks are exact, the same as
  `plugin-api`: a library built for another version is skipped with a console warning.
* **Engine-thread rule:** the extension is called only on the engine thread, from inside a client
  export.
* **Callbacks the extension fills in** (`ClientExt`; all optional):

  | Callback | Called from | Notes |
  |---|---|---|
  | `init()` | `HUD_Init` | register commands and cvars here |
  | `vid_init()` | `HUD_VidInit` | new map or video mode; GL textures are gone |
  | `frame(time)` | `HUD_Frame` | |
  | `redraw(time, intermission)` | `HUD_Redraw`, after the port's HUD and VGUI | GL state saved and restored by the port |
  | `key_event(down, key, binding) -> swallow` | `HUD_Key_Event`, before the port | non-zero swallows the key |
  | `mouse_active(active)` | `IN_ActivateMouse` / `IN_DeactivateMouse` | |
  | `wants_cursor() -> bool` | `IN_*`, each frame | true releases the mouse to the cursor, as the port's VGUI menus do |
  | `user_message(channel, data, len)` | `TSExt` hook (§4.2) | only messages for channels the extension claimed |
  | `usercmd(cmd)` | `CL_CreateMove`, after the port has built the command | read-only copy: `buttons` (TS bit map, `button_bits`), view angles, `impulse`, `weaponselect` |
  | `game_event(kind, data)` | the port's HUD message handlers | spawn, death, weapon change, health/armour change, team change, map start (§7.3) |
  | `message_pre(name, data, len) -> Verdict` | the port's `on_message` dispatcher, before any handling | every port HUD user message, for observing and for interception (§7.7) |
  | `command_pre(argc, argv) -> Verdict` | each port-owned client command, before it runs | `slot1`–`slot10`, `invnext`, `invprev`, `buy`, `special`, `impulse`, `+commandmenu`, `togglebrowser`, `cancelselect`, ... |
  | `server_cmd_pre(cmd) -> Verdict` | wherever the port sends a command to the server | `weapon_*` from `hud::weapons::select_slot`, `menuselect`, `cutwire_*`, buy items |
  | `shutdown()` | `HUD_Shutdown` | |

* **Functions the port provides** (`ClientExtApi`):
  * `add_command`, `register_cvar`, `cvar_value`, `cvar_string`, `set_cvar_value`
    (`Cvar_SetValue`, which sets synchronously; see the HL25 userinfo note in §8.4)
  * `print`, `server_cmd`, `client_cmd`
  * `screen_size`, `mouse_position`
  * `gl_proc(name)` (the port's `goldsrc::ext::gl` loader), `renderer_kind` (GL, GLES or
    software)
  * `claim_channel(name)`, `server_address`, `game_dir`, `port_version`
  * read-only game state, for the game API (§7.4). Each returns a plain `#[repr(C)]` copy, not
    an engine pointer:
    * `local_state()`: alive, team, health, armour, eye origin, view angles, velocity, on
      ground, ducking, current weapon id, clip and reserve ammo, from the port's HUD state and
      `GetLocalPlayer`
    * `player(slot)`: name, team, frags, deaths, ping (`GetPlayerInfo` and the scoreboard),
      plus origin, alive and whether the entity was in this frame's packet (`curstate.messagenum`)
    * `trace_line(start, end, players_solid)`: `goldsrc::ext::render::player_trace` inside
      `with_solid_players`, returning fraction, end point and the hit entity index
    * `world_to_screen(point)`: `pTriAPI->WorldToScreen`, plus the current FOV
    * `weapon_name(id)`, `map_name`, `max_clients`

  GoldSrc has no call to remove a client command, so extensions register commands once and
  dispatch them themselves (§7.2).

* **Interception** (`Verdict`):
  * `Pass`, `Block`, or `Replace(buf, len)` (for messages, a new payload in the same wire format;
    for commands, a new command line).
  * `Defer(token)`, for presentation messages only (§7.7). The port keeps the payload, and the
    extension later calls `release(token, verdict)` with `Pass`, `Block` or `Replace`. The port
    then runs its normal display handler on that payload, in arrival order per message.
  * The port applies a message's state side effects at once and defers or blocks only its
    display. A blocked or held `DeathMsg` still marks the victim dead on the scoreboard at once;
    a held `SayText` plays its sound only when released, and a blocked one plays none.
  * A `Replace` the port's decoder rejects counts as `Block`, with a console warning.
  * Verdicts are ignored for the messages and commands §7.10 protects.
* **HUD control:** `hud_set_hidden(mask)` hides port HUD elements (§7.9) and `hud_elements()`
  lists them with their screen rectangles, so a page can place its own drawing there.
* **Menu answers:** `menu_reply(kind, choice)` sends the port's own answer to a text or VGUI
  menu the extension took over (`menuselect N`, `jointeam N`, a buy item), the same command the
  port's menu would have sent.
* **Test:** the port gets a stub extension and an `xtask` check on the client roots, as
  `xtask plugins` does for the server.

### 4.2 `TSExt`: a server-to-client channel message

AMXX can only send user messages the game library has registered. The port server registers one
generic message for extensions:

* name `TSExt`, variable length
* body: `string channel` (≤ 15 bytes), then up to 175 bytes of payload. The engine limit for a
  user message is 192 bytes.

The port client hooks `TSExt` and passes each message to the extension that claimed the channel.
Messages for an unclaimed channel are dropped silently, so servers can send them to vanilla port
clients safely. This repo uses the channel `webview`.

The port server also announces the API in the server's `SrvSett` so plugins can check support.

## 5. Components

### 5.1 `tsmod_webview` (client plugin)

* Built for the same four targets as the port client: linux-i686, linux-x86_64, windows-i686 and
  windows-x86_64. The i686 builds still start the 64-bit host.
* Starts the host lazily, on the first `webview_toggle`, the first server open, or `webview_home`
  at startup when `webview_preload 1` is set.
* Owns policy: the allowlist, the consent prompt and rate limits (§8). The host only does what
  the plugin tells it.
* **Drawing:**
  * One GL texture per visible surface.
  * Each frame, upload the dirty rects that changed since the last frame (`glTexSubImage2D`
    from shared memory), then draw a textured quad in `redraw`.
  * GLES renderers (FWGS `ref_gles`, gl4es) use the same path with GLES calls. On `ref_soft` the
    plugin prints once and stays off.
* Releases its GL objects in `vid_init` and `shutdown`.

### 5.2 `tsmod-webview-host` (helper)

* 64-bit only: linux-x86_64 and windows-x86_64 (also used under Wine).
* **CEF:** pinned to one stable CEF branch per release (`xtask cef-fetch` downloads and checks
  the Spotify CDN minimal distribution). Settings:
  * windowless rendering, BGRA `OnPaint`, `external_begin_frame` off
  * `windowless_frame_rate` set from `webview_fps` (default 30, maximum 60)
* **Lifetime:**
  * Started with the plugin's PID and the IPC names.
  * Exits when the pipe closes or the parent PID dies. On Linux it also uses
    `PR_SET_PDEATHSIG`; on Windows, a job object.
* **Profile:**
  * Cache and cookies live in `<game dir>/webview/profile/`, kept apart from any system browser.
  * Persistence is controlled by `webview_persist` (default 1).
* **Locked down:** downloads, popups and new windows (`OnBeforePopup` cancels), `file://`
  outside `web/` and the bundled pages, and `localhost`/private addresses unless
  `webview_allow_local 1` is set.
* **Sandbox:** on Linux, user namespaces if available, otherwise `--no-sandbox` with a console
  warning. On Windows, the normal CEF sandbox.
* **Under Wine:** `--disable-gpu --disable-gpu-compositing`, detected through `wine_get_version`.
* **Hidden surfaces:** `WasHidden(true)` throttles to near-zero CPU. The host keeps running unless
  `webview_idle_exit` seconds pass with nothing visible (default 300).

### 5.3 Surfaces and display modes

A surface is one browser. v1 allows two at a time: the player's own overlay, and one server
surface.

| Mode | Input | Use |
|---|---|---|
| `full` | takes keyboard and mouse; Esc or the toggle key closes it | browser, server pages |
| `panel` | as `full`, in a rectangle (percent of screen) | votes, shops, map pickers |
| `hud` | none (click-through, the game keeps input) | widgets: scoreboard strip, timers |

The `Esc` key always reaches the plugin first. A server can't capture a player in a page.

## 6. Client-side IPC and input

### 6.1 Shared memory

There is one region per surface, named `tsmod-webview-<pid>-<surface>` (`shm_open` on Linux, a
named file mapping on Windows):

```
header (64 B): magic "TSWV", version u16, width u16, height u16, stride u32,
               seq u32 (seqlock: odd while writing), front u8, dirty rects count u8,
               dirty rects [8] { x, y, w, h: u16 }
buffers[2]:    width × height × 4 bytes BGRA
```

The host writes the back buffer, then flips `front` under the seqlock. The plugin reads `front`;
if `seq` changed while it read, it uses the old texture this frame. When a page resizes, the host
recreates the region and sends `Resized` over the pipe.

### 6.2 Pipe messages

The pipe is a Unix domain socket on Linux and a named pipe on Windows. Messages are
length-prefixed `postcard` frames (`crates/protocol`), with a protocol version check on connect.

* **Plugin → host:**
  * surfaces: `Open { surface, url, mode, rect }`, `Navigate`, `Show`, `Hide`, `Close`, `Resize`
  * input: `Key { down, code, mods }`, `Char { ch }`, `Mouse { x, y, buttons }`, `Wheel { dy }`
  * from the server: `ServerEvent { name, json }`
  * game API (§7.2–7.4): `CommandRun { surface, name, args, down }`,
    `GameEvents { surface, events }` (one batch per frame), `QueryResult { surface, id, json }`,
    `Grants { surface, caps }`
  * interception (§7.7–7.9): `Held { surface, token, kind, fields }`,
    `MenuOpened { surface, token, kind, fields }`, `MenuClosed { token }`,
    `HudElements { elements }`
  * `Shutdown`
* **Host → plugin:**
  * `Ready`, `Resized`, `TitleChanged`, `LoadError`
  * `CursorChanged`
  * `PageEvent { surface, name, json }` (§7), `Crashed`
  * game API: `RegisterCommand { surface, name, button, help }`, `UnregisterCommand`,
    `Subscribe { surface, kinds }`, `Query { surface, id, what, args }`,
    `Watch { surface, id, what, interval_ms }`, `Unwatch`
  * interception: `SetRules { surface, rules }` (the whole rule set, replaced each time),
    `SetHolds { surface, holds }`, `Verdict { surface, token, verdict }`,
    `MenuReply { surface, token, choice }`, `HudHide { surface, elements }`

### 6.3 Input

* **Keys:** `HUD_Key_Event` gives engine key numbers (`K_*`), not characters. The plugin maps them
  to Windows virtual key codes for CEF, and turns printable keys plus the shift state into
  `Char` with a US layout. Other layouts and IME are out of scope for v1.
* **Paste:** `Ctrl+V` is passed through. CEF reads the system clipboard itself, but under
  X11/Wayland without a window this is unreliable; this is open question 3 in §12.
* **Mouse:**
  * While a `full` or `panel` surface has focus, `wants_cursor` releases the mouse.
  * `mouse_position` is read each frame and scaled to the surface.
  * Buttons arrive as `K_MOUSE1..5` and `K_MWHEELUP/DOWN` through `key_event`.
* **Swallowing:** the plugin swallows every key while an interactive surface has focus, except
  the console key, Esc and the toggle key.

## 7. Page bridge (`window.tsmod`)

The host injects the bridge into every page in an allowed origin (§8) and nowhere else. The
types ship as `web/tsmod.d.ts`.

### 7.1 Interface

```ts
interface TsmodBridge {
  version: number;                        // protocol version
  surface: "player" | "server";
  player: { name: string; slot: number }; // the local player
  server?: { address: string; name: string };
  caps: Cap[];                            // what this page was granted (§7.5)
  on(event: string, cb: (data: unknown) => void): void;   // server → page
  send(event: string, data?: unknown): void;              // page → server (server surface only)
  close(): void;                                          // asks the plugin to close this surface
  show(): void; hide(): void;                             // a hidden page may show() only from a command handler
  requestCaps(caps: Cap[]): Promise<Cap[]>;               // shows the consent card if needed
  commands: Commands;                     // §7.2, needs "commands"
  game: Game;                             // §7.3 and §7.4, needs "game.read" / "game.input"
  intercept: Intercept;                   // §7.7, needs "intercept.display" / "intercept.actions"
  menus: Menus;                           // §7.8, needs "intercept.display"
  hud: Hud;                               // §7.9, needs "hud"
}

type Cap = "commands" | "game.read" | "game.input" | "intercept.display" | "intercept.actions" | "hud";
```

* Event names match `[a-z0-9_.-]{1,31}`. Data is JSON, at most 4 KB after encoding.
* `send` is rate-limited to 4 per second and 16 per 10 seconds per surface. It is dropped with a
  console warning past that.
* The player's own surface has no `send`: personal pages never talk to game servers.
* Calls that need a capability the page doesn't hold reject with `TsmodError("cap")`.

### 7.2 Console commands

A page can add console commands, so the player can bind keys to it (`bind g wv_buy`):

```ts
interface Commands {
  // Adds `wv_<name>`. With button: true adds `+wv_<name>` and `-wv_<name>` instead,
  // and the handler gets down = true/false (for hold-to-show widgets).
  register(name: string, handler: (args: string[], down?: boolean) => void,
           opts?: { button?: boolean; help?: string }): boolean;
  unregister(name: string): void;
  list(): string[];
}
```

* **Names:** `[a-z0-9_]{1,24}`, always given the `wv_` prefix, so a page can never shadow an
  engine, port or alias command. `register` returns false for a bad or taken name.
* **Ownership:** a name belongs to one surface at a time; the first page to register it holds it
  until it unregisters, navigates away or closes. A server surface can't take a name the player
  surface holds.
* **No removal in the engine:** GoldSrc can't remove a client command (§4.1). The plugin adds each
  engine command the first time any page registers that name, and keeps a table from name to
  owner. A command with no owner prints `wv_<name>: no page handles this` and does nothing.
* **Limits:** 32 names per surface, 128 engine commands per game session; past that, `register`
  returns false.
* **Arguments:** `Cmd_Argv(1..)`, at most 8 arguments and 256 bytes in total.
* **Discovery:** `webview_commands` lists the commands, their owners and help text.
* **Runs while hidden:** commands reach a hidden surface too, so a bound key can show it (the
  handler can call `tsmod.show()`, which a hidden page may do only from inside a command
  handler).

### 7.3 Game events

```ts
interface Game {
  on(kind: GameEventKind, cb: (e: GameEvent) => void): void;   // subscribes; first on() per kind
  off(kind: GameEventKind, cb?: (e: GameEvent) => void): void;
  query<K extends QueryKind>(what: K, args?: QueryArgs[K]): Promise<QueryResult[K]>;  // §7.4
  watch<K extends QueryKind>(what: K, intervalMs: number, cb: (r: QueryResult[K]) => void): () => void;
}
```

| Kind | Cap | Payload | Source |
|---|---|---|---|
| `input` | `game.input` | `{ button, down }` | edges of `usercmd.buttons` between commands (§4.1 `usercmd`) |
| `spawn` | `game.read` | `{}` | `game_event` |
| `death` | `game.read` | `{ killer?: slot, weapon? }` | `game_event` (`DeathMsg` for the local player) |
| `weapon` | `game.read` | `{ id, name }` | `game_event` (current weapon changes) |
| `health` | `game.read` | `{ health, armour }` | `game_event` |
| `team` | `game.read` | `{ team }` | `game_event` |
| `map` | `game.read` | `{ map }` | `vid_init` |
| `aim` | `game.read` | the `aim` query result (§7.4) | the plugin, when the aimed-at player changes |

* **Buttons** for `input`: `attack`, `attack2`, `use`, `jump`, `duck`, `reload`, `alt1` (stunt),
  `alt2`, `forward`, `back`, `moveleft`, `moveright`, `left`, `right`, `score`. These are the bits
  of the port's `button_bits` table, so a `+attack` from a key, a mouse button or a script all
  look the same. The event says what the command asked for, not whether the server accepted it
  (a shot when out of ammo still fires `attack`).
* While an interactive surface has focus, the plugin swallows keys (§6.3), so no game buttons
  change and no `input` events fire.
* **Delivery:** the plugin collects events during a frame and sends one `GameEvents` batch. At most
  64 events per batch and 240 per second per surface; past that it drops the oldest `input`
  events and sets `dropped` on the next batch.

### 7.4 Queries

`query` is answered on the next client frame. `watch` re-runs a query every `intervalMs`
(at least 50 ms, so at most 20 per second) and calls back only when the result changed.

| Query | Cap | Result |
|---|---|---|
| `self` | `game.read` | `{ alive, team, health, armour, origin, angles, forward, velocity, speed, onGround, ducking, weapon: { id, name, clip, ammo } }` |
| `aim` | `game.read` | `{ hit: "none" \| "world" \| "player" \| "entity", distance, point, player?: { slot, name, team, sameTeam } }` |
| `players` | `game.read` | `{ count, players: [{ slot, name, team, sameTeam, distance, bearing, elevation, screen: [x, y] }] }`, visible players only |
| `scores` | `game.read` | the scoreboard: `[{ slot, name, team, frags, deaths, ping }]` |
| `view` | none | `{ width, height, fov, map }` |

Units are game units; angles are degrees (`angles` is pitch, yaw, roll as the engine gives them;
`forward` is the unit view vector).

* **`aim`:** a trace from the eye along the view angles, 8192 units, with players solid
  (`trace_line(.., players_solid = true)`). A hit on a player slot reports that player. TS
  servers don't send `StatusValue` (the port's `hud/statusbar.rs`), so there is no
  server-provided target ID to use instead.
* **`players`:** a player is visible when all of these hold:
  1. it's alive, isn't the local player, and was in this frame's packet (`messagenum`);
  2. it's inside the view frustum (`world_to_screen` succeeds and is on screen);
  3. a world-only trace from the eye reaches its head or its centre.

  `bearing` is the yaw from the view direction to the player (−180..180, positive left) and
  `elevation` the pitch difference. Teammates and enemies are both listed; `sameTeam` tells them
  apart.
* **Cost:** `players` is at most two traces per candidate, so 62 traces with 32 players. Results
  are cached for the frame, so many watches cost one evaluation.

### 7.5 Capabilities and consent

| Surface and origin | `commands` | `game.read` | `game.input` | `intercept.display` | `intercept.actions` | `hud` |
|---|---|---|---|---|---|---|
| bundled pages (`webview/pages/`) | yes | yes | yes | yes | yes | yes |
| server surface, server's FastDL origin | yes | yes | yes | yes | yes | yes |
| server surface, other allowed origin | yes | ask | ask | ask | ask | ask |
| player surface, origin in `webview_allowlist` | yes | ask | ask | ask | ask | ask |
| player surface, any other origin | no | no | no | no | no | no |

* "Ask" shows the consent card the first time `requestCaps` asks for it, with the same choices as
  §8.5. The card names each capability in plain words ("hide or change chat and kill messages",
  "stop weapon switches and menu commands"). Choices are stored in `webview/consent.json` by
  origin (and server address, for the server surface).
* `webview_game_api 0` on the client turns `game.read` and `game.input` off everywhere;
  `webview_intercept 0` turns off `intercept.display`, `intercept.actions` and `hud`.
* The server can turn any of them off too (§8.1 `policy`), per surface: a competitive server can
  keep game data and interception away from the player's own pages. The plugin obeys it until
  the player disconnects.
* A server-surface page with `game.read` can pass what it learns to the server with `send`. That
  tells the server nothing it doesn't already know.

### 7.6 Event catalogue

Every client event a page can see or stop, in one table. The kind name is used by `game.on`
(observe, §7.3), `intercept.rule` and `intercept.hold` (§7.7). Fields are the port's decoded
message, named in `web/tsmod.d.ts`.

Actions: **B** block, **R** rewrite, **H** hold (asynchronous verdict), **T** take over (§7.8),
**O** observe only.

**From the server** (user messages, through `message_pre`); observing needs `game.read`, acting
needs `intercept.display`:

| Kind | Messages | Fields | Actions |
|---|---|---|---|
| `chat` | `SayText` | `from` (slot, 0 for the server), `team`, `teamOnly`, `text` | B R H |
| `killfeed` | `DeathMsg` | `killer`, `victim` (slots), `weapon`, plus the TS flags the port decodes | B R H |
| `textmsg` | `TextMsg` | `dest` (`notify`, `console`, `talk`, `center`), `text`, `args` | B R H |
| `hudtext` | `HudText`, `TSMessage` | `text` and the message's position and colour fields | B R H |
| `objective` | `Objective`, `ShowObj` | `text`, `shown` | B R H |
| `award` | `TSPAward` | `award`, `value` | B H |
| `pickup` | `WeapPickup`, `AmmoPickup`, `ItemPickup` | `what` (`weapon`, `ammo`, `item`), `id` or `name`, `count` | B |
| `menu` | `ShowMenu` (text menus) | `keys` (valid choices), `time`, `text` | B R H T |
| `vguimenu` | `VGUIMenu` | `menu` (team select, class, buy, MOTD, ... by the port's menu ids) | B T |
| `motd` | `MOTD` | `text` (or the URL, when the text is one) | B T |
| `state` | `Health`, `Battery`, `TSHealth`, `TSArmor`, `CurWeapon`, `WeaponList`, `AmmoX`, `ClipInfo`, `WStatus`, `ScoreInfo`, `TeamInfo`, `TeamScore`, `TSState`, `TSCash`, `KFuPower`, `PwUp`, `PwUpTime`, `RoundTime`, `NextStunt`, `TSSpace`, `ActItems`, `TSFlag`, `Spectator`, `SetFOV`, `HideWeapon`, `Wires`, `ResetHUD`, `InitHUD`, `ServerName`, `GameTitle`, `TeamNames` | `message` and its decoded fields | O |
| `effect` | `ScreenFade`, `ScreenShake`, `TSFade`, `TSMBlur`, `Damage`, `PTakeDam`, `Geiger`, `TSBTime`, `TSSlowMo` | `message` and its decoded fields | O |

**From the player** (client actions, through `command_pre` and `server_cmd_pre`); observing needs
`game.input`, acting needs `intercept.actions`:

| Kind | Source | Fields | Actions |
|---|---|---|---|
| `weapon_select` | `slot1`–`slot10`, `invnext`, `invprev`, and the `weapon_*` command they produce | `slot`, `weapon` (name), `command` | B |
| `menu_select` | `menuselect N` from the port's text menu, VGUI menu answers | `menu`, `choice` | B |
| `buy` | `buy` and the buy menu's purchase commands | `item` | B |
| `command` | the other port client commands: `special`, `impulse`, `+commandmenu`, `togglebrowser`, `cancelselect`, `specmode`, `thirdperson`, `firstperson` | `name`, `args` | B |
| `chat_send` | `say`, `say_team` (open question 9) | `text`, `teamOnly` | B H, and R on the player surface only |
| `input` | button edges (§7.3) | `button`, `down` | O |

* **Not in the table, not reachable:** engine-only traffic (`svc_print`, kick and disconnect
  reasons, `svc_stufftext`, voice), `SrvSett`, `TSExt` itself, and every message the port doesn't
  hook. A message added to the port later is `O` until this table lists it.
* **Wire to catalogue:** the plugin keeps one decoder per message, the same layout as the port's
  `shared::protocol::hud` types, and checks it against the port's message table at `init`. A
  message whose size or registration doesn't match is passed through untouched and reported by
  `webview_status`.

### 7.7 Interception

The page runs in another process, at least a frame away, but a verdict is needed inside the game
callback. So there are two ways to intercept:

* **Rules** are declarative and evaluated synchronously in the plugin, with no page round trip.
  Every kind with B or R takes rules.
* **Holds** send the event to the page and wait for its verdict, up to a timeout. Only kinds with
  H take holds: presentation, where a short delay is harmless. Weapon selection, menu answers,
  buying and commands are rules only, because a delay there is felt.

```ts
interface Intercept {
  rule(r: Rule): string;                  // returns a rule id
  hold(kind: HoldKind, handler: (e: CatalogueEvent) => Verdict | Promise<Verdict>,
       opts?: { match?: Match; timeoutMs?: number }): string;
  remove(id: string): void;
  clear(): void;
  list(): (Rule | HoldInfo)[];
}

type Rule = {
  kind: InterceptKind;
  match?: Match;                          // all entries must match
  action: "block" | { rewrite: Rewrite } | "takeover";
  once?: boolean;                         // removed after its first match
  ttlMs?: number;                         // removed after this long
};

// Per field: exact value, or { contains }, { prefix }, { in: [...] }, { not: ... }.
// Slot fields also take "self", "teammate", "enemy".
type Match = Record<string, Value | { contains: string } | { prefix: string } | { in: Value[] } | { not: Value }>;

// Only text fields: chat/textmsg/hudtext/objective `text` and `args`, killfeed `weapon`,
// menu `text`. Slot fields (who said or did it) can't be rewritten.
type Rewrite = Record<string, string>;

type Verdict = "pass" | "block" | "takeover" | { rewrite: Rewrite };
```

* **Order:** for each event the plugin first checks the protected list (§7.10). Then it runs the
  server surface's rules, then the player surface's, in the order they were added. Then it runs
  the holds, server surface first. The first `block` or `takeover` ends the run; rewrites
  accumulate, and each later rule sees the rewritten fields.
* **Rewrites:** the plugin re-encodes the message with the port's encoder and hands it to the
  port as `Replace`. Text is limited to what the message carries (chat to 190 bytes). The port's
  colour codes are kept; other control bytes are stripped.
* **Holds:**
  * `timeoutMs` is 100 by default and at most 500. The total for one event, over all holds, is
    capped at 500 ms.
  * **Fail open:** on timeout, an exception in the handler, a crash, a navigation or a closed
    surface, the event passes unchanged. The plugin logs the timeout once per hold.
  * **Order:** a held event of a kind holds back later events of that kind, so chat and the kill
    feed never show out of order. At most 32 can queue per kind; past that, the oldest passes.
  * A hidden surface's holds still run.
* **Lifetime:** rules and holds belong to the page that added them. They're removed when it
  navigates, reloads, closes or crashes, when the host restarts, and, for the server surface,
  on disconnect.
* **Limits:** 64 rules and 16 holds per surface; strings in a match at most 64 bytes.
* **Seeing it happen:**
  * `webview_intercepts` lists the active rules and holds and their owners.
  * `webview_intercept_reset` removes them all.
  * `webview_intercept_log 1` prints each block, rewrite and takeover.
  * While anything is intercepting, the plugin draws a small marker in a screen corner
    (`webview_intercept_indicator`, default 1).

Examples:

```ts
// Hide join/leave notices and restyle the kill feed on the page instead.
tsmod.intercept.rule({ kind: "textmsg", match: { text: { contains: "#Game_joined" } }, action: "block" });
tsmod.intercept.rule({ kind: "killfeed", action: "block" });
tsmod.game.on("killfeed", e => feed.push(e));          // observed events still arrive when blocked

// A chat filter that can look things up before deciding.
tsmod.intercept.hold("chat", async e => (await isSpam(e.text)) ? "block" : "pass", { timeoutMs: 150 });

// No weapon switching while the shop panel is open.
const r = tsmod.intercept.rule({ kind: "weapon_select", action: "block" });
shop.onClose(() => tsmod.intercept.remove(r));
```

### 7.8 Taking over menus

A `takeover` verdict on a `menu`, `vguimenu` or `motd` event stops the port's menu. The page then
gets the menu as a handle it can answer:

```ts
interface Menus {
  on(cb: (m: MenuHandle) => void): void;  // fires for each menu this page took over
}
interface MenuHandle {
  kind: "menu" | "vguimenu" | "motd";
  fields: CatalogueEvent;                 // keys, text, menu id, MOTD text
  reply(choice: number | string): void;   // a key from `keys`, a team, a buy item
  cancel(): void;                         // as Esc on the port's menu
  closed: Promise<void>;                  // the server closed it, it timed out, or it was answered
}
```

* `reply` goes through `menu_reply` (§4.1), so the server gets exactly the command the port's
  own menu would have sent. A choice the menu doesn't offer (a key not in `keys`, a team the
  team menu doesn't list) is refused.
* A handle answers once. It expires with the menu's `time`, when a newer menu replaces it, or on
  `ResetHUD`.
* **Fail open:** if the page closes or crashes while it holds an open menu, the port shows its
  own menu again.
* For a URL MOTD, `takeover` and then `tsmod.show()` lets the page display it. This covers open
  question 4 without a port hook.

### 7.9 HUD control

```ts
interface Hud {
  elements(): Promise<{ name: HudElement; rect: [number, number, number, number]; visible: boolean }[]>;
  hide(names: HudElement[]): void;
  show(names: HudElement[]): void;
}

type HudElement =
  | "health" | "armour" | "ammo" | "weapon_select" | "crosshair" | "killfeed" | "chat"
  | "notify" | "centre" | "hudtext" | "timer" | "cash" | "kungfu" | "stunt" | "powerup"
  | "objective" | "pickups" | "status_icons" | "spectator" | "scoreboard";
```

* This goes through `hud_set_hidden` (§4.1). It only stops drawing: the port keeps the element's
  state, so `show` brings it back as it would be now.
* `elements()` gives each element's rectangle in screen pixels, so a `hud` surface can draw its
  own version in the same place.
* Hidden elements come back when the page that hid them navigates, closes or crashes, and with
  `webview_hud_reset`.

### 7.10 What pages can never see or do

The API is limited to what the player sees on screen and to undoing what already happened. So it
can't be used as a wallhack, an aimbot or a triggerbot, and it can't speak for the player:

* **Seeing:**
  * Players who aren't visible (§7.4) never appear, not even as a count. No query lists other
    entities, items or projectiles.
  * Positions of other players come only as distance, bearing and screen position, rounded to
    whole units and degrees.
* **Writing:** no view angles, buttons, console commands or cvars. The only commands a page
  causes are menu answers within what the menu offers (§7.8), and chat the player typed and the
  page rewrote (player surface only).
* **Protected events** take no rules or holds, whatever the page asks:
  * `state` messages, which keep the HUD and prediction in step with the server;
  * `effect` messages, so a page can't remove fades, shakes, blur, damage indicators or slow
    motion meant to hinder the player;
  * `input` buttons: blocking `+attack` until the crosshair is on an enemy would be a
    triggerbot;
  * `webview_*` commands, Esc, the console key and the toggle key;
  * everything outside the catalogue (§7.6).
* **No forgery:** slot fields can't be rewritten, so a page can't make a line look as if another
  player said or did it. Nor can it add events to the port's HUD; it can only draw its own.
* A page can still draw over visible players in a `hud` surface, the same as any external
  overlay. The `policy` switch is there for servers that don't want that.

## 8. Server ⇄ client protocol

### 8.1 Server → client: `TSExt` channel `webview`

Payload (after the channel string):

```
byte  op
byte  msg_id      wraps at 256; groups chunks
byte  chunk       index, high bit set on the last chunk
...   body chunk  ≤ 172 bytes
```

The client reassembles the chunks for each `msg_id` and gives up after 2 seconds or 16 chunks
(about 2.7 KB). The reassembled body depends on `op`:

| op | name | body |
|---|---|---|
| 1 | `open` | `mode\0url\0[x y w h]` (rect in percent, `panel` only) |
| 2 | `navigate` | `url` |
| 3 | `show` | none |
| 4 | `hide` | none |
| 5 | `close` | none |
| 6 | `event` | `name\0json` |
| 7 | `hello` | none; the client answers with `wv hello` (§8.2) |
| 8 | `policy` | two bytes of flags, little-endian: 1 `game.read`, 2 `game.input`, 4 `intercept.display`, 8 `intercept.actions`, 16 `hud` on the player surface; the same shifted left by 8 (256, 512, 1024, 2048, 4096) on the server surface. A cleared bit removes the capability whatever consent says. Default before any `policy`: everything (`0x1F1F`) |

### 8.2 Client → server: client command `wv`

```
wv hello <proto> <state>                  state: off | allow | prompt | denied
wv state <opened|closed|denied|error> [reason]
wv ev <msg_id> <chunk> <name> "<json chunk>"
```

* Commands are kept under 128 bytes, and JSON longer than that is chunked the same way as §8.1.
* Quotes and semicolons in the JSON are escaped as `"` and `;` before sending.
* The server plugin rate-limits and length-limits whatever it receives. A client can send
  anything it likes.

### 8.3 Fallback without `TSExt`

On a server whose port is too old to register `TSExt`, the AMXX plugin can fall back to
`client_cmd(id, "wv_srv <op> ...")`. That command is accepted only when the client sets
`webview_allow_stuffcmd 1` (default 0).

Stuffed text is also subject to HL25's `cl_filterstuffcmd`, and it can't be told apart from
other stuffed commands. That is why it is opt-in.

### 8.4 Capability discovery

* The plugin registers the userinfo cvar `_wv` and sets it with `Cvar_SetValue` at `init`, to
  the protocol version, or 0 when the player has set `webview_allow_server 0`. Steam GoldSrc puts
  a client cvar in the userinfo only once it has been set (the port's #196), hence the explicit
  set.
* AMXX reads it with `get_user_info(id, "_wv", ...)`. An empty value means no plugin.

### 8.5 Client policy

| cvar | default | meaning |
|---|---|---|
| `webview_allow_server` | 1 | 0 never; 1 allowlisted origins, asks the first time per server; 2 allowlisted origins without asking |
| `webview_allowlist` | `""` | extra origins, space-separated (`https://stats.example.org`) |
| `webview_allow_local` | 0 | allow `localhost` and private addresses |
| `webview_allow_stuffcmd` | 0 | §8.3 fallback |
| `webview_home` | bundled start page | the player surface's home page |
| `webview_fps` | 30 | frame rate cap |
| `webview_persist` | 1 | keep cookies and cache |
| `webview_preload` | 0 | start the host at launch |
| `webview_idle_exit` | 300 | seconds with nothing visible before the host exits; 0 never |
| `webview_game_api` | 1 | 0 turns the game API's `game.read` and `game.input` off for every page (§7.5) |
| `webview_intercept` | 1 | 0 turns `intercept.display`, `intercept.actions` and `hud` off for every page |
| `webview_intercept_indicator` | 1 | draw the marker while anything intercepts (§7.7) |
| `webview_intercept_log` | 0 | print each block, rewrite and takeover |

Origin rules for server `open` and `navigate`:

* Allowed by default: the server's own FastDL host (`sv_downloadurl`, which the client already
  trusts for content), plus origins in `webview_allowlist`.
* Allowed only after consent: any other `https://` origin. The consent card shows the origin and
  three choices: Allow once, Always for this server, Deny. Choices are stored in
  `webview/consent.json`, keyed by server address and origin.
* Never allowed: `http://` (unless `webview_allow_local`), `file:`, `data:`, `javascript:`,
  `chrome:`, `devtools:` and anything else.
* Navigation inside the page that leaves allowed origins closes the server surface.

### 8.6 Client commands

* `webview_toggle`: shows or hides the player surface (bind it, e.g. `bind F7 webview_toggle`;
  nothing is bound by default).
* `webview_open <url>`, `webview_close`.
* `webview_server_close`: closes the server surface.
* `webview_status`: prints the host state, surfaces and protocol version.
* `webview_consent_reset`: clears stored consent choices.
* `webview_commands`: lists the `wv_*` commands pages registered, their owners and help (§7.2).
* `webview_intercepts`, `webview_intercept_reset`: list or remove every rule and hold (§7.7).
* `webview_hud_reset`: shows every HUD element pages hid (§7.9).
* `wv_*`: commands registered by pages.

## 9. AMX Mod X plugin (`amxx/`)

`tsmod_webview.amxx` is a library plugin: it owns the protocol and offers natives and forwards to
other plugins, plus admin commands. It needs only stock AMXX 1.9 or 1.10 modules (`amxmodx`;
`engine` isn't needed) and works on Metamod-R and Metamod-P, the same as tsmod-amxx-rs.

### 9.1 Include (`tsmod_webview.inc`)

```pawn
#if defined _tsmod_webview_included
  #endinput
#endif
#define _tsmod_webview_included
#pragma reqlib tsmod_webview

enum WebviewMode { WV_FULL = 0, WV_PANEL, WV_HUD };

enum WebviewState { WV_NONE = 0, WV_OFF, WV_PROMPT, WV_ALLOW, WV_DENIED };

// Client support, from the `_wv` userinfo and the last `wv hello`.
native WebviewState:webview_state(id);

// id = 0 sends to every client with support. Return the number of clients sent to.
native webview_open(id, const url[], WebviewMode:mode = WV_FULL, x = 0, y = 0, w = 100, h = 100);
native webview_navigate(id, const url[]);
native webview_show(id);
native webview_hide(id);
native webview_close(id);
native webview_send(id, const event[], const json[]);   // json ≤ 2 KB

// Game API and interception policy (§7.5, `policy` op). id = 0 sets it for everyone,
// including players who join later.
#define WV_CAP_READ      1    // game.read
#define WV_CAP_INPUT     2    // game.input
#define WV_CAP_DISPLAY   4    // intercept.display
#define WV_CAP_ACTIONS   8    // intercept.actions
#define WV_CAP_HUD       16   // hud
#define WV_CAP_ALL       31
native webview_set_game_policy(id, player_caps = WV_CAP_ALL, server_caps = WV_CAP_ALL);

// Fired for each reassembled `wv ev` from a client (already length/rate-checked).
forward webview_on_event(id, const event[], const json[]);
// Fired on `wv state`: opened, closed, denied, error.
forward webview_on_state(id, const state[], const reason[]);
```

### 9.2 Implementation notes

* Uses `get_user_msgid("TSExt")` in `plugin_init`. If the message is missing, it logs once and
  uses the §8.3 fallback, or refuses with cvar `webview_fallback 0` (the default).
* Sends with `message_begin(MSG_ONE, msgid, _, id)`, or `MSG_ALL` for id 0, followed by
  `write_string("webview")` and `write_byte` for the chunks. It splits bodies into chunks of at
  most 172 bytes.
* Registers `register_clcmd("wv", ...)`, reassembles chunks per player, and applies its own
  limits: `webview_max_event_len` (default 2048) and `webview_max_events` (default 8 per 10
  seconds). It kicks with a reason only when `webview_kick_flood 1` is set.
* Sends `hello` on `client_putinserver`, then gives the client 5 seconds before treating it as
  `WV_NONE`. A `policy` follows the `hello` when the cvars `webview_player_caps` and
  `webview_server_caps` (default 31 each) or `webview_set_game_policy` have set anything other
  than everything. A competitive config sets `webview_player_caps 0`.
* Page commands, game events and interception stay on the client. The server sees only what a server page
  chooses to `send`.

### 9.3 Admin commands

| Command | Access | Effect |
|---|---|---|
| `amx_webview_open <target> <url> [full\|panel\|hud] [x y w h]` | `ADMIN_CFG` | open on a player, `@team` or `@all` |
| `amx_webview_close <target>` | `ADMIN_CFG` | close |
| `amx_webview_show <target>` / `amx_webview_hide <target>` | `ADMIN_CFG` | |
| `amx_webview_send <target> <event> <json>` | `ADMIN_RCON` | send an event to the page |
| `amx_webview_status` | `ADMIN_ADMIN` | list players with their support state |

These are also server console commands (`register_concmd`), so `rcon`, `server.cfg` and other
plugins' `server_cmd` can drive the overlay. Targets follow AMXX's `cmd_target`: name, `#userid`,
`@all`, `@1`/`@2` for teams.

### 9.4 Demo plugin (`tsmod_webview_demo.sma`)

* `say /stats` opens `<sv_downloadurl>/webview/stats.html` as a panel and sends
  `stats.update` events with the player's kills and deaths every 5 seconds while it is open.
* `say /vote` opens a vote page, and the plugin tallies the `vote.cast` events it receives.
* `say /hud` opens `<sv_downloadurl>/webview/hud.html` as a `hud` surface. The page uses the game
  API: a hit counter on `input` `attack`, the aimed-at player's name from the `aim` event, a
  "players in view" count from `watch("players", 250)`, and `wv_hud` (`+wv_hud` held shows the
  full panel). It also hides the port's `health`, `armour` and `ammo` elements and draws its
  own in their rectangles, and blocks the `killfeed` to show a restyled one fed by observed
  `killfeed` events.
* `say /menu` sends a stock `ShowMenu` text menu; the open `hud.html` page takes it over, shows
  it as clickable buttons and answers it with `reply`. The plugin prints the `menuselect` it
  receives, which is the same as the port's own menu would have sent.

This shows both directions, the game API, interception and the FastDL hosting that needs no
consent. The pages live in `web/demo/` and the server's FastDL tree.

## 10. Build, test, release

* **`cargo xtask build|dist`:** builds the four client-plugin targets, the two host targets and
  `amxxpc` for `amxx/` (`LD_LIBRARY_PATH` set to amxxpc's directory).
* **`cargo xtask test`:** unit tests. Covers protocol framing round trips, chunking at 172/128
  bytes, the escaping rules, the origin policy table and key mapping. Also covers the game API's
  pure parts: command name rules and the ownership table, button-edge detection from `buttons`
  words, the capability table (§7.5) and the visibility filter (§7.4) on recorded snapshots.
  For interception: rule matching, the evaluation order and rewrite chaining, the protected list,
  hold timeouts failing open, per-kind ordering behind a held event, and a decode/re-encode round
  trip for every catalogue message against payloads recorded from the port.
* **`cargo xtask e2e [roots]`:**
  1. *Client:* on the port's FWGS root (Xvfb, as the port's screenshot runs do), with a
     cfg-driven run. It loads the plugin, runs `webview_open` with a bundled test page that
     paints a known colour, takes a `screenshot`, and checks the pixels. It then closes the page
     and checks the host exits after `webview_idle_exit 1`.

     A second test page registers `wv_e2e` and subscribes to `input`. The cfg runs `wv_e2e 1`,
     `+attack` / `-attack` and `+use` / `-use`, and the page paints one colour per event it saw.
     The page also runs `self` and `aim` queries on a known map spot and logs the results to the
     console, which the run checks.

     A third page blocks `slot2` (`weapon_select`) and hides `health`; the cfg runs `slot2` and
     the run checks the weapon didn't change and the screenshot has no health digits.
  2. *Server and client:* the AMXX test roots (`amxx_rehlds_linux` and `amxx_rehlds_win`) plus an
     FWGS client. `webview_e2e.sma` opens a page on the client; the page answers `e2e.ping` with
     `e2e.pong`, which the plugin logs. The run also checks the fallback refusal and the flood
     limit.

     For interception, the plugin sends `SayText`, `DeathMsg`, `TextMsg` and a `ShowMenu`. The
     page blocks one chat line, rewrites another, holds the kill feed past its timeout (which
     must fail open), and takes over the menu and answers key 2; the run checks the client
     console and the `menuselect 2` the server receives. With `webview_player_caps 0` and
     `webview_server_caps 0`, the same page must get `cap` errors and change nothing.
  3. *Steam HL25:* manual for v1, following the port's `docs/validation.md` §5 (Steam status
     invisible while testing).
* **Releases:** a `v*` tag runs `release.yml`. It publishes:
  * `tsmod-webview-<tag>-<platform>.{zip,tar.gz}`: the client plugin, host, CEF and the bundled
    pages.
  * `tsmod-webview-amxx-<tag>.zip`: `.amxx`, `.sma` and `.inc`.
* **Version pinning:** the crate pins the port's `client-ext-api` by tag, the same way
  tsmod-amxx-rs pins `plugin-api`. A release states the port versions it works with. The launcher
  offers the client archive as an optional download, as it does the HD pack.
* **Chromium updates:** a scheduled workflow checks for a new CEF stable build weekly and opens a
  PR with the bump.

### Install layout (client)

```
ts/cl_dlls/tsmod_webview<suffix>          client plugin
ts/webview/bin/tsmod-webview-host[.exe]   helper + CEF (libcef, paks, locales, icudtl.dat, ...)
ts/webview/pages/                         bundled pages (start page, consent card, test page)
ts/webview/profile/                       created at run time
```

## 11. Budgets

| | Target |
|---|---|
| Client archive | ≤ 130 MB compressed per platform (CEF minimal, stripped) |
| Plugin cost while closed | no host running, or a throttled host; < 0.1 ms per frame in the game |
| Plugin cost while open | ≤ 1 ms per frame for upload and draw at 1080p with a full-screen repaint |
| Host memory | ≤ 300 MB with one simple page |
| First open | ≤ 1.5 s cold; ≤ 100 ms with `webview_preload 1` |
| Server → page event latency | one server frame plus one page frame |
| Game event latency (`input`, `aim`) | one client frame plus one page frame |
| Game API cost in the game | ≤ 0.2 ms per frame with `players` watched on a 32-player server; 0 when nothing subscribes or watches |
| Rule evaluation | ≤ 5 µs per event with 64 rules; nothing when no rule or hold names the kind |
| Held event delay | the page's answer time, at most the hold's `timeoutMs` (100 ms by default, 500 ms in total) |

## 12. Open questions

1. Does HL25's `cl_filterstuffcmd` block commands that a client library registered with
   `pfnAddCommand`? This only affects the §8.3 fallback.
2. Does FWGS's GLES renderer give a usable `glTexSubImage2D` with BGRA (`GL_EXT_texture_format_BGRA8888`),
   or does the plugin need to swizzle?
3. Clipboard paste in a windowless CEF on Linux without an X11/Wayland window of its own.
4. URL MOTDs: §7.8's `motd` takeover lets a page show one. Should the plugin also offer a
   built-in "open URL MOTDs in the overlay" setting, without a page, subject to §8.5's origin
   rules?
5. Windows i686 host for 32-bit Windows systems: drop it, or ship CEF's win32 build?
6. Does FWGS offer a way to remove a client command (`pfnDelCommand` or similar)? If so, the port's
   `client-ext-api` could expose it on FWGS and §7.2 could drop its no-owner stubs there.
7. Should `players` be on by default for the player surface, or should `policy` default to
   `game.read` off for the player's own pages (flags 12)? On-screen-only data argues for on; a
   `hud` page highlighting visible enemies argues for off on competitive servers.
8. `death.killer` and `weapon` names: take them from the port's `DeathMsg` handling as the kill
   feed shows them, or from `weapon_name(id)`? They must match what the HUD prints.
9. Outgoing chat: `say` and `say_team` aren't port commands. The engine forwards them, and
   `messagemode` is the engine's own. Can the port register `say`/`say_team` in `client.dll` and
   forward them itself (`ServerCmd`) on both HL25 and FWGS, without the engine rejecting the name
   or changing behaviour? If not, `chat_send` is dropped from v1.
10. The port's handlers mix state and display for some messages (`DeathMsg` updates the
    scoreboard and the kill feed; `TextMsg` checks `AllowedToPrintText`). §4.1 needs them split
    so a block or hold only touches the display. Which handlers need splitting is a port-side
    audit, done in milestone 1.
11. TS's buy menu commands: which commands does the port's buy VGUI send, so `buy` and
    `menu_reply` can name them exactly?
12. Should a server surface be able to block the player's own pages' interception (it can, with
    `policy`), and should the player be told when a server has done so? The proposal is a line in
    `webview_status` and the consent card.

## 13. Milestones

1. **Port:** client extension API with a stub extension and an xtask check, including the
   `usercmd` and `game_event` callbacks, the read-only game state functions, and the
   interception hooks (`message_pre`, `command_pre`, `server_cmd_pre`, `Defer`/`release`,
   `hud_set_hidden`, `menu_reply`) with the state/display split; the `TSExt` message. Two port
   issues; the interception hooks may be a third.
2. **Player overlay:** the host with CEF, shared memory and the pipe; the plugin with drawing,
   input and `webview_toggle`, on FWGS Linux first, then Windows, Wine and Steam HL25.
3. **Server control:** `TSExt` handling, the policy and consent card, `_wv` discovery, the bridge.
4. **Game API:** page commands, game events, queries, capabilities and the `policy` op; the
   e2e test pages.
5. **Interception:** the catalogue decoders, rules, holds, menu takeover, HUD control, the
   marker and the console commands; the e2e pages.
6. **AMXX:** `tsmod_webview.sma` and `.inc`, admin commands, the demo plugin, e2e on the AMXX
   roots.
7. **Release:** release workflow, CEF bump automation, launcher integration.
