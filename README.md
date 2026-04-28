# 🤖 LeKiwi Voice Controller

Control a 3-wheeled Kiwi-drive robot with your voice! This project combines a **Seeed Studio XIAO ESP32** (motor controller) with a **Raspberry Pi** (voice brain) to create a robot you can command hands-free using natural language.

---

## How It Works

```
You speak → Wake word detected → Audio recorded → Whisper STT → LLaMA LLM → Orpheus TTS speaks back → ESP32 moves the robot
```

1. You say the wake word (default: **"Hey Jarvis"**)
2. The Raspberry Pi records your command
3. **Groq Whisper** transcribes your speech to text
4. **LLaMA 3** decides what action the robot should take
5. **Groq Orpheus TTS** speaks a reply out loud
6. A serial command is sent to the ESP32, which drives the motors

---

## What You Need

### Hardware
| Part | Purpose |
|------|---------|
| Seeed Studio XIAO ESP32 (S3 / C3 / C6) | Motor controller |
| Raspberry Pi (3B+ / 4 / 5) | Voice processing brain |
| 3× Feetech STS3215 smart servos | Kiwi-drive wheels |
| ReSpeaker Flex (USB mic array) | Microphone input |
| LeKiwi chassis + wheels | 3D-printed frame |
| USB-A to USB-C cable | Pi ↔ ESP32 serial |

