# Raspberry Pi & Robotics at ECU ATMAE

## Welcome to the Brain of the Robot! 🧠

If the Arduino is the robot's hands and muscles, the Raspberry Pi is its brain. This guide explains how we use the Pi to make intelligent decisions, process camera feeds, handle controllers, and coordinate everything together.

---

## What Is the Raspberry Pi?

The Raspberry Pi is a small, full-featured computer running Linux. Unlike the Arduino (which runs one program sequentially), the Pi can:

- Run multiple programs simultaneously (multithreading)
- Process complex images from cameras (computer vision)
- Communicate with servers over WiFi/Ethernet
- Make AI decisions using machine learning models
- Handle high-level logic and game states

It's basically a mini-PC.

**Why use it?**
- We need something powerful enough for stuff like video processing and Bluetooth, but small enough to fit on a robot.
- While attachments for Arduinos exist to allow for extra functionality like camera recording or Bluetooth connectivity, an Arduino has only so many pins and so much compute. Things get slow fast.

---

## Our System Architecture (Detailed)

```
┌──────────────────────────────────────────────┐
│                RASPBERRY PI                    │
│      Example Architecture Reflects 2025 Bot     │
├──────────────────────────────────────────────┤
│                                                │
│  ┌─────────────────────────────────────┐     │
│  │ 1. Controller Input (pygame)         │     │
│  │    Xbox Controller → USB/Bluetooth   │     │
│  └─────────────────────────────────────┘     │
│                                                │
│  ┌─────────────────────────────────────┐     │
│  │ 2. Camera Processing                 │     │
│  │    - PiCamera2 (internal)            │     │
│  │    - USB Camera (external)           │     │
│  └─────────────────────────────────────┘     │
│                                                │
│  ┌─────────────────────────────────────┐     │
│  │ 3. Serial Communication              │     │
│  │    Pi ←→ Arduino (9600 baud)         │     │
│  └─────────────────────────────────────┘     │
│                                                │
│  ┌─────────────────────────────────────┐     │
│  │ 4. Network (WiFi/Ethernet)           │     │
│  │    - Stream camera to outside server │     │
│  │    - Talk to ML Server               │     │
│  │    - Get AI instructions             │     │
│  └─────────────────────────────────────┘     │
│                                                │
└──────────────────────────────────────────────┘
                    ↓
               Serial Port
                  (USB)
                    ↓
┌──────────────────────────────────────────────┐
│         ARDUINO (Low-Level Control)            │
│  Receives instructions & controls motors       │
└──────────────────────────────────────────────┘
```

---

## The Main Roles of the Raspberry Pi Across Our Repos

### 🎮 Controller Input (The Driver)

The Pi reads the Xbox controller and converts player input into commands.

```python
# From ATMAE-2025 RaspberryPi/client/robotmodes.py
from ControllerCommon.XboxController import XboxController
import pygame

# Initialize the controller
pygame.init()
controller = XboxController(deadZone=0.2)  # Ignore tiny joystick movements

# Get controller input
instructions = controller.getControllerInput()
# Returns: ["5:0.8\n", "10:0.5\n", "11:1\n"]
# Button A = 11, Left Trigger = 10, Left Stick = 5

for instruction in instructions:
    inputID = int(controller.getInputID(instruction))       # "5"
    inputValue = float(controller.getInputValue(instruction))  # 0.8
    # Send to Arduino
    arduino.write(instruction.encode("utf-8"))
```

**Key Concept:** The controller library maps physical inputs to standardized IDs:
- More information in the ControllerCommon page

**Three Robot Modes:**

| Mode | What It Does |
|---|---|
| Teleop | Human controls robot with Xbox controller |
| Autonomous | Robot uses an algorithm to fulfill a task |
| Neutral | Safe idle state, no movement |

---

### 📷 Camera Processing (The Vision System)

The Pi can have multiple cameras and processes video in real-time.

**Example: Internal Sorting (Color Detection)**

