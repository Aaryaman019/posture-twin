# Posture Twin

ESP32 + 3× MPU6050 (one I2C bus per sensor) streams spine angles over Wi-Fi. A seated 3D human mirrors them live in **Blender** and in a **browser dashboard**, which also runs online so anyone with the link can watch.

**Live dashboard:** https://aaryaman019.github.io/posture-twin/ — opens with demo motion; click **Connect** to watch the vest live.

```
blender/build_posture_model.py      builds the seated human, chair, 26-bone rig, vertebrae, IMU boxes
blender/posture_twin_live.py        live UDP receiver → drives the rig (N-panel "Posture" tab)
firmware/PostureTwin_ESP32/         Arduino sketch
index.html                          the dashboard (same file as dashboard/PostureTwin_Dashboard.html)
dashboard/PostureTwin_Dashboard.html  self-contained, works offline (three.js, MQTT client and model embedded)
dashboard/posture_model.glb         same model, for reuse
tools/sim_esp32.py                  fake ESP32 (stdlib Python) for testing without hardware
```

## 1. Blender (5.0+)
1. Open the **Scripting** tab, then **Open** `build_posture_model.py` and click **Run Script** (▶). This builds the `PostureTwin` collection. Re-running rebuilds it.
2. **Open** `posture_twin_live.py` and click **Run Script**.
3. In the 3D Viewport press **N**, open the **Posture** tab and click **Start**.
   - **Demo motion** animates the model with no ESP32.
   - **Calibrate** sends `cal` to the ESP32.
   - **Alt+Z** (X-ray) shows the vertebrae.
4. The first time you click Start, Windows Firewall asks about Blender. Allow it on **Private** networks (inbound UDP 4210).

## 2. Firmware (Arduino IDE)
- Board: **ESP32 Dev Module**. No extra libraries: the WebSocket server, MQTT client and third I2C bus are built into the sketch.
- Wiring: **each sensor has its own I2C bus**, so all three stay at address 0x68. The ESP32 has two hardware I2C controllers; the third bus is software I2C built into the sketch.

| Sensor | SDA | SCL | AD0 | Bus |
|---|---|---|---|---|
| Upper back (T2–T3) | GPIO21 | GPIO22 | GND | hardware I2C0 |
| Mid back (T8) | GPIO32 | GPIO33 | GND | hardware I2C1 |
| Lower back (L3) | GPIO18 | GPIO19 | GND | software I2C |

- Every MPU6050: VCC → 3V3, GND → GND, AD0 → GND; leave XDA, XCL and INT unconnected. GY-521 boards already have SDA/SCL pull-ups.
- Optional vibration motor: GPIO25 → 1 kΩ → NPN base, with a flyback diode across the motor.
- At boot, the Serial Monitor (115200) prints a scan of each sensor's bus and `WHO_AM_I` for each sensor. Set `SERIAL_DEBUG 2` to print raw acceleration in g instead of angles (at rest, the axis pointing up reads about +1).
- **Mounting** (default `AXMAP`): board flat on the back, with **+X toward the head**, **+Y to the person's left**, and **+Z pointing out of the back**. If you mount it differently, edit `AXMAP`. It takes a signed raw-axis index and must stay a proper rotation.
- **Wi-Fi:** put your network in `STA_SSID` / `STA_PASS` (2.4 GHz only). The ESP32 joins it and prints its IP on Serial (115200), repeating `IP: …` every 10 s until a dashboard connects. The laptop must be on the same network. If joining fails for 15 s, it starts its own network **PostureTwin** / `posture123` at `192.168.4.1`. `USE_AP_MODE 1` always uses its own network.
- **Boot:** keep the sensors still for about 1 s (gyro bias). On the very first boot, sit upright: the upright-zero calibration runs automatically and is saved to NVS.
- **Recalibrate:** use the dashboard/Blender button, or hold the BOOT button for 1 s.

Packet (50 Hz, UDP 4210 + WebSocket 81):
`{"t":ms,"u":[p,r],"m":[p,r],"l":[p,r],"st":0-5,"al":0/1,"ok":mask,"cal":0/1}`
p = flexion (+ forward), r = lateral bend (+ to the right), in degrees relative to the calibrated upright position.
st: 0 good, 1 slouching, 2 leaning forward, 3 leaning back, 4 leaning left, 5 leaning right.

## 3. Dashboard
What it shows:
- **3D twin:** a coloured seated human (skin, shirt, trousers, hair, shoes, sensor strap) on the same 26-bone rig as Blender; Side / Back / 3/4 / Front views and X-ray to see the vertebrae, coloured by each segment's bend.
- **Posture score (0–100):** 100 = upright, 50 = the worst metric is exactly at its limit, 0 = twice the limit.
- **Spine profile:** live side (flexion) and back (lateral) drawings of the spine from the three sensors, with the upright reference, the allowed-lean cone and the sensor positions.
- **Angles:** per sensor, or per segment (upper thoracic, thoraco-lumbar, lumbar).
- **Last 60 seconds:** curvature, forward lean and side lean with their limits, plus a posture-state ribbon.
- **Session:** time in good posture, tracked time, alerts, average score, current and best good streak.

The dashboard has two connection modes; the vest feeds both at once.

| Mode | Works from | How it connects |
|---|---|---|
| **Online** | anywhere (GitHub Pages link or the local file) | The ESP32 publishes to the public MQTT broker `broker.hivemq.com` (topic `posturetwin/<DEVICE_ID>/data`, 20 Hz); the dashboard subscribes over secure WebSocket. Needs internet on the ESP32's Wi-Fi, e.g. a phone hotspot. |
| **Same Wi-Fi** | the downloaded HTML file only | Direct WebSocket to the ESP32 IP (port 81), 50 Hz, no internet needed. Browsers block this from the https GitHub page. |

- **Online:** open the link, keep Device ID `vest-7f3k` (must equal `DEVICE_ID` in the firmware) and click **Connect**. The status shows *Live* when packets arrive, or *waiting for vest* when the broker is reachable but the vest is not publishing.
- **Calibrate** works in both modes (online it goes back through the broker).
- The broker is public: anyone who knows the Device ID can read the stream or send `cal`. Change `DEVICE_ID` (and type the same ID on the dashboard) to make it hard to guess.
- **Same Wi-Fi:** double-click `PostureTwin_Dashboard.html`, choose **Same Wi-Fi**, enter the IP the ESP32 printed (or `192.168.4.1` on its own network) and click **Connect**.

## Test without hardware
```
python tools/sim_esp32.py
```
- **Blender:** Start in the Posture tab. It receives packets on 127.0.0.1:4210.
- **Dashboard:** set the host to `localhost` and click Connect (WebSocket on port 81).

## Posture rules (same in firmware, Blender and dashboard)
| State | Condition (deg) |
|---|---|
| Leaning left/right | \|upper roll\| > 12 |
| Slouching | upper pitch − lower pitch > 15 |
| Leaning forward | upper pitch > 20 |
| Leaning back | upper pitch < −15 |

The haptic motor pulses after 3 s in a bad state. Thresholds are `TH_*` in all three code files.

## Limits
- Twist (yaw) isn't measured, because the MPU6050 has no magnetometer.
- The chair is treated as fixed, so the pelvis doesn't move on the model.
- Shoulder protraction on the model is estimated from spinal rounding (there's no shoulder sensor). Turn it off in the Blender panel.
- Neck/head movement is cosmetic (partial gaze compensation), not measured.
