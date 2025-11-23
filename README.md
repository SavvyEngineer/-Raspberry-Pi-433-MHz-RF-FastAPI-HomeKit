
📘 Raspberry Pi 433 MHz RF → FastAPI → HomeKit (Siri Control)

Control cheap 433 MHz RF lights using a Raspberry Pi, FastAPI, and Apple HomeKit via Homebridge.

This project lets you:
	•	Capture your RF remote signals
	•	Replay them from the Pi
	•	Expose them as a local HTTP API
	•	Control everything using Siri (“Hey Siri, turn on the lights”)
	•	Add brightness buttons, color-change buttons, or power buttons

⸻

🖼 System Diagram

        ┌──────────────────┐
        │ 433 MHz Remote   │
        └───────┬──────────┘
                │ (sniff)
                ▼
    ┌──────────────────────────┐
    │ Raspberry Pi + RX Module │
    └───────┬──────────────────┘
            │ Capture JSON pulse files
            ▼
    ┌──────────────────────────┐
    │   FastAPI Web Server     │
    └───────┬──────────────────┘
            │ HTTP POST /btn/power
            ▼
    ┌──────────────────────────┐
    │   TX Module (433 MHz)    │
    └───────┬──────────────────┘
            │ Sends RF command
            ▼
    ┌──────────────────────────┐
    │    Light / Device        │
    └──────────────────────────┘


⸻

📡 Hardware Needed
	•	Raspberry Pi (Zero, 3, 4, anything works)
	•	433 MHz RXB6 receiver
	•	433 MHz FS1000A transmitter
	•	Jumper wires
	•	Your original RF remote
	•	Optional: USB power supply with good stability

⸻

🛠 Step 1 — Install pigpio

```
sudo apt update
sudo apt install pigpio python3-pigpio
sudo systemctl enable pigpiod
sudo systemctl start pigpiod
```


⸻

🧲 Step 2 — Capture Remote Buttons

This script captures a full RF frame (no splitting) and saves it as clean_<button>.json.

📌 capture_button.py
```
import pigpio, time, json

RX_PIN = 20
GAP_US = 15000

pi = pigpio.pi()
pi.set_mode(RX_PIN, pigpio.INPUT)
pi.set_glitch_filter(RX_PIN, 50)

frames = []
pulses = []
last = None

btn = input("Enter button name (e.g. power, bright_up): ").strip()
print(f"Press the '{btn}' button once...")
time.sleep(0.3)

def cbf(gpio, level, tick):
    global last, pulses
    if last is None:
        last = tick
        return
    dt = pigpio.tickDiff(last, tick)
    last = tick

    if dt > GAP_US:
        if len(pulses) > 20:
            print("Captured:", len(pulses))
            frames.append(pulses)
        pulses = []
    pulses.append([level, dt])

cb = pi.callback(RX_PIN, pigpio.EITHER_EDGE, cbf)

time.sleep(2)
cb.cancel()
pi.stop()

if frames:
    filename = f"clean_{btn}.json"
    with open(filename, "w") as f:
        json.dump(frames[0], f, indent=2)
    print(f"Saved {filename}")
else:
    print("NO FRAME")

```


⸻

🚀 Step 3 — Send RF Signals

This script replays the captured JSON pulses exactly as recorded.

📌 send_raw.py

```
import pigpio, time, json, sys

TX_PIN = 21
REPEATS = 2

filename = sys.argv[1]
pulses = json.load(open(filename))

pi = pigpio.pi()
pi.set_mode(TX_PIN, pigpio.OUTPUT)
pi.write(TX_PIN, 1)  # idle HIGH

wf = []
cur = 1

for lvl, dt in pulses:
    dt = int(dt)
    lvl = 1 - lvl  # invert

    if lvl != cur:
        if lvl == 1:
            wf.append(pigpio.pulse(1<<TX_PIN, 0, 0))
        else:
            wf.append(pigpio.pulse(0, 1<<TX_PIN, 0))
        cur = lvl

    wf.append(pigpio.pulse(0, 0, dt))

if cur == 1:
    wf.append(pigpio.pulse(0, 1<<TX_PIN, 0))

pi.wave_add_generic(wf)
wid = pi.wave_create()

print(f"Sending {filename}…")
for _ in range(REPEATS):
    pi.wave_send_once(wid)
    while pi.wave_tx_busy():
        time.sleep(0.005)
    time.sleep(0.05)

pi.wave_delete(wid)
pi.write(TX_PIN, 0)
pi.stop()
```

⸻

🌐 Step 4 — FastAPI HTTP Server

📌 server.py

```
from fastapi import FastAPI
import subprocess

app = FastAPI()

@app.post("/btn/{name}")
def press_button(name: str):
    file = f"clean_{name}.json"
    subprocess.call(["python3", "send_raw.py", file])
    return {"status": "ok", "button": name}

Run it:

uvicorn server:app --host 0.0.0.0 --port 8000
```

⸻

🍎 Step 5 — HomeKit via Homebridge

Install plugin:
```
sudo npm install -g homebridge-http-switch
```

⸻

🏡 Example Homebridge config

✔ Power button (stateless)

```

{
  "accessory": "HTTP-SWITCH",
  "name": "Main Lights Power",
  "switchType": "stateless",
  "onUrl": "http://10.0.0.91:8000/btn/power",
  "httpMethod": "POST"
}

✔ Brightness Up (as a button)

{
  "accessory": "HTTP-SWITCH",
  "name": "Main Lights Brightness Up",
  "switchType": "stateless",
  "onUrl": "http://10.0.0.91:8000/btn/bright_up",
  "httpMethod": "POST"
}

✔ Color Change Button

{
  "accessory": "HTTP-SWITCH",
  "name": "Color Change (Main)",
  "switchType": "stateless",
  "onUrl": "http://10.0.0.91:8000/btn/color_change",
  "httpMethod": "POST"
}

```


⸻

🗣 Use Siri
	•	“Hey Siri, turn on the main lights”
	•	“Hey Siri, increase brightness in the living room”
	•	“Hey Siri, change the light color”

⸻

🎉 That’s it!

You’re now controlling super-cheap 433 MHz RF lights through:
	•	Raspberry Pi
	•	FastAPI
	•	Homebridge
	•	HomeKit
	•	Siri