```python
# From ATMAE-2025 RaspberryPi/client/robotmodes.py
from picamera2 import Picamera2
import colordetect
import cv2

def internal_sort_mode(arduino):
    """Sort balls by color and route them to correct bins."""
    # Initialize internal Pi camera
    capture = Picamera2(0)
    capture.start()

    # Set camera exposure for consistent lighting
    capture.set_controls({
        "AeEnable": False,       # Disable auto-exposure
        "AwbEnable": False,      # Disable auto-white balance
        "ExposureTime": 10000,
        "AnalogueGain": 1.0
    })

    ball_count = 0
    color_instruction = {
        "blue": "toBlue\n",
        "red": "toRed\n",
        "green": "toGreen\n",
        "yellow": "toYellow\n"
    }

    while ball_count < 12:  # Sort 12 balls
        # Capture frame
        img = capture.capture_array()
        img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)

        # Detect ball color
        color = colordetect.detect_color(img)

        if color != "black":  # "black" = no ball detected
            # Send servo command to route ball to correct bin
            instruction = color_instruction[color]
            arduino.write(instruction.encode("utf-8"))
            ball_count += 1
            time.sleep(2.5)  # Wait for servo to move
        else:
            # No ball detected, try again
            time.sleep(0.1)

    capture.stop()
```

**What's happening:**
1. Capture frame from camera
2. Analyze pixel colors
3. Identify ball color
4. Send servo command to Arduino
5. Wait for ball to drop
6. Repeat for next ball

---

### 🌐 Networking (The Strategic Brain)

In autonomous mode for the 2025 Competition, the Pi sends camera frames to a remote server running YOLO (AI object detection), gets movement instructions back, and executes them.

```python
# From ATMAE-2025 RaspberryPi/client/robotmodes.py
import socket
import struct

def send_video(arduino):
    """Stream video to AI server and receive instructions."""
    # Setup camera
    cap = cv2.VideoCapture(0)

    # Connect to external ML server
    SERVER_IP = '192.168.0.227'
    PORT = 9999

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((SERVER_IP, PORT))

        while True:
            # 1. CAPTURE: Read frame from camera
            ret, frame = cap.read()

            # 2. ENCODE: Compress to JPEG
            _, buffer = cv2.imencode('.jpg', frame)
            frame_bytes = buffer.tobytes()

            # 3. SEND: Pack with size header and send over network
            msg = struct.pack('>I', len(frame_bytes)) + frame_bytes
            s.sendall(msg)

            # 4. RECEIVE: Get AI instructions from server
            # Server sends back: "5:0.8\n|7:0.3\n|..."
            instructions = s.recv(1050).decode().split('|')

            # 5. FORWARD: Send instructions to Arduino
            for instruction in instructions:
                arduino.write(instruction.encode('utf-8'))

                # Special handling: when robot reaches object, start sorting
                if instruction == "9:-1\n":  # Trigger value = -1
                    arduino.write("screwDown\n".encode("utf-8"))

                    # Wait for limit switch confirmation
                    while arduino.readline().rstrip() != "lowerHit":
                        pass

                    # Start sorting logic
                    internal_sort_mode(arduino)
```

**The Flow:**

```
Pi Camera Frame
       ↓
    Network
       ↓
ML Server (YOLO)  ← Runs on powerful computer
       ↓
"Move forward, turn left, approach object"
       ↓
Pi receives instructions
       ↓
Arduino executes motor commands
```

---

## Key Python Libraries We Use

| Library | Purpose |
|---|---|
| `pygame` | Read Xbox controller input |
| `serial` | Communicate with Arduino over USB |
| `picamera2` | Access Pi camera |
| `cv2` (OpenCV) | Process images (crop, resize, color detection) |
| `socket` | Network communication with ML server |
| `threading` | Run multiple tasks simultaneously |
| `logging` | Debug logs with timestamps |

---

## Threading: Running Multiple Things at Once

Unlike an Arduino, a Pi can run multiple programs at once as well as multiple threads at once, just like any other computer.

The Pi in 2025 needed to:
- Listen for controller input
- Stream video to server
- Sort balls with camera
- All at the same time!

Python's `threading` module handled this:

```python
import threading

# Main task: Send video to server
video_thread = threading.Thread(target=send_video, args=[arduino])
video_thread.start()

# Secondary task: Listen for controller "stop" button
while not stop_event.is_set():
    instructions = controller.getControllerInput()
    if neutral_button_pressed:
        stop_event.set()  # Signal video thread to stop
    time.sleep(0.2)

# Wait for video thread to finish
video_thread.join()
```

**Key Concept:** `threading.Event()` acts like a traffic light between threads.

---

## Questions? Resources

- [PySerial Docs](https://pyserial.readthedocs.io/)
- Common Libraries used in ATMAE: Common ATMAE Libraries - Confluence
- Team Discord