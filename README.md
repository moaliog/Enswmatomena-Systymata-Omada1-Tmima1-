# Autonomous Line Follower Robot

## 📝 Description
This project was developed as part of the **Embedded Systems** course (Group 1 - Section 1). The objective is to design and build an autonomous robotic vehicle capable of tracking a black line on a white background using infrared (IR) sensors and a microcontroller for real-time navigation and motor control.

---

## 🛠️ Hardware Components
The following hardware was used to build the Line Follower:
* **Microcontroller:** RP2040 (Raspberry Pi)
* **Sensors:** Infrared (IR) Sensor Array for line tracking.
* **Shield/Board:** BrashBoard / Breadboard
* **Motors:** [N20 500RPM, 2x DC Geared Motors]
* **Motor Driver:** [integrated into the board]
* **Power Supply:** [3.7V Lipo_Battery]

---

## 💻 Software & Tools
The project's firmware is written in **C++** using the standard Arduino framework.
* **Development Environment (IDE):** Arduino IDE / VS Code with PlatformIO
* **Core Libraries:** `Arduino.h` (Standard Arduino Core API for RP2040)
* **Libraries Used:** None (Custom high-performance PID control and low-pass filter implemented from scratch)

---

## 🔌 Pinout Configuration

The following table details how the components (motors, sensors, and buttons) are connected to the Raspberry Pi Pico (RP2040) microcontroller based on the firmware definitions:

| Component | RP2040 Pin | Function | Description |
| :--- | :--- | :--- | :--- |
| **Left Motor PWM** | GP8 | Output | PWM speed control for Left Motor (0-4095) |
| **Left Motor DIR** | GP9 | Output | Direction control for Left Motor (LOW=Forward, HIGH=Backward) |
| **Right Motor PWM** | GP10 | Output | PWM speed control for Right Motor (0-4095) |
| **Right Motor DIR** | GP11 | Output | Direction control for Right Motor (LOW=Forward, HIGH=Backward) |
| **Digital IR (Left)** | GP0 | Input | Digital reading for extreme Left alignment |
| **Digital IR (Right)** | GP1 | Input | Digital reading for extreme Right alignment |
| **Analog IR (Left)** | GP27 (A1) | Input | Analog reading for Left sensor (`SENS_L`) |
| **Analog IR (Center)** | GP26 (A0) | Input | Analog reading for Center sensor (`SENS_C`) |
| **Analog IR (Right)** | GP28 (A2) | Input | Analog reading for Right sensor (`SENS_R`) |
| **Button 1** | GP20 | Input (Pullup) | Starts Calibration & Runs Track 1 (Turbo Mode) |
| **Button 2** | GP21 | Input (Pullup) | Starts Calibration & Runs Track 2/3 (Stable Mode) |
---

## 🚀 Getting Started / How to Run
1. Clone the repository to your local machine:
2. 
   ```bash
   git clone [[https://github.com/moaliog/Enswmatomena-Systymata-Omada1-Tmima1-.git](https://github.com/moaliog/Enswmatomena-Systymata-Omada1-Tmima1-.git](https://github.com/moaliog/Enswmatomena-Systymata-Omada1-Tmima1-/blob/main/main.cpp/main.cpp.ino))
