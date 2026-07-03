# Code Cheat Sheet

**Platform assumption:** ESP32 / ESP8266 with the Arduino framework (the most common stack for IoT technical tests). Concepts transfer directly to STM32, Raspberry Pi Pico, or plain embedded C if the test uses a different MCU — only the HAL calls change.


---

## 1. WiFi — Station Mode (connect to a router)

```cpp
#include <WiFi.h>  // ESP32. Use <ESP8266WiFi.h> on ESP8266

const char* ssid     = "YourSSID";
const char* password = "YourPassword";

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected!");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}
```

## 2. WiFi — Auto-Reconnect Pattern (non-blocking)

Interviewers love testing this because naive code hangs forever in `while(WiFi.status() != WL_CONNECTED)`. A robust version uses a timeout + retry, and re-checks status in `loop()`.

```cpp
unsigned long lastReconnectAttempt = 0;
const unsigned long reconnectInterval = 5000; // ms

void ensureWiFiConnected() {
  if (WiFi.status() != WL_CONNECTED) {
    unsigned long now = millis();
    if (now - lastReconnectAttempt > reconnectInterval) {
      lastReconnectAttempt = now;
      Serial.println("WiFi lost, reconnecting...");
      WiFi.disconnect();
      WiFi.reconnect();
    }
  }
}

void loop() {
  ensureWiFiConnected();
  // ... rest of loop, never blocks here
}

// Optional: event-driven approach instead of polling
void WiFiEventHandler(WiFiEvent_t event) {
  switch (event) {
    case ARDUINO_EVENT_WIFI_STA_DISCONNECTED:
      Serial.println("WiFi disconnected");
      break;
    case ARDUINO_EVENT_WIFI_STA_GOT_IP:
      Serial.println("WiFi (re)connected");
      break;
    default: break;
  }
}
// register with: WiFi.onEvent(WiFiEventHandler);
```

## 3. WiFi — Access Point (AP) Mode

Useful for "first-boot configuration" patterns (device hosts its own AP so a phone can configure it).

```cpp
#include <WiFi.h>

const char* ap_ssid = "ESP32-Config";
const char* ap_pass = "12345678"; // min 8 chars, or "" for open AP

void setup() {
  Serial.begin(115200);
  WiFi.softAP(ap_ssid, ap_pass);
  Serial.print("AP IP address: ");
  Serial.println(WiFi.softAPIP()); // usually 192.168.4.1
}
```

---

## 4. MQTT — Connect & Reconnect (PubSubClient)

The de-facto standard library: `PubSubClient` by Nick O'Leary.

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

const char* mqtt_server = "broker.example.com";
const int   mqtt_port   = 1883;
const char* mqtt_user   = "device01";
const char* mqtt_pass   = "secret";
const char* client_id   = "esp32-iot-engineer-test";

WiFiClient espClient;
PubSubClient mqttClient(espClient);

void connectMQTT() {
  mqttClient.setServer(mqtt_server, mqtt_port);
  while (!mqttClient.connected()) {
    Serial.print("Connecting to MQTT...");
    if (mqttClient.connect(client_id, mqtt_user, mqtt_pass)) {
      Serial.println("connected");
    } else {
      Serial.print("failed, rc=");
      Serial.print(mqttClient.state()); // negative = network, positive = protocol error
      Serial.println(" retrying in 2s");
      delay(2000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  // ... WiFi.begin() here ...
  connectMQTT();
}

void loop() {
  if (!mqttClient.connected()) connectMQTT();
  mqttClient.loop(); // MUST be called frequently — handles keepalive/pings
}
```

## 5. MQTT — Publish

```cpp
// Simple publish
mqttClient.publish("sensors/esp32-01/temperature", "23.5");

// Publish with QoS-like retain flag (PubSubClient only supports QoS 0)
mqttClient.publish("sensors/esp32-01/status", "online", true /* retained */);

// Publish a formatted numeric value
char payload[16];
float temperature = 23.47;
dtostrf(temperature, 4, 2, payload); // float -> string, width 4, 2 decimals
mqttClient.publish("sensors/esp32-01/temperature", payload);
```

## 6. MQTT — Subscribe & Callback

```cpp
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  String message;
  for (unsigned int i = 0; i < length; i++) message += (char)payload[i];

  Serial.printf("Message on [%s]: %s\n", topic, message.c_str());

  if (String(topic) == "actuators/esp32-01/relay") {
    digitalWrite(RELAY_PIN, message == "ON" ? HIGH : LOW);
  }
}

void setup() {
  // ...
  mqttClient.setCallback(mqttCallback);
}

void connectMQTT() {
  // ... after mqttClient.connect succeeds:
  mqttClient.subscribe("actuators/esp32-01/relay");
}
```

## 7. MQTT + JSON Payload (ArduinoJson v7)

Real devices almost never send raw strings — they send structured JSON.

```cpp
#include <ArduinoJson.h>

void publishSensorData(float temp, float humidity, int rssi) {
  JsonDocument doc; // ArduinoJson 7: no capacity template needed (grows on heap)
  doc["device_id"]   = "esp32-01";
  doc["temperature"] = temp;
  doc["humidity"]     = humidity;
  doc["rssi"]         = rssi;
  doc["timestamp"]    = millis();

  char buffer[256];
  size_t n = serializeJson(doc, buffer);
  mqttClient.publish("sensors/esp32-01/data", buffer, n);
}

void mqttCallback(char* topic, byte* payload, unsigned int length) {
  JsonDocument doc;
  DeserializationError err = deserializeJson(doc, payload, length);
  if (err) {
    Serial.print("JSON parse failed: ");
    Serial.println(err.c_str());
    return;
  }
  const char* command = doc["command"];
  int value            = doc["value"] | 0; // "| 0" = default if key missing
  Serial.printf("cmd=%s value=%d\n", command, value);
}
```

---

## 8. Digital I/O (GPIO)

```cpp
#define LED_PIN   2
#define BUTTON_PIN 4

void setup() {
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP); // internal pull-up, button reads LOW when pressed
}

void loop() {
  if (digitalRead(BUTTON_PIN) == LOW) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
}
```

## 9. Interrupt-Driven Input with Software Debounce

A very common "show me you understand hardware" test question.

```cpp
volatile bool buttonPressedFlag = false;
volatile unsigned long lastInterruptTime = 0;
const unsigned long debounceDelay = 50; // ms

void IRAM_ATTR handleButtonInterrupt() {
  unsigned long now = millis();
  if (now - lastInterruptTime > debounceDelay) {
    buttonPressedFlag = true;
    lastInterruptTime = now;
  }
}

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), handleButtonInterrupt, FALLING);
}

void loop() {
  if (buttonPressedFlag) {
    buttonPressedFlag = false; // clear flag, do the real work OUTSIDE the ISR
    Serial.println("Button pressed (debounced)");
  }
}
```

## 10. Analog Input (ADC)

```cpp
#define SENSOR_PIN 34 // ESP32 ADC1 pin

void setup() {
  Serial.begin(115200);
  analogReadResolution(12); // ESP32 ADC: 0-4095 (12-bit)
}

void loop() {
  int raw = analogRead(SENSOR_PIN);
  float voltage = (raw / 4095.0) * 3.3; // convert to volts (3.3V reference)
  Serial.printf("Raw: %d  Voltage: %.3f V\n", raw, voltage);
  delay(500);
}
```

## 11. PWM Output

ESP32 Arduino core **3.x** changed the PWM (LEDC) API — `ledcSetup`/`ledcAttachPin` were merged into a single `ledcAttach()`. Know both in case the test rig uses an older core.

```cpp
// --- Core 3.x (current) ---
#define LED_PIN 5
void setup() {
  ledcAttach(LED_PIN, 5000, 8); // pin, frequency (Hz), resolution (bits)
}
void loop() {
  for (int duty = 0; duty <= 255; duty++) {
    ledcWrite(LED_PIN, duty); // duty 0-255 for 8-bit resolution
    delay(10);
  }
}

// --- Legacy Core 2.x (channel-based) ---
// #define LED_PIN 5
// #define LEDC_CHANNEL 0
// ledcSetup(LEDC_CHANNEL, 5000, 8);
// ledcAttachPin(LED_PIN, LEDC_CHANNEL);
// ledcWrite(LEDC_CHANNEL, duty);
```

---

## 12. I2C Communication (Wire library)

```cpp
#include <Wire.h>

