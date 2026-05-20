# ShakeABLE ♿️👋

An open-source, low-cost assistive technology device that translates adaptive switch inputs into dynamic shaking and waving motions. Designed for wheelchair integration, **ShakeABLE** allows users to independently play noisemakers, wave pompoms, or flutter sensory scarves.

Built using an Arduino Nano, a high-torque MG995 servo, and 3D-printed parts, the device features a teacher-controlled speed slider and an intentional TPU "mechanical fuse" to improve safety.  The TPU lever arm will bend if obstructed or if too heavy of a load is placed on it.

---

## 📋 Bill of Materials (BOM)

To keep production costs accessible, **ShakeABLE** relies on widely available hobbyist components. The total hardware cost per unit is roughly **$10.00 USD**.


| Component | Description | Qty | Purpose | Approximate Cost |
| :--- | :--- | :--- | :--- | :--- |
| **Arduino Nano** | ATmega328P Microcontroller (Clone/Knockoff) | 1 | System Brains & Logic | $2.50 |
| **TowerPro MG995** | High-Torque Metal Gear Servo (12kg-cm at 6V) | 1 | Actuator / Driving Motor | $4.50 |
| **4S NiMH Battery Pack** | 4-Cell AA Rechargeable Pack (~4.8V–5.2V) | 1 | High-current Mobile Power Source | $1.00 (Cells extra) |
| **3.5mm Stereo Jack** | Panel-Mount Female Audio Connector | 1 | Adaptive Switch Interface Port | $1.25 |
| **10kΩ Linear Potentiometer** | Panel-Mount Miniature Dial | 1 | Teacher's Speed Control Slider | $0.40 |
| **100Ω Resistor** | 1/4 Watt Through-Hole Resistor | 1 | Electrical Safety Fuse for Switch Jack | <$0.01 |
| **SPST Toggle Switch** | Small Panel-Mount Rocker or Slide Switch | 1 | Main Hardware Power Switch | $0.20 |
| **Velcro Wire Straps** | Standard 13mm Hook-and-Loop Cable Keepers | 2-3 | Object Fastener at Tip of Arm | $0.10 |


## 🌟 Features
* **Adaptive Switch Input:** Global standard 3.5mm jack interface.
* **Direct Cause-and-Effect Control:** Shakes continuously while the switch is held down.
* **Safety Failsafe Timeout:** Automatically shuts down and centers after 5 seconds of continuous activation to protect the hardware from stuck switches.
* **Teacher Speed Slider:**  Adjustable oscillation frequency from 1.5 Hz to 2.5 Hz.
* **The TPU Whip Fuse:** A flat, 3D-printed flexible blade that serves as an intentional mechanical clutch—safely buckling during collisions or if an object is too heavy.
* **Wheelchair Mountable:** Dimensions match a standard smartphone, allowing the device to clamp into common open-source flexible phone mounts.

---

## 📊 Technical Specifications
* **Enclosure Size:** ~160mm x 75mm x 28mm (Smartphone footprint)
* **Sweep Window:** 60-degree total sweep ($\pm30^\circ$)
* **Default Center Angle:** $22.5^\circ$ offset from vertical (optimized for gravitational relief)
* **Lever Radius:** 4 inches (1-inch rigid PETG bracket + 3-inch 95A TPU blade)
* **Maximum Safe Payload:** ~250 grams at 1.5 Hz / ~150 grams at 2.5 Hz before TPU fuse buckling.

---

## 🔌 Circuit Diagram & Hardware

### Power Architecture
The high-torque MG995 servo draws heavy current spikes when reversing direction at 2.5 Hz. To prevent Arduino brownouts, power must be routed in **parallel** from a high-current battery pack. **Do not power the servo from the Arduino's 5V pin.**

===================================================================
POWER DISTRIBUTION (PARALLEL)
===================================================================

  [ Battery Pack (+) ] ───► [ Power Switch ] ───┬───► Servo RED Wire
                                               │
                                               └───► Arduino NANO "5V" Pin

  [ Battery Pack (-) ] ─────────────────────────┬───► Servo BROWN Wire
                                               │
                                               ├───► Arduino NANO "GND" Pin
                                               │
                                               ├───► 3.5mm Jack SLEEVE / RING
                                               │
                                               └───► Potentiometer LEFT Pin


