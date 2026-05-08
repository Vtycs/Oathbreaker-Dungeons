# Oathbreaker Dungeons V2.0.2 :)

> 2D Roguelite with online co-op published on Steam — independently developed.
![2222](https://github.com/user-attachments/assets/84543e4f-5afe-4b46-bd32-5d9eb764985a)


[![Steam](https://img.shields.io/badge/Steam-Publicado-1b2838?style=flat&logo=steam)](https://store.steampowered.com/app/4461820/Oathbreak_Dungeons/)
[![Stack](https://img.shields.io/badge/Stack-Electron%20%2B%20p5.js-61DAFB?style=flat)](#stack)
[![Rede](https://img.shields.io/badge/Rede-P2P%20Steam-informational?style=flat)](#sistema-de-rede-p2p)

-----

## About the project

Oathbreaker Dungeons is a 2D action roguelite featuring procedural room generation, a buff and archetype system, and **online co-op via Steam** for up to 4 players. The project was independently developed — from the engine to platform deployment — and is available on Steam.

> **This repository is a technical portfolio.** It showcases the architecture, engineering decisions, and representative code snippets. The full source code is private.

-----

## Stack

| Layer | Technology | Reason for choice |
|---|---|---|
| Runtime | **Electron** (Node.js) | Native access to the Steam API via C++ bindings |
| Render | **p5.js 1.9** | Controllable game loop, simple canvas API |
| Networking | **steamworks.js** | Bindings for the Steamworks SDK (lobby, P2P, achievements) |
| Distribution | **Steam** | Build + deploy pipeline with Steamworks |

Using p5.js as the game engine published on Steam is an unconventional choice that required solving problems the library wasn't designed to support — from lighting shaders with pixel density correction on HiDPI/Retina displays, to real-time game state synchronization via a custom P2P protocol.

-----

## Module architecture

```
├── main_js.js      # Main Electron process — Steamworks init, overlay, invites
├── steam.js        # Lobby state, network protocol, achievements, stats
├── globals.js      # Dictionary of weapons, buffs, settings, controls
├── sketch.js       # Main game loop (setup/draw), camera, HUD, state machine
├── player.js       # Player class — physics, combat, buffs, rendering
├── enemies.js      # Entity hierarchy — projectiles, enemies, bosses
├── dungeon.js      # Procedural generation of rooms, platforms, shops, arenas
├── ui.js           # Menus, lobby screen, buff screen, settings
└── audio.js        # Sound manager with Web Audio context
```

-----

## Technical systems

### P2P Network System

The game implements a custom P2P protocol over Steam's socket layer, with the following packet types:

```js
const PKT = {
    // Session
    HANDSHAKE: 10, HANDSHAKE_ACK: 11,
    LOBBY_SETTINGS: 1, GAME_START: 2,

    // Game state
    HOST_STATE: 20,      // Full snapshot sent by the host
    CLIENT_INPUT: 21,    // Client inputs with sequence
    PLAYER_STATES: 22,   // Position/velocity of all players

    // World events
    ENEMY_SPAWN: 30, ENEMY_DEATH: 31,
    BOSS_SPAWN: 32, BOSS_PHASE: 33,
    ROOM_CHANGE: 34, BUFF_SELECT: 39,

    // Synchronized effects
    TIME_STOP: 70, SLOW_MO: 73, SCREEN_EFFECT: 71,

    // Heartbeat and reconnection
    HEARTBEAT: 50, PING_REQ: 51, PING_RES: 52,
    HOST_LOST: 54, PLAYER_DROPPED: 55
};
```

**Protocol features:**

  - Host authority with state reconciliation on clients
  - Snapshot buffer with interpolation (5-frame buffer, 100ms delay)
  - Heartbeat with an 8-second timeout and lost host detection
  - Client input sequencing to prevent desync
  - Slow-motion and time stop effects synchronized via dedicated packets

-----

### Steamworks Integration

The integration goes beyond initializing the API. The main process handles **runtime lobby invites** received via the `GameLobbyJoinRequested` callback, with defensive parsing of the SteamID64:

```js
// Extracts SteamID64 from any format returned by the API
// (bigint, number, string, object with variable fields)
function _extractId(v) {
    if (typeof v === 'bigint') return String(v);
    if (typeof v === 'string') {
        let s = v.replace(/n$/, '');
        if (/^\d{10,25}$/.test(s)) return s;
    }
    if (typeof v === 'object') {
        // Tries known API fields
        let candidates = [
            v.steamId64, v.steamId, v.m_steamid,
            v.lobbyId, v.m_steamIDLobby, v.lobby
        ];
        for (let c of candidates) {
            let r = _extractId(c);
            if (r) return r;
        }
    }
    return null;
}
```

This handling was necessary because different versions of steamworks.js return the same data in distinct formats (native bigint, string with an `n` suffix, object with variable fields).

**Other integration points:**

  - Steam Overlay enabled with `electronEnableSteamOverlay()`
  - Callback loop running at 30hz via `setInterval`
  - `+connect_lobby` via argv for direct join when opening the game through an invite
  - JavaScript injection in the renderer via `executeJavaScript` for main↔renderer bridge

-----

### Game Feel

Physics and feedback details that make movement responsive:

| System | Implementation |
|---|---|
| **Coyote time** | 8 frames of tolerance after walking off a platform |
| **Jump buffer** | Jump input stored for 8 frames |
| **Hit stop** | Frame freeze on impact (`hitStopFrames`) |
| **Screen shake** | Maximum intensity with a 0.80 decay per frame |
| **Squash & stretch** | Character scale based on vertical velocity and dash |
| **Afterimages** | Visual trail during dash |
| **Slow motion** | Multiplier factor on delta time, synchronized over the network |

-----

### Platform entity hierarchy

All platforms inherit from a base class and override `update()` and `display()` with their own behavior:

```
Platform (base)
├── LavaFloor     — moves up/down with a sine wave, deals damage and knockup
├── HazardSpike   — state machine: sleeping → alerting → firing
├── BouncyPad     — detects vertical collision and applies an impulse
├── IcePlatform   — reduces the player's horizontal friction
├── ConveyorBelt  — applies continuous horizontal velocity
└── DisappearPad  — vanishes after contact, respawns with a timer
```

-----

### Buff and archetype system

The game features over 60 unique buffs and 8 archetypes. The `addBuff()` method coordinates all chain effects:

1.  Adds the buff to the player's array
2.  Triggers particles and floating text
3.  Applies stat modifiers immediately (HP, damage, speed)
4.  Synchronizes over the network to all clients
5.  Checks Steam achievements tied to the archetype
6.  Persists meta-stats to disk

Some buffs affect **all players simultaneously** — `B_LAST_STAND` reduces everyone to 1 HP and triples the damage multiplier; `B_LIFE_DEBT` gives a random buff to everyone but doubles the HP and damage of enemies in the room.

-----

### Procedural room generation

The dungeon is generated as an infinite grid map (X, Y). Each cell is built on demand when visited for the first time, with its type defined by weights adjusted according to progression:

  - **Combat rooms** with 10+ enemy types and varied compositions
  - **Miniboss rooms** with 4 unique intermediate bosses (Blacksmith, Priestess, Centipede, Octopus)
  - **Final boss** deterministic, with 2 options (Dragon / Joker) — specific music and phases
  - **Shops** with a pricing system, random items, and network interaction
  - **Buff rooms** with card selection — synchronized across all players

-----

### Lighting shader and HiDPI correction

The lighting system uses a separate graphics buffer (`lightBuf`) with a blend mode to simulate darkness around the players. A non-obvious issue arose on HiDPI/Retina displays: p5.js applies `pixelDensity(2)` by default, causing the light buffer to misalign with the main canvas. The fix required forcing `pixelDensity(1)` globally and on auxiliary buffers.

```js
function setup() {
    pixelDensity(1); // Critical: forces 1:1 — fixes shader on HiDPI/Retina displays
    createCanvas(GAME_WIDTH, GAME_HEIGHT);

    let lightBuf = createGraphics(GAME_WIDTH, GAME_HEIGHT);
    lightBuf.noSmooth(); // Same pixel density as the main canvas
    if (typeof initLightSystem !== 'undefined') initLightSystem(lightBuf);
}
```

-----

### Music state machine

The soundtrack changes dynamically according to the game context:

```
MENU / LOBBY      → menu track
PLAY - common room→ exploration track
PLAY - MINIBOSS   → specific track by boss type
    SMITH         → miniboss_smith
    PRIESTESS     → miniboss_priest
    CENTIPEDE     → miniboss_centi
    OCTOPUS       → miniboss_octo
PLAY - FINAL_BOSS → specific track
    DRAGON        → boss_dragon
    JOKER         → boss_joker
PLAY - SHOP       → calm shop track
GAMEOVER          → game over track
CREDITS           → credits track
```

-----

## Implemented Steam achievements

20 achievements integrated via Steamworks, including:

| Achievement | Condition |
|---|---|
| First Blood | First enemy killed |
| Parry Master | 50 parries in a single run |
| Za Warudo\! | Use time stop for the first time |
| JACKPOT\! | Roll 777 on the Gambler archetype's slot machine |
| Untouchable | Defeat a miniboss without taking damage |
| Versatile | Play with all 8 archetypes |
| Brothers in Arms | Complete a run in online co-op |
| Lightning Fast | Complete a run in under 15 minutes |
| Overkill | Deal over 500 damage in a single hit |

-----

## Solved challenges

  - **Pixel density + misaligned shader** — fixed by forcing global `pixelDensity(1)`
  - **SteamID64 in multiple formats** — defensive parser that handles bigint, string, object
  - **Steam invites at runtime** — main process → renderer bridge via `executeJavaScript`
  - **Network-synchronized slow motion** — dedicated packet with client-side application
  - **P2P without a dedicated server** — host-authoritative protocol over Steamworks sockets
  - **Resync on room change** — host sends the complete layout of the new room to the client after generation
  - **Engine not designed for games** — p5.js adapted with hit stop, coyote time, multiple graphic buffers

-----

## Images
![23334](https://github.com/user-attachments/assets/a0d77c57-6f96-40fb-a9ca-4c9c44062832)
![02299](https://github.com/user-attachments/assets/bf2c0fd7-55fa-4bbd-83bc-97ae19cb2593)

*Independently developed. Published on Steam.*
*Developed by Empty TEAM -Poggerz*
