# Findings: what worked, what didn't, and why

Umbral was developed and tested in a real apartment (≈ 3.5 × 6 m open living room and kitchen, router in an adjacent room). This document records the approaches that were evaluated and discarded, since they explain the current design.

Numbers marked **(sim)** come from the built-in simulator, not from the real room. They are useful to compare methods against each other, not as real-world accuracy.

---

### 1. Starting point: no ESP32-S3

RuView's firmware requires an ESP32-S3. The available boards were an original ESP32 (NodeMCU ESP-32S) and an ESP32-2432S028. The original ESP32 does expose CSI (`esp_wifi_set_csi`), so the whole pipeline was rebuilt on ESP-IDF for it.

### 2. Power is part of the design

The first board kept rebooting exactly at "RF calibration" when powered from a laptop USB port — a brownout from the radio's current peak. Fixes: disable the brownout detector, cap TX power, run the CPU at 160 MHz, and use decent chargers. A second board later "disappeared" for the same reason.

### 3. Motion index works — but only on the line

The first metric (coefficient of variation of subcarrier amplitude over 0.5 s) clearly showed someone entering and leaving. But it only reacted when the person **crossed the straight path** between transmitter and receiver: the sensitive region is a thin ellipse (first Fresnel zone).

### 4. The radio's AGC fakes motion

The ESP32 adjusts its gain automatically and does not report it, so amplitude jumps with nobody around. Normalizing each packet by its mean fixes it — but over-normalizing also removed the part of the signal a body causes, and detection got "deaf". The firmware now computes **five metrics in parallel** so they can be compared on real data instead of guessed.

### 5. "Compare against the empty room" failed — for a physical reason

Idea: record the CSI of the empty room and flag any deviation. It detects people even when still, so it was very attractive. In practice:

- The two router lines drifted far from their fingerprint within seconds (deviation ≈ 8.5 vs ≈ 0.6 on the node-to-node line), even with nobody home.
- Moving a chair **15 cm** left the deviation at ≈ 2.0 permanently, above the 0.39 threshold: "someone is in the room", forever.

At 2.4 GHz the wavelength is ≈ 12 cm, so any object displaced by a few centimetres rewrites the multipath pattern. A static fingerprint is too fragile for a home where things move. **Abandoned.**

### 6. "Assume they are still there" failed too

Next design: detect entry by motion, then assume the person stays until they are seen leaving. It broke as soon as someone left through an area no line covers: the panel kept saying *"someone is sitting still"*. Inferring presence from the absence of evidence is not measuring.

**Decision:** report only what is measured — *activity now*, *recent activity (N min ago, last zone)*, *no activity*. Reliable detection of a person who stays completely still requires a different sensor, such as mmWave radar.

### 7. A precise-looking dot is worse than an honest zone

Geometric tomography (intersecting the ellipses of the active lines) produced a dot with a trail. It looked great and was wrong: one real step drew an "S" on screen, and the dot drifted back to positions the geometry favours. With only 3 lines, the system has information for a handful of regions, not coordinates. The dot and trail were removed; the panel shows **zones with probabilities**, and says *"between X and Y"* when unsure. Geometry alone: **≈ 44 % exact zone (sim)**.

### 8. Node placement mattered more than any algorithm

Both nodes were on the same wall, so all three lines ran along the right side and everything was "detected on the right". Moving one node:

| Layout | Triangle area | Mean distance, left wall → nearest line |
|---|---|---|
| Original | 7.4 m² | 0.95 m |
| First proposal (move the other node) | 3.2 m² | 0.85 m |
| **Adopted** (kitchen node to the middle) | **8.3 m²** | **0.50 m** |

The first proposal looked reasonable and would have shrunk coverage by more than half — computing it before moving anything avoided a bad change.

### 9. Learning the room beats modelling it

Geometry assumes straight lines; a real room has walls, a TV and a fridge, so estimates were systematically shifted. Training zones from real movement corrected that:

| Classifier / features | Exact zone (sim) |
|---|---|
| Geometry only | ≈ 44 % |
| kNN, 2 motion metrics per line | ≈ 69 % |
| kNN, per-subcarrier signature | ≈ 77–88 % ¹ |
| **MLP, per-subcarrier signature** | **≈ 82–85 %** (4 training runs) |
| MLP with a 1 s time window | ≈ 69 % — worse, mixes moments crossing between zones |

¹ Depends on how training movement is simulated; the simulator was written so that this information exists, so treat it as an upper bound.

In the real room, training noticeably improved zone estimates. Real-world accuracy figures require a labelled dataset and are not reported here.

### 10. Practical notes

- A router may silently stop answering pings from one node; pinging the **other node** through the router keeps traffic flowing.
- An open window with wind registers as activity near the window. Record the background noise with the window as it usually is.
- A web panel served by the board means every UI change requires reflashing; running the same HTML from disk removed that loop.