void setup() {
  Wire.begin(); // ESP32 default SDA=21, SCL=22 (override: Wire.begin(SDA, SCL))
  Serial.begin(115200);
}

// --- I2C bus scanner (extremely common interview/test task) ---
void scanI2C() {
  Serial.println("Scanning I2C bus...");
  for (byte addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
      Serial.printf("Found device at 0x%02X\n", addr);
    }
  }
}

// --- Read 2 bytes from a register (typical sensor pattern) ---
int16_t readRegister16(uint8_t deviceAddr, uint8_t reg) {
  Wire.beginTransmission(deviceAddr);
  Wire.write(reg);
  Wire.endTransmission(false); // false = keep connection (repeated start)
  Wire.requestFrom(deviceAddr, (uint8_t)2);
  int16_t value = Wire.read() << 8 | Wire.read();
  return value;
}

// --- Write a single byte to a register ---
void writeRegister(uint8_t deviceAddr, uint8_t reg, uint8_t value) {
  Wire.beginTransmission(deviceAddr);
  Wire.write(reg);
  Wire.write(value);
  Wire.endTransmission();
}
```

## 13. SPI Communication

```cpp
#include <SPI.h>

#define CS_PIN 5

void setup() {
  pinMode(CS_PIN, OUTPUT);
  digitalWrite(CS_PIN, HIGH); // idle high
  SPI.begin(); // ESP32 default: SCK=18, MISO=19, MOSI=23
}

uint8_t spiTransfer(uint8_t reg, uint8_t value) {
  SPI.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));
  digitalWrite(CS_PIN, LOW);
  SPI.transfer(reg);
  uint8_t result = SPI.transfer(value);
  digitalWrite(CS_PIN, HIGH);
  SPI.endTransaction();
  return result;
}
```

## 14. UART / Serial Communication (talking to another MCU or module)

```cpp
// ESP32 has 3 hardware UARTs. Serial = UART0 (USB debug).
// Use UART1/UART2 for talking to a GPS module, modem, or another MCU.
HardwareSerial gpsSerial(1); // UART1

void setup() {
  Serial.begin(115200);              // debug console
  gpsSerial.begin(9600, SERIAL_8N1, 16, 17); // baud, config, RX pin, TX pin
}

void loop() {
  while (gpsSerial.available()) {
    char c = gpsSerial.read();
    Serial.write(c); // bridge: print whatever the module sends
  }
}
```

---

## 15. JSON Parse & Build — Quick Reference (ArduinoJson v7)

```cpp
#include <ArduinoJson.h>

// PARSE
const char* json = "{\"sensor\":\"gps\",\"lat\":-6.21,\"lon\":106.85}";
JsonDocument doc;
DeserializationError error = deserializeJson(doc, json);
if (!error) {
  const char* sensor = doc["sensor"];
  double lat = doc["lat"];
  double lon = doc["lon"];
}

// BUILD
JsonDocument out;
out["device"] = "esp32-01";
out["values"][0] = 23.5;
out["values"][1] = 45.1;
JsonArray tags = out["tags"].to<JsonArray>();
tags.add("indoor");
tags.add("calibrated");

String result;
serializeJson(out, result);
Serial.println(result);
```

---

## 16. HTTP GET / POST (HTTPClient)

```cpp
#include <HTTPClient.h>

void httpGetExample() {
  HTTPClient http;
  http.begin("http://api.example.com/status");
  int httpCode = http.GET();
  if (httpCode == 200) {
    Serial.println(http.getString());
  } else {
    Serial.printf("GET failed, code: %d\n", httpCode);
  }
  http.end();
}

void httpPostExample(float temperature) {
  HTTPClient http;
  http.begin("http://api.example.com/telemetry");
  http.addHeader("Content-Type", "application/json");

  JsonDocument doc;
  doc["temperature"] = temperature;
  String body;
  serializeJson(doc, body);

  int httpCode = http.POST(body);
  Serial.printf("POST response: %d\n", httpCode);
  Serial.println(http.getString());
  http.end();
}
```

## 17. NTP Time Sync

```cpp
#include <time.h>

