# IOT---Based-Smart-Street-Light-System
Using ESP32 we designed an automated street light
Hackathon Project Report: IoT-based Smart Street Light System
Track: Internet of Things (IoT) & Hardware
Team ID: — T053
Team Members: —4
● Member Name : Nikhil R Gunagi
                                        Rahul L B
                                        Dhyan S
                                        Ananya N
Repository Link: 
 https://github.com/DhyanSzzzzmachh28/IOT---Based-Smart-Street-Light-System
Demo Link: 
https://jumpshare.com/share/cxJWxBkjp6gyNkMfBlH2
________________________________________
1. Project Abstract
The IoT-based Smart Street Light System automates street lighting based on real-time ambient brightness using an ESP32 microcontroller and an LDR sensor.
 When the environment becomes dark, the system turns the street light ON; during daylight, it turns it OFF automatically. 
The ESP32’s Wi-Fi capability also enables IoT-based monitoring, significantly reducing energy wastage and manual operation.
________________________________________
2. Problem Statement
Inefficient street lighting leads to significant energy wastage and increased operational costs. 
This project presents an IoT-based Smart Street Light System designed to automate lighting control based on ambient environmental conditions. 
Using an ESP32 microcontroller interfaced with an LDR (Light Dependent Resistor), the system continuously monitors sunlight intensity. 
When natural light falls below a specific threshold, the ESP32 automatically activates the LED street light; conversely, it switches the light off during the day. 
The system also leverages IoT connectivity to transmit status data for remote monitoring. 
This solution effectively reduces power consumption and eliminates the need for manual operation.
________________________________________
3. Solution Overview
The system uses an ESP32 connected to an LDR sensor to measure light intensity. 
Based on the readings, it switches the LED street light ON or OFF through GPIO23.
 The ESP32 continuously monitors the brightness via its ADC pin (GPIO34). 
If light falls below a set threshold, the LED turns ON; if brightness rises, it turns OFF.
Below is the Arduino IDE code used to implement the smart street light logic:
________________________________________
⭐ Code for IoT Smart Street Light System (ESP32 + LDR + LED)
/*
  ESP32 – LDR + Ultrasonic motion street light
  + Web UI (HTML) for Manual ON/OFF and Motion Arm/Disarm
  + Telegram Bot notifications using UniversalTelegramBot
  - Sends a Telegram message ONLY when the ultrasonic sensor newly detects an object/human
  - Timing tuned for faster reaction (polling & ultrasonic timeouts reduced)
*/

#include <WiFi.h>
#include <WebServer.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>

// ----------- WIFI (EDIT) -----------
const char* WIFI_SSID = "iPhone";
const char* WIFI_PASS = "123456789";

// ----------- TELEGRAM (EDIT) -----------
const char* TELEGRAM_BOT_TOKEN = "8341484904:AAEmdR6pTisTzsAgcUAWAc2A88ISnYoF7IQ";
const char* TELEGRAM_CHAT_ID = "1912799044";

// ----------- WEB SERVER -----------
WebServer server(80);

// ----------- PINS -----------
const int LDR_DIG_PIN = 4;      // digital LDR OUT
const int LED_PIN     = 5;      // LED / relay output
const int US_TRIG_PIN = 18;
const int US_ECHO_PIN = 19;

// ----------- SETTINGS ----------
// Note: tuned timing values for faster reaction (see comments above)
const bool LDR_ACTIVE_LOW = true;
const bool USE_LDR_PULLUP = false;

const unsigned int MOTION_DISTANCE_CM = 120;
const unsigned long MOTION_KEEP_MS = 30000;

// Reduced max echo wait from 30ms to 15ms to avoid long blocking waits
const unsigned long US_MAX_ECHO_TIME = 15000UL; // microseconds

// Poll faster: 50ms between sensor samples (was 100ms)
const unsigned long SAMPLE_INTERVAL = 50;

// Faster LDR response: require only 1 stable sample (was 2)
const int STABLE_COUNT_REQUIRED = 1;

// Shorter no-motion debounce so system recovers faster (was 3)
const int NO_MOTION_STABLE_REQUIRED = 2;

