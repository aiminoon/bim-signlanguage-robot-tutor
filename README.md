# BIM Sign Language Tutor Robot

An AI-powered tutoring system built on the Juno robot platform to help learners practice **Bahasa Isyarat Malaysia (BIM)** — Malaysian Sign Language. The robot tracks hand and body movements in real time, classifies signs using a CNN-LSTM model, and delivers synchronized visual and spoken feedback, with an LLM-powered mode for freeform, natural-language practice requests.

---

## Overview

Learning sign language from books or pre-recorded videos lacks the interactive feedback needed to correct hand positioning. This project addresses that gap with a robot that:

- Tracks a learner's upper body and hands via camera (**MediaPipe Holistic**, 258 landmarks/frame)
- Classifies BIM signs in real time using a **hybrid CNN-LSTM** model (one model per sentence, for higher accuracy)
- Gives **dual-channel feedback**: on-screen visual cues + spoken audio (TTS)
- Supports **hands-free voice navigation** ("skip", "back") via speech recognition
- Generates **personalized practice sequences** from natural language (e.g. *"I want to go home and pray"*) using **Gemini 2.5 Flash**

Built as a modular **ROS (Robot Operating System)** application with six independent nodes.

---

## Architecture

```
┌──────────┐    /camera/rgb    ┌─────────────────────┐
│  camera  │ ────────────────▶ │  handsign_detector   │
└──────────┘                   │  (MediaPipe+CNN-LSTM)│
                                └──────────┬───────────┘
                                           │ /handsign/detection
                                           │ /handsign/viz
                                           ▼
┌──────────┐  /handsign/voice_command  ┌────────────┐  /handsign/speak  ┌─────┐
│   stt    │ ─────────────────────────▶│    gui     │──────────────────▶│ tts │
└──────────┘                           │ (coordinator)│                  └─────┘
                                        └──────┬──────┘
                              /handsign/switch_model
                                        │      ▲
                                        │      │ /handsign/llm_sequence
                          /handsign/llm_text_input
                                        ▼      │
                                    ┌───────────────┐
                                    │      llm      │──▶ Gemini 2.5 Flash API
                                    └───────────────┘
```

### ROS Nodes

| Node | Responsibility |
|---|---|
| `camera_node` | Captures RGB frames from the robot's camera (640×480 @ 30 FPS), publishes to `/camera/rgb` |
| `handsign_detector_node` | Extracts 258 MediaPipe landmarks, runs CNN-LSTM inference over a 30-frame sliding window, publishes predictions and annotated video |
| `stt_node` | Captures microphone audio, transcribes voice commands via Google Speech-to-Text |
| `tts_node` | Converts text to speech via `pyttsx3` for instructions and feedback |
| `llm_node` | Sends freeform text requests to Gemini 2.5 Flash, returns a sequence of sign models to practice |
| `gui_node` / `gui_app` | Central coordinator — GUI, practice-state management, module/LLM practice modes |

### Key ROS Topics

| Topic | Publisher → Subscriber | Purpose |
|---|---|---|
| `/camera/rgb` | camera → detector | Raw video frames |
| `/handsign/detection` | detector → gui | Detected label, confidence, status |
| `/handsign/viz` | detector → gui | Annotated frame with skeleton overlay |
| `/handsign/switch_model` | gui → detector | Load a specific sign model |
| `/handsign/voice_command` | stt → gui | Transcribed voice command |
| `/handsign/speak` | gui → tts | Text for the robot to speak |
| `/handsign/llm_text_input` | gui → llm | Natural language practice request |
| `/handsign/llm_sequence` | llm → gui | Generated sequence of sign models |

---

## Tech Stack

- **Robotics:** ROS Noetic, Ubuntu 20.04
- **Computer Vision:** MediaPipe Holistic, OpenCV
- **Deep Learning:** TensorFlow/Keras (CNN-LSTM)
- **Speech:** Google Speech-to-Text (STT), pyttsx3 (TTS)
- **LLM:** Gemini 2.5 Flash
- **GUI:** Python Tkinter
- **Language:** Python 3.8.10

---

## Repository Structure

```
bim-signlanguage-robot-tutor/
├── scripts/
│   ├── camera_node.py
│   ├── handsign_detector_node.py
│   ├── stt_node.py
│   ├── tts_node.py
│   ├── llm_node.py
│   ├── gui_node.py
│   └── gui_app.py
├── launch/
│   └── handsign.launch
├── model/                  # sample trained models (see note below)
│   ├── saya_demam/
│   │   ├── model.keras
│   │   ├── scaler.pkl
│   │   └── label_encoder.pkl
│   └── mari_makan/
│   │   ├── model.keras
│   │   ├── scaler.pkl
│   │   └── label_encoder.pkl
│   └── apa_khabar/
│       ├── model.keras
│       ├── scaler.pkl
│       └── label_encoder.pkl
├── package.xml
├── CMakeLists.txt
├── .env.example
└── docs/
    └── architecture.png
```

> **Note on models:** Each subfolder in `model/` corresponds to one sentence-level sign model ("1 sentence = 1 model" design, to reduce confusion between visually similar signs). Only a couple of sample models are included here to keep the repo lightweight — the full set (~25 signs) is available on request.

---

## Setup

### Prerequisites
- Ubuntu 20.04.6 LTS
- ROS Noetic
- Python 3.8.10
- A working camera and microphone

### Installation

```bash
# 1. Create a catkin workspace
mkdir -p ~/catkin_ws/src && cd ~/catkin_ws
catkin_make

# 2. Clone this repo into src
cd src
git clone https://github.com/aiminoon/bim-signlanguage-robot-tutor.git handsign_detector

# 3. Configure environment variables
cd handsign_detector
cp .env.example .env
# then edit .env and add your Gemini API key

# 4. Install Python dependencies
pip install mediapipe opencv-python tensorflow rospkg SpeechRecognition pyttsx3 python-dotenv requests

# 5. Build the workspace
cd ~/catkin_ws
catkin_make
source devel/setup.bash
```

### Running

Make sure `roscore` is running in a separate terminal, then:

```bash
roslaunch handsign_detector handsign.launch
```

---

## Usage

1. The robot greets the user and displays the main menu: **Module Practice** or **LLM Practice**
2. **Module Practice** — select a category (e.g. Medical Emergency, Greetings) and a sentence model, then perform each sign as prompted; the robot gives real-time correct/incorrect feedback
3. **LLM Practice** — type a natural-language request (e.g. *"fever and headache"*); Gemini 2.5 Flash generates a personalized multi-sign practice sequence
4. Use voice commands ("skip", "back") to navigate hands-free during practice

---

## Limitations

- Supports a limited, trained vocabulary (~20–25 signs across 3 categories)
- Requires moderate lighting and a 0.8–1.2 m fixed distance from the camera
- LLM mode requires internet connectivity (falls back to keyword matching if unavailable)
- CPU-only inference (~20–30 FPS); very fast gestures may not be captured accurately

---

## Team

Built as a group project for WID3010 Autonomous Robot, Faculty of Computer Science and Information Technology, Universiti Malaya.

- Muhammad Imran bin Shuhanizal
- Nur Laila Nahwah binti Mohd Rostam
- Wan Qistina Damia binti Wan Mohd Azmi
- Dennis Aimin Oon bin Jeffrey Oon (Me)
- Muhammad Imran bin Ilias

## Acknowledgements

- [ROS Noetic](https://www.ros.org)
- [MediaPipe Holistic](https://ai.google.dev/edge/mediapipe)
- [OpenCV](https://opencv.org)
