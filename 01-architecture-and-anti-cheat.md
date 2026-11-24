# Technical Architecture & 60 Hz Server-Authoritative Anti-Cheat

## 1. Zero-Install WebGPU / WebGL 2.0 Client

Traditional PC and console combat racers lock players behind 40–70 GB downloads, while legacy Unity WebGL ports suffer from 100+ MB WASM cold starts and mobile browser memory crashes.

ASTRORACE runs on a native **TypeScript + Babylon.js 9.28 + React 19** architecture featuring:
- **Sub-3.5-Second Cold Start:** Instant click-to-hangar admission from X, Discord, Telegram, Twitch, or YouTube links with one-click guest onboarding and zero mandatory wallet popups at entry.
- **Zero-Allocation Render Loop:** Cached PBR shader materials, adaptive device quality scaling across desktop and mobile viewports, and a procedural WebAudio Doppler engine.

---

## 2. Full Simulation Isomorphism (`@astrorace/simulation`)

In high-velocity combat racing (`216.6 m/s` cruise, `333.3 m/s` inside `HYPER` gates), splitting client prediction and server authority across different programming languages introduces floating-point drift and collision desynchronizations.

ASTRORACE executes the **exact same TypeScript physics package (`rules-v7`)** on both the browser client (for zero-lag 60 Hz client-side prediction and Hermite spline reconciliation) and the authoritative Node.js Colyseus room server:
- **Deterministic 60 Hz Symplectic Integration:** Fixed `16.666 ms` simulation step with sub-tick finish-line arbitration (`0.001 ms` precision).
- **2.5D Frenet-Serret Clothoid Track Solver:** Evaluates longitudinal progress (`s`), lateral offset (`d`), hover altitude (`h`), and heading (`theta`) along continuous clothoid splines.
- **Swept-Capsule Continuous Collision Detection (CCD):** Prevents high-speed tunneling during hull rams, missile impacts, and barrier grazes, paired with diminishing-returns shield mitigation:

$$DR = \min\left(0.70,\ \frac{S}{S + 250}\right)$$

---

## 3. Cryptographic Admission & Anti-Cheat Pipeline

The browser client is treated strictly as an **untrusted input terminal**:

1. **Compact Bitmask Inputs Only:** Clients transmit 30 Hz `16-byte` steering and ability bitmasks (`ClientInputPacket`). Position, speed, energy, cooldown, or damage claims from the client are never accepted.
2. **HMAC/SHA-256 Signed Loadout Tickets:** Upon matchmaking admission, `server-api` reads the pilot's locked PostgreSQL inventory, computes the canonical `RaceLoadoutSnapshot { rulesVersion, modules }`, hashes it with `SHA-256`, and signs a short-lived HMAC admission JWT.
3. **Bandwidth-Efficient Binary Delta Sync:** `server-room` verifies the cryptographic ticket before spawning the craft and broadcasts 20 Hz Colyseus binary delta snapshots (`80–180 bytes` per tick, `< 30 kbps` downstream).
4. **Double-Entry PostgreSQL 18.6 ACID Ledger:** Every currency transfer, container opening, enhancement, and marketplace listing executes inside serializable transactions (`ledger_transactions` + `ledger_entries`). Listing a module on the P2P marketplace atomically sets `IS_LOCKED = TRUE`, immediately rejecting concurrent equip, upgrade, or salvage attempts with `409 Conflict`.
