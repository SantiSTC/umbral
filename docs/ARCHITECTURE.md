# Architecture

Umbral has two parts: a **firmware** that runs on each ESP32 node and measures the radio channel, and a **web panel** that turns those measurements into detection, zones and history.

## 1. Firmware (`firmware/main/app_main.c`)

ESP-IDF 5.4, target `esp32`. Single source file.

### Links measured by each node

| Link | Traffic source | Rate |
|---|---|---|
| `router` | ICMP pings to the **other node** (fallback: to the gateway). Both the request relayed to the other node and its reply go out of the router's antenna, so the router only *forwards* — it does not rate-limit replies the way it does when it answers pings itself. | 100 Hz |
| `peer` | ESP-NOW broadcast beacons from the other node, sent at **MCS0 (OFDM)**. At 1 Mbps DSSS the ESP32 reports no CSI. | 50 Hz |

A watchdog switches the ping target if a link starves for 5 s. Nodes discover each other through the ESP-NOW beacons, which carry the sender's name and IP.

### Per-packet processing (`csi_cb`)

- Keeps only packets whose source MAC is the router (BSSID) or the known peer.
- Converts the 64 L-LTF subcarriers to amplitude and stores them in a 50-packet ring buffer per link (~0.5 s).
- Tracks the PHY format of received packets (11b/g/n, 20/40 MHz) and only accepts the dominant one, because CSI from different formats is not comparable.

### Metrics (`metric_task`, every 100 ms)

Each packet is normalized by its mean amplitude to cancel the automatic gain control (AGC) of the radio. Over the window:

| Field | Meaning |
|---|---|
| `m[0]` | Coefficient of variation of raw amplitude, averaged over subcarriers |
| `m[1]` | Robust variant: drops the 10 % most atypical packets and noisiest subcarriers |
| `m[2]` | Coefficient of variation of AGC-normalized amplitude |
| `m[3]` | Shape change between consecutive packets: `1 − correlation` (gain-invariant) |
| `m[4]` | Variation of total packet power |
| `p[64]` | Mean normalized amplitude per subcarrier (static "fingerprint") |
| `s[64]` | **Per-subcarrier motion signature**: std/mean of each subcarrier over the window |

Each metric is median-filtered over the last 5 computations.

### HTTP API

`GET /` → the panel (`web/index.html`, embedded at build time).
`GET /data` → JSON, CORS open:

```json
{
  "name": "Node A", "ip": "192.168.1.20",
  "peer": { "name": "Node B", "ip": "192.168.1.21", "age_ms": 13 },
  "links": [
    { "id": "router", "pps": 72, "rssi": -54,
      "m": [0.06, 0.03, 0.04, 0.05, 0.04],
      "fmt": [0, 0, 13826, 0], "lock": 2,
      "p": [/* 64 × 1000 */], "s": [/* 64 × 10000 */] },
    { "id": "peer", "...": "..." }
  ]
}
```

### Serial protocol

Used by `tools/configure-board.html` (Web Serial, 921600 baud). WiFi credentials and the node name are stored in NVS; the password never touches a file on the PC.

## 2. Web panel (`web/index.html`)

A single dependency-free HTML file. It runs from the board (`http://<ip>/`), from disk (`file://`, it asks for a board IP) or as a demo (`?demo`, or on GitHub Pages) with a built-in simulator.

### Pipeline (every 200 ms)

```
/data from both nodes
   │
   ├─► per-line motion ratio = max(m2/noise, m3/noise) × sensitivity
   │      line "active" if > 1 in ≥3 of the last 5 readings (≥4 for router lines)
   │
   ├─► state machine:  activity ─(2 s quiet)─► recent activity ─(N min)─► no activity
   │
   └─► zone (only while there is activity)
          features = log(1 + s/noise) for 4 measurements × 32 subcarrier pairs,
                     split into pattern (minus mean) + intensity
          classifier = small MLP (32 hidden, ReLU, softmax, Adam, trained in-browser)
                       or k-nearest neighbours (k = 9)
          fallback   = geometric tomography over the 3 lines (before any training)
          output smoothed over time; shows "between X and Y" when unsure
```

### Calibration and training

- **Background noise** (15 s, empty room) sets each measurement's noise floor, per subcarrier pair.
- **Zone training** (30 s per zone) stores motion signatures; up to 160 per zone, oldest dropped first.
- **"I'm here"** (5 s) adds labelled samples exactly where the model failed.
- **Quality report**: leave-one-out accuracy per zone and the zone it is most confused with, with a one-click *merge*.
- Everything is stored in `localStorage` of the browser that did the training.

### Simulator

`?demo` generates CSI-like data for a scripted scene (enter, walk, sit, move a chair, leave), including router antenna spikes, so the panel and the classifiers can be exercised without hardware.
