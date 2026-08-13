# MetaQuestGuard

**Paid anti-cheat for Meta Quest VR games — currently in active development.**

Standalone C# package built on Meta Attestation, identity, and entitlement — not PC kernel drivers or third-party cloud scanners. Designed especially for competitive and Gorilla Tag–style locomotion titles.

[![Status](https://img.shields.io/badge/Status-In%20Development-orange?style=flat-square)](#status)
[![Quest](https://img.shields.io/badge/Platform-Meta%20Quest%202%2FPro%2F3%2F3S-00A4E4?style=flat-square)](https://developers.meta.com/horizon/)
[![Unity](https://img.shields.io/badge/Engine-Unity%20(IL2CPP)-000000?style=flat-square&logo=unity)](https://unity.com/)
[![Photon](https://img.shields.io/badge/Netcode-Photon%20%7C%20Mirror%20%7C%20Agnostic-7B68EE?style=flat-square)](https://www.photonengine.com/)
[![License](https://img.shields.io/badge/License-Commercial%20(Paid)-red?style=flat-square)](#license--access)

> **In development.** APIs, detection coverage, and packaging may change before a stable release.  
> Paid / early-access builds are planned as an **obfuscated DLL** (ConfuserEx / Obfuscar / commercial IL obfuscator + string encryption). Source access will be available to licensed studios.  
> Product docs site: `MetaQuestGuard.html` in this repo.

---

## Why MetaQuestGuard?

Competitive Quest titles (especially Gorilla Tag–style locomotion) face:

- Client-side authority abuse (speed, fly, stretch, memory edits)
- Mod loaders & injectors (MelonLoader, BepInEx, Harmony, Frida, QuestPatcher)
- Identity / economy abuse and ban evasion
- False positives from naive “max speed” checks on legitimate momentum flings

Meta’s own guidance is a **layered model**: entitlement → Attestation → server-side validation. Photon (and most netcode) is **not** inherently authoritative. MetaQuestGuard implements that model as a **standalone C# package** that talks only to *your* backend.

| Problem | Guard’s answer |
|--------|----------------|
| Pirated / repackaged APK | Entitlement + Meta Attestation (server-verified) |
| Spoofed identity | Meta User Verification → org-scoped ID → backend play token |
| Speed / fly / stretch hacks | Server-side kinematics + GorillaLocomotion-aware validator |
| Mod loaders & injection | Reflection + `/proc/self/maps` + Harmony type-graph scans |
| RPC / spawn floods | Opt-in Photon `AntiSpamRpc` |
| Master Client seizure | Opt-in Photon `AntiMasterClientSwitch` |
| False positives on flings | Per-type thresholds, suppression windows, history re-validation |

---

## Design pillars

- **Quest-native trust root** — Meta Attestation, not PC anti-cheat. Zero PC-only detections in the Quest build (they would be incompatible with Attestation and the platform model).
- **Attestation tokens are cached** for their full validity window. **No heartbeat that mints new tokens** (Meta rate-limits Attestation; a heartbeat would waste budget for no security gain).
- **Advanced** device integrity always passes gated actions. **Basic** is restricted unless you opt in. Nothing below Basic.
- **Security patch age** (default 30 days) is gated the same way a failed attestation is.
- **Read-only, user-mode, worker-thread only** — never patches processes, never blocks the main thread, never tips off the player by default.
- **Silent reporting** — signed reports go to *your* server. Guard itself does not kick (Photon helpers that do are opt-in).
- **Netcode-agnostic core** — Mirror bridge and Photon helpers are included in the current tree; Fusion and other transports are planned on the same pattern.

---

## What’s included

```
MetaQuestGuard/
├── MetaQuestGuard.html          # Product / docs site
└── Scripts/
    ├── MetaQuestGuard.cs                 # Core Guard + bootstrap + generic VR validator
    ├── MetaQuestGuardGorillaLocomotion.cs # Gorilla Tag–style locomotion validator (v2)
    ├── MetaQuestGuardMirrorIntegration.cs # Mirror server bridge
    ├── AntiSpamRpc.cs                    # Photon: rate-limit RPC + Instantiation
    └── AntiMasterClientSwitch.cs         # Photon: block illicit Master Client switches
```

### Core capabilities

| Area | Features |
|------|----------|
| **Session** | Bootstrap via entitlement + user proof + attestation → short-lived play token + HMAC key |
| **Gates** | `IsAllowedHighValueAction()` for ranked / trades / progression |
| **Self-protection** | Guard method-body integrity hash vs baseline in token; signed worker “alive” pings |
| **Runtime scans** | Mod loaders, Harmony graphs, injected libs, unusual `libunity`/`libil2cpp` maps, dynamic codegen, debuggers, emulator signals *(expanding)* |
| **Generic VR kinematics** | Sequence, rate, clock skew, speed envelopes, reach, rubber-band, joint limits, short history + prediction |
| **GorillaLocomotion v2** | Surface contact, release/decay, mid-air accel, air time + height gain, rubber-band, statistical outliers, history buffer, physics prediction, long-arm / joint checks |
| **Anti-false-positive** | Per-detection confidence thresholds, post-action suppression windows, history for server re-validation, coalesced “detection storm” reports |
| **Reporting** | Versioned schema (v2), HMAC-signed, ring buffer, POST to your endpoint only |

---

## Quick start

### 1. Bootstrap (once at launch)

After Meta Platform SDK init (entitlement within ~10s, user verification, attestation with server nonce):

```csharp
StartCoroutine(MetaQuestGuardBootstrap.Run(
    guard: MetaQuestGuard.Instance,
    backendUrl: "https://your-server.example.com/anticheat/bootstrap",
    metaUserId: metaUserId,
    userProofNonce: userProofNonce,
    attestationToken: attestationToken,
    onComplete: (ok, playToken) =>
    {
        if (!ok) { /* fail closed for ranked */ return; }
        // Session is cached in Guard — no refresh heartbeat
    }));
```

Your backend should verify nonce, package, cert hash, app/device integrity, map to an org-scoped ID, and return a short-lived play token + HMAC key (+ optional Guard integrity baseline).

### 2. Gate high-value actions

```csharp
if (!MetaQuestGuard.Instance.IsAllowedHighValueAction(out string reason))
{
    // reason: no_valid_session | security_patch_outdated | device_integrity_insufficient
    return;
}
// join ranked queue / claim reward / trade ...
```

### 3. Server-side movement validation

**Gorilla-style locomotion** (authoritative server):

```csharp
var loco = new GorillaLocomotionGuard();
loco.NearestClimbableSurfaceDistance = pos => /* distance to climbable */;

var result = loco.ValidateFrame(playerId, sample);
if (!result.IsAccepted)
{
    MetaQuestGuard.Instance.ReportDetection(
        confidence: 0.8f,
        type: $"gorilla_locomotion_{result.Reason}",
        evidence: result.Detail,
        historyJson: loco.GetHistoryJson(playerId));
    return; // do not apply state
}

// Optional: reduce false positives after a legit fling
MetaQuestGuard.Instance.NotifyHighIntensityAction();
```

**Generic VR envelope** (non-Gorilla games):

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

## Architecture (recommended)

```
Quest client
  ├─ Meta entitlement (≤10s of launch)
  ├─ GetUserProof → your backend → user_nonce_validate → org_scoped_id
  ├─ Attestation (server nonce) → your backend verifies token claims
  └─ MetaQuestGuard (cached session, worker scans, silent reports)
         │
         ▼
Your backend
  ├─ Play token + HMAC + optional Guard integrity baseline
  ├─ Photon Custom Auth / webhooks (trusted session facts)
  └─ Authoritative game server
         ├─ GorillaLocomotionGuard / ValidateInputCommand
         ├─ Scoring, inventory, cooldowns, progression
         └─ Signed detection reports → moderation / bans
```

Treat **every** client pose, RPC, and custom property as untrusted unless your server can re-derive or validate it. Photon room membership ≠ gameplay authority.

---

## Detection overview

**Runtime (client worker, read-only)**  
Mod loaders · Harmony type graphs · library injection · unusual native maps · dynamic codegen · instruction prologue hashes · Guard integrity · worker alive · debuggers · emulator signals · multi-instance mutex  

**Server (authoritative)**  
Session / sequence / rate / clock · head/hand speed · reach · rubber-band · joint limits · Gorilla surface contact · release & drag decay · mid-air accel · air time & height gain · statistical outliers · history spikes · physics prediction  

**Photon (opt-in)**  
RPC spam · Instantiation spam · unauthorized Master Client switch  

Confidence thresholds and short suppression windows after legitimate high-intensity actions keep false positives down without weakening real cheats.

---

## Privacy & platform notes

- Reports go **only** to the URL you configure. No third-party anti-cheat cloud.
- Prefer **less invasive on-device, stronger on server** — aligned with Meta Data Use Checkup, privacy policy, and Attestation guidance (proportional enforcement, device ban with your own moderation pipeline).
- VR motion can be identifying; design telemetry as if it may be regulated biometric-adjacent data under GDPR/CCPA when used for linkage.
- Meta Attestation is Android Quest–native (Quest 2 / Pro / 3 / 3S). Do not rely on Google Play Integrity as the primary Quest trust signal.

---

## Requirements

- Unity (IL2CPP recommended for Quest)
- Meta Platform / Horizon SDK (entitlement, user verification, Attestation API)
- Your own HTTPS backend for bootstrap + report ingestion
- Optional: Photon PUN / Realtime (for the two helper scripts), Mirror (bridge included)

---

## Status

MetaQuestGuard is a **paid anti-cheat in active development**.

| Phase | Meaning |
|-------|---------|
| **Now** | Core Guard, GorillaLocomotion validator, Mirror bridge, Photon helpers, and docs are being built and iterated |
| **Next** | Hardening, early studio pilots, obfuscated package pipeline |
| **Later** | Stable paid release, broader transport adapters, continued detection updates |

Expect breaking changes to APIs and config until a versioned stable release is tagged. Feedback from Quest / Gorilla-style projects is welcome and directly shapes the roadmap.

---

## License & access

**Commercial (paid) product — not open source.**

- Development builds and early access are offered under a commercial license.
- Release packages are intended to ship as an **obfuscated DLL**; source access under studio agreement.
- Do not redistribute the package, scripts, or source without a valid license.

For early access, licensing, or pilot interest, contact the author via the channels on the product page (`MetaQuestGuard.html`) or the GitHub profile linked to this repository.

---

## Disclaimer

No anti-cheat is perfect. Determined attackers with full client control can still attempt bypasses. MetaQuestGuard aims to raise attacker cost, bind sessions to Meta-verified identity and device integrity, and move competitive outcomes to a server-authoritative boundary — the same layered approach Meta and Photon document.

Because the project is **in development**, treat current builds as pre-release: validate thoroughly in your own environment, keep server authority as the real trust boundary, and use proportional enforcement with a clear appeals path.

---

<p align="center">
  <strong>MetaQuestGuard</strong> · Paid · In development · Attestation-native · GorillaLocomotion-aware<br/>
  <sub>Quest 2 / Pro / 3 / 3S · Unity · Photon &amp; Mirror helpers · Your backend only</sub>
</p>

