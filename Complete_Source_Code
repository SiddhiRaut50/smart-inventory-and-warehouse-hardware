#include <SPI.h>
#include <MFRC522.h>
#include "HX711.h"

// ========================================
// SMART INVENTORY AND WAREHOUSE SYSTEM
// ESP32 + RFID RC522 + LOAD CELL + HX711
// ========================================

// RFID pins
#define RFID_SS_PIN   5
#define RFID_RST_PIN  27

// RFID SPI pins
#define RFID_SCK_PIN  18
#define RFID_MISO_PIN 19
#define RFID_MOSI_PIN 23

// HX711 pins
#define HX711_DOUT_PIN 21
#define HX711_SCK_PIN  25

// Create objects
MFRC522 rfid(RFID_SS_PIN, RFID_RST_PIN);
HX711 scale;

// ========================================
// LOAD CELL SETTINGS
// ========================================

float calibrationFactor = 1.0;

bool loadCellCalibrated = false;

// ========================================
// STOCK MOVEMENT SETTINGS
// ========================================

enum MovementType {
  STOCK_IN,
  STOCK_OUT
};

MovementType selectedMovement = STOCK_IN;

String lastUID = "";
unsigned long lastScanTime = 0;
const unsigned long scanDelay = 2000;


unsigned long lastWeightTime = 0;
const unsigned long weightInterval = 1000;


float previousWeight = 0.0;
bool firstWeightReading = true;

// ========================================
// SETUP
// ========================================

void setup() {

  Serial.begin(115200);
  delay(1000);

  Serial.println();
  Serial.println("====================================");
  Serial.println(" SMART INVENTORY SYSTEM");
  Serial.println(" Hardware Initialization");
  Serial.println("====================================");

  // Initialize RFID
  SPI.begin(
    RFID_SCK_PIN,
    RFID_MISO_PIN,
    RFID_MOSI_PIN,
    RFID_SS_PIN
  );

  rfid.PCD_Init();
  delay(100);

  Serial.println("RFID RC522 initialized.");
  rfid.PCD_DumpVersionToSerial();

  // Initialize load cell
  scale.begin(HX711_DOUT_PIN, HX711_SCK_PIN);

  if (scale.is_ready()) {
    Serial.println("HX711 detected.");
  } else {
    Serial.println("WARNING: HX711 not ready.");
    Serial.println("Check DT, SCK, VCC and GND.");
  }

  Serial.println();
  Serial.println("Movement selection:");
  Serial.println("Type IN for incoming stock.");
  Serial.println("Type OUT for outgoing stock.");
  Serial.println("Place an RFID card near the reader.");
  Serial.println("====================================");
}

// ========================================
// MAIN LOOP
// ========================================

void loop() {

  readSerialCommands();
  readRFIDCard();
  monitorLoadCell();
}

// ========================================
// SELECT STOCK MOVEMENT
// ========================================

void readSerialCommands() {

  if (Serial.available()) {

    String command = Serial.readStringUntil('\n');
    command.trim();
    command.toUpperCase();

    if (command == "IN") {

      selectedMovement = STOCK_IN;
      Serial.println("Movement selected: STOCK IN");

    } else if (command == "OUT") {

      selectedMovement = STOCK_OUT;
      Serial.println("Movement selected: STOCK OUT");

    } else {

      Serial.println("Invalid command. Type IN or OUT.");
    }
  }
}

// ========================================
// RFID PRODUCT IDENTIFICATION
// ========================================

void readRFIDCard() {

  if (!rfid.PICC_IsNewCardPresent()) {
    return;
  }

  if (!rfid.PICC_ReadCardSerial()) {
    return;
  }

  String currentUID = "";

  for (byte i = 0; i < rfid.uid.size; i++) {

    if (rfid.uid.uidByte[i] < 0x10) {
      currentUID += "0";
    }

    currentUID += String(rfid.uid.uidByte[i], HEX);

    if (i < rfid.uid.size - 1) {
      currentUID += ":";
    }
  }

  currentUID.toUpperCase();

  unsigned long currentTime = millis();

  // Ignore a repeated scan during the delay
  if (currentUID != lastUID ||
      currentTime - lastScanTime >= scanDelay) {

    Serial.println();
    Serial.println("========== STOCK MOVEMENT ==========");

    Serial.print("Product RFID UID: ");
    Serial.println(currentUID);

    if (selectedMovement == STOCK_IN) {
      Serial.println("Movement: STOCK IN");
    } else {
      Serial.println("Movement: STOCK OUT");
    }

    Serial.print("Time since startup (ms): ");
    Serial.println(currentTime);

    if (scale.is_ready()) {

      long rawReading = scale.read_average(5);

      Serial.print("Load Cell Raw Reading: ");
      Serial.println(rawReading);

      if (loadCellCalibrated) {

        float weight = scale.get_units(5);

        Serial.print("Measured Weight: ");
        Serial.print(weight, 2);
        Serial.println(" calibrated units");

      } else {

        Serial.println(
          "Weight not calibrated; raw reading only."
        );
      }

    } else {

      Serial.println("Weight unavailable: HX711 not ready.");
    }

    Serial.println("====================================");

    lastUID = currentUID;
    lastScanTime = currentTime;
  }

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}

// ========================================
// LOAD CELL STOCK MONITORING
// ========================================

void monitorLoadCell() {

  if (millis() - lastWeightTime < weightInterval) {
    return;
  }

  lastWeightTime = millis();

  if (!scale.is_ready()) {
    Serial.println("HX711 not ready.");
    return;
  }

  long rawReading = scale.read_average(5);

  Serial.print("Load Cell Raw Reading: ");
  Serial.println(rawReading);

  if (loadCellCalibrated) {

    float currentWeight = scale.get_units(5);

    Serial.print("Current Weight: ");
    Serial.println(currentWeight, 2);

    if (!firstWeightReading) {

      float difference = currentWeight - previousWeight;

      if (difference > 0.5) {
        Serial.println("Weight increased.");
      } else if (difference < -0.5) {
        Serial.println("Weight decreased.");
      }
    }

    previousWeight = currentWeight;
    firstWeightReading = false;

  } else {

    Serial.println(
      "Calibrate load cell to obtain actual weight."
    );
  }
}