### Accounts & API Keys
- [**Groq**](https://groq.com/) account — free tier is enough to get started

---

## Part 1 — Prepare the XIAO ESP32 (Motor Controller)

The ESP32 runs the Arduino sketch that physically drives the three wheels using **Kiwi-drive kinematics**. It receives single-character serial commands (`w`, `s`, `a`, `d`, etc.) from the Raspberry Pi and translates them into coordinated motor speeds.

### Step 1 — Set Motor IDs

> ⚠️ **Important:** The sketch expects servo IDs **1, 2, 3**. Before assembling your robot, connect each servo individually and assign the correct ID using the Feetech configuration tool.
>
> 📄 [Feetech setup guide (PDF)](https://www.feetechrc.com/Data/feetechrc/upload/file/20201127/start%20%20tutorial201015.pdf)

### Step 2 — Assemble the LeKiwi

Follow the official Seeed Studio video tutorial to assemble the chassis, mount the wheels, and wire the servos:

📺 [LeKiwi Assembly Tutorial](https://wiki.seeedstudio.com/lerobot_lekiwi/#assembly)

> You only need to complete the **physical assembly** — skip any steps about cloning the LeRobot GitHub repository, as this project uses a different setup.

### Step 3 — Upload the Arduino Sketch

The sketch (`lekiwi_motor_control.ino`) does the following:

- Initialises all three servos in **position mode**, centres them, then switches to **wheel (continuous rotation) mode**
- Listens on the USB serial port for single-character commands
- Uses **Kiwi-drive kinematics** to calculate the correct speed for each wheel
- Supports both **nudge mode** (short burst, then auto-stop) and **continuous mode** (runs until you send a stop)

**Serial command reference:**

| Key | Action |
|-----|--------|
| `w` | Forward nudge |
| `s` | Backward nudge |
| `a` | Turn left nudge |
| `d` | Turn right nudge |
| `q` | Strafe left nudge |
| `e` | Strafe right nudge |
| `W/S/A/D/Q/E` | Same movements, continuous |
| `x` / Space | Emergency stop |
| `+` / `-` | Increase / decrease nudge duration |
| `*` / `/` | Increase / decrease nudge speed |

Upload the sketch via the Arduino IDE with the **SCServo** library installed. Open the Serial Monitor at **115200 baud** to verify all three servos respond with `OK`.

---

## Part 2 — Prepare the Raspberry Pi (Voice Brain)

### Step 1 — Get a Groq API Key

1. Register at [console.groq.com](https://console.groq.com/)
2. Create a new API key and copy it — you'll need it in Step 4

### Step 2 — Clone This Repository

```bash
git clone https://github.com/KasunThushara/Lekiwi-voice
cd Lekiwi-voice
```

### Step 3 — Create a Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

> Run `source venv/bin/activate` every time you open a new terminal before using this project.

### Step 4 — Install Dependencies

```bash
pip install -r requirements.txt
```

Then download the wake word model:

```bash
python3 download_model.py
```

This downloads the pre-trained **"Hey Jarvis"** model (and others) from openwakeword into `~/.openwakeword/`.

### Step 5 — Find Your Microphone Index

Plug in your **ReSpeaker Flex** via USB, then run:

```bash
python3 list_mics.py
```

You'll see a list like:

```
Available audio INPUT devices:

  [0] bcm2835 Headphones  (rate=44100Hz)
  [1] ReSpeaker 4 Mic Array  (rate=16000Hz)
  [2] USB PnP Sound Device  (rate=16000Hz)
```

Note the number in brackets next to your ReSpeaker — that's your `MIC_INDEX`.

### Step 6 — Find Your ESP32 Serial Port

Plug the XIAO ESP32 into the Pi via USB, then run:

```bash
python3 list_ports.py
```

Example output:

```
Available serial ports:

  /dev/ttyACM0         USB Serial Device
  /dev/ttyUSB0         CP2102 USB to UART Bridge
```

Note the device path (usually `/dev/ttyACM0` for XIAO).

### Step 7 — Configure the Project

Copy the example config and fill in your values:

```bash
cp config.env.example config.env
nano config.env
```

The key settings to update:

```env
# Your Groq API key (required)
GROQ_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxxxxxx

# The number from list_mics.py
MIC_INDEX=1

# The port from list_ports.py
SERIAL_PORT=/dev/ttyACM0
```

Everything else can be left at its default to get started.

---

## Run the Robot

Make sure your virtual environment is active, then:

```bash
python3 pipeline.py
```

You should see:

```
======================================================
  LeKiwi Voice Controller — Ready
  Wake word  : hey jarvis
  LLM model  : llama-3.1-8b-instant
  STT model  : whisper-large-v3-turbo
  TTS voice  : autumn
  Serial     : /dev/ttyACM0 @ 115200
======================================================
[WakeWord] Listening for 'hey jarvis' ...
```

Now say **"Hey Jarvis"** and give a movement command!

### Example Commands

> "Hey Jarvis, move forward"
> "Hey Jarvis, turn left"
> "Hey Jarvis, strafe right"
> "Hey Jarvis, keep going forward" *(continuous mode)*
> "Hey Jarvis, stop"
> "Hey Jarvis, what can you do?"

---

## Project File Overview

```
Lekiwi-voice/
├── pipeline.py          # Main entry point — orchestrates the full flow
├── wakeword.py          # Listens for "Hey Jarvis" in a background thread
├── audio_recorder.py    # Records N seconds of audio after wake word
├── stt.py               # Sends audio to Groq Whisper → returns text
├── llm.py               # Sends text to LLaMA → returns action + speech JSON
├── tts.py               # Sends reply text to Groq Orpheus → plays audio
├── robot_serial.py      # Sends serial commands to the ESP32
├── config.py            # Loads all settings from config.env
├── config.env.example   # Template — copy to config.env and fill in
├── list_mics.py         # Helper: find your MIC_INDEX
├── list_ports.py        # Helper: find your SERIAL_PORT
└── download_model.py    # Downloads the openwakeword model files
```

---

## Troubleshooting

**Servos not found at startup**
Check your wiring and make sure the servo bus is powered. Verify IDs are set to 1, 2, 3 using the Feetech tool before assembling.

**Wake word never triggers**
Run `list_mics.py` again and confirm `MIC_INDEX` in `config.env` matches your ReSpeaker. Speak clearly and within ~1 metre of the mic. Try lowering `WAKEWORD_THRESHOLD` to `0.3`.

**Robot not moving after a command**
Check that `SERIAL_PORT` is correct. Try `python3 list_ports.py` again — the port can change if you replug the USB. Also make sure the ESP32 sketch is uploaded and running (LED should be on).

**TTS / STT errors**
Double-check your `GROQ_API_KEY` in `config.env`. Make sure you have internet access from the Pi. The Groq free tier has rate limits — wait a few seconds between commands if you hit errors.

**Audio plays but sounds distorted**
Set your Pi audio output to the correct device using `raspi-config` → System Options → Audio.

---

## Configuration Reference

| Variable | Default | Description |
|---|---|---|
| `GROQ_API_KEY` | *(required)* | Your Groq API key |
| `WAKEWORD_MODEL` | `hey jarvis` | Wake word phrase |
| `MIC_INDEX` | `1` | PyAudio device index |
| `WAKEWORD_THRESHOLD` | `0.5` | Detection sensitivity (0.0–1.0) |
| `WAKEWORD_COOLDOWN` | `2` | Seconds between re-triggers |
| `RECORDING_SECONDS` | `3` | How long to record after wake word |
| `LLM_MODEL` | `llama-3.1-8b-instant` | Groq LLM model |
| `STT_MODEL` | `whisper-large-v3-turbo` | Groq Whisper model |
| `TTS_VOICE` | `autumn` | Voice for speech output |
| `SERIAL_PORT` | *(required)* | ESP32 port (e.g. `/dev/ttyACM0`) |
| `SERIAL_BAUD` | `115200` | Must match Arduino `Serial.begin()` |

---

## Credits

Built with:
- [Seeed Studio XIAO ESP32](https://wiki.seeedstudio.com/xiao_esp32s3_getting_started/) — compact ESP32 board
- [Feetech STS3215](https://www.feetechrc.com/) — smart serial bus servos
- [openwakeword](https://github.com/dscripka/openWakeWord) — local wake word detection
- [Groq](https://groq.com/) — ultra-fast Whisper STT, LLaMA LLM, and Orpheus TTS
- [LeKiwi](https://wiki.seeedstudio.com/lerobot_lekiwi/) — open-source Kiwi-drive robot platform