void setupTime() {
  configTime(7 * 3600, 0, "pool.ntp.org", "time.nist.gov"); // GMT+7 (Jakarta)
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) {
    Serial.println("Waiting for NTP sync...");
    delay(500);
  }
  Serial.println(&timeinfo, "%Y-%m-%d %H:%M:%S");
}
```

## 18. OTA Firmware Update (ArduinoOTA)

```cpp
#include <ArduinoOTA.h>

void setupOTA() {
  ArduinoOTA.setHostname("esp32-iot-device");
  ArduinoOTA.onStart([]() { Serial.println("OTA Start"); });
  ArduinoOTA.onEnd([]()   { Serial.println("OTA End"); });
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\n", (progress * 100) / total);
  });
  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("OTA Error[%u]\n", error);
  });
  ArduinoOTA.begin();
}

void loop() {
  ArduinoOTA.handle(); // must be called every loop
}
```

---

## 19. Deep Sleep / Power Saving

```cpp
#define WAKEUP_PIN GPIO_NUM_33
#define SLEEP_SECONDS 60

void goToDeepSleep() {
  esp_sleep_enable_timer_wakeup(SLEEP_SECONDS * 1000000ULL); // microseconds
  esp_sleep_enable_ext0_wakeup(WAKEUP_PIN, 1); // wake on pin HIGH (optional)
  Serial.println("Going to sleep...");
  Serial.flush();
  esp_deep_sleep_start();
}

void checkWakeReason() {
  esp_sleep_wakeup_cause_t reason = esp_sleep_get_wakeup_cause();
  switch (reason) {
    case ESP_SLEEP_WAKEUP_TIMER: Serial.println("Woke from timer"); break;
    case ESP_SLEEP_WAKEUP_EXT0:  Serial.println("Woke from GPIO");  break;
    default: Serial.println("Normal boot/reset"); break;
  }
}
```

## 20. Watchdog Timer (recover from a hung loop)

```cpp
#include <esp_task_wdt.h>

#define WDT_TIMEOUT 10 // seconds

void setup() {
  esp_task_wdt_init(WDT_TIMEOUT, true); // true = panic/reset on timeout
  esp_task_wdt_add(NULL);               // watch current task (loopTask)
}

void loop() {
  esp_task_wdt_reset(); // "I'm alive" — call this regularly
  // ... normal work ...
  // If this loop ever hangs (e.g., blocking I2C call), the chip reboots
  // automatically after WDT_TIMEOUT seconds instead of staying frozen.
}
```

## 21. FreeRTOS — Dual-Core Multitasking

ESP32 runs FreeRTOS under the hood. Knowing this is a common differentiator in IoT technical tests.

```cpp
TaskHandle_t sensorTaskHandle;

void sensorTask(void* parameter) {
  for (;;) {
    float temp = readTemperature(); // your sensor function
    Serial.printf("[Core %d] Temp: %.2f\n", xPortGetCoreID(), temp);
    vTaskDelay(pdMS_TO_TICKS(1000)); // non-blocking delay, yields to scheduler
  }
}

void setup() {
  Serial.begin(115200);
  xTaskCreatePinnedToCore(
    sensorTask,        // function
    "SensorTask",       // name
    4096,                // stack size (bytes)
    NULL,                // parameter
    1,                   // priority
    &sensorTaskHandle,   // task handle
    1                    // core (0 or 1; ESP32 core 1 is usually free for app code)
  );
}

void loop() {
  // runs on core 1 by default (Arduino loop task) alongside other tasks
}
```

---

## 22. GPS / NMEA Parsing (TinyGPS++)

Highly relevant given this role sits under a **fleet/transportation management** product — vehicle GPS tracking is a likely test topic.

```cpp
#include <TinyGPS++.h>
#include <HardwareSerial.h>

TinyGPSPlus gps;
HardwareSerial gpsSerial(1);

void setup() {
  Serial.begin(115200);
  gpsSerial.begin(9600, SERIAL_8N1, 16, 17); // RX, TX to GPS module
}

