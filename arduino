#include <Wire.h>
#include "MAX30105.h"
#include "heartRate.h"
#include <SimpleTimer.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

#define BLYNK_TEMPLATE_ID "TMPL3TKJBXTDn"
#define BLYNK_TEMPLATE_NAME "oximeter"
#define BLYNK_AUTH_TOKEN "l66lVPJdhHr4IrVQ3zwu9XjxoN-KQMHn"
#include <BlynkSimpleEsp8266.h> // Use BlynkSimpleEsp32.h for ESP32

char ssid[] = "GalaxyA16";     
char pass[] = "Genius68";  

SimpleTimer timer;
MAX30105 particleSensor;

long lastBeat = 0;
float beatsPerMinute = 0.0;
float spO2 = 0.0;

#define BPM_VPIN V1
#define AVG_BPM_VPIN V2
#define IR_VALUE_VPIN V3
#define SPO2_VPIN V4

bool isInhaling = false;
bool isExhaling = false;
long irMovingAverage = 0;
const int smoothingFactor = 20;
const long MIN_IR_VALUE = 5000;
bool fingerDetected = false;

float calculateSpO2Simple(long redValue, long irValue) {
  float ratio = (float)redValue / irValue;
  float spO2 = 110.0 - (25.0 * ratio);
  return constrain(spO2, 0.0, 100.0);
}

void sendToBlynk() {
  if (fingerDetected) {
    Blynk.virtualWrite(BPM_VPIN, beatsPerMinute);
    Blynk.virtualWrite(SPO2_VPIN, spO2);
    Blynk.virtualWrite(IR_VALUE_VPIN, irMovingAverage);
  }
}

void displayOnOLED(float bpm, float spo2) {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);

  display.setCursor(0, 0);
  display.println("Pulse Oximeter");

  display.setTextSize(2);
  display.setCursor(0, 20);
  display.print("BPM: ");
  display.println(bpm, 1);

  display.setCursor(0, 45);
  display.print("SpO2: ");
  display.print(spo2, 1);
  display.println(" %");

  display.display();
}

void setup() {
  Serial.begin(115200);
  Serial.println("Initializing...");

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 allocation failed");
    while (1);
  }

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Connecting WiFi...");
  display.display();

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30105 not found. Check wiring.");
    display.clearDisplay();
    display.setCursor(0, 0);
    display.println("Sensor not found!");
    display.display();
    while (1);
  }

  byte ledBrightness = 0x1F;
  byte sampleAverage = 8;
  byte ledMode = 3;
  int sampleRate = 100;
  int pulseWidth = 411;
  int adcRange = 4096;

  particleSensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange);
  particleSensor.enableDIETEMPRDY();

  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("Sensor Ready!");
  display.display();

  timer.setInterval(1000L, sendToBlynk);
}

void loop() {
  Blynk.run();
  timer.run();

  long irValue = particleSensor.getIR();
  long redValue = particleSensor.getRed();

  fingerDetected = (irValue > MIN_IR_VALUE);

  if (fingerDetected) {
    if (checkForBeat(irValue)) {
      long delta = millis() - lastBeat;
      if (delta > 300) {
        lastBeat = millis();
        float bpm = 60 / (delta / 1000.0);
        if (bpm > 40 && bpm < 180) {
          beatsPerMinute = bpm;
        }
      }
    }

    spO2 = calculateSpO2Simple(redValue, irValue);

    Serial.print("BPM: ");
    Serial.print(beatsPerMinute);
    Serial.print(" | SpO2: ");
    Serial.print(spO2);
    Serial.println(" %");

    displayOnOLED(beatsPerMinute, spO2);

    irMovingAverage = (irMovingAverage * (smoothingFactor - 1) + irValue) / smoothingFactor;

    if (irValue < irMovingAverage - 300) {
      if (!isInhaling) {
        isInhaling = true;
        isExhaling = false;
      }
    } else if (irValue > irMovingAverage + 300) {
      if (!isExhaling) {
        isExhaling = true;
        isInhaling = false;
      }
    }

  } else {
    display.clearDisplay();
    display.setTextSize(2);
    display.setCursor(0, 20);
    display.println("Insert");
    display.setCursor(0, 40);
    display.println("Finger");
    display.display();
  }
}

boolean checkForBeat(long irValue) {
  static long prevIrValue = 0;
  static boolean prevState = false;

  boolean beatDetected = (irValue > prevIrValue + 20) && prevState && (millis() - lastBeat > 300);
  prevIrValue = irValue;
  prevState = (irValue > 5000);

  return beatDetected;
}
