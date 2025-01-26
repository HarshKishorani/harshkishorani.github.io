---
title: "Universal IR remote | Hardware."
date: "2024-12-18"
summary: "Making an Universal IR Remote Control using an ESP32."
categories: ["esp32","iot"]
---

### Hardware Setup for IR Remote Repeater Using ESP-IDF

To complement the software implementation explained in Part 1, we now focus on the hardware setup for building the IR remote repeater using ESP-IDF and the IRremoteESP8266 library. By the end of this guide, you’ll have a clear understanding of the components required and the wiring details to ensure proper functionality.

#### Components Required:

1. **ESP32 Development Board**

2. **IR Receiver Module**
   - Common IR receiver modules like the TSOP1738 or TSOP38238 work well for this project.

3. **IR LED (Infrared LED)**
   - A 940nm IR LED is suitable for transmitting the IR signals. Optionally, a high-power IR LED can be used for extended range.

4. **Resistors**
   - A 220-ohm resistor for the IR LED.
   - A pull-up resistor (10kΩ) for the IR receiver (optional if it’s not integrated into the module).

5. **Transistor**
   - A 2N2222 or BC547 transistor to drive the IR LED. This is essential to amplify the current and ensure strong transmission.

6. **Diode**
   - A 1N4148 diode (optional) to protect the circuit in case of reverse polarity.

7. **Capacitors**
   - Decoupling capacitors (10μF and 0.1μF) near the power pins of the IR receiver for stability.

8. **Breadboard and Jumper Wires**
   - For prototyping and connecting components together.

9. **Power Source**
   - Use a USB cable to power the ESP32 development board or an external 5V power supply.

#### Hardware Connections:

Below are the detailed connections for setting up the IR remote repeater:

1. **IR Receiver Module**
   - Connect the **OUT** pin of the IR receiver module to GPIO 18 on the ESP32 (this is defined as `kRecvPin` in the code).
   - Connect the **VCC** pin of the IR receiver to the 3.3V pin on the ESP32.
   - Connect the **GND** pin of the IR receiver to the GND pin on the ESP32.

   *(Note: If your IR receiver does not include an onboard pull-up resistor, connect a 10kΩ pull-up resistor between the OUT and VCC pins.)*

2. **IR LED Circuit**
   - Connect the **anode** (longer leg) of the IR LED to the **collector** of the transistor (2N2222 or BC547).
   - Connect the **cathode** (shorter leg) of the IR LED to the GND via a 220Ω resistor.
   - Connect the **emitter** of the transistor to GND.
   - Connect the **base** of the transistor to GPIO 19 on the ESP32 (this is defined as `kIrLedPin` in the code) via a 1kΩ resistor.

   *(Note: Using a transistor ensures the IR LED gets enough current for effective signal transmission.)*

3. **ESP32 Development Board**
   - Connect the ESP32 to your computer using a USB cable for power and programming.
   - Ensure the GPIO pins specified in the code (18 for receiver and 19 for transmitter) are correctly wired to the respective components.

#### Assembly Tips:

1. **IR LED Orientation**
   - Ensure the IR LED’s anode and cathode are connected properly. Reverse polarity may damage the LED.

2. **IR Receiver Placement**
   - Position the IR receiver so it’s exposed to incoming IR signals. Avoid blocking it with components or wires.

3. **Power Supply**
   - Use a stable power source for the ESP32 and connected components. Fluctuations may cause erratic behavior.

4. **Decoupling Capacitors**
   - Place decoupling capacitors close to the power pins of the IR receiver for added stability.

#### Testing the Setup:

1. After completing the wiring, upload the code from Part 1 to the ESP32 using ESP-IDF.
2. Open the serial monitor in your IDE (e.g., VSCode or ESP-IDF terminal) at a baud rate of 115200.
3. Point a remote control at the IR receiver and press a button. You should see the IR signal details printed in the serial monitor.
4. The IR LED should retransmit the captured signal. You can verify this by pointing it toward another device (e.g., a TV or an air conditioner) and observing if it reacts as expected.

#### Troubleshooting:

- **No Signal Detected:**
  - Double-check the connections to the IR receiver module.
  - Verify the GPIO pin configuration in the code.

- **Weak IR Transmission:**
  - Ensure the IR LED is correctly wired and not damaged.
  - Replace the 220Ω resistor with a lower value (e.g., 150Ω) to increase the current if the transistor supports it.

- **ESP32 Resets Frequently:**
  - Check if the IR LED circuit is drawing too much power.
  - Use an external 5V power supply if necessary.

#### Conclusion:

With the hardware setup complete, your IR remote repeater is ready to capture and retransmit IR signals. This project provides a solid foundation for further development, such as decoding specific protocols, adding a web interface, or controlling devices via Wi-Fi. Stay tuned for future updates on enhancing this project!