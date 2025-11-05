# 🛰️ FiveM Network Desync Analysis — Lag vs Exploit Misunderstanding

> A detailed technical and human breakdown of a real FiveM desync incident — exploring how network jitter, packet loss, and interpolation buffer drains can mimic exploit behavior, and how to properly identify and mitigate it.

> Why I made this: Because I want people to understand that there’s a big difference between genuine lag and actual exploit abuse — and that technical issues shouldn’t be mistaken for cheating.

---

## 📑 Table of Contents

- [🎯 Purpose](#-purpose)
- [⚙️ Background](#️-background)
- [🧩 Technical Breakdown](#-technical-breakdown)
- [🔎 Example Case Analysis](#-example-case-analysis)
- [🧠 Discussion Highlights](#-discussion-highlights)
- [🛠️ Diagnostic Toolkit](#️-diagnostic-toolkit)
- [🔧 Mitigation Guidelines](#-mitigation-guidelines)
- [💬 Human Element](#-human-element)
- [🧾 Summary](#-summary)
- [🧪 Reproduction Steps](#-reproduction-steps)
- [❓ FAQs](#-faqs)
- [📖 Glossary](#-glossary)
- [🤝 Contributing](#-contributing)
- [📜 License](#-license)
- [👥 Credits](#-credits)
- [📚 Further Reading](#-further-reading)

---

## 🎯 Purpose

This repository documents a real case where a **player was accused of exploit abuse** due to visual snapping and positional desync — but the root cause was **lag, jitter, and RAGE netcode limitations**.

### Goals

✅ Explain **how FiveM networking actually works**  
✅ Show **why desync ≠ cheating**  
✅ Provide **diagnostic methods** for developers and server admins  
✅ Suggest **fixes and mitigations** for smoother gameplay  
✅ Highlight the importance of **technical literacy** in moderation decisions  

---

## ⚙️ Background

FiveM's multiplayer layer is built atop Rockstar's **RAGE engine**. To hide latency, it uses:

- **Client-side prediction** (guessing where entities will be)
- **Server reconciliation** (correcting the client with truth)
- **Interpolation buffers** (smoothing between snapshots)

Under stable conditions, this hides latency effectively. Under **packet loss**, **jitter**, or **server hitching**, the interpolation buffer can **run dry**, and the client begins **extrapolating outdated data**.

### Common Symptoms

| Symptom | Technical Cause |
|---------|-----------------|
| 🔁 Snapping / Rubber-banding | Buffer drain + reconciliation correction |
| 🔫 Delayed or ghost hits | Server-client state mismatch |
| 🚀 Sudden position corrections | Extrapolation correction snap |
| 🎭 Animation desyncs | Animation state packet loss |

⚠️ **These symptoms are often misread as exploits or animation abuse**, but they're actually network artifacts.

---

## 🧩 Technical Breakdown

### 1️⃣ Client-Side Prediction & Interpolation

The client predicts future positions from last known state. It renders smoothly by interpolating between recent **authoritative** server snapshots.

**Normal Flow:**
1. Server sends snapshots every tick (≈30–60 Hz)
2. Client buffers 2–3 snapshots (~100 ms buffer)
3. Renderer interpolates between the last two snapshots
4. If snapshots are missing → **extrapolation** starts
5. When a new snapshot arrives → **reconciliation** nudges or snaps to truth

**Key Concept:** If prediction error exceeds ~100 ms, the client **snaps** to server truth → visible "teleport."

### 2️⃣ Server Reconciliation & Tick Drift

The server ticks at a fixed rate (commonly 60 Hz, ~16.6 ms/tick). If **server frame time exceeds tick budget**, snapshot intervals widen and **drift** accumulates.

**Causes & Effects:**

| Cause | Effect | Impact |
|-------|--------|--------|
| Server frame > 17 ms | Sync delay & widened intervals | Buffer starvation |
| Resource stutter/loop | Snapshot queue stalls | Extrapolation kicks in |
| Network jitter/loss | Out-of-order snapshots | Reconciliation snaps |
| High entity count (OneSync) | Slower delta compression | Increased latency |

### 3️⃣ RAGE Engine Networking Constraints

RAGE predates modern rollback netcode. Limitations include:

❌ No full rollback (snapshot delta compression without state rewind)  
❌ Limited per-tick reconciliation exposure to FiveM  
❌ Non-deterministic tick alignment across threads  

🔒 **These are engine-level constraints**, not just script bugs.

---

## 🔎 Example Case Analysis

### Real Incident Report

**Incident:** Player appears to "snap" mid-emote; admins interpret as animation-cancel exploit.

**Observed:**
- Position correction during emote under load
- Other players see sudden shift ("teleport")

**Diagnostic Data:**

| Tool | Finding | Significance |
|------|---------|--------------|
| `netgraph` | 2-5% loss, 50-120ms jitter | Network instability |
| **Profiler** | Tick delta 60–80 ms | Server hitching |
| **frameTime vs serverTime** | 40-60ms mismatch | Desync window |

**Root Cause:**
1. Server enters high load → tick budget exceeded
2. Snapshot interval widens → buffer drains
3. Client extrapolates from stale data
4. Server sends delayed snapshot → hard correction during animation
5. **Appears as exploit, but is LAG ARTIFACT**

**Conclusion:** ✅ No malicious behavior — **Lag + engine constraints** caused false positive

---

## 🧠 Discussion Highlights

**Person1 — 12:52 AM**  
> "If reconciliation uses an interpolation buffer, packet loss alone won't cause snapping unless thresholds are exceeded."

**Chumbis20 — 12:53 AM**  
> "On FiveM/RAGE, buffers drain when jitter spikes; the client extrapolates stale data. That was my case — not an exploit."

**Person1 — 1:09 AM**  
> "Watch `cl_drawPerf 1`. Once the scheduler hits ~17 ms/frame, `frameTime` vs `serverTime` diverge."

**Chumbis20 — 1:10 AM**  
> "On runtime 6248 with load, tick delta drifted to 60–80 ms — buffer drains faster than refills."

**Person1 — 1:15 AM**  
> "Add tick delta logging; flag anomalies > 40 ms. That's your smoking gun for false positives."

---

## 🛠️ Diagnostic Toolkit

### Client-Side Tools

| Command | Purpose | What to Look For |
|---------|---------|------------------|
| `netgraph 1` | Network stats overlay | Ping spikes, packet loss %, choke |
| `cl_drawPerf 1` | Frame timing display | frameTime vs serverTime divergence |
| `resmon 1` | Resource monitor | High CPU/memory usage |

### Server-Side Tools

| Command | Purpose | What to Look For |
|---------|---------|------------------|
| `resmon 1` | Resource monitor | CPU spikes, thread stalls |
| `profile record` | Capture performance trace | Tick budget overruns, hitches |
| `txAdmin` metrics | Server health dashboard | Tick rate drops, desync warnings |

### Custom Logging (Lua)

**Server-Side: Tick Delta Monitor**

```lua
local lastTick = GetGameTimer()

CreateThread(function()
    while true do
        Wait(0)
        local currentTick = GetGameTimer()
        local delta = currentTick - lastTick
        
        if delta > 40 then -- Expected ~16ms at 60 FPS
            print(string.format("^3[DESYNC WARNING] Tick delta: %dms^0", delta))
        end
        
        lastTick = currentTick
    end
end)
```

---

## 🔧 Mitigation Guidelines

### For Server Owners

**Performance:**
- ✅ Keep server tick rate stable at 30-60 Hz
- ✅ Monitor `resmon` for resource spikes
- ✅ Use `txAdmin` performance monitoring
- ✅ Limit entity count (vehicles, NPCs, objects)

**Network Configuration:**

```bash
# server.cfg
sv_maxclients 32              # Don't exceed bandwidth capacity
sv_enforceGameBuild 2802      # Prevent client mismatches
onesync enabled               # Enable OneSync
```

**Script Optimization:**
- ✅ Audit resources for heavy loops (Wait(0) abuse)
- ✅ Batch network events instead of spamming
- ✅ Use server-side validation for critical actions

### For Developers

**Best Practices:**

✅ Add tick delta logging (flag > 40 ms)  
✅ Avoid client-only authority for critical movement  
✅ Implement smoothing for position corrections  
✅ Use appropriate Wait() intervals  

**Position Correction Smoothing:**

```lua
-- Client-side smooth correction
local targetPos = nil
local smoothingAlpha = 0.2

RegisterNetEvent('updatePosition', function(serverPos)
    targetPos = serverPos
end)

CreateThread(function()
    while true do
        Wait(0)
        if targetPos then
            local ped = PlayerPedId()
            local currentPos = GetEntityCoords(ped)
            
            -- Smoothly interpolate
            local newX = currentPos.x + (targetPos.x - currentPos.x) * smoothingAlpha
            local newY = currentPos.y + (targetPos.y - currentPos.y) * smoothingAlpha
            local newZ = currentPos.z + (targetPos.z - currentPos.z) * smoothingAlpha
            
            local distance = #(currentPos - targetPos)
            if distance > 0.1 then
                SetEntityCoords(ped, newX, newY, newZ, false, false, false, false)
            else
                targetPos = nil -- Close enough
            end
        end
    end
end)
```

### For Players

**Network Optimization:**
- ✅ Use wired Ethernet (reduces jitter by 60-80%)
- ✅ Close bandwidth-heavy applications
- ✅ Disable Windows Update during gameplay
- ✅ Close Discord/Steam overlay if causing issues

**Client Configuration:**
- ✅ Cap FPS to stable value (60 or 120, not unlimited)
- ✅ Keep FiveM and game files up to date
- ✅ Verify game file integrity periodically

**Diagnostics Before Reporting:**
- ✅ Use `netgraph 1` to verify connection
- ✅ Screenshot netgraph when experiencing desync
- ✅ Note exact time of incident
- ✅ If persistent: reconnect, restart router, change route

**Communication:**

| ❌ Don't Say | ✅ Do Say |
|-------------|----------|
| "Player X is cheating!" | "I experienced desync around Player X" |
| "They're exploiting!" | "I had 5% packet loss (netgraph attached)" |
| "Ban them!" | "Could admin check logs at [timestamp]?" |

---

## 💬 Human Element

> "Why would I work this hard to explain if I didn't care? This was lag, not an exploit. I'm asking for a fair read of the data."

### Key Takeaways

1. **False positives happen** when technical artifacts are misunderstood
2. **Moderation should be evidence-based** (logs/profiles), not assumptions
3. **Desync is a networking problem**, not always malicious
4. **Education prevents witch hunts** — understanding netcode helps everyone
5. **Benefit of the doubt** should be given under poor network conditions

---

## 🧾 Summary

| Category | Detail |
|----------|--------|
| **Engine** | RAGE (FiveM) |
| **Primary Issue** | Interpolation buffer drain & server tick drift |
| **Symptoms** | Snapping, rubber-banding, false exploit flags |
| **Root Cause** | Jitter, packet loss, server hitching |
| **Detection** | Netgraph, profiler, tick logs |
| **Mitigation** | Smoothing, optimized ticks, buffer stabilization |
| **Verdict** | Lag artifact, not exploit |

---

## 🧪 Reproduction Steps

### Controlled Test

**1. Controlled Load:**
```lua
-- Spawn 50 vehicles to simulate load
RegisterCommand('spawnload', function()
    local coords = GetEntityCoords(PlayerPedId())
    for i = 1, 50 do
        local model = GetHashKey('adder')
        RequestModel(model)
        while not HasModelLoaded(model) do Wait(0) end
        CreateVehicle(model, coords + vector3(math.random(-50, 50), math.random(-50, 50), 0), 0.0, true, false)
    end
end)
```

**2. Induce Jitter:**
- Use network traffic shaper (clumsy.exe on Windows)
- Simulate 2% loss + 50ms jitter

**3. Observe:**
```bash
netgraph 1
cl_drawPerf 1
resmon 1
```

**4. Trigger Desync:**
- Perform emote + movement
- Watch for position snaps

**5. Collect Evidence:**
- Screenshot netgraph
- Save profiler traces
- Log tick anomalies

---

## ❓ FAQs

**Q: Isn't snapping always proof of cheats?**  
A: No. Snapping is common with buffer drains, jitter, or misordered snapshots.

**Q: Why didn't netgraph show huge spikes?**  
A: Micro-desync can occur from thread timing issues not visible in aggregate stats.

**Q: Can we "fix" RAGE networking?**  
A: You can't rewrite the engine, but you can mitigate through optimization and smoothing.

**Q: Does high FPS help with desync?**  
A: Not directly. Stable server tick rate matters more than client FPS.

---

## 📖 Glossary

| Term | Definition |
|------|------------|
| **Interpolation** | Smoothing between server snapshots |
| **Extrapolation** | Guessing position when snapshots missing |
| **Reconciliation** | Correcting client state to match server |
| **Tick** | Server frame/update cycle (~16.6ms at 60Hz) |
| **Snapshot** | Server state broadcast to clients |
| **Buffer** | Queue of recent snapshots (~100ms) |
| **Jitter** | Variance in network latency |
| **Packet Loss** | Network packets failing to arrive |
| **Desync** | Client and server disagreeing on state |
| **Snapping** | Sudden position correction |

---

## 🤝 Contributing

Contributions welcome!

- Share profiler traces, netgraph screenshots, tick logs
- Open issues with reproduction steps
- PRs with smoothing strategies or diagnostics
- Improve documentation clarity

---

## 📜 License

**MIT License** — See LICENSE for details

---

## 👥 Credits

| Contributor | Role |
|------------|------|
| **Leon** | Technical analysis, profiler/tick insights |
| **Chumbis20** | Case study, reproduction testing, documentation |
| **Person1** | Diagnostic tools, discussion insights |

**Special Thanks:** FiveM Development Team, QBCore Framework, The FiveM Community

---

## 📚 Further Reading

- [Valve's Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- [Gaffer on Games — Fix Your Timestep](https://gafferongames.com/post/fix_your_timestep/)
- [Glenn Fiedler — Networked Physics](https://gafferongames.com/post/introduction_to_networked_physics/)
- [FiveM Documentation — Networking](https://docs.fivem.net/docs/scripting-reference/networking/)

---

**Last Updated:** November 5, 2025  
**Version:** 2.0.0  
**Status:** Living Document

---

*For questions, contributions, or case studies, please open an issue or discussion.*
