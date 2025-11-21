
SmartRF HomeKit Bridge

Control any cheap 433 MHz RF lights using a Raspberry Pi, FastAPI, pigpio, and Homebridge — fully Siri-enabled and 100% local.

This project replaces a physical RF remote (like GY-RC01) and exposes all buttons (Power, Brightness, Color Change, etc.) as Apple Home Kit accessories.

⸻

⭐ Features

✔ Fully works with Apple Home, Siri, Automations, Scenes
✔ Web API powered by FastAPI
✔ Raw RF waveform replay using pigpio
✔ Supports any number of buttons
✔ Works with FS1000A TX and XY-MK-5V RX
✔ Fully offline — no cloud, no internet required
✔ Very cheap hardware (less than €5)

⸻

📦 Hardware Required

Component	Price	Notes
Raspberry Pi (any with GPIO)	–	Pi 3A+/3B/4/Zero 2
433 MHz RX (XY-MK-5V)	€2	For capturing signals
433 MHz TX (FS1000A)	€2	For transmitting signals
Jumper wires	€2	Female-to-female
17 cm wire	Free	Antenna for TX


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

📡 System Architecture Overview

          Apple Home / Siri
       ┌─────────────────────┐
       │ Scenes, Automations │
       └──────────┬──────────┘
                  │ HTTP POST
                  ▼
          Homebridge HTTP-SWITCH
       ┌─────────────────────┐
       │  homebridge plugin  │
       └──────────┬──────────┘
                  │
                  ▼
        FastAPI Server (Raspberry Pi)
       ┌────────────────────────────┐
       │ GET /buttons               │
       │ POST /btn/<button>         │
       └──────────┬─────────────────┘
                  │ Shell call
                  ▼
           send_raw.py (pigpio)
       ┌────────────────────────────┐
       │ 433 MHz waveform replay    │
       └──────────┬─────────────────┘
                  │ GPIO21
                  ▼
            FS1000A RF Transmitter
                  │
                  ▼
          Your 433 MHz Light Receiver


⸻

🎯 Step 1 — Capture RF Signals

Create capture_button.py:

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

Run it:

python3 capture_button.py

Do this for each button:

Button	File generated
power	clean_power.json
bright_up	clean_bright_up.json
bright_down	clean_bright_down.json
color_change	clean_color_change.json
white	clean_set_color_white.json


⸻

🚀 Step 2 — Send RF Signals

send_raw.py:

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
    lvl = 1 - lvl  # invert levels

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

To test:

python3 send_raw.py power
python3 send_raw.py bright_up
python3 send_raw.py color_change


⸻

🌐 Step 3 — FastAPI Web Server

Create FastAPI server main.py:

from fastapi import FastAPI
import subprocess, os

app = FastAPI()

@app.get("/buttons")
def list_buttons():
    return {
        "buttons": [
            f.replace("clean_", "").replace(".json", "")
            for f in os.listdir()
            if f.startswith("clean_")
        ]
    }

@app.post("/btn/{name}")
def press(name: str):
    subprocess.Popen(["python3", "send_raw.py", name])
    return {"status": "ok", "button": name}

Run:

uvicorn main:app --host 0.0.0.0 --port 8000


⸻

🛠 Step 4 — Systemd Service (Autostart)

/etc/systemd/system/rf-api.service

[Unit]
Description=RF Smart Light FastAPI server
After=network.target pigpiod.service
Wants=pigpiod.service

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/root/smartLight/web-server
Environment="PATH=/root/rfenv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/root/rfenv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target

Enable it:

sudo systemctl daemon-reload
sudo systemctl enable rf-api
sudo systemctl start rf-api


⸻

🏠 Step 5 — Homebridge Setup

Install plugin:

npm install -g homebridge-http-switch

Your config.json:

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
      "name": "Main Brightness Down",
      "onUrl": "http://10.0.0.91:8000/btn/bright_down",
      "switchType": "stateless",
      "httpMethod": "POST"
    },
    {
      "accessory": "HTTP-SWITCH",
      "name": "Color Change",
      "onUrl": "http://10.0.0.91:8000/btn/color_change",
      "switchType": "stateless",
      "httpMethod": "POST"
    }
  ],

  "platforms": []
}


⸻

🎉 Final Result

You now have:
	•	Fully working RF → Siri → HomeKit control
	•	All buttons exposed as tappable HomeKit tiles
	•	Smooth brightness, color, power control
	•	Entire system runs automatically on Pi boot
	•	Cheap hardware and completely local control
