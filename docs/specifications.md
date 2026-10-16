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
* Player control: the player decides whether servers may open pages at all, and which ones.
* Shipped and updated on its own release cadence, so Chromium security updates don't need a port
  release.

Non-goals (v1):

* Replacing the port's own VGUI menus (MOTD, buy menu, scoreboard). A later version may offer
  that through the same interface.
* Letting a server run arbitrary JavaScript on the client.
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
  | `shutdown()` | `HUD_Shutdown` | |

* **Functions the port provides** (`ClientExtApi`):
  * `add_command`, `register_cvar`, `cvar_value`, `cvar_string`, `set_cvar_value`
    (`Cvar_SetValue`, which sets synchronously; see the HL25 userinfo note in §8.4)
  * `print`, `server_cmd`, `client_cmd`
  * `screen_size`, `mouse_position`
  * `gl_proc(name)` (the port's `goldsrc::ext::gl` loader), `renderer_kind` (GL, GLES or
    software)
  * `claim_channel(name)`, `server_address`, `game_dir`, `port_version`
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
  * `Shutdown`
* **Host → plugin:**
  * `Ready`, `Resized`, `TitleChanged`, `LoadError`
  * `CursorChanged`
  * `PageEvent { surface, name, json }` (§7), `Crashed`

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

The host injects the bridge into every page in an allowed origin (§8) and nowhere else:

```ts
interface TsmodBridge {
  version: number;                        // protocol version
  surface: "player" | "server";
  player: { name: string; slot: number }; // the local player
  server?: { address: string; name: string };
  on(event: string, cb: (data: unknown) => void): void;   // server → page
  send(event: string, data?: unknown): void;              // page → server (server surface only)
  close(): void;                                          // asks the plugin to close this surface
}
```

* Event names match `[a-z0-9_.-]{1,31}`. Data is JSON, at most 4 KB after encoding.
* `send` is rate-limited to 4 per second and 16 per 10 seconds per surface. It is dropped with a
  console warning past that.
* The player's own surface has no `send`: personal pages never talk to game servers.

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
  `WV_NONE`.

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

This shows both directions and the FastDL hosting that needs no consent. The pages live in
`web/demo/` and the server's FastDL tree.

## 10. Build, test, release

* **`cargo xtask build|dist`:** builds the four client-plugin targets, the two host targets and
  `amxxpc` for `amxx/` (`LD_LIBRARY_PATH` set to amxxpc's directory).
* **`cargo xtask test`:** unit tests. Covers protocol framing round trips, chunking at 172/128
  bytes, the escaping rules, the origin policy table and key mapping.
* **`cargo xtask e2e [roots]`:**
  1. *Client:* on the port's FWGS root (Xvfb, as the port's screenshot runs do), with a
     cfg-driven run. It loads the plugin, runs `webview_open` with a bundled test page that
     paints a known colour, takes a `screenshot`, and checks the pixels. It then closes the page
     and checks the host exits after `webview_idle_exit 1`.
  2. *Server and client:* the AMXX test roots (`amxx_rehlds_linux` and `amxx_rehlds_win`) plus an
     FWGS client. `webview_e2e.sma` opens a page on the client; the page answers `e2e.ping` with
     `e2e.pong`, which the plugin logs. The run also checks the fallback refusal and the flood
     limit.
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

## 12. Open questions

1. Does HL25's `cl_filterstuffcmd` block commands that a client library registered with
   `pfnAddCommand`? This only affects the §8.3 fallback.
2. Does FWGS's GLES renderer give a usable `glTexSubImage2D` with BGRA (`GL_EXT_texture_format_BGRA8888`),
   or does the plugin need to swizzle?
3. Clipboard paste in a windowless CEF on Linux without an X11/Wayland window of its own.
4. Should the port's MOTD (`VGUIMenu` 5) be able to hand an `http(s)://` MOTD to the overlay when
   the plugin is installed? This needs a port hook; the proposal is v2.
5. Windows i686 host for 32-bit Windows systems: drop it, or ship CEF's win32 build?

## 13. Milestones

1. **Port:** client extension API with a stub extension and an xtask check; the `TSExt`
   message. Two port issues.
2. **Player overlay:** the host with CEF, shared memory and the pipe; the plugin with drawing,
   input and `webview_toggle`, on FWGS Linux first, then Windows, Wine and Steam HL25.
3. **Server control:** `TSExt` handling, the policy and consent card, `_wv` discovery, the bridge.
4. **AMXX:** `tsmod_webview.sma` and `.inc`, admin commands, the demo plugin, e2e on the AMXX
   roots.
5. **Release:** release workflow, CEF bump automation, launcher integration.