void loop() {
  while (gpsSerial.available()) {
    gps.encode(gpsSerial.read());
  }

  if (gps.location.isUpdated()) {
    Serial.printf("Lat: %.6f  Lon: %.6f  Speed: %.1f km/h  Sats: %d\n",
                  gps.location.lat(),
                  gps.location.lng(),
                  gps.speed.kmph(),
                  gps.satellites.value());
  }

  // Manual NMEA sentence parsing (in case the test forbids libraries):
  // $GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47
  // fields are comma-separated: time, lat, N/S, lon, E/W, fix quality, sats, hdop...
}
```

## 23. CAN Bus (MCP2515) — Automotive / Vehicle Telematics

The job mentions "IoT/automotive hardware" — CAN bus is the standard vehicle data bus (OBD-II runs on top of it). Most test rigs use an MCP2515 SPI-to-CAN module.

```cpp
#include <SPI.h>
#include <mcp_can.h>

#define CAN_CS_PIN 5
MCP_CAN CAN(CAN_CS_PIN);

void setup() {
  Serial.begin(115200);
  if (CAN.begin(MCP_ANY, CAN_500KBPS, MCP_8MHZ) == CAN_OK) {
    Serial.println("CAN init OK");
  } else {
    Serial.println("CAN init FAILED");
  }
  CAN.setMode(MCP_NORMAL);
}

void loop() {
  // --- Receive ---
  if (CAN.checkReceive() == CAN_MSGAVAIL) {
    long unsigned int rxId;
    unsigned char len = 0;
    unsigned char buf[8];
    CAN.readMsgBuf(&rxId, &len, buf);

    Serial.printf("ID: 0x%lX  Data:", rxId);
    for (int i = 0; i < len; i++) Serial.printf(" %02X", buf[i]);
    Serial.println();
  }

  // --- Transmit ---
  unsigned char data[8] = {0x01, 0x02, 0x03, 0x04, 0, 0, 0, 0};
  CAN.sendMsgBuf(0x100, 0, 8, data); // CAN ID 0x100, standard frame, 8 bytes
  delay(500);
}
```

---

## 24. Non-Volatile Storage (Preferences — ESP32's key-value flash storage)

```cpp
#include <Preferences.h>

Preferences prefs;

void saveConfig(int deviceId, const char* wifiSsid) {
  prefs.begin("config", false); // namespace, read-write
  prefs.putInt("device_id", deviceId);
  prefs.putString("ssid", wifiSsid);
  prefs.end();
}

void loadConfig() {
  prefs.begin("config", true); // read-only
  int id = prefs.getInt("device_id", -1);     // -1 = default if not found
  String ssid = prefs.getString("ssid", "");
  prefs.end();
  Serial.printf("Loaded device_id=%d ssid=%s\n", id, ssid.c_str());
}
```

---

## 25. Data Logging Pattern for DAQ / DoE Testing

Given the job explicitly mentions **Data Acquisition (DAQ) testing** and **Design of Experiment (DoE)**, expect a task like "log N samples at fixed intervals and report basic stats."

```cpp
const int SAMPLE_COUNT = 100;
float samples[SAMPLE_COUNT];

void runDataAcquisition() {
  Serial.println("timestamp_ms,raw_adc,voltage"); // CSV header

  for (int i = 0; i < SAMPLE_COUNT; i++) {
    int raw = analogRead(SENSOR_PIN);
    float voltage = (raw / 4095.0) * 3.3;
    samples[i] = voltage;

    Serial.printf("%lu,%d,%.4f\n", millis(), raw, voltage); // CSV row
    delay(20); // 50 Hz sample rate
  }
  reportStats();
}

void reportStats() {
  float sum = 0, sumSq = 0;
  float minVal = samples[0], maxVal = samples[0];

  for (int i = 0; i < SAMPLE_COUNT; i++) {
    sum += samples[i];
    if (samples[i] < minVal) minVal = samples[i];
    if (samples[i] > maxVal) maxVal = samples[i];
  }
  float mean = sum / SAMPLE_COUNT;

  for (int i = 0; i < SAMPLE_COUNT; i++) {
    sumSq += (samples[i] - mean) * (samples[i] - mean);
  }
  float stdDev = sqrt(sumSq / SAMPLE_COUNT);

  Serial.printf("Mean: %.4f  StdDev: %.4f  Min: %.4f  Max: %.4f\n",
                mean, stdDev, minVal, maxVal);
}
```

---

## 26. Simple State Machine Pattern

Common for "implement a device that goes through states X → Y → Z" test questions.

```cpp
enum DeviceState { IDLE, CONNECTING, RUNNING, ERROR_STATE };
DeviceState currentState = IDLE;

