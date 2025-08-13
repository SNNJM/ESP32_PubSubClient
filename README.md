# ESP32_PubSubClient
ESP32 publish and subscribe to MQTT (mosquitto) broker

# Solution Architecture 
```
┌─────────────────────────────────────────────────────────────────┐
│                         Boot & Identity                         │
│─────────────────────────────────────────────────────────────────│
│ • Setup() starts, Serial @115200                                │
│ • LED pin mode: GPIO 2 (active LOW), set HIGH (OFF)             │
│ • DEVICE_ID = "ESP32-" + MAC (uppercase hex, 12 chars)          │
│   e.g., ESP32-3C71BF12ABCD                                      │
└─────────────────────────────────────────────────────────────────┘
               ↓
┌─────────────────────────────────────────────────────────────────┐
│                           Wi-Fi Connect                         │
│─────────────────────────────────────────────────────────────────│
│ • ensureWiFi():                                                 │
│   – WiFi.mode(STA), WiFi.begin(WIFI_SSID, WIFI_PASSWORD)        │
│   – Loop until WL_CONNECTED                                     │
│   – Log local IP                                                │
└─────────────────────────────────────────────────────────────────┘
               ↓
┌─────────────────────────────────────────────────────────────────┐
│                           MQTT Connect                          │
│─────────────────────────────────────────────────────────────────│
│ • client.setServer("test.mosquitto.org", 1883)                  │
│ • client.setCallback(mqttCallback)                              │
│ • ensureMQTT():                                                 │
│   – Connect with clientId: DEVICE_ID + "-" + millis()           │
│   – On success: subscribe("smartcity/control/streetlights")     │
│   – On failure: wait 2s and retry                               │
└─────────────────────────────────────────────────────────────────┘
               ↓
┌─────────────────────────────────────────────────────────────────┐
│                          Runtime Loop                           │
│─────────────────────────────────────────────────────────────────│
│ loop():                                                         │
│ • ensure Wi-Fi / MQTT still connected                           │
│ • client.loop()                                                 │
│ • Every ~2000 ms:                                               │
│   – Build JSON: {                                               │
│       "id": DEVICE_ID,                                          │
│       "temp": 25.0 + rand()/10.0 (fake),                        │
│       "ts": millis()                                            │
│     }                                                           │
│   – Publish to "smartcity/env"                                  │
└─────────────────────────────────────────────────────────────────┘
               ↓                       ↑
      (telemetry out)                  │ (control in via callback)
┌─────────────────────────────────────────────────────────────────┐
│                        MQTT Message Path                        │
│─────────────────────────────────────────────────────────────────│
│ mqttCallback(topic, payload):                                   │
│ • Log: [MQTT] <topic> => <payload>                              │
│ • If payload starts with '{' → JSON control:                    │
│     handleJsonCommand(json):                                    │
│       – Parse with ArduinoJson                                  │
│       – Require fields: "id", "cmd"                             │
│       – If id == DEVICE_ID or id == "ALL":                      │
│           controlLED(cmd)  // ON→LOW, OFF→HIGH                 │
│           publishAck(cmd,"OK")                                  │
│         else:                                                   │
│           publishAck(cmd,"IGNORED","ID not matched")            │
│       – On bad JSON / missing fields:                           │
│           publishAck("UNKNOWN","ERROR",reason)                  │
│ • Else (raw text "ON"/"OFF"):                                   │
│     controlLED(payload), publishAck(payload,"OK")               │
└─────────────────────────────────────────────────────────────────┘
               ↓
┌─────────────────────────────────────────────────────────────────┐
│                          Acknowledgment                         │
│─────────────────────────────────────────────────────────────────│
│ publishAck(cmd,status,reason?):                                 │
│ • Build JSON: { "id": DEVICE_ID,                                │
│                 "cmd": <cmd>,                                   │
│                 "status": "OK" | "IGNORED" | "ERROR",           │
│                 "reason": <optional> }                          │
│ • Publish to "smartcity/ack/streetlights"                       │
└─────────────────────────────────────────────────────────────────┘
               ↓
┌─────────────────────────────────────────────────────────────────┐
│                         Libraries & Pins                        │
│─────────────────────────────────────────────────────────────────│
│ • WiFi.h, PubSubClient.h, ArduinoJson.h                         │
│ • LED_PIN = 2 (ESP32-WROOM onboard LED; ACTIVE LOW)             │
│ • Broker: test.mosquitto.org:1883                               │
│ • Topics:                                                       │
│   – Publish telemetry: "smartcity/env"                          │
│   – Subscribe control: "smartcity/control/streetlights"         │
│   – Publish ack: "smartcity/ack/streetlights"                   │
└─────────────────────────────────────────────────────────────────┘


```