===================================================================
CONTROL & SIGNAL LINES (WITH IN-LINE SAFETY RESISTOR)
===================================================================

  [ Arduino NANO ]
   ├─── Pin D9  ────────────────────────────────────► Servo YELLOW Wire
   │
   ├─── Pin D2  ───► [ 100Ω RESISTOR ] ─────────────► 3.5mm Jack TIP
   │                  (Short-Circuit Fuse)
   │
   ├─── Pin A0  ◄───────────────────────────────────► Potentiometer CENTER Pin
   │
   └─── Pin 5V  ────────────────────────────────────► Potentiometer RIGHT Pin


*Note: Tying the Ring and Sleeve together on the cheaper-to-buy stereo jack allows the use of standard mono adaptive switches. A **100Ω safety resistor** is added to the Tip line to eliminate the risk of a dead short if a cable is plugged in while powered on.*

---

## 🛠 3D Printing Guidelines

### 1. Main Enclosure Shell
* **Material:** PLA or PETG
* **Infill:** 15% - 20%
* **Notes:** Lay flat. Incorporate a standard headphone jack mount and side potentiometer mount. 

### 2. 90-Degree Conversion Bracket
* **Material:** PETG (highly recommended for mechanical stiffness)
* **Infill:** 40% - 50% Gyroid
* **Notes:** Converts the vertical rotation of the standard servo horn $90^\circ$ into a flat, horizontal clamping shelf. Uses an M4 or M5 brass heat-set insert to accept a hand-tightened thumb screw.

### 3. The Flexible Arm (The Whip/Fuse)
* **Material:** **95A TPU** (Do not use ultra-soft 85A/NinjaFlex)
* **Infill:** 100% Solid
* **Perimeters:** 4 to 5 walls
* **Geometry:** Print flat as a 1-inch wide, 5mm thick blade. Include 13mm x 2.5mm rectangular cutouts along the length to easily thread standard dual-sided Velcro cable straps for object attachment.

---

## 💻 Firmware (Arduino Sketch)

```cpp
#include <Servo.h>

// Pin Definitions
const int SWITCH_PIN = 2;   // Adaptive switch (INPUT_PULLUP)
const int SLIDER_PIN = A0;  // Speed dial potentiometer
const int SERVO_PIN = 9;    // MG995 Signal

Servo shakerServo;

// Movement Parameters
int centerAngle = 90;       
int sweepRange = 30;        // 60-degree total sweep
unsigned long lastMoveTime = 0;
bool movingForward = true;

// Safety Failsafe Variables
unsigned long switchPressedTime = 0;
bool isFailsafeActive = false;
const unsigned long MAX_RUN_TIME = 5000; // 5-second timeout

void setup() {
  pinMode(SWITCH_PIN, INPUT_PULLUP);
  shakerServo.attach(SERVO_PIN);
  shakerServo.write(centerAngle); 
}

void loop() {
  bool switchPressed = (digitalRead(SWITCH_PIN) == LOW);

  // Read Slider and map to milliseconds per half-sweep 
  // 1.5 Hz (approx 333ms per sweep) to 2.5 Hz (200ms per sweep)
  int sliderVal = analogRead(SLIDER_PIN);
  unsigned long moveInterval = map(sliderVal, 0, 1023, 333, 200);

  // Monitor continuous pressing for failsafe trigger
  if (switchPressed) {
    if (switchPressedTime == 0) {
      switchPressedTime = millis(); 
    }
    if (millis() - switchPressedTime > MAX_RUN_TIME) {
      isFailsafeActive = true;
    }
  } else {
    switchPressedTime = 0;
    isFailsafeActive = false;
  }

  // Non-blocking servo sweep execution
  if (switchPressed && !isFailsafeActive) {
    if (millis() - lastMoveTime >= moveInterval) {
      lastMoveTime = millis();
      
      if (movingForward) {
        shakerServo.write(centerAngle + sweepRange);
      } else {
        shakerServo.write(centerAngle - sweepRange);
      }
      movingForward = !movingForward; 
    }
  } else {
    shakerServo.write(centerAngle); // Fail-to-center architecture
  }
}
```

---

## 🤝 Contributing & Non-Profit Scale
This project is dedicated to affordable, open-source assistive hardware. If you are scaling this for giveaways or classroom bundles, please see the `PCB/` folder for a simple KiCad layout designed to minimize hand-soldering and eliminate manual wire routing. 
