---
title: "Making an Universal IR remote using ESP32 (Part-1)."
date: "2024-04-14"
summary: "Michael's weeknotes for November 4-10, 2024."
categories: ["esp32","iot"]
---

### We will be using esp idf to implement IRremoteesp8266 Arduino Library in esp-idf to build an IR remote repeater.


The [**IRremoteESP8266**](https://github.com/crankyoldgit/IRremoteESP8266) is an Arduino Framework which can be used to **send and receive** infra-red signals on an **ESP8266** or an **ESP32** using common 940nm IR LEDs and common IR receiver modules. e.g. TSOP{17,22,24,36,38,44,48}* demodulators etc.


Firstly we will learn how to implement this Arduino Framework into ESP-IDF.


*(Note : You can implement any Arduino Library or Framework in ESP_IDF using this approach. We won't we using Platform-IO to do this step, although you can try that approach too.)*


So, to implement [**IRremoteESP8266**](https://github.com/crankyoldgit/IRremoteESP8266) Arduino Library into esp-idf (not using Platform-IO) first we will need to import the arduino-esp32 library as a component in our esp-idf project.


Follow the following steps to add arduino-esp32 library as your global component for your esp-idf projects.
1. Go to your `esp-idf/components` directory in your terminal.
2. Run the following commands:
  - `git clone https://github.com/espressif/arduino-esp32.git arduino`
  - `cd arduino`
  - `git submodule update --init --recursive`


You have successfully added arduino-esp32 library as your idf component.
Now, create a new idf project in vscode using the sample project template.


Change your *main.c* file to *main.cpp* in your `main` folder and also update your `idf_component_register` in `main/CmakeLists.txt`.


Let's test out the arduino library. Paste the following code into your `main.cpp`.


(Note : In `sdkconfig.defaults` of project set `CONFIG_FREERTOS_HZ=1000`.)


```C++
#include "Arduino.h"


extern "C" void app_main()
{
  initArduino();


  // Arduino-like setup()
  Serial.begin(115200);
  while(!Serial){
    ; // wait for serial port to connect
  }


  // Arduino-like loop()
  while(true){
    Serial.println("loop");
  }
}
```


Now you can similarly create a separate idf component for the **IRremoteESP8266** library, but I will be copying the source code of the library and importing it in my project.


I will copy the `src` folder from [**IRremoteESP8266**](https://github.com/crankyoldgit/IRremoteESP8266) folder and paste it in `main\`. And update the `main/CmakeLists.txt` as following.


```C++
# Define where the source files are located
set(SOURCE_DIR "${CMAKE_CURRENT_LIST_DIR}/src")


# Collect all .cpp source files from the source directory
file(GLOB_RECURSE SOURCE_FILES "${SOURCE_DIR}/*.cpp")


idf_component_register(SRCS "main.cpp" ${SOURCE_FILES}
                    INCLUDE_DIRS "." ${SOURCE_DIR}
                    REQUIRES "arduino")
```


### Your Project structure will look like this:
    .
    ├── build                  
    ├── main                    
          ├── src    # IRremoteESP8266 source files.
          ├── main.cpp
          ├── CmakeLists.txt              
    ├── CmakeLists.txt                    
    ├── sdkconfig                  
    ├── sdkconfig.defaults
    └── README.md




Our Library is successfully added to the IDF environment. Copy the following code IR repeater code in the `main.cpp` of the project. This code captures the incoming IR signal and retransmits it back using the transmitter.


*(Note : I will be using an ESP32 development board and an IR receiver and transmitter at GPIO 18 and 19 respectively.)*


```C++
#include "Arduino.h"
#include "src/IRrecv.h"
#include "src/IRsend.h"
#include "IRutils.h"


const uint16_t kRecvPin = 18;
const uint16_t kIrLedPin = 19;


// The Serial connection baud rate.
const uint32_t kBaudRate = 115200;


// As this program is a special purpose capture/resender, let's use a larger
// than expected buffer so we can handle very large IR messages.
const uint16_t kCaptureBufferSize = 1024; // 1024 == ~511 bits


// kTimeout is the Nr. of milli-Seconds of no-more-data before we consider a
// message ended.
const uint8_t kTimeout = 50; // Milli-Seconds


// kFrequency is the modulation frequency all UNKNOWN messages will be sent at.
const uint16_t kFrequency = 38000; // in Hz. e.g. 38kHz.


// The IR transmitter.
IRsend irsend(kIrLedPin);
// The IR receiver.
IRrecv irrecv(kRecvPin, kCaptureBufferSize, kTimeout, false);
// Somewhere to store the captured message.
decode_results results;


extern "C" void app_main()
{
  initArduino();


  irrecv.enableIRIn(); // Start up the IR receiver.
  irsend.begin();      // Start up the IR sender.


  Serial.begin(kBaudRate);
  while (!Serial) // Wait for the serial connection to be establised.
    delay(50);
  Serial.println();


  Serial.print("SmartIRRepeater is now running and waiting for IR input "
               "on Pin ");
  Serial.println(kRecvPin);
  Serial.print("and will retransmit it on Pin ");
  Serial.println(kIrLedPin);


  while (true)
  {
    // Check if an IR message has been received.
    if (irrecv.decode(&results))
    { // We have captured something.
      // The capture has stopped at this point.
      decode_type_t protocol = results.decode_type;
      uint16_t size = results.bits;
      bool success = true;
      // Is it a protocol we don't understand?
      if (protocol == decode_type_t::UNKNOWN)
      { // Yes.
        // Convert the results into an array suitable for sendRaw().
        // resultToRawArray() allocates the memory we need for the array.
        uint16_t *raw_array = resultToRawArray(&results);
        // Find out how many elements are in the array.
        size = getCorrectedRawLength(&results);
#if SEND_RAW
        // Send it out via the IR LED circuit.
        irsend.sendRaw(raw_array, size, kFrequency);
#endif // SEND_RAW
        // Deallocate the memory allocated by resultToRawArray().
        delete[] raw_array;
      }
      else if (hasACState(protocol))
      { // Does the message require a state[]?
        // It does, so send with bytes instead.
        success = irsend.send(protocol, results.state, size / 8);
      }
      else
      { // Anything else must be a simple message protocol. ie. <= 64 bits
        success = irsend.send(protocol, results.value, size);
      }
      // Resume capturing IR messages. It was not restarted until after we sent
      // the message so we didn't capture our own message.
      irrecv.resume();


      // Display a crude timestamp & notification.
      uint32_t now = millis();
      Serial.printf(
          "%06lu.%03lu: A %d-bit %s message was %ssuccessfully retransmitted.\n",
          now / 1000, now % 1000, size, typeToString(protocol).c_str(),
          success ? "" : "un");
    }
    yield(); // Or delay(milliseconds); This ensures the ESP doesn't WDT reset.
  }
}
```

Part 2 Coming soon : {{< backlink "universal-ir-part-2" >}}