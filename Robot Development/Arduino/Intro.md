# Arduino & Robotics at ECU ATMAE

## Welcome to the Team! 👋

This guide is designed to help you understand how the ECU ATMAE Robotics team uses Arduino microcontrollers to bring our competition robots to life. Whether you're just starting or joining mid-season, this will give you the foundation you need.

---

## What Is Arduino?

Arduino is a small, programmable microcontroller board that acts as the "brain" of our robot's hardware. Think of it as a tiny computer that:

- Reads sensor data (distance, color, buttons, etc.)
- Controls motors and servos (movement, rotation, gripping)

**Why do we use it?** It's perfect for real-time hardware control because it runs without an operating system—it just executes your code instantly upon turning on.

---

## System Architecture: How Everything Connects

```
┌─────────────────────┐
│   Xbox Controller    │
│  (Bluetooth/USB)     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Raspberry Pi       │
│  (Main Computer)     │
└──────────┬──────────┘
           │
       Serial UART
       (9600 baud)
           │
           ▼
┌─────────────────────┐
│  Arduino Mega/Giga    │
│  (Microcontroller)   │  ◄── Runs C++, controls hardware
└──────────┬──────────┘
           │
        GPIO Pins
           │
     ┌──────┴──────┐
     ▼             ▼
┌────────┐    ┌────────┐
│ Motors │    │Sensors │
└────────┘    └────────┘
```

**Key Takeaway:** The Pi makes high-level decisions, the Arduino handles low-level hardware control. They talk through serial communication.

---

## How Data Flows Between Pi and Arduino

### 1. Pi Sends Instructions to Arduino (Serial)

```python
# Example from ATMAE-2025 RaspberryPi/main.py
arduino = serial.Serial("/dev/ttyACM0", 9600, timeout=1)
instruction = "5:1.0\n"  # Format: ID:Value
arduino.write(instruction.encode('utf-8'))
```

**What's happening:**
- Pi opens a serial connection at 9600 baud (bits per second)
- Sends formatted commands like `"5:1.0\n"` (instruction ID 5, value 1.0)
  - This is a formatted instruction from ControllerCommon
- Arduino reads these on its end

### 2. Arduino Receives and Parses Data

```cpp
// Example from ATMAE-2024 Arduino/Shopping_Cart.ino
void loop() {
  if (Serial.available()) {
    char receivedChar = Serial.read();
    if (receivedChar == '\n') {  // End of instruction
      parseData(receivedData);
      receivedData = "";  // Clear for next instruction
    } else {
      receivedData += receivedChar;  // Build the message
    }
  }
}
```

**What's happening:**
- Arduino continuously checks for incoming data
- Reads character-by-character until it hits `\n` (newline)
- Parses the complete instruction and executes it

### 3. Arduino Sends Status Back to Pi

```cpp
// From ATMAE-2025 main loop
Serial.println("lowerHit");  // Arduino reports limit switch triggered
```

```python
# Pi listens for responses
if arduino.in_waiting:
    response = arduino.readline().rstrip()
    print(response)  # "lowerHit"
```

---

## Common Arduino Tasks in Our Robots

### Motor Control

```cpp
const int motorPin = 10;
Servo motor;

void setup() {
  motor.attach(motorPin);  // Attach servo to pin 10
}

void loop() {
  motor.write(1500);  // 1500 = neutral (no movement)
  motor.write(1750);  // Forward
  motor.write(1250);  // Backward
}
```

### Sensor Reading (Limit Switch)

```cpp
const int limitSwitchPin = 12;

void setup() {
  pinMode(limitSwitchPin, INPUT);
}

if (digitalRead(limitSwitchPin) == LOW) {  // Switch pressed
  Serial.println("limitSwitchHit");
}
```

### Sending Sensor Data to Pi

```cpp
Serial.print("distance:");
Serial.println(sensorValue);  // Pi reads this as "distance:450"
```

---

## Real Example: Robot Movement

**Scenario:** Driver pushes right trigger to accelerate.

1. **Controller Input** → Xbox detects trigger press (value: 0.8)
2. **Pi Receives** → `"10:0.8\n"` arrives via serial
3. **Arduino Parses** → ID = 10, so it's right trigger, value = 0.8
4. **Arduino Calculates** → Convert 0.8 to motor PWM (e.g., 1750 out of 1500-2000)
5. **Arduino Sends** → `motor.write(1750);`
6. **Hardware Moves** → Motor spins at 80% power forward

---

## Important Pins & Connections

### Common GPIO Pins Used

- **Digital Pins (0-53):** Motors, servos, limit switches, LEDs
- **Analog Pins (A0-A15):** Sensors, voltage readings

### Servo Signal

- Send PWM signal to servo pin
- Range: 1000-2000 microseconds (1500 = center)

---

## Troubleshooting Tips

| Problem | Solution |
|---|---|
| Arduino not receiving data | Check serial connection, verify baud rate (9600), ensure `\n` at end of message |
| Motor not responding | Verify pin number and pin connection, check PWM range (1000-2000), test servo with simple sweep code |
| Can't upload to Arduino | Select correct board & COM port in Arduino IDE, check USB cable |

---

## Next Steps

1. **Read the Arduino Code** → Start with Shopping Cart
   - Very simple bot, literally a box

---

## Key Takeaways

- **Arduino is the hardware controller** — it handles real-time motor/sensor control
- **Raspberry Pi is the decision maker** — it handles Cameras, Bluetooth, Servers, and high-level logic
- **They communicate via serial** — format: `instructionID:value\n`
- **Everything is documented** → Check repos for README and Arduino comments
- **Start small** → Test components individually before combining them

---

## Questions? Resources

- [Arduino Docs](https://www.arduino.cc/reference/)
- Team Discord