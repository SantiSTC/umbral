# Setup

## 1. Get the firmware

- **Prebuilt:** download `umbral_esp32_full_0x0.bin` from the latest GitHub Actions run or release.
- **From source:** ESP-IDF v5.4, then:
  ```bash
  cd firmware
  idf.py set-target esp32
  idf.py build
  # optional: single image for the web flasher
  cd build && esptool.py --chip esp32 merge_bin -o umbral_esp32_full_0x0.bin @flash_args
  ```

## 2. Flash (from Chrome, no toolchain)

1. Open <https://espressif.github.io/esptool-js/> in Chrome or Edge, baudrate **115200**.
2. Hold **BOOT**, plug the board in (or press EN/RST), click **Connect** and pick the port (CP210x / CH340). Release BOOT once it says "Chip is ESP32".
3. Flash address **`0x0`**, file `umbral_esp32_full_0x0.bin`, **Program**.

To update a board that is already configured without erasing its WiFi settings, flash only the app image (`build/umbral.bin`) at **`0x10000`**.

If the transfer fails mid-way (`status 199`, `No serial data received`), retry without moving the cable; it is a USB link problem, not the image.

## 3. Configure WiFi

Open `tools/configure-board.html` in Chrome, press **Connect board over USB**, then set a node name, the 2.4 GHz network and its password and press **Save to board**. The board stores them and reports its IP.

## 4. Place the nodes

- Both nodes on the same 2.4 GHz network, 0.8–1.7 m high, antenna facing the room, not on metal.
- Make the triangle *router – node A – node B* cross the area you care about; avoid putting both nodes on the same wall.
- Use chargers that can supply the radio's current peaks.

## 5. Use the panel

Open `http://<node-ip>/` (or `web/index.html` from disk and enter an IP).

1. **Move router / boards** — drag the three points to their real positions.
2. **Record baseline noise** — 15 s with the room empty (windows as they usually are).
3. **Train zones** — select a zone, press Train and move inside it for 30 s. Repeat for every zone, ideally on different days.
4. **Train network** — trains the neural network on your samples (a few seconds, in the browser).
5. When it misses, select the real zone and press **I'm here**.
