# MetaQuestGuard

**Paid anti-cheat for Meta Quest & Android VR — commercial license.**

Standalone Unity package for Quest and Android VR. Runtime integrity scans, FridaShield instrumentation detection, native-library baselines, GorillaLocomotion-aware server validation, and multiplayer helpers for Mirror, Photon, and PlayFab.

**Meta Platform Attestation is not required or included.** This edition uses your own authentication / session model (or anonymous scanning) instead.

[![Status](https://img.shields.io/badge/Status-Paid%20Commercial-blue?style=flat-square)](#license--access)
[![Quest](https://img.shields.io/badge/Platform-Meta%20Quest%20%2F%20Android%20VR-00A4E4?style=flat-square)](https://developers.meta.com/horizon/)
[![Unity](https://img.shields.io/badge/Engine-Unity%20(IL2CPP)-000000?style=flat-square&logo=unity)](https://unity.com/)
[![Netcode](https://img.shields.io/badge/Netcode-Mirror%20%7C%20Photon%20%7C%20PlayFab-7B68EE?style=flat-square)](#whats-included)
[![License](https://img.shields.io/badge/License-Commercial%20(Paid)-red?style=flat-square)](#license--access)

> **Paid product.** Licensing and purchase access are handled through your sales / download flow.  
> Product docs site: `MetaQuestGuard.html` in this repo (or the hosted docs page).

---

## Why MetaQuestGuard?

Competitive Quest titles (especially Gorilla Tag–style locomotion) face:

- Client-side authority abuse (speed, fly, stretch, memory edits)
- Mod loaders & injectors (MelonLoader, BepInEx, Harmony, Frida, QuestPatcher)
- False positives from naive “max speed” checks on legitimate momentum flings
- RPC / spawn floods and Master Client abuse on Photon

MetaQuestGuard raises the cost of client-side cheating and moves important gameplay decisions to an authoritative server — without depending on Meta Platform Attestation.

| Problem | Guard’s answer |
|--------|----------------|
| Mod loaders & injection | Runtime integrity scans + FridaShield multi-vector checks |
| Native library tampering | Encrypted Android native-library baseline (Editor-generated) |
| Speed / fly / stretch / long-arm | Server-side kinematics + GorillaLocomotion-aware validator |
| Rubber-band teleports & sustained flight | Sequence, release/decay, airborne, and geometry checks |
| RPC / spawn floods | Opt-in Photon `AntiSpamRpc` |
| Master Client seizure | Opt-in Photon `AntiMasterClientSwitch` |
| False positives on flings | Per-signal confidence thresholds + suppression windows |

---

## Design pillars

- **No Meta Attestation dependency** — works with your existing auth/session stack or anonymous scanning.
- **Read-only, user-mode, worker-thread** — never patches processes, never blocks the main thread, never tips off the player by default.
- **Silent reporting** — detections are reported (optionally HMAC-signed) to *your* backend. Guard itself does not kick; Photon helpers that do are opt-in.
- **Server authority first** — client signals are supporting evidence; competitive outcomes live on the server.
- **Netcode-friendly** — Mirror bridge and Photon helpers included; PlayFab CloudScript path for high-confidence enforcement.
- **Configurable thresholds** — per-detection confidence values and anti-false-positive windows.

---

## What’s included

```
MetaQuestGuard/
├── MetaQuestGuard.html          # Product / docs site
└── Scripts/                     # (package contents under license)
    ├── Core Guard + bootstrap + generic VR validator
    ├── GorillaLocomotion-aware movement validator
    ├── Mirror server bridge
    ├── AntiSpamRpc              # Photon: rate-limit RPC + Instantiation
    └── AntiMasterClientSwitch   # Photon: block illicit Master Client switches
```

### Core capabilities

| Area | Features |
|------|----------|
| **Session** | Anonymous scanning *or* `StudioSession` (SessionId, optional HMAC key, player UniqueId, expiry, build version, integrity baseline) |
| **Runtime integrity** | Mod loaders, library injection, instruction patching, FridaShield, native-library baseline, Guard self-integrity, debuggers, virtualized device, multi-instance, dynamic code generation |
| **FridaShield** | Native multi-vector checks: ports, memory maps, threads, RWX regions, Unix sockets, ELF hashes, TracerPid, memory tracing signals |
| **Generic VR validation** | Sequence monotonicity, head/hand speed, clock skew, command rate, rubber-band, joint/reach limits |
| **GorillaLocomotion** | Arm span, grip/release speed, momentum decay, sustained flight, grab geometry, rubber-band teleports, long-arm behaviour |
| **PlayFab** | Optional CloudScript path for high-confidence detections → Discord + configurable bans |
| **Photon helpers** | AntiSpamRpc, AntiMasterClientSwitch |
| **Mirror bridge** | Forwards validated locomotion frames; exposes detections to moderation hooks |
| **Reporting** | Optional HMAC signing; silent detection designed for server enforcement rather than automatic client quit |

---

## Package compatibility

| Target | Status |
|--------|--------|
| Meta Quest / Android VR | ✓ Primary target |
| Mirror Networking | ✓ Bridge included |
| Photon PUN | ✓ Helpers included |
| PlayFab | ✓ CloudScript support |

This edition does **not** include or require Meta Platform Attestation. You can run anonymously, or supply a `StudioSession` with your own session ID, optional HMAC key, player identifier, expiry, build version, and integrity baseline.

---

## Quick start

No Meta Attestation bootstrap is required.

### 1. Add MetaQuestGuard

Place `MetaQuestGuard` in your scene. With `allowHighValueWithoutSession` enabled, runtime scanning starts automatically.

### 2. Connect your session (optional)

```csharp
// No Meta Attestation required
var session = new MetaQuestGuard.StudioSession {
    SessionId = myMatchToken,
    UniqueId = myPlayerId,
    HmacKey = myOptionalHmacKey,           // optional — enables signed reports
    TokenExpiresAtUtc = DateTime.UtcNow.AddHours(4)
};

MetaQuestGuard.Instance.SetStudioSession(session);

// Or simply allow anonymous scanning and keep enforcement server-side.
```

Alternatively, use the PlayFab CloudScript backend for high-value detections.

### 3. Validate gameplay on the server

**Gorilla-style locomotion** (authoritative server):

```csharp
var result = GorillaLocomotionGuard.ValidateFrame(playerId, sample);
if (!result.IsAccepted)
{
    // Forward detection via OnDetected / your moderation pipeline
    return; // do not apply state
}
```

**Generic VR input:**

```csharp
var result = MetaQuestGuard.Instance.ValidateInputCommand(cmd, gameState, nowMs);
```

### 4. Listen for detections

```csharp
MetaQuestGuard.Instance.OnDetected += (confidence, type, evidence) =>
{
    // Your moderation / telemetry pipeline
};
```

### 5. Photon helpers (optional)

Attach `AntiSpamRpc` and/or `AntiMasterClientSwitch` on a persistent object. They report through Guard and can enforce kicks only if you enable it.

---

## Runtime detection overview

A worker thread performs configurable read-only sweeps. Reports expose confidence, type, evidence, session/build identifiers, and optional history.

| Detection | What the package checks | Default threshold |
|-----------|-------------------------|-------------------|
| Mod loaders | MelonLoader, BepInEx, Harmony, Il2CppInterop, QuestPatcher and related signatures | 0.90 |
| Library injection | Injection / instrumentation artifacts and loaded-library signals | 0.90 |
| Instruction patching | Critical native function pointers / inline-hook indicators | 0.95 |
| Frida instrumentation | FridaShield native multi-vector checks | 0.60 |
| Native library tamper | Encrypted native-library baseline comparison (Android) | 0.92 |
| Guard integrity | Self-integrity hash verification | 0.99 |
| Debuggers | Debugger attachment signals | Configurable |
| Virtualized device | Device / build soft integrity indicator | Configurable |
| Multiple clients | Named single-instance mutex | Configurable |
| Dynamic code generation | Suspicious dynamic-code behaviour | Configurable |

**Important limitation:** Without Meta Attestation there is no platform-level proof that the binary is genuine or that the device is untampered. Client-side signals should be combined with authoritative server validation.

---

## Server validation overview

| Validation | Implementation | Strength |
|------------|----------------|----------|
| Sequence monotonicity | Per-player command sequencing and replay detection | High |
| Head / hand kinematics | Default max head ~20 m/s, hand ~25 m/s + time-delta checks | Medium |
| Arm span & grip speed | Reach cap from the head; grip-speed limits while gripped | Medium |
| Release & momentum | Caps release speed; validates expected post-release decay | High |
| Sustained flight | Airborne movement without grip/ground contact beyond expected arc | High |
| Grab geometry | Optional callback: new grip near valid climbable geometry | Medium |
| Rubber-band teleport | Short-delta position spike detection | High |
| Long-arm exploit | Reach / joint configuration checks for abnormal extension | High |
| Rate limiting | Commands-per-second cap and flood detection | Medium |
| Clock synchronization | Client timestamp sanity with configurable skew tolerance | Low |

---

## Deployment model

| Setup | How it works | Best fit |
|-------|--------------|----------|
| **Anonymous** | Guard starts scanning automatically; reports can be unsigned | Projects that already identify players elsewhere |
| **StudioSession** | Supply SessionId, optional HMAC key, UniqueId, expiry, build version, baseline | Games with their own authentication / session service |
| **PlayFab** | CloudScript receives high-value detections and can ban | PlayFab-backed multiplayer titles |
| **Photon** | Anti-spam and Master Client protection helpers | Photon PUN multiplayer |
| **Mirror** | Bridge submits movement frames for server validation | Mirror networking projects |

---

## Architecture (recommended)

```
Quest / Android VR client
  └─ MetaQuestGuard
         ├─ Runtime integrity scans (worker thread, read-only)
         ├─ Optional StudioSession (or anonymous)
         └─ Silent detection reports (optionally HMAC-signed)
                │
                ▼
Your backend
  ├─ Session / auth / play token (your model)
  ├─ Authoritative game server
  │     ├─ GorillaLocomotionGuard / ValidateInputCommand
  │     ├─ Scoring, inventory, cooldowns, progression
  │     └─ Detection reports → moderation / bans
  └─ Optional: PlayFab CloudScript, Photon Custom Auth / webhooks
```

Treat **every** client pose, RPC, and custom property as untrusted unless your server can re-derive or validate it. Photon room membership ≠ gameplay authority.

---

## Privacy & platform notes

- Reports go **only** to the URL / backend you configure. No third-party anti-cheat cloud.
- Prefer less invasive on-device checks and stronger server authority.
- VR motion data can be identifying; design telemetry with GDPR/CCPA in mind when used for linkage.
- This package does not provide platform-level device attestation.

---

## Requirements

- Unity (IL2CPP recommended for Quest)
- Your own HTTPS backend (or PlayFab) for report ingestion and enforcement
- Optional: Photon PUN / Realtime (helper scripts), Mirror (bridge included), PlayFab

---

## License & access

**Commercial (paid) product — not open source.**

- This is a paid anti-cheat package. A commercial license is required.
- Do not redistribute the package, scripts, or source without a valid license.
- Pricing / purchase details should be added to your sales or license link (not specified in the package itself).

For licensing or access, use the channels on the product page (`MetaQuestGuard.html`) or the contact method linked to this repository.

---

## Disclaimer

No anti-cheat is perfect. Determined attackers with full client control can still attempt bypasses. MetaQuestGuard aims to raise attacker cost, surface integrity signals, and move competitive outcomes to a server-authoritative boundary.

Validate thoroughly in your own environment, keep server authority as the real trust boundary, and use proportional enforcement with a clear appeals path.

---

<p align="center">
  <strong>MetaQuestGuard</strong> · Paid commercial package · No Meta Attestation dependency · Quest / Android VR · GorillaLocomotion-aware
</p>
  <strong>MetaQuestGuard</strong> · Paid · In development · Attestation-native · GorillaLocomotion-aware<br/>
  <sub>Quest 2 / Pro / 3 / 3S · Unity · Photon &amp; Mirror helpers · Your backend only</sub>
</p>