// Use single ultrasonic sample (was 3) for speed — increase if noisy
const int US_MEDIAN_SAMPLES = 1;
const unsigned long US_SAMPLE_SPACING_MS = 2; // small spacing (ms)

// ----------- STATE ----------
bool ledState = false;
bool manualOverride = false;
bool motionArmed = true;

int stableCount = 0;
int lastLdrSample = -1;
unsigned long lastSampleMillis = 0;
unsigned long lastMotionMillis = 0;
int noMotionCount = 0;
long lastDistanceCm = -1;

// object presence flag — used to send one notification per appearance
bool objectPresent = false;

// secure client and UniversalTelegramBot instance
WiFiClientSecure secureClient;
UniversalTelegramBot bot(TELEGRAM_BOT_TOKEN, secureClient);

// forward declaration
void sendTelegram(const String &text);

// ----------- HELPERS ----------
void applyLed(bool on) {
  if (ledState != on) {
    ledState = on;
    digitalWrite(LED_PIN, on ? HIGH : LOW);
    Serial.printf("[LED] %s\n", on ? "ON" : "OFF");
  }
}

long measureDistanceOnce() {
  // trigger pulse
  digitalWrite(US_TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(US_TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(US_TRIG_PIN, LOW);

  // wait for echo but with reduced timeout to avoid long blocking
  unsigned long duration = pulseIn(US_ECHO_PIN, HIGH, US_MAX_ECHO_TIME);
  if (duration == 0) return -1;
  return (long)(duration / 58.0);
}

long measureDistanceMedian(int samples = US_MEDIAN_SAMPLES) {
  long vals[11];
  int v = 0;

  for (int i = 0; i < samples; i++) {
    vals[v++] = measureDistanceOnce();
    delay(US_SAMPLE_SPACING_MS);
  }

  long good[11];
  int g = 0;

  for (int i = 0; i < v; i++) {
    if (vals[i] >= 0) good[g++] = vals[i];
  }

  if (g == 0) return -1;

  // if only one sample, return it quickly
  if (g == 1) return good[0];

  // sort and pick median (kept for sample>1)
  for (int i = 0; i < g - 1; i++)
    for (int j = i + 1; j < g; j++)
      if (good[j] < good[i]) {
        long t = good[i];
        good[i] = good[j];
        good[j] = t;
      }

  return good[g / 2];
}

// Send simple message to Telegram Bot using UniversalTelegramBot
void sendTelegram(const String &text) {
  if (strlen(TELEGRAM_BOT_TOKEN) == 0 || strlen(TELEGRAM_CHAT_ID) == 0) {
    Serial.println("[TELEGRAM] Token or Chat ID not set!");
    return;
  }

  Serial.print("[TELEGRAM] Sending: ");
  Serial.println(text);

  secureClient.setInsecure(); // simpler for many ESP32 setups
  bool ok = bot.sendMessage(String(TELEGRAM_CHAT_ID), text, "");
  if (ok) {
    Serial.println("[TELEGRAM] Sent");
  } else {
    Serial.println("[TELEGRAM] Failed to send");
  }
}

// ----------- WEB HANDLER WITH NEW UI ----------
void handleRoot() {
  String page = R"rawliteral(
<!doctype html>
<html>
<head>
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>ESP32 Smart Street Light</title>
<style>
body { margin:0; font-family: 'Segoe UI', Roboto, Arial; background: linear-gradient(135deg,#121212,#1f1f1f); color:#fff; text-align:center; padding:18px; }
h2{margin-top:10px;font-size:28px;letter-spacing:1.2px;text-shadow:0 0 8px #00d4ff;color:#00eaff}
.card{background:rgba(255,255,255,0.06);border-radius:18px;backdrop-filter:blur(8px);padding:18px;margin-top:20px;border:1px solid rgba(255,255,255,0.12);box-shadow:0 4px 14px rgba(0,0,0,0.5)}
button{background:linear-gradient(135deg,#00d4ff,#0066ff);border:none;padding:13px 20px;margin:10px;border-radius:12px;font-size:16px;color:white;font-weight:bold;cursor:pointer;transition:0.25s;box-shadow:0 4px 10px rgba(0,170,255,0.4)}
button:hover{transform:scale(1.07);box-shadow:0 6px 18px rgba(0,170,255,0.7)}
.stat-item{font-size:19px;margin:10px 0}.stat-value{font-weight:bold;color:#00eaff}footer{margin-top:22px;font-size:13px;opacity:0.6}
</style>
</head>
<body>
<h2>ESP32 Smart Street Light</h2>
<div class="card"><h3>Manual Control</h3><button onclick="fetch('/ledon').then(update)">Turn ON</button><button onclick="fetch('/ledoff').then(update)">Turn OFF</button></div>
<div class="card"><h3>Motion Control</h3><button onclick="fetch('/motionon').then(update)">Enable Motion</button><button onclick="fetch('/motionoff').then(update)">Disable Motion</button></div>
<div class="card"><h3>Status</h3>
<p class="stat-item">LED State: <span id="led" class="stat-value">-</span></p>
<p class="stat-item">Motion Armed: <span id="motion" class="stat-value">-</span></p>
<p class="stat-item">Manual Override: <span id="manual" class="stat-value">-</span></p>
<p class="stat-item">Ambient: <span id="ldr" class="stat-value">-</span></p>
<p class="stat-item">Distance (cm): <span id="dist" class="stat-value">-</span></p>
</div>
<footer>ESP32 Smart Light UI • Powered by WebServer</footer>
<script>
function update(){
  fetch('/status')
    .then(r => r.json())
    .then(j => {
      document.getElementById('led').innerText    = j.led ? "ON" : "OFF";
      document.getElementById('motion').innerText = j.motionArmed ? "ENABLED" : "DISABLED";
      document.getElementById('manual').innerText = j.manualOverride ? "YES" : "NO";
      document.getElementById('ldr').innerText    = j.isDark ? "DARK" : "LIGHT";
      document.getElementById('dist').innerText   = (j.distanceCm >= 0) ? j.distanceCm : "-";
    });
}
setInterval(update, 1000);
window.onload = update;
</script>
</body>
</html>
  )rawliteral";

  server.send(200, "text/html", page);
}

void handleStatus() {
  int raw = digitalRead(LDR_DIG_PIN);
  bool isDark = LDR_ACTIVE_LOW ? (raw == LOW) : (raw == HIGH);

  String json = "{";
  json += "\"led\":"; json += (ledState ? "1" : "0"); json += ",";
  json += "\"manualOverride\":"; json += (manualOverride ? "true" : "false"); json += ",";
  json += "\"motionArmed\":"; json += (motionArmed ? "true" : "false"); json += ",";
  json += "\"isDark\":"; json += (isDark ? "true" : "false"); json += ",";
  json += "\"distanceCm\":"; json += lastDistanceCm;
  json += "}";

  server.send(200, "application/json", json);
}

void handleLedOn() {
  applyLed(true);
  manualOverride = true;
  server.send(200, "text/plain", "OK");
  // no Telegram notification for manual
}
void handleLedOff() {
  applyLed(false);
  manualOverride = true;
  server.send(200, "text/plain", "OK");
  // no Telegram notification for manual
}
void handleMotionOn() {
  motionArmed = true;
  manualOverride = false;
  server.send(200, "text/plain", "OK");
  // no notification here
}
void handleMotionOff() {
  motionArmed = false;
  manualOverride = false;
  server.send(200, "text/plain", "OK");
  // no notification here
}

// ----------- WIFI ----------
void setupWiFi() {
  Serial.print("Connecting to WiFi");
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(100);
    Serial.print(".");
  }
  Serial.println("\nConnected! IP Address:");
  Serial.println(WiFi.localIP());
}

// ----------- SETUP ----------
void setup() {
  Serial.begin(115200);
  delay(10);

  pinMode(LED_PIN, OUTPUT);
  if (USE_LDR_PULLUP) pinMode(LDR_DIG_PIN, INPUT_PULLUP);
  else pinMode(LDR_DIG_PIN, INPUT);

  pinMode(US_TRIG_PIN, OUTPUT);
  pinMode(US_ECHO_PIN, INPUT);

  applyLed(false);

  setupWiFi();

  server.on("/", handleRoot);
  server.on("/status", handleStatus);
  server.on("/ledon", handleLedOn);
  server.on("/ledoff", handleLedOff);
  server.on("/motionon", handleMotionOn);
  server.on("/motionoff", handleMotionOff);
  server.begin();

  secureClient.setInsecure(); // for many ESP32 setups; replace with setCACert for production
}

// ----------- MAIN LOOP ----------
void loop() {
  server.handleClient();

  unsigned long now = millis();
  if (now - lastSampleMillis >= SAMPLE_INTERVAL) {
    lastSampleMillis = now;

    int raw = digitalRead(LDR_DIG_PIN);
    bool isDark = LDR_ACTIVE_LOW ? (raw == LOW) : (raw == HIGH);
    int sample = isDark ? 0: 1;

    if (sample == lastLdrSample) stableCount++;
    else stableCount = 1;

    lastLdrSample = sample;

    if (stableCount >= STABLE_COUNT_REQUIRED) {
      if (sample == 1) {  // DARK
        if (motionArmed && !manualOverride) {
          long dist = measureDistanceMedian();
          lastDistanceCm = dist;

          // DETECTION: if distance is valid and within threshold
          if (dist > 0 && dist <= MOTION_DISTANCE_CM) {
            // reset no-motion counter
            lastMotionMillis = now;
            noMotionCount = 0;

            // send notification only when object newly appears (debounced by objectPresent)
            if (!objectPresent) {
              objectPresent = true;
              String msg = String("ESP32 StreetLight: Object detected — distance: ") + String(dist) + " cm";
              sendTelegram(msg);
            }

            applyLed(true);
          } else {
            // no detection — increment no-motion debounce counter
            noMotionCount++;
            if (noMotionCount >= NO_MOTION_STABLE_REQUIRED) {
              // object has left; reset flag so next arrival will trigger notification
              if (objectPresent) {
                objectPresent = false;
                Serial.println("[DETECT] Object left (reset flag)");
              }
              applyLed(false);
              lastMotionMillis = 0;
              noMotionCount = 0;
            } else if (lastMotionMillis && (now - lastMotionMillis >= MOTION_KEEP_MS)) {
              // timeout: switch off and reset detection flag
              if (objectPresent) {
                objectPresent = false;
                Serial.println("[DETECT] Object left by timeout (reset flag)");
              }
              applyLed(false);
              lastMotionMillis = 0;
              noMotionCount = 0;
            }
          }
        }
      } else { // LIGHT
        lastDistanceCm = -1;
        if (!manualOverride) applyLed(false);
        noMotionCount = 0;
        lastMotionMillis = 0;
        // also reset objectPresent because environment is bright (no expected detection)
        if (objectPresent) {
          objectPresent = false;
          Serial.println("[DETECT] Reset objectPresent due to light");
        }
      }
    }
  }
}
________________________________________
4. Hardware & Technical Stack
4.1 Components List
•	ESP32 microcontroller
•	LDR sensor
•	1k-ohm resistor
•	220-ohm resistor
•	LED (street light simulation)
•	Micro USB cable
4.2 Communication Protocols
•	Wi-Fi (optional IoT monitoring)
•	ADC (LDR reading)
•	GPIO output (LED control)
________________________________________
5. Circuit / Architecture Diagram
Connections:
•	LDR → GPIO34
•	LDR second leg → 3.3V
•	LDR + 1kΩ resistor → GND (voltage divider)
•	LED anode → GPIO23 via 220Ω resistor
•	LED cathode → GND

________________________________________
6. Prototype & Implementation
•	Assembled on a breadboard using ESP32, LDR, and LED.
•	Programmed using Arduino IDE.
•	ESP32 reads light levels and toggles LED automatically.

________________________________________
7. Challenges & Solutions
Challenge	Solution
LDR values fluctuating	Added threshold and smoothing
LED not switching reliably	Adjusted resistor values
Wi-Fi dropouts (optional IoT)	Added reconnection logic
Light sensitivity too high/low	Tuned ADC threshold

________________________________________
8. Scalability & Commercial Viability
•	Cost per unit is low, making mass deployment feasible.
•	In real street lights, LED can be replaced with a relay driver to control AC lamps.
•	Can integrate solar charging for smart energy saving.
•	IoT features make it suitable for smart city infrastructure.
 
 
                                               **************************************
