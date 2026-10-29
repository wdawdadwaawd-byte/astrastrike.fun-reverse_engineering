# AstraStrike.fun — JS Source Code Analysis

> Deobfuscation and reverse engineering of [astrastrike.fun](https://astrastrike.fun/) — a browser-based CS2 fan clone.  
> For educational and research purposes only. Not affiliated with Valve Corporation.

---

## Table of Contents

1. [About](#about)
2. [Technical Stack](#technical-stack)
3. [Weapon System](#weapon-system)
4. [Game Modes & Maps](#game-modes--maps)
5. [Economy System](#economy-system)
6. [Round System](#round-system)
7. [Bot AI](#bot-ai)
8. [Anticheat System](#anticheat-system)
9. [WebSocket Protocol](#websocket-protocol)
10. [Fingerprint System](#fingerprint-system)
11. [Asset Encryption](#asset-encryption)

---

## About

**AstraStrike.fun** is a fully browser-based CS2 fan clone. There is no separate backend for game logic — the majority runs as **client-side JavaScript**. Despite showing "Still in development", bomb defusal, the buy menu, round system and 5v5 modes are already functional.

The main bundle (`main.js`) was obfuscated using **string array rotation + hex encoding** (`_0x409c` / `_0xf62f` pattern). After deobfuscation the output is ~79,000 lines / ~2.5 MB of readable JavaScript.

---

## Technical Stack

| Component | Detail |
|-----------|--------|
| 3D Engine | **Three.js** (custom-patched, `__ASTRA_TOOL__` debug flag present) |
| Renderer | **WebGL2** — ASTC / S3TC texture compression |
| Network | **WebSocket** — custom JSON-based protocol |
| Physics | Client-side BVH (Bounding Volume Hierarchy) ray casting |
| Build Tool | Vite (`vendor-DmpaChgWKs.js`, `modulepreload-polyfill`) |
| Obfuscation | String array rotation + hex encoding |

---

## Weapon System

All weapons are present with full stats in plaintext after deobfuscation:

| Weapon | Damage | Headshot Mult. | Penetration | Spread | Reload |
|--------|--------|---------------|-------------|--------|--------|
| AK-47 | 36 | x4 | 2.0 | 0.0006 | ~2.4s |
| M4A4 | 33 | x4 | 2.5 | 0.0003 | ~3.1s |
| M4A1-S | 38 | x4 | 2.5 | 0.0003 | ~3.1s |
| AWP | 115 | x4 | 2.5 | 0.0003 | ~3.7s |
| FAMAS | 30 | x4 | 2.0 | 0.0006 | 3.3s |
| Desert Eagle | 53 | x4 | 1.0 | — | — |
| SCAR-20 | 80 | x4 | 2.5 | 0.0003 | 3.1s |
| G3SG1 | 80 | x4 | 2.5 | 0.0003 | 4.7s |
| M249 | — | x4 | 1.0 | — | 5.7s |
| Negev | — | x4 | 1.0 | — | 5.7s |

Each weapon also exposes `recoilSeed`, `recoilAngle`, `recoilMagnitude`, `recoilMagnitudeVariance`, `recoveryStartBullet`, and `attackMovespeed`.

---

## Game Modes & Maps

```
Aim Training   →  aim_arena         (solo, unlimited time, no bots)
Surf           →  surf_boreas / surf_utopia_njv
AWP Duel       →  awp_lego_2        (1v1, first to 5 rounds)
Rifle 2v2      →  aim_usp_anubis    (2v2, first to 9 rounds, 30 max)
Retake 3v3     →  de_dust2_b        (3v3, first to 7 rounds, B bombsite)
```

Radar metadata (origin, scale, size) is also embedded in the source for every map.

---

## Economy System

Nearly identical to CS2:

```
Starting money   : $800
Maximum money    : $16,000
Kill reward      : $300
Round win        : $3,250
Loss bonus base  : $1,400  (+$500 per consecutive loss, max $3,400)
Bomb plant       : $300
Bomb defuse      : $300
Buy time         : 20 seconds
```

**Buy Menu items:**

| Item | Price |
|------|-------|
| AK-47 | $2,700 |
| M4A4 | $2,900 |
| AWP | $4,750 |
| Desert Eagle | $700 |
| Molotov | $400 |
| Smoke Grenade | $300 |
| Kevlar + Helmet | $1,000 |
| Defuse Kit | $400 |

---

## Round System

```
Freeze time      : 3 seconds
Round time       : 115 seconds
End time         : 5 seconds
Wins needed      : 13 (full match)
Bomb timer       : 40 seconds
Plant time       : 3.2 seconds
Defuse time      : 10 seconds
Defuse with kit  : 5 seconds
```

---

## Bot AI

All bot parameters are exposed in plaintext:

```js
runSpeed         : 215 * unit
walkSpeed        : 110 * unit
health           : 100
armor            : 100
damage           : 33
reaction         : [0.4, 0.55] seconds
aimErrorStart    : [6, 10] degrees
aimErrorMin      : 0.5 degrees
aimSettle        : 0.92
teamwork         : 0.75
aggression       : 0.6
headAim          : 0.12   // probability of targeting the head
burst            : [3, 7] shots
burstPause       : [0.22, 0.55] seconds
fov              : cos(75°) ≈ 0.259
viewDist         : 90 units
hearDist         : 26 units
shotHearDist     : 60 units
investigators    : 2
```

Bot names are also hardcoded: `Dragomir, Goran, Zoran, Branko, Milan` (T-side), `Hawkins, Reyes, Ward, Doyle` (CT-side).

---

## Anticheat System

The most interesting part of the codebase. It combines **client-side tamper detection** with **server-side hit validation**.

### Client-Side Detection — `ng()` function

Runs every 5 seconds via `reportTamper()`. Results are collected in a `Set` (`Yh`) and sent to the server.

| Flag | What triggers it |
|------|-----------------|
| `three-patched` | Three.js native methods overridden (`add`, `updateMatrixWorld`, `onBeforeRender`…) |
| `game-wrapped` | `fire_k7arb`, `inaccuracy_k7arb`, `applyCamera_k7arb` overridden |
| `chams` | Bot materials have `depthTest === false` or `wireframe === true` → wallhack/chams detection |
| `ws-patched` | `WebSocket.prototype.send` or `onmessage` setter is not native |
| `webgl-patched` | `drawElements`, `drawArrays`, `uniformMatrix4fv` overridden |
| `tostring-patched` | `Function.prototype.toString` is not native (classic hook detection) |
| `builtins-patched` | Native JS built-ins modified |
| `known-esp` | `__botOverlay` in window or `#bot-overlay-canvas` in DOM |
| `overlay-canvas` | Foreign canvas covering >25% of the screen |
| `overlay-svg` | Foreign SVG covering >25% of the screen |
| `automation` | `navigator.webdriver === true` (Selenium / Puppeteer) |
| `astra-tool` | `__ASTRA_TOOL__` in window or `__es*`-prefixed property on game object |
| `three-devtools` | `__THREE_DEVTOOLS__` browser extension present |
| `debug-url` | `?debug` present in the URL |

### `_k7arb` Suffix Trick

Critical game functions are intentionally named with an unusual suffix:

```
fire_k7arb, inaccuracy_k7arb, stdInaccuracy_k7arb,
punchPitch_k7arb, punchYaw_k7arb, applyCamera_k7arb,
viewPunchPitch_k7arb, flinchStack_k7arb, shotBvh_k7arb …
```

If someone tries to monkey-patch these via the prototype, `eg()` catches it via `hasOwnProperty`.

### `tg()` — Native Code Checker

```js
tg = (fn) => /\{\s*\[native code\]\s*\}\s*$/.test(
    Function.prototype.toString.call(fn)
)
```

Applied to WebSocket, WebGL methods, and `Function.prototype.toString` itself.

### Server-Side Hit Validation

- Client sends a hit → server replies with **`hitok`** or **`hitno`**
- If `hitno` is received, `refuseHit()` is called → logs `"a shot the anti-cheat threw out"`
- **Even with an aimbot, hits are worthless unless the server confirms them**

### Tamper Reporting

```js
// Runs every 5 seconds
setInterval(() => this.reportTamper(), 5000)

// Payload sent to server:
{ "t": "tamper", "k": ["chams", "ws-patched", "overlay-canvas"] }
```

---

## WebSocket Protocol

All messages are JSON. The `t` field identifies the message type.

**Client → Server:**

| Message | Description |
|---------|-------------|
| `hello` | Sent on connect — `name`, `platform`, `fp`, `gpu`, `ticket` |
| `queue` | Enter matchmaking queue |
| `cancel` | Leave queue |
| `host` | Create a room |
| `join` | Join a room by code |
| `accept` | Accept a match |
| `tamper` | Report AC flags |
| `defuse` | Defuse the bomb |
| `rematch` | Request rematch |
| `ticket` | Request auth ticket |

**Server → Client:**

| Message | Description |
|---------|-------------|
| `welcome` | Connection confirmed |
| `ticket` | Auth token |
| `hitok` | Hit confirmed |
| `hitno` | Hit rejected (anticheat) |
| `roomvote` | Vote opened / result |
| `roomrated` | Match rating updated |

---

## Fingerprint System

On every WebSocket connection, `li()` runs and sends `fp` + `gpu` to the server.

**Data collected for the fingerprint:**

1. WebGL renderer string (GPU name, capped at 120 chars)
2. Canvas 2D render fingerprint — custom gradient + two font renders → `toDataURL()` → SHA-256
3. `screen.width × screen.height × colorDepth`
4. `navigator.hardwareConcurrency` (CPU core count)
5. `navigator.deviceMemory` (RAM amount)
6. `Intl.DateTimeFormat().resolvedOptions().timeZone`
7. `navigator.languages`
8. `navigator.platform`
9. `navigator.maxTouchPoints`

**Output:** SHA-256 → first 16 bytes → 32-character hex string.  
Sent on every connection. Purpose: identify returning users even after a ban.

---

## Asset Encryption

3D models (`.glb`, `.bin`) and effect files (`.rgba`) are encrypted with a **custom XOR stream cipher**.

**Magic headers:**
- `AST1` → `[65, 83, 84, 49]` — standard encryption
- `AST2` → `[65, 83, 84, 50]` — skip-mode encryption

**Algorithm:**

```js
// key derivation (per Uint32 word)
let key = (seed ^ Math.imul(offset, 0x9E3779B1) ^ 0x27E9C6E1) >>> 0;
key = Math.imul(key ^ (key >>> 16), 0x85EBCA6B);
key = Math.imul(key ^ (key >>> 13), 0x4029B020);
data[i] ^= key ^ (key >>> 16);
```

In skip-mode (`AST2`), within every 1024-byte block, the first 4 bytes are left unencrypted.  
The first 8 bytes of each file are a header (magic + 32-bit seed) and are stripped before decryption.

---

*Analysis based on: `main.deobf.js` (~79k lines, ~2.5 MB) and `deobfuscated.js` (~44k lines, ~2.1 MB)*
