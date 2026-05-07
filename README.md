# سَكينة — Sakina
### Stress Monitoring using Federated Learning

Sakina is a privacy-preserving, real-time stress monitoring system built as a capstone project (CPCS499). It combines edge AI inference on an ESP32 microcontroller, a cross-platform Flutter mobile application, and a Federated Learning server powered by Flower (flwr) — all without sharing raw physiological data.

> **Group C02** — Tariq Areesh | Majd Al-farasani  
> King Abdulaziz University — Computer Science

---

## How It Works

1. The **ESP32** simulates BVP and temperature sensor data, runs a quantized TFLite MLP model locally, and broadcasts the stress prediction over **Bluetooth Low Energy (BLE)**
2. The **Flutter app** connects to the ESP32 via BLE, displays real-time health data, and collects training samples
3. The app fine-tunes the model locally on the user's own data (**on-device FL training**)
4. Updated weights are pushed to a central **Flower FL server** for privacy-preserving aggregation using the **FedAdam** strategy — no raw data ever leaves the device

---

## App Screenshots

### Connect Page
Scan for nearby BLE devices, select the ESP32, and establish a connection.

<img src="screenshots/connect.png" width="280"/>

---

### Home Page
Displays the connected device name, current stress status, health advice, skin temperature, and heart rate in real-time.

<img src="screenshots/home.png" width="280"/>

---

### Health Dashboard
Live scrolling charts for Blood Volume Pulse (BVP) and Body Temperature, updated as data streams in over BLE.

<img src="screenshots/dashboard.png" width="280"/>

---

### Stress History
A searchable and filterable log of all past stress readings with timestamps, score, HR, and temperature.

<img src="screenshots/history.png" width="280"/>

---

### FL Local Training — Data & Settings
Visualizes collected BLE training samples (Normal vs Stressed), window accumulation progress, and training hyperparameters (epochs, learning rate, batch size).

<img src="screenshots/fl_training.jpg" width="280"/>

---

### FL Local Training — Results & Server
Shows the last training result (accuracy, loss, FL round) and allows pushing local weights to the Flower FL server or pulling the latest global model.

> ⚠️ Server URL blurred for privacy.

<img src="screenshots/fl_server.png" width="280"/>

---

## System Architecture

| Component | Technology |
|-----------|-----------|
| Edge Device | ESP32 + TensorFlow Lite Micro (INT8) |
| Mobile App | Flutter (Dart) |
| BLE Communication | flutter_blue_plus |
| On-Device Training | Pure Dart MLP (5 → 64 → 32 → 1) |
| FL Server | Flower (flwr) + Flask REST API |
| FL Strategy | FedAdam |
| Dataset | WESAD (17 subjects) + 10,000-subject synthetic cGAN |
| Model Accuracy | ~96% on expanded WESAD dataset |

---

## Project Structure

```
lib/
├── main.dart               # App entry point
├── appShell.dart           # Bottom nav + BLE state management
├── homePage.dart           # Home screen
├── healthDashboardPage.dart# Live BVP & temp charts
├── stressHistoryPage.dart  # History log with search & filter
├── blu.dart                # BLE scan, connect, receive logic
├── fl_local_trainer.dart   # On-device FL training (Dart MLP)
└── fl_training_page.dart   # FL training UI

assets/
└── sakina_initial_weights.json  # Pre-trained Keras weights (float32)

Sakina-Server/
├── server.py               # Flower FL server + Flask REST bridge
├── client.py               # WESAD simulation client
└── run_clients.bat         # Launch all 15 WESAD clients at once
```

---

## How to Run

### Prerequisites

- Flutter SDK (3.x)
- Android device with Bluetooth (API 23+), developer mode enabled
- Python 3.10+ (for the FL server)
- An ESP32 flashed with the Sakina firmware

---

### 1. Flutter App

```bash
# Clone the repo
git clone https://github.com/tariq-areesh/Sakina-App.git
cd Sakina-App

# Install dependencies
flutter pub get

# Run on connected Android device
flutter run
```

Make sure `assets/sakina_initial_weights.json` is present and listed in `pubspec.yaml`:

```yaml
flutter:
  assets:
    - assets/sakina_initial_weights.json
```

---

### 2. FL Server

```bash
cd Sakina-Server

# Install dependencies
pip install flwr tensorflow flask numpy

# Start the server (FedAdam, port 8080 for Flower / port 5050 for REST)
python server.py
```

---

### 3. Simulation Clients (WESAD dataset)

Place the WESAD dataset under `Sakina-Server/WESAD/` with the structure:
```
WESAD/
├── S2/S2.pkl
├── S3/S3.pkl
...
└── S17/S17.pkl   (S12 excluded)
```

Then run all 15 clients at once:
```bash
run_clients.bat
```

---

### 4. Connect Flutter App to FL Server

1. Find your PC's local IP: run `ipconfig` on Windows, look for IPv4 under WiFi
2. In the app → **Train** tab → enter `http://<your-ip>:5050` as the server URL
3. Tap **Pull global model** to download weights, **Push local weights** to upload after training

---

## FL Strategy Comparison

| Strategy | Epoch 3 / 100 Rounds — Accuracy | Recall |
|----------|----------------------------------|--------|
| FedAvg | 0.883 | 0.625 |
| FedProx | 0.880 | 0.607 |
| FedAdagrad | 0.857 | 0.735 |
| FedYogi | 0.728 | 0.990 |
| **FedAdam** ✓ | **0.819** | **0.826** |

FedAdam was selected for providing the most balanced accuracy and recall — avoiding the false high-accuracy / low-recall trap seen in FedAvg and FedProx.

---

## Privacy Design

- Raw physiological data **never leaves the device**
- Only model weight updates (gradients) are shared with the server
- The FL server aggregates updates without seeing any user data
- On-device inference on the ESP32 means no cloud dependency for predictions