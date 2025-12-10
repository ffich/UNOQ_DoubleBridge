# UNOQ DoubleBridge
Repositgory that implement the UNOQ DoubleBridge.

## Install script
```
curl -L https://github.com/ffich/UNOQ_DoubleBridge/archive/refs/heads/main.tar.gz \
| tar -xz --strip-components=2 -C /home/arduino/ArduinoApps UNOQ_DoubleBridge-main/DoubleBridge
```

## UNOQ TCP Control Server + Node-RED Integration

This project provides a **robust and extensible control infrastructure for Arduino UNO Q**, based on:

-   **Node-RED** for automation logic and UI/dashboard
-   **Python (App Lab)** as a **TCP control server**
-   **Bridge** for communication with the MCU sketch
-   **JSON line-based protocol over TCP**

The goal is to create a **clean, scalable, and production-ready bridge**
between high-level automation (Node-RED) and low-level hardware control
(UNO Q).

------------------------------------------------------------------------

## System Architecture

<img width="1067" height="292" alt="image" src="https://github.com/user-attachments/assets/de78f9a8-e0cc-424d-8af1-ac1e9c2e5690" />

### Data Flow

1.  **Node-RED**
    -   Builds JSON commands
    -   Sends them via TCP
2.  **Python App Lab Server**
    -   Receives and validates JSON
    -   Dispatches commands to the Bridge
    -   Sends structured JSON responses back
3.  **UNO Q Sketch**
    -   Executes hardware operations (GPIO, ADC, LED Matrix)

------------------------------------------------------------------------

## Communication Protocol

-   **Transport:** TCP
-   **Format:** JSON
-   **Message termination:** newline (`\n`) -- **mandatory**
-   **Model:** request → response

### Example

``` json
{"cmd":"set_io","pin":"D13","value":true}
```

(with `\n` appended at the end)

------------------------------------------------------------------------

# Implemented Features

------------------------------------------------------------------------

## 1. Digital GPIO Control

### Set Digital Output

**Command:**

``` json
{ "cmd": "set_io", "pin": "D13", "value": true }
```

**Python Action:**

``` python
Bridge.call("set_pin_by_name", pin, True)
```

**Response:**

``` json
{
  "ok": true,
  "event": "io_updated",
  "pin": "D13",
  "value": true
}
```

------------------------------------------------------------------------

### Get Digital Input

**Command:**

``` json
{ "cmd": "get_io", "pin": "D13" }
```

**Python Action:**

``` python
Bridge.call("get_pin_by_name", pin)
```

**Response:**

``` json
{
  "ok": true,
  "event": "io_status",
  "pin": "D13",
  "value": true
}
```

------------------------------------------------------------------------

## 2. Global Digital Control (Set All Outputs)

This command allows enabling or disabling **all digital outputs at
once**.

**Command:**

``` json
{ "cmd": "set_all_io", "value": false }
```

**Python Action:**

``` python
Bridge.call("set_all_io", False)
```

**Response:**

``` json
{
  "ok": true,
  "event": "io_all_updated",
  "value": false
}
```

------------------------------------------------------------------------

## 3. LED Matrix Text Output

The UNO Q LED Matrix is driven using the **standard Arduino UNO R4 LED
Matrix library**.

### Command

``` json
{
  "cmd": "led_matrix_print",
  "text": "HELLO UNO Q"
}
```

### Python

``` python
Bridge.call("led_matrix_print", text)
```

### MCU Implementation (Arduino_LED_Matrix)

``` cpp
#include <Arduino_LED_Matrix.h>

ArduinoLEDMatrix matrix;

void setup() {
  Bridge.begin();
  matrix.begin();
  Bridge.addCommand("led_matrix_print", led_matrix_print);
}

void led_matrix_print(String text) {
  matrix.clear();
  matrix.textFont(Font_5x7);
  matrix.textScrollSpeed(75);
  matrix.beginDraw();
  matrix.stroke(255);
  matrix.text(0, 1, text);
  matrix.endDraw();
}
```

### Response

``` json
{
  "ok": true,
  "event": "led_matrix_updated",
  "text": "HELLO UNO Q"
}
```

------------------------------------------------------------------------

## 4. Analog Input Reading

Analog inputs are read using a Bridge function implemented on the MCU:

``` cpp
int get_an_pin_by_name(String pin);
```

### Command

``` json
{ "cmd": "get_an", "pin": "A0" }
```

### Python

``` python
Bridge.call("get_an_pin_by_name", pin)
```

### Response

``` json
{
  "ok": true,
  "event": "an_value",
  "pin": "A0",
  "value": 734
}
```

------------------------------------------------------------------------

# TCP Server Robustness

The Python TCP server is designed to be **fault-tolerant and
production-safe**:

-   Handles:
    -   Invalid JSON
    -   Missing newline termination
    -   Socket timeouts
    -   Bridge runtime errors
-   Never blocks indefinitely
-   Always returns structured error responses

### Error Examples

``` json
{ "ok": false, "error": "Invalid JSON" }
```

``` json
{ "ok": false, "error": "Timeout waiting for complete line" }
```

``` json
{ "ok": false, "error": "Bridge call set_all_io failed: ..." }
```

------------------------------------------------------------------------

# Node-RED Integration

Each command is encapsulated inside **dedicated subflows**:

-   ✅ Set IO via TCP\
-   ✅ Get IO via TCP\
-   ✅ Set All IO via TCP\
-   ✅ LED Matrix Print via TCP\
-   ✅ Get Analog via TCP

Each subflow:

-   Uses an **environment variable `PIN_NAME`** when needed
-   Automatically builds:

``` json
{ "cmd": "...", ... }
```

-   Sends data through a `tcp request` node

## Node-RED Control Panel
A Node-RED sample control panel is also provided:
<img width="1805" height="860" alt="image" src="https://github.com/user-attachments/assets/55f1db99-a45a-412a-82e6-16eee1d70f62" />


------------------------------------------------------------------------

# Extensibility

This architecture allows you to easily add:

-   PWM outputs
-   DAC control
-   I²C / SPI via Bridge
-   Advanced LED effects
-   State synchronization
-   Safe boot modes
-   Remote diagnostics

To add a new feature:

1.  Implement the function in the MCU sketch
2.  Add a new branch in `handle_command` (Python)
3.  Create a new Node-RED subflow

------------------------------------------------------------------------

# Conclusion

This project provides a **clean, scalable, and industrial-grade control
bridge** between:

-   **Automation layer:** Node-RED\
-   **Control layer:** Python App Lab\
-   **Hardware layer:** Arduino UNO Q

It is designed to be:

-   ✅ Reliable\
-   ✅ Scalable\
-   ✅ Maintainable\
-   ✅ Ready for dashboards and remote control

