
SmartRF HomeKit Bridge

Control any cheap 433 MHz RF lights using a Raspberry Pi, FastAPI, pigpio, and Homebridge — fully Siri compatible, no cloud, no subscription, no custom hardware beyond one TX module.

This project replaces a physical RF remote (e.g., GY-RC01) and exposes every button (Power, White Set, Color Change, Brightness Up/Down, etc.) as tappable buttons or switches in the Apple Home app, usable with Siri, automation, and scenes.

⸻

⭐ Features

✔ Works with any 433 MHz non-rolling code light remote
✔ Fully compatible with Apple Home / Siri
✔ Web API using FastAPI
✔ Native RF transmission using pigpio waveforms
✔ Supports multiple buttons, including dimming and mode buttons
✔ Homebridge plugin configuration included
✔ 100% local, private, no cloud
✔ Uses a cheap FS1000A TX + XY-MK-5V RX

⸻

📦 Hardware Required

Item	Price	Notes
Raspberry Pi (any model with GPIO)	–	Pi 3A+, 3B, 4, Zero 2 all work
433 MHz RX (XY-MK-5V)	€2	For capturing signals
433 MHz TX (FS1000A)	€2	For transmitting signals
17 cm wire	Free	Used as the antenna for TX
Jumper wires	–	Female-to-female recommended


⸻

🖧 Wiring Diagram

        Raspberry Pi GPIO
        ┌───────────────────────────────┐
        │                               │
        │   5V ─────────────────┐       │
        │                       │       │
        │  GND ─────────────┐   │       │
        │                    │   │       │
        │ GPIO20 (RX DATA) ──┼───┘       │
        │                    │           │
        │ GPIO21 (TX DATA) ──┼───────────┐
        │                               │
        └───────────────────────────────┘

                 433 MHz RX (XY-MK-5V)
                 ┌─────────────────────┐
                 │   VCC ──────────────┘
                 │   DATA ───────── GPIO20
                 │   GND ─────────── GND
                 └─────────────────────┘

                 433 MHz TX (FS1000A)
                 ┌─────────────────────┐
                 │   VCC ───────── 5V
                 │   DATA ───────── GPIO21
                 │   GND ────────── GND
                 │  ANT  (17 cm wire)
                 └─────────────────────┘


⸻

📡 System Architecture

                ┌────────────────────────────┐
                │     Apple Home / Siri      │
                │  (Voice, Home App, Scenes) │
                └──────────────┬─────────────┘
                               │
                               ▼
                     ┌──────────────────┐
                     │   Homebridge     │
                     │  HTTP-SWITCH     │
                     └─────────┬────────┘
                               │ HTTP POST
                               ▼
                   ┌──────────────────────────┐
                   │   FastAPI RF Server      │
                   │  http://pi:8000/btn/...  │
                   └───────────┬─────────────┘
                               │
                    Calls send_raw.py
                               │
                               ▼
                   ┌──────────────────────────┐
                   │      pigpio Waveform     │
                   │   (Replays captured RF)  │
                   └───────────┬─────────────┘
                               │ GPIO 21
                               ▼
                     ┌────────────────────┐
                     │ 433 MHz TX Module  │
                     │     (FS1000A)      │
                     └─────────┬──────────┘
                               │ RF 433 MHz
                               ▼
                    ┌─────────────────────────┐
                    │  Ceiling Lights / Lamp  │
                    │  (Original RF Receiver) │
                    └─────────────────────────┘


⸻

🎯 Step 1 — Capture the RF Signals

Create & run:

capture_button.py

import pigpio, time, json, sys

RX_PIN = 20
GAP_US = 15000

name = input("Enter button name: ").strip()
filename = f"clean_{name}.json"

pi = pigpio.pi()
pi.set_mode(RX_PIN, pigpio.INPUT)
pi.set_pull_up_down(RX_PIN, pigpio.PUD_OFF)
pi.set_glitch_filter(RX_PIN, 50)

frames = []
pulses = []
last = None

print(f"Press '{name}' once...")
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
            frames.append(pulses)
            print("Captured:", len(pulses))
        pulses = []
    pulses.append([level, dt])

cb = pi.callback(RX_PIN, pigpio.EITHER_EDGE, cbf)

time.sleep(2)

cb.cancel()
pi.stop()

if frames:
    with open(filename, "w") as f:
        json.dump(frames[0], f, indent=2)
    print(f"[✓] Saved {filename}")
else:
    print("NO FRAME")

Run this once per button:

python capture_button.py

Examples:

Button name → saved JSON
	•	power → clean_power.json
	•	bright_up → clean_bright_up.json
	•	color_change → clean_color_change.json
	•	etc.

⸻

🚀 Step 2 — Sending RF (Replay Waveform)

send_raw.py

import pigpio, time, json, sys

TX_PIN = 21
REPEATS = 2

name = sys.argv[1]
file = f"clean_{name}.json"

pi = pigpio.pi()
if not pi.connected:
    sys.exit("pigpiod not running")

with open(file) as f:
    pulses = json.load(f)

pi.set_mode(TX_PIN, pigpio.OUTPUT)
pi.write(TX_PIN, 1)   # idle HIGH

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

print(f"Sending {file} x{REPEATS}")
for _ in range(REPEATS):
    pi.wave_send_once(wid)
    while pi.wave_tx_busy():
        time.sleep(0.005)
    time.sleep(0.05)

pi.wave_delete(wid)
pi.write(TX_PIN, 0)
pi.stop()

Send a command manually:

python send_raw.py power
python send_raw.py color_change
python send_raw.py bright_up


⸻

🌐 Step 3 — FastAPI Web Server

main.py

from fastapi import FastAPI
import subprocess, json, os

app = FastAPI()

@app.get("/buttons")
def list_buttons():
    files = [f.replace("clean_","").replace(".json","")
             for f in os.listdir()
             if f.startswith("clean_")]
    return {"buttons": files}

@app.post("/btn/{name}")
def press(name: str):
    subprocess.Popen(["python3", "send_raw.py", name])
    return {"status": "ok", "button": name}

Run API:

uvicorn main:app --host 0.0.0.0 --port 8000


⸻

🏠 Step 4 — Homebridge Configuration

Install plugin:

npm install -g homebridge-http-switch

Add to config.json:

{
  "bridge": {
    "name": "Homebridge RF",
    "username": "0E:8D:6E:1F:F4:8B",
    "port": 51594,
    "pin": "242-16-287"
  },

  "accessories": [
    {
      "accessory": "HTTP-SWITCH",
      "name": "Main Lights Power",
      "onUrl": "http://10.0.0.91:8000/btn/power",
      "switchType": "stateless",
      "httpMethod": "POST"
    },

    {
      "accessory": "HTTP-SWITCH",
      "name": "Main Brightness Up",
      "onUrl": "http://10.0.0.91:8000/btn/bright_up",
      "switchType": "stateless",
      "httpMethod": "POST"
    },

    {
      "accessory": "HTTP-SWITCH",
      "name": "Main Color Change",
      "onUrl": "http://10.0.0.91:8000/btn/color_change",
      "switchType": "stateless",
      "httpMethod": "POST"
    }
  ],

  "platforms": []
}

🟢 Stateless = tap button, auto resets
Perfect for brightness + color buttons.

⸻

🎉 Result

You now have:
	•	Fully working HomeKit controls
	•	Siri voice commands
	•	A web API
	•	RF replay that matches your physical remote
	•	No cloud, local-only automation
