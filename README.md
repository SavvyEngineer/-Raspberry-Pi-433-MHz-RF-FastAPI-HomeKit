Raspberry Pi 433 MHz RF → FastAPI → HomeKit

Control Any 433 MHz Remote-Controlled Light With Siri

This project lets you take any cheap 433 MHz RF light (or any RF remote) and turn it into a smart light you can control with:
```
	•	Your phone
	•	Siri (“Hey Siri, turn on the lights”)
	•	HomeKit automations
	•	A simple web API
```

The Raspberry Pi records the RF signals from your original remote and later replays them exactly, so you don’t have to decode or understand any protocols.
If your remote uses 433 MHz ASK/OOK (which 99% of cheap remotes do), this works.

⸻

Why This Works

Most cheap 433 MHz remotes aren’t secure.
They simply send a burst of high/low pulses with very specific timing (microseconds).
If you capture those timings and replay them with good accuracy, the light behaves exactly like you pressed the original remote.

This project:
```
	1.	Captures the raw pulses from your remote using a 433 MHz receiver
	2.	Saves them to JSON files
	3.	Lets the Raspberry Pi simulate the remote using a transmitter
	4.	Wraps everything in a FastAPI server
	5.	Integrates it into HomeKit via Homebridge
	6.	Lets you replace the remote entirely
```

⸻

Hardware Required
```
	•	Raspberry Pi (Zero, 3, 4 — anything works)
	•	RXB6 433 MHz Receiver
	•	FS1000A 433 MHz Transmitter
	•	Jumper wires
	•	Your original RF remote
	•	Optional: a 17 cm copper wire as antenna (makes range 10× better)
```

⸻

Wiring Diagram (Important!)

433 MHz modules are sensitive. Correct wiring matters.

```

Receiver (RXB6)

RXB6      → Raspberry Pi
-------------------------
VCC (5V)  → 5V pin
GND       → GND
DATA      → GPIO 20
ANT       → 17 cm wire (optional but recommended)

Transmitter (FS1000A)

FS1000A   → Raspberry Pi
-------------------------
VCC       → 5V
GND       → GND
DATA      → GPIO 21
ANT       → 17 cm wire (recommended)

```

Both modules share the same ground — if you forget this, nothing works.

⸻

System Diagram
```
       [ 433 MHz Remote ]
                |
                | Capture pulses
                v
   [ Raspberry Pi + RXB6 Receiver ]
                |
                | Save JSON pulses
                v
        [ FastAPI Web Server ]
                |
                | HTTP /btn/power
                v
   [ FS1000A 433 MHz Transmitter ]
                |
                | Send RF command
                v
         [ Your Light / Device ]
```

⸻

Step 1 — Install pigpio
```

sudo apt update
sudo apt install pigpio python3-pigpio
sudo systemctl enable pigpiod
sudo systemctl start pigpiod
```


⸻

Step 2 — Capture Remote Buttons

This script records a full RF frame, stores it in clean_<name>.json, and keeps all pulses intact so the Pi can replay them accurately.

capture_button.py

```
import pigpio, time, json

RX_PIN = 20
GAP_US = 15000  # frame break

pi = pigpio.pi()
pi.set_mode(RX_PIN, pigpio.INPUT)
pi.set_pull_up_down(RX_PIN, pigpio.PUD_OFF)
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

Run it for each button:

python3 capture_button.py
```

⸻

Step 3 — Send RF Signals

The Pi sends the pulses exactly as recorded, with the correct inverted idle level.

send_raw.py

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
    lvl = 1 - lvl  # invert signal

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

print(f"Sending {filename}")
for _ in range(REPEATS):
    pi.wave_send_once(wid)
    while pi.wave_tx_busy():
        time.sleep(0.005)
    time.sleep(0.05)

pi.wave_delete(wid)
pi.write(TX_PIN, 0)
pi.stop()

```

Test it:

python3 send_raw.py clean_power.json


⸻

Step 4 — Run the FastAPI Web Server

server.py

```
from fastapi import FastAPI
import subprocess

app = FastAPI()

@app.post("/btn/{name}")
def btn(name: str):
    subprocess.call(["python3", "send_raw.py", f"clean_{name}.json"])
    return {"status": "ok", "button": name}
```

Start the API:
```
uvicorn server:app --host 0.0.0.0 --port 8000
```
Now you can test:
```
curl -X POST http://YOUR_PI_IP:8000/btn/power
```

⸻

Step 5 — Add HomeKit Support with Homebridge

Install the plugin:
```
sudo npm install -g homebridge-http-switch
```
Your config.json section:

Power Button
```
{
  "accessory": "HTTP-SWITCH",
  "name": "Main Lights Power",
  "switchType": "stateless",
  "onUrl": "http://10.0.0.91:8000/btn/power",
  "httpMethod": "POST"
}

Brightness Up

{
  "accessory": "HTTP-SWITCH",
  "name": "Main Lights Brightness Up",
  "switchType": "stateless",
  "onUrl": "http://10.0.0.91:8000/btn/bright_up",
  "httpMethod": "POST"
}

Color Change

{
  "accessory": "HTTP-SWITCH",
  "name": "Color Change",
  "switchType": "stateless",
  "onUrl": "http://10.0.0.91:8000/btn/color_change",
  "httpMethod": "POST"
}
```

Homebridge will show them as tappable buttons in HomeKit.

⸻

Use Siri Commands

Once added:
```
	•	“Hey Siri, turn on the lights”
	•	“Hey Siri, increase the brightness”
	•	“Hey Siri, change the light color”
```

⸻

This Works With ANY 433 MHz Remote

It doesn’t matter what brand your lights are.

If they use 433 MHz RF (most remotes do), this method lets the Raspberry Pi clone the remote and act as a fully-featured smart controller.
