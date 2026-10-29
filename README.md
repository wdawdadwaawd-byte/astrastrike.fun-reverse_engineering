# AstraStrike.fun — JS Kaynak Kodu Analizi

> **astrastrike.fun** sitesinin obfuscate edilmiş JavaScript dosyaları deobfuscate edilerek incelenmiştir.  
> Fan projesi, Valve ile ilgisi yok. Analiz tamamen eğitim/araştırma amaçlıdır.

---

## İçindekiler

1. [Site Hakkında](#site-hakkında)
2. [Teknik Altyapı](#teknik-altyapı)
3. [Silah Sistemi](#silah-sistemi)
4. [Oyun Modları ve Haritalar](#oyun-modları-ve-haritalar)
5. [Ekonomi Sistemi](#ekonomi-sistemi)
6. [Round Sistemi](#round-sistemi)
7. [Bot AI](#bot-ai)
8. [Anticheat Sistemi](#anticheat-sistemi)
9. [Network Protokolü (WebSocket)](#network-protokolü-websocket)
10. [Fingerprint Sistemi](#fingerprint-sistemi)
11. [Asset Şifreleme](#asset-şifreleme)
12. [CheatGlobal Konu Başlığı](#cheatglobal-konu-başlığı)

---

## Site Hakkında

**AstraStrike.fun** — tamamen tarayıcıda çalışan bir CS2 fan klonu.  
Backend/sunucu kodu yok, oyun mantığının büyük kısmı **client-side JavaScript** ile yürütülüyor.  
"Still in development" diyor ama bomb defuse, buy menu, round sistemi, 5v5 modlar çalışır durumda.

---

## Teknik Altyapı

| Bileşen | Detay |
|---------|-------|
| 3D Engine | **Three.js** (custom patch'li, `__ASTRA_TOOL__` debug flag) |
| Renderer | **WebGL2** (ASTC / S3TC texture compression) |
| Network | **WebSocket** — JSON tabanlı özel protokol |
| Fizik | Client-side BVH (Bounding Volume Hierarchy) ray casting |
| Build | Vite (`vendor-DmpaChgWKs.js`, `modulepreload-polyfill`) |
| Obfuscation | String array rotation + hex encoding (`_0x409c`, `_0xf62f`) |

---

## Silah Sistemi

Tüm silahlar tam istatistikleriyle kaynak kodda mevcut:

| Silah | Hasar | Headshot Çarpanı | Penetrasyon | Spread | Reload |
|-------|-------|-----------------|-------------|--------|--------|
| AK-47 | 36 | x4 | 2.0 | 0.0006 | ~2.4s |
| M4A4 | 33 | x4 | 2.5 | 0.0003 | ~3.1s |
| AWP | 115 | x4 | 2.5 | 0.0003 | ~3.7s |
| FAMAS | 30 | x4 | 2.0 | 0.0006 | 3.3s |
| Deagle | 53 | x4 | 1.0 | — | — |
| SCAR-20 | 80 | x4 | 2.5 | 0.0003 | 3.1s |
| G3SG1 | 80 | x4 | 2.5 | 0.0003 | 4.7s |
| M249 | — | x4 | 1.0 | — | 5.7s |
| Negev | — | x4 | 1.0 | — | 5.7s |

Her silahın ayrıca **recoil seed**, `recoilAngle`, `recoilMagnitude`, `recoveryStartBullet` değerleri de kaynak kodda mevcut.

---

## Oyun Modları ve Haritalar

```
Aim Training   → aim_arena        (solo, 3600s, bot yok)
Surf           → surf_boreas / surf_utopia_njv
AWP Duel       → awp_lego_2       (1v1, first to 5 rounds)
Rifle 2v2      → aim_usp_anubis   (2v2, first to 9 rounds, 30 max)
Retake 3v3     → de_dust2_b       (3v3, first to 7 rounds, B bombsite)
```

Harita radar verileri de kaynak kodda mevcut (`posX`, `posY`, `scale`, `size`).

---

## Ekonomi Sistemi

CS2 ile neredeyse birebir:

```
Başlangıç parası : $800
Maksimum para    : $16,000
Kill bonusu      : $300
Round kazanma    : $3,250
Loss bonus base  : $1,400  (+$500 her kaybedilen roundda, max $3,400)
Bomba plant      : $300
Bomba defuse     : $300
Buy time         : 20 saniye
```

**Buy Menu:**
- AK-47 → $2,700
- M4A4 → $2,900
- AWP → $4,750
- Desert Eagle → $700
- Molotov → $400
- Smoke → $300
- Kevlar + Helmet → $1,000
- Defuse Kit → $400

---

## Round Sistemi

```
Freeze time  : 3 saniye
Round time   : 115 saniye
End time     : 5 saniye
Wins needed  : 13 (tam maç)
Bomb timer   : 40 saniye
Plant time   : 3.2 saniye
Defuse time  : 10 saniye
Defuse (kit) : 5 saniye
```

---

## Bot AI

Bot parametreleri kaynak kodda açık:

```js
runSpeed       : 215 * birim
walkSpeed      : 110 * birim
health         : 100
armor          : 100
damage         : 33
reaction       : [0.4, 0.55] saniye
aimErrorStart  : [6, 10] derece
aimErrorMin    : 0.5 derece
aimSettle      : 0.92
teamwork       : 0.75
aggression     : 0.6
headAim        : 0.12 (kafa hedefleme ihtimali)
burst          : [3, 7] atış
burstPause     : [0.22, 0.55] saniye
fov            : cos(75°) = ~0.259
viewDist       : 90 birim
hearDist       : 26 birim
shotHearDist   : 60 birim
```

Bot isimleri bile kodda: `Dragomir, Goran, Zoran, Branko, Milan` (T tarafı), `Hawkins, Reyes, Ward, Doyle` (CT tarafı).

---

## Anticheat Sistemi

En ilginç kısım burası. Hem **client-side detection** hem **server-side hit validation** var.

### Client-Side: `ng()` Fonksiyonu

Her 5 saniyede bir çalışan `reportTamper()` aracılığıyla sunucuya raporlanıyor.

| Flag | Tetikleyici |
|------|------------|
| `three-patched` | Three.js native methodları override edilmişse (`add`, `updateMatrixWorld`, `onBeforeRender`...) |
| `game-wrapped` | `fire_k7arb`, `inaccuracy_k7arb`, `applyCamera_k7arb` override edilmişse |
| `chams` | Bot modellerinde `depthTest === false` veya `wireframe === true` → wallhack/chams tespiti |
| `ws-patched` | `WebSocket.prototype.send` veya `onmessage` setter'ı native değilse |
| `webgl-patched` | `drawElements`, `drawArrays`, `uniformMatrix4fv` override edilmişse |
| `tostring-patched` | `Function.prototype.toString` native değilse (en klasik hook tespiti) |
| `builtins-patched` | Native JS built-in'ler değiştirilmişse |
| `known-esp` | `__botOverlay` window'da varsa veya `#bot-overlay-canvas` DOM'da varsa |
| `overlay-canvas` | Ekranın %25'inden büyük yabancı canvas varsa |
| `overlay-svg` | Ekranın %25'inden büyük yabancı SVG varsa |
| `automation` | `navigator.webdriver === true` (Selenium/Puppeteer tespiti) |
| `astra-tool` | `__ASTRA_TOOL__` window'da varsa veya `__es*` prefix'li property varsa |
| `three-devtools` | `__THREE_DEVTOOLS__` browser extension mevcut |
| `debug-url` | URL'de `?debug` parametresi varsa |

### `_k7arb` Suffix Trick

Kritik oyun fonksiyonları kasıtlı garip suffix ile isimlendirilmiş:
```
fire_k7arb, inaccuracy_k7arb, stdInaccuracy_k7arb,
punchPitch_k7arb, punchYaw_k7arb, applyCamera_k7arb,
viewPunchPitch_k7arb, flinchStack_k7arb ...
```
Biri bunları monkey-patch etmeye kalkarsa `hasOwnProperty` check ile yakalanıyor.

### `tg()` — Native Code Checker
```js
tg = (fn) => /\{\s*\[native code\]\s*\}\s*$/.test(
    Function.prototype.toString.call(fn)
)
```
WebSocket, WebGL ve diğer native API'lara uygulanıyor.

### Server-Side Hit Validation

- Client hit gönderir → server **`hitok`** veya **`hitno`** döner
- `hitno` gelirse `refuseHit()` devreye girer → konsola `"a shot the anti-cheat threw out"` yazılır
- Aimbot kullansa bile **server hit'i onaylamazsa geçerli sayılmaz**

### Raporlama

```js
// Her 5 saniyede bir
setInterval(() => this.reportTamper(), 5000)

// Sunucuya giden mesaj:
{ t: "tamper", k: ["chams", "ws-patched", "overlay-canvas"] }
```

---

## Network Protokolü (WebSocket)

Tüm mesajlar JSON, `t` field'i mesaj tipini belirliyor.

**Client → Server:**

| Mesaj | Açıklama |
|-------|---------|
| `hello` | Bağlantı açılınca — `name`, `platform`, `fp`, `gpu`, `ticket` |
| `queue` | Matchmaking kuyruğuna gir |
| `cancel` | Kuyruktan çık |
| `host` | Oda oluştur |
| `join` | Oda kodyla katıl |
| `accept` | Maç kabul et |
| `tamper` | AC flag'lerini raporla |
| `defuse` | Bomba defuse et |
| `rematch` | Tekrar oyna |
| `ticket` | Auth ticket iste |

**Server → Client:**

| Mesaj | Açıklama |
|-------|---------|
| `welcome` | Bağlantı onayı |
| `ticket` | Auth token |
| `hitok` | Hit onaylandı |
| `hitno` | Hit reddedildi (AC) |
| `roomvote` | Oy açıldı/sonuçlandı |
| `roomrated` | Maç rating güncellendi |

---

## Fingerprint Sistemi

Bağlantı açılırken `li()` fonksiyonu çalışır, sunucuya `fp` ve `gpu` gönderilir.

**Fingerprint için toplanan veriler:**
1. **WebGL renderer string** (GPU adı, max 120 karakter)
2. **Canvas 2D fingerprint** — özel gradient + font render → `toDataURL()` → SHA-256
3. `screen.width x screen.height x colorDepth`
4. `navigator.hardwareConcurrency` (CPU çekirdek sayısı)
5. `navigator.deviceMemory` (RAM miktarı)
6. `Intl.DateTimeFormat().resolvedOptions().timeZone`
7. `navigator.languages`
8. `navigator.platform`
9. `navigator.maxTouchPoints`

Sonuç: **SHA-256 → ilk 16 byte → hex string** (32 karakter)  
Bu fingerprint sunucuya her WebSocket bağlantısında iletiliyor.

---

## Asset Şifreleme

3D model (`.glb`, `.bin`) ve efekt (`.rgba`) dosyaları **özel XOR stream cipher** ile şifrelenmiş.

**Magic Header:**
- `AST1` (byte: `[65, 83, 84, 49]`) → normal şifreleme
- `AST2` (byte: `[65, 83, 84, 50]`) → skip-mode şifreleme

**Algoritma özeti:**
```
key = seed XOR imul(offset, 0x9E3779B1) XOR 0x27E9C6E1
key = imul(key ^ (key >>> 16), 0x85EBCA6B)
key = imul(key ^ (key >>> 13), 0x4029B020)
data[i] ^= key ^ (key >>> 16)
```
Skip-mode'da her 1024 byte'lık blokta ilk 4 byte şifrelenmeden geçiliyor.

---

## CheatGlobal Konu Başlığı

Önerilen başlıklar:

```
[Kaynak Analiz] astrastrike.fun — Browser CS2 Klonunun Deobf JS Analizi
(Silah Stats, Ekonomi, AC Sistemi, Fingerprint, WebSocket Protokolü)
```

veya kısa:

```
astrastrike.fun JS deobf — AC sistemi, fingerprint, WS protokolü ve tüm oyun verisi
```

---

*Analiz: main.deobf.js (~79k satır, ~2.5MB) ve deobfuscated.js (~44k satır, ~2.1MB)*
