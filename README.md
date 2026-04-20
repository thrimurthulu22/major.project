# major.project
IoT-based Smart Helmet that detects accidents and alcohol levels, sending real-time SMS alerts with GPS location for enhanced rider safety.
#include <SoftwareSerial.h>
#include <TinyGPS++.h>
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_ADXL345_U.h>

// ----------- ADXL345 -----------
Adafruit_ADXL345_Unified accel = Adafruit_ADXL345_Unified(12345);

// ----------- GPS -----------
static const int gpsRX = 7, gpsTX = 8;
SoftwareSerial gpsPort(gpsRX, gpsTX);
TinyGPSPlus gps;

// ----------- SIM800L -----------
static const int simRX = 2, simTX = 3;
SoftwareSerial sim800l(simRX, simTX);

// ----------- MQ3 DIGITAL -----------
#define MQ3_DIGITAL_PIN 9   // DO pin connected here

// ----------- VARIABLES -----------
float lat = 0.0, lon = 0.0;
int flagImpact = 0;
int flagAlcohol = 0;

unsigned long last;
unsigned long current;
unsigned long elapsed;
int refreshRate = 15;

// ----------- FLOAT TO STRING -----------
String floatToString(float x, byte precision = 6) {
  char buffer[20];
  dtostrf(x, 0, precision, buffer);
  return String(buffer);
}

// ----------- SETUP -----------
void setup() {

  Serial.begin(9600);

  gpsPort.begin(9600);
  sim800l.begin(9600);

  pinMode(MQ3_DIGITAL_PIN, INPUT);

  // ADXL345 init
  if (!accel.begin()) {
    Serial.println("ADXL345 not detected!");
    while (1);
  }

  accel.setRange(ADXL345_RANGE_16_G);

  Serial.println("Warming MQ3...");
 

  Serial.println("Initializing SIM800L...");
  delay(20000);

  Serial.println("System Ready");

  last = millis();
}

// ----------- IMPACT DETECTION -----------
bool impactDetected() {

  sensors_event_t event;
  accel.getEvent(&event);

  float totalAccel = sqrt(
    event.acceleration.x * event.acceleration.x +
    event.acceleration.y * event.acceleration.y +
    event.acceleration.z * event.acceleration.z
  );

  if (totalAccel > 14) {

    delay(100);
    accel.getEvent(&event);

    float confirm = sqrt(
      event.acceleration.x * event.acceleration.x +
      event.acceleration.y * event.acceleration.y +
      event.acceleration.z * event.acceleration.z
    );

    if (confirm > 14) {
      Serial.println("Impact Detected!");
      return true;
    }
  }

  return false;
}

// ----------- ALCOHOL DETECTION (DIGITAL) -----------
bool alcoholDetected() {

  int state = digitalRead(MQ3_DIGITAL_PIN);

  Serial.print("MQ3 Digital State: ");
  Serial.println(state);

  // Most modules: LOW = alcohol detected
  if (state == LOW) {

    Serial.println("Alcohol Detected!");

    delay(2000); // confirm delay

    if (digitalRead(MQ3_DIGITAL_PIN) == LOW) {
      return true;
    }
  }

  return false;
}

// ----------- GPS PROCESS -----------
void processGPSData() {

  while (gpsPort.available() > 0) {
    gps.encode(gpsPort.read());

    if (gps.location.isUpdated()) {
      lat = gps.location.lat();
      lon = gps.location.lng();

      Serial.print("Lat: ");
      Serial.print(lat, 6);
      Serial.print(" Lon: ");
      Serial.println(lon, 6);
    }
  }

  if (!gps.location.isValid()) {
    lat = -1;
    lon = -1;
  }
}

// ----------- LOOP -----------
void loop() {

  current = millis();
  elapsed += current - last;
  last = current;

  processGPSData();

  if (impactDetected()) {
    flagImpact = 1;
  }

  if (alcoholDetected()) {
    flagAlcohol = 1;
  }

  if (elapsed >= (refreshRate * 1000)) {

    if (flagImpact == 1) {
      sendSMS("Accident Detected!");
      flagImpact = 0;
    }

    if (flagAlcohol == 1) {
      sendSMS("Alcohol Detected! Unsafe Driving!");
      flagAlcohol = 0;
    }

    elapsed -= (refreshRate * 1000);
  }
}

// ----------- SEND SMS -----------
void sendSMS(String alertType) {

  sim800l.listen();

  sim800l.println("AT+CMGF=1");
  delay(100);

  sim800l.println("AT+CMGS=\"+918309154070\"");
  delay(100);

  String message = alertType + " Location: http://maps.google.com/maps?q=" +
                   floatToString(lat, 6) + "," +
                   floatToString(lon, 6);

  Serial.println(message);

  sim800l.println(message);
  sim800l.write(26);

  delay(5000);

  Serial.println("SMS Sent");

  gpsPort.listen();
}