void updateStateMachine() {
  switch (currentState) {
    case IDLE:
      if (WiFi.status() == WL_CONNECTED) currentState = CONNECTING;
      break;
    case CONNECTING:
      if (mqttClient.connected()) currentState = RUNNING;
      break;
    case RUNNING:
      // normal operation
      if (someErrorCondition()) currentState = ERROR_STATE;
      break;
    case ERROR_STATE:
      Serial.println("Error! Attempting recovery...");
      currentState = IDLE;
      break;
  }
}
```

## 27. Software Debounce / Moving Average Filter (signal conditioning)

```cpp
// Moving average filter — smooths noisy ADC/sensor readings
#define FILTER_SIZE 10
float readingBuffer[FILTER_SIZE];
int filterIndex = 0;

float movingAverage(float newReading) {
  readingBuffer[filterIndex] = newReading;
  filterIndex = (filterIndex + 1) % FILTER_SIZE;

  float sum = 0;
  for (int i = 0; i < FILTER_SIZE; i++) sum += readingBuffer[i];
  return sum / FILTER_SIZE;
}
```

---

## 28. Quick Reference — Common ESP32 Pinouts & Defaults

| Peripheral | Default Pins (ESP32) |
|---|---|
| I2C | SDA=21, SCL=22 |
| SPI (VSPI) | SCK=18, MISO=19, MOSI=23, CS=5 |
| UART0 (debug) | RX=3, TX=1 (don't use — wired to USB) |
| UART2 | RX=16, TX=17 (common free choice) |
| ADC1 (safe with WiFi) | GPIO32–39 |
| Built-in LED | GPIO2 (board-dependent) |
| Common baud rates | 9600 (GPS/old modules), 115200 (debug/USB) |

---

## 29. PC-Side: Quick MQTT Test Script (Python, paho-mqtt)

Useful if the test asks you to write a script that publishes/subscribes to verify your device — common for "simulate the cloud side" tasks.

```python
import paho.mqtt.client as mqtt
import json
import time

BROKER = "broker.example.com"
PORT = 1883
TOPIC_SUB = "sensors/esp32-01/data"
TOPIC_PUB = "actuators/esp32-01/relay"

def on_connect(client, userdata, flags, rc, properties=None):
    print(f"Connected with result code {rc}")
    client.subscribe(TOPIC_SUB)

def on_message(client, userdata, msg):
    try:
        payload = json.loads(msg.payload.decode())
        print(f"[{msg.topic}] {payload}")
    except json.JSONDecodeError:
        print(f"[{msg.topic}] {msg.payload.decode()} (not JSON)")

client = mqtt.Client(callback_api_version=mqtt.CallbackAPIVersion.VERSION2)
client.on_connect = on_connect
client.on_message = on_message
client.connect(BROKER, PORT, keepalive=60)

client.publish(TOPIC_PUB, "ON")
client.loop_forever()
```

---

## Quick Notes (not code, but worth remembering)

- **Blocking vs non-blocking:** Avoid `delay()` in production loops if anything else needs to run concurrently (WiFi, MQTT keepalive, button reads). Use `millis()`-based timing instead.
- **`mqttClient.loop()` must be called frequently** or the connection drops on keepalive timeout.
- **ESP32 ADC1 pins (32-39)** are safe to use alongside WiFi; **ADC2 pins** can conflict with WiFi and may return -1 when WiFi is active.
- **`IRAM_ATTR`** is required on ISR functions on ESP32 so the interrupt code stays in fast internal RAM.
- Know the difference between **I2C** (2-wire, addressable, slower, good for many sensors), **SPI** (4-wire, faster, no addressing, one CS per device), and **UART** (point-to-point, no addressing).
- For DAQ-style questions: be ready to explain **sampling rate vs Nyquist**, and basic stats (mean, std dev, min/max) on a sample set — this maps directly to section 25 above.
