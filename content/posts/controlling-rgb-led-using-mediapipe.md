---
title: "Transform Your RGB Lights with Finger Gestures: A Mediapipe & ESP32 Project"
date: 2024-12-18
summary: Learn how to control RGB LED Strip Lights with your fingers using Mediapipe and ESP32.
categories:
  - Artificial Intelligence
  - esp32
  - iot
  - Mediapipe
---
### **Introduction**

Bring your RGB lights to life with simple hand gestures! This project combines **Mediapipe’s Hand Tracking** and an **ESP32 microcontroller** to let you control your lights effortlessly. Follow along to build a fun, interactive lighting system.

---
## **1. The Hardware**

Firstly a five-meter 12 volt RGB LED strip has to be connected to an **ESP32 NodeMCU board** because the NodeMCU board cannot directly drive the LED strip. We'll use a NPN transistor(TIP120 or  TIP122) to drive the LED strip using the NodeMCU. The transistors are connected to the PWM supported pins of the ESP32 development board each for red, green and blue lines of the LED strip. We'll send serial data to ESP32 using a Python script to control individual red, green and blue signal.

### Parts required

| **Part**              | **Description**                                       | **Affiliate Link** |
| --------------------- | ----------------------------------------------------- | ------------------ |
| ESP32-DevKit-V1       | Microcontroller for handling inputs and outputs       | [Buy on Amazon](#) |
| 12V RGB Strip Light   | RGB LED strip compatible with 12V power supply        | [Buy on Amazon](#) |
| 3 TIP-122 Transistors | Transistors for driving RGB channels of the LED strip | [Buy on Amazon](#) |
| 12V Power Supply      | Power source for the RGB strip and ESP32              | [Buy on Amazon](#) |
| Barrel Adaptor        | Connects the 12V power supply to the RGB strip        | [Buy on Amazon](#) |

### Circuit

![rgb-strip-schematic](/rgb-strip-schematic.png)
- Connect the R, G & B terminals of the LED strip to the collector of the transistor as shown.
- Through a 10Kohm resistance connect the base of the transistors to the D25, D26 & D27 pins of the ESP32 NodeMCU.
- All the grounds have to be connected together as shown, the ground of ESP32, the ground of transistors, and the ground of the 12v power supply used for the RGB LED strip.
- You can power the NodeMCU board either through the USB cable or through the VIN terminal.
### Code
```C
// Initialize the GPIO pins for analogWrite
const int redPin = 26;
const int greenPin = 27;
const int bluePin = 25;

void setup() {
  Serial.begin(115200);
  
  pinMode(redPin, OUTPUT);
  pinMode(greenPin, OUTPUT);
  pinMode(bluePin, OUTPUT);
}

void loop() {
  if (Serial.available()) {
    String colorData = Serial.readStringUntil('\n');
    
    int red, green, blue;
    // Parse RGB values from the received data
    sscanf(colorData.c_str(), "%d,%d,%d", &red, &green, &blue);
    
    // Set RGB LED color
    analogWrite(redPin, red);
    analogWrite(greenPin, green);
    analogWrite(bluePin, blue);
  }
}
```

The code is designed to control an RGB LED using PWM [(Pulse Width Modulation)](https://en.wikipedia.org/wiki/Pulse-width_modulation) signals. It initializes GPIO pins 26, 27, and 25 as output pins to drive the red, green, and blue channels of the RGB LED, respectively. 

Inside the `loop()` function, the program continuously checks for incoming data via the serial port. When data is available, it reads a string containing RGB values formatted as "R,G,B" (e.g., `255,128,64`) and parses these values using `sscanf()`. The parsed values are then used to set the intensity of the red, green, and blue channels via `analogWrite()`, which adjusts the LED's color by changing the duty cycle of the PWM signals sent to each pin. This setup enables dynamic color changes for the RGB LED based on user input.

---
## **2.** **Mediapipe Hand landmarks detection and Serial Communication.**

### Setting Up the Environment

Before we dive into the code, let’s understand what we are building. We aim to detect hand landmarks using **MediaPipe**, a powerful machine learning framework, and then use these landmarks for further processing. 
### Importing Necessary Libraries

First, we import the essential libraries:

- **OpenCV (`cv2`)**: Used for image processing and video capture.
- **MediaPipe (`mediapipe`)**: Provides pre-trained models for hand tracking.
- **time**: Helps with performance measurement, like calculating FPS.
- **math**: For any mathematical operations (e.g., distance calculations).
- **serial**: Used if you plan to send hand gesture data to a device like Arduino.

```python
import cv2
import mediapipe as mp
import time
import math
import serial

```

### Creating the Hand Detector Class

The `HandDetector` class encapsulates the functionality for detecting hands and extracting landmark positions. This modular design makes it reusable and easy to expand.

#### 1. Initialization (`__init__` Method)

The constructor initializes MediaPipe's **Hands** solution with customizable parameters:

- `maxHands`: The maximum number of hands to detect (default is 2).
- `detection_confidence`: The confidence threshold for detecting hands.
- `tracking_confidence`: The confidence threshold for tracking hand landmarks.


```python

class HandDetector():
    def __init__(self, maxHands=2, detection_confidence=0.8, tracking_confidence=0.5):
        # Initialize MediaPipe Hands
        self.mpHands = mp.solutions.hands
        self.mpDraw = mp.solutions.drawing_utils
        self.hands = self.mpHands.Hands(
            min_detection_confidence=detection_confidence,
            min_tracking_confidence=tracking_confidence,
            max_num_hands=maxHands)

```

#### 2. Detecting Hands (`findHands` Method)

The `findHands` method processes an image, detects hands, and optionally draws landmarks and connections.

- Converts the input image to **RGB** format (MediaPipe processes only RGB).
- Flips the image for a mirror-like effect, as it's common in user interfaces.
- Uses the MediaPipe model to detect hands.
- Optionally draws the landmarks on the processed image.

**Key Steps:**

1. Convert and flip the image for hand tracking.
2. Use `self.hands.process()` to detect hands.
3. Draw the landmarks using MediaPipe’s drawing utilities.

```python
def findHands(self, img, draw=True):
    imgRGB = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    
    imgRGB = cv2.flip(imgRGB, 1)
    imgRGB.flags.writeable = False
    self.results = self.hands.process(imgRGB)
    imgRGB.flags.writeable = True
    
    imgRGB = cv2.cvtColor(imgRGB, cv2.COLOR_RGB2BGR)

    if draw and self.results.multi_hand_landmarks:
        for hand in self.results.multi_hand_landmarks:
            self.mpDraw.draw_landmarks(imgRGB, hand, self.mpHands.HAND_CONNECTIONS)
    return imgRGB

```

#### 3. Extracting Landmark Positions (`findPosition` Method)

Once hands are detected, we use `findPosition` to extract the positions of specific landmarks (e.g., fingertips). Each landmark is represented by its **ID** and (x, y) coordinates on the image.

**Key Steps:**

1. Retrieve the hand landmarks for the specified hand (`handNo`).
2. Loop through the landmarks and calculate their pixel positions.
3. Optionally, draw circles at each landmark position for visualization.

```python
def findPosition(self, img, handNo=0, draw=True, color=(255, 0, 255), z_axis=False):
    lmList = []
    if self.results.multi_hand_landmarks:
        myHand = self.results.multi_hand_landmarks[handNo]
        for id, lm in enumerate(myHand.landmark):
            h, w, c = img.shape
            cx, cy = int(lm.x * w), int(lm.y * h)
            lmList.append([id, cx, cy])
            if draw:
                cv2.circle(img, (cx, cy), 5, color, cv2.FILLED)
    return lmList
```
**Parameters:**

- `img`: The image to process.
- `handNo`: Index of the hand (0 for the first detected hand).
- `draw`: Boolean flag to control landmark visualization.
- `color`: Circle color for landmarks.

### Real-Time Hand Detection

Next, we'll use the **`HandDetector`** class and *findHands* function for real-time hand detection using **OpenCV**. 

```python
def main():
    cap = cv2.VideoCapture(0)
    detector = HandDetector(maxHands=1)
    
    while True:
        _, img = cap.read()
        img = detector.findHands(img, draw=True)
        
        # Display the result
        cv2.imshow("Glowing Sphere", img)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    cap.release()
    cv2.destroyAllWindows()
main()
```

First we initialize a video capture object to access the webcam and creates an instance of `HandDetector` to process the video frames. Inside the loop, each frame is read from the webcam and passed to the `findHands` method for hand detection. The hand landmarks are detected, the `draw` parameter is set to `True`, so visual markers are displayed on the image of your hand. The processed frames are then displayed in a window. The program continues to process and display frames until the **'q'** key is pressed, at which point the resources are released and the window is closed.

### Adding Fingertip and Wrist Detection

In this section, we’ll extend our hand detection pipeline to identify specific landmarks, such as the fingertips and wrist, using the `findPosition` method from the `HandDetector` class. These landmarks will later be processed to send meaningful data to an ESP32 for controlling the RGB LED. This approach bridges hand gestures with hardware interaction.

Here is the updated code:

```python
def main():
    cap = cv2.VideoCapture(0)
    detector = HandDetector(maxHands=1)

    while True:
        _, img = cap.read()
        img = detector.findHands(img, draw=False)
        lmList = detector.findPosition(img, z_axis=True, draw=True)

        if len(lmList) != 0:
            # Get positions of the wrist and fingertips
            x1, y1 = lmList[4][1], lmList[4][2]   # Thumb tip
            x2, y2 = lmList[8][1], lmList[8][2]   # Index finger tip
            x3, y3 = lmList[12][1], lmList[12][2]  # Middle finger tip
            x4, y4 = lmList[16][1], lmList[16][2]  # Ring finger tip
            x5, y5 = lmList[20][1], lmList[20][2]  # Pinky finger tip

            x6, y6 = lmList[0][1], lmList[0][2]   # Wrist

            x7, y7 = lmList[9][1], lmList[9][2]   # Middle of palm
            x8, y8 = lmList[13][1], lmList[13][2]  # Base of middle finger
            
            # TODO Process these landmarks to convert to color and send to ESP32

        # Display the result
        cv2.imshow("Glowing Sphere", img)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    cap.release()
    cv2.destroyAllWindows()

main()
```

##### Fingertip and Wrist Detection:

![hand-landmarks.png](/hand-landmarks.png)

The `findPosition` method is used with `z_axis=True` to extract 3D positions of specific landmarks. we extract the following:
   - **Thumb Tip:** `lmList[4]`
   - **Index Finger Tip:** `lmList[8]`
   - **Middle Finger Tip:** `lmList[12]`
   - **Ring Finger Tip:** `lmList[16]`
   - **Pinky Finger Tip:** `lmList[20]`
   - **Wrist:** `lmList[0]`
   - **Middle of Palm:** `lmList[9]`
   - **Base of Middle Finger:** `lmList[13]`
The indices for specific fingers and landmarks can be found using this image provided by this [MediaPipe documentation](https://ai.google.dev/edge/mediapipe/solutions/vision/hand_landmarker).
### Mapping Hand Movements to LED Colors

In this section, we’ll use the hand landmark coordinates to calculate distances, convert them into meaningful color data, and send the values to an ESP32. The primary focus is to calculate the average distance from the wrist to the fingertips and use it to determine the hue of an RGB LED color.

Here’s the complete code:

```python
def main():
    cap = cv2.VideoCapture(0)
    detector = HandDetector(maxHands=1)

    while True:
        _, img = cap.read()
        img = detector.findHands(img, draw=False)
        lmList = detector.findPosition(img, z_axis=True, draw=False)

        if len(lmList) != 0:
            # Get positions of the wrist and fingertips
            x1, y1 = lmList[4][1], lmList[4][2]   # Thumb tip
            x2, y2 = lmList[8][1], lmList[8][2]   # Index finger tip
            x3, y3 = lmList[12][1], lmList[12][2]  # Middle finger tip
            x4, y4 = lmList[16][1], lmList[16][2]  # Ring finger tip
            x5, y5 = lmList[20][1], lmList[20][2]  # Pinky finger tip

            x6, y6 = lmList[0][1], lmList[0][2]   # Wrist

            x7, y7 = lmList[9][1], lmList[9][2]   # Middle of palm
            x8, y8 = lmList[13][1], lmList[13][2]  # Base of middle finger

            # Calculate distances from the wrist to each fingertip
            distances = [
                math.hypot(x6 - x1, y6 - y1),
                math.hypot(x6 - x2, y6 - y2),
                math.hypot(x6 - x3, y6 - y3),
                math.hypot(x6 - x4, y6 - y4),
                math.hypot(x6 - x5, y6 - y5)
            ]

            # Calculate the average distance
            avg_distance = sum(distances) / len(distances)

            # Map the average distance to a hue value for the color
            hue = int(min(max((avg_distance / 300) * 179, 0), 179))
            color = cv2.cvtColor(
                cv2.UMat(1, 1, cv2.CV_8UC3, (hue, 255, 255)), cv2.COLOR_HSV2BGR
            ).get()[0, 0].tolist()

            red, green, blue = color

            # Send color values to ESP32
            ser.write(f"{red},{green},{blue}\n".encode())

        # Display the result
        cv2.imshow("Glowing Sphere", img)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    cap.release()
    cv2.destroyAllWindows()

main()
```

1. **Distance Calculation:**
   Using the wrist position as a reference point, we calculate the Euclidean distance to each fingertip. These distances represent the hand’s openness or specific gesture.
2. **Average Distance Mapping:**
   The average of these distances is computed and scaled to a hue value in the HSV color space. The `min` and `max` functions ensure that the hue stays within valid bounds.
3. **Hue to RGB Conversion:**
   The HSV color (hue, saturation, and value) is converted to an RGB color using OpenCV. This color is then used to set the LED color.
4. **Serial Communication:**
   The RGB values are sent to the ESP32 using a serial connection, enabling real-time LED control based on hand movements.

### Adding a Glowing Sphere to Visualize the Color

In this step, we’ll enhance the visual feedback by displaying a glowing sphere in the middle of the palm using the calculated color values. This sphere not only represents the color mapped from the hand’s distance but also adds an aesthetic glow effect to make the visualization more dynamic.

```python

def main():
    cap = cv2.VideoCapture(0)
    detector = HandDetector(maxHands=1)

    while True:
        _, img = cap.read()
        img = detector.findHands(img, draw=False)
        lmList = detector.findPosition(img, z_axis=True, draw=False)
        if len(lmList) != 0:
            # Get positions of the wrist and fingertips
            x1, y1 = lmList[4][1], lmList[4][2]   # Thumb tip
            x2, y2 = lmList[8][1], lmList[8][2]   # Index finger tip
            x3, y3 = lmList[12][1], lmList[12][2]  # Middle finger tip
            x4, y4 = lmList[16][1], lmList[16][2]  # Ring finger tip
            x5, y5 = lmList[20][1], lmList[20][2]  # Pinky finger tip
            x6, y6 = lmList[0][1], lmList[0][2]   # Wrist
            x7, y7 = lmList[9][1], lmList[9][2]   # Middle of palm
            x8, y8 = lmList[13][1], lmList[13][2]  # Base of middle finger

            # Calculate distances from the wrist to each fingertip
            distances = [
                math.hypot(x6 - x1, y6 - y1),
                math.hypot(x6 - x2, y6 - y2),
                math.hypot(x6 - x3, y6 - y3),
                math.hypot(x6 - x4, y6 - y4),
                math.hypot(x6 - x5, y6 - y5)
            ]
            
            # Calculate the average distance
            avg_distance = sum(distances) / len(distances)
            
            # Map the average distance to a hue value for the color
            hue = int(min(max((avg_distance / 300) * 179, 0), 179))
            color = cv2.cvtColor(
                cv2.UMat(1, 1, cv2.CV_8UC3, (hue, 255, 255)), cv2.COLOR_HSV2BGR
            ).get()[0, 0].tolist()
            red, green, blue = color

            # Send color values to ESP32
            ser.write(f"{red},{green},{blue}\n".encode())

            # Calculate the center point between landmarks 0, 9, and 13
            center_x = (x6 + x7 + x8) // 3
            center_y = (y6 + y7 + y8) // 3

            # Draw a filled circle (main sphere) at the wrist
            cv2.circle(img, (center_x, center_y), 20, color, cv2.FILLED)

            # Create a glow effect by drawing larger circles with decreasing opacity
            for i in range(1, 6):
                radius = 20 + i * 10  # Increase radius for the outer glow
                alpha = 255 - i * 40  # Reduce alpha for fading effect
                glow_color = [int(c * (alpha / 255)) for c in color]
                overlay = img.copy()
                cv2.circle(overlay, (center_x, center_y),
                           radius, glow_color, cv2.FILLED)

                # Blend the overlay with the original image
                img = cv2.addWeighted(overlay, 0.3, img, 0.7, 0)
        # Display the result
        cv2.imshow("Glowing Sphere", img)

        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    cap.release()
    cv2.destroyAllWindows()

main()

```
#### Key Steps in the Code:

1. **Calculate the Center Point**:  The center of the palm is determined as the average position of three landmarks: the wrist (0), the middle of the palm (9), and the base of the middle finger (13).

```python
center_x = (x6 + x7 + x8) // 3 
center_y = (y6 + y7 + y8) // 3
```

2. **Draw the Main Sphere**:  A filled circle is drawn at the calculated center point using the color derived from the distance.

```python
cv2.circle(img, (center_x, center_y), 20, color, cv2.FILLED)
```

3. **Create the Glow Effect**:  To make the sphere appear as if it’s glowing, larger circles with decreasing opacity are drawn around the main sphere. This creates a gradient-like effect.

```python
for i in range(1, 6):
    radius = 20 + i * 10  # Increase radius for the outer glow
    alpha = 255 - i * 40  # Reduce alpha for fading effect
    glow_color = [int(c * (alpha / 255)) for c in color]
    overlay = img.copy()
    cv2.circle(overlay, (center_x, center_y), radius, glow_color, cv2.FILLED)
    img = cv2.addWeighted(overlay, 0.3, img, 0.7, 0)
```

### **Conclusion**

You’ve successfully created a gesture-controlled RGB lighting system using **Mediapipe** and **ESP32**. This simple yet powerful project showcases how AI and IoT can come together for interactive solutions. Now, let your imagination guide your next creation!