/*
 * Solar-Powered IoT Weather Monitoring System
 * 
 * A sustainable weather monitoring solution using Arduino, multiple sensors,
 * and solar power for autonomous environmental data collection.
 * 
 * Authors: Vinay Kokkisa, Shashivadhan Cheepiri, Gadipelli Vishnu Vardhan,
 *          Bashaboina Rahul Thrinethra, Kailasakoti Rahul
 * Institution: SR University, Warangal, India
 * 
 * Hardware Components:
 * - Arduino Uno/ESP32
 * - DHT11/DHT22 Temperature & Humidity Sensor
 * - LDR (Light Dependent Resistor)
 * - MQ-135 Gas Sensor
 * - Capacitive Touch Sensor
 * - 16x2 LCD Display
 * - Solar Panel (5V, 3W)
 * - Li-ion Battery (3.7V, 2000mAh)
 */

#include <LiquidCrystal.h>
#include <DHT.h>
#include <LowPower.h>

// Pin Definitions
#define DHT_PIN 2              // DHT11/DHT22 sensor pin
#define LDR_PIN A0             // Light sensor pin
#define GAS_SENSOR_PIN A1      // MQ-135 gas sensor pin
#define TOUCH_SENSOR_PIN 3     // Capacitive touch sensor pin
#define BATTERY_VOLTAGE_PIN A2 // Battery voltage monitoring pin
#define SOLAR_VOLTAGE_PIN A3   // Solar panel voltage monitoring pin

// DHT Sensor Configuration
#define DHT_TYPE DHT11         // Change to DHT22 if using DHT22
DHT dht(DHT_PIN, DHT_TYPE);

// LCD Configuration (RS, Enable, D4, D5, D6, D7)
LiquidCrystal lcd(12, 11, 5, 4, 7, 8);

// System Configuration
#define SLEEP_INTERVAL 8       // Sleep cycles (8s each, total ~64s between readings)
#define DISPLAY_MODES 4        // Number of display modes
#define DISPLAY_TIME 3000      // Time to show each reading (ms)
#define BATTERY_LOW_THRESHOLD 3.2  // Low battery voltage threshold
#define GAS_CALIBRATION_TIME 20000 // Gas sensor warm-up time (ms)

// Global Variables
volatile bool touchPressed = false;
int currentDisplayMode = 0;
unsigned long lastDisplayUpdate = 0;
unsigned long lastSensorRead = 0;
bool gasCalibrated = false;
float baselineGas = 0;

// Sensor Data Structure
struct SensorData {
  float temperature;
  float humidity;
  int lightLevel;
  int gasLevel;
  float batteryVoltage;
  float solarVoltage;
  bool systemHealthy;
};

SensorData currentData;

// Custom LCD Characters
byte batteryChar[8] = {
  B01110,
  B11111,
  B11111,
  B11111,
  B11111,
  B11111,
  B11111,
  B01110
};

byte solarChar[8] = {
  B00100,
  B10101,
  B01110,
  B11111,
  B01110,
  B10101,
  B00100,
  B00000
};

byte degreeChar[8] = {
  B01100,
  B10010,
  B10010,
  B01100,
  B00000,
  B00000,
  B00000,
  B00000
};

byte dropChar[8] = {
  B00100,
  B00100,
  B01010,
  B01010,
  B10001,
  B10001,
  B10001,
  B01110
};

void setup() {
  Serial.begin(9600);
  Serial.println(F("Solar Weather Monitoring System Starting..."));
  
  // Initialize LCD
  lcd.begin(16, 2);
  lcd.createChar(0, batteryChar);
  lcd.createChar(1, solarChar);
  lcd.createChar(2, degreeChar);
  lcd.createChar(3, dropChar);
  
  // Initialize sensors
  dht.begin();
  
  // Configure pins
  pinMode(TOUCH_SENSOR_PIN, INPUT_PULLUP);
  pinMode(LDR_PIN, INPUT);
  pinMode(GAS_SENSOR_PIN, INPUT);
  pinMode(BATTERY_VOLTAGE_PIN, INPUT);
  pinMode(SOLAR_VOLTAGE_PIN, INPUT);
  
  // Setup touch interrupt
  attachInterrupt(digitalPinToInterrupt(TOUCH_SENSOR_PIN), touchInterrupt, FALLING);
  
  // Display startup message
  displayStartupMessage();
  
  // Calibrate gas sensor
  calibrateGasSensor();
  
  Serial.println(F("System initialized successfully!"));
}

void loop() {
  // Check if it's time to read sensors
  if (millis() - lastSensorRead >= 60000 || lastSensorRead == 0) {
    readAllSensors();
    lastSensorRead = millis();
  }
  
  // Handle display updates
  handleDisplay();
  
  // Check system health
  checkSystemHealth();
  
  // Handle touch input
  if (touchPressed) {
    handleTouchInput();
    touchPressed = false;
  }
  
  // Enter sleep mode for power conservation
  if (millis() - lastDisplayUpdate > DISPLAY_TIME && !touchPressed) {
    enterSleepMode();
  }
  
  delay(100); // Small delay to prevent excessive processing
}

void readAllSensors() {
  Serial.println(F("Reading sensors..."));
  
  // Read temperature and humidity
  currentData.temperature = dht.readTemperature();
  currentData.humidity = dht.readHumidity();
  
  // Check for DHT reading errors
  if (isnan(currentData.temperature) || isnan(currentData.humidity)) {
    Serial.println(F("Failed to read from DHT sensor!"));
    currentData.temperature = -999;
    currentData.humidity = -999;
  }
  
  // Read light level (convert to percentage)
  int rawLight = analogRead(LDR_PIN);
  currentData.lightLevel = map(rawLight, 0, 1023, 0, 100);
  
  // Read gas level
  int rawGas = analogRead(GAS_SENSOR_PIN);
  if (gasCalibrated) {
    currentData.gasLevel = rawGas - baselineGas;
    if (currentData.gasLevel < 0) currentData.gasLevel = 0;
  } else {
    currentData.gasLevel = rawGas;
  }
  
  // Read battery voltage
  int rawBattery = analogRead(BATTERY_VOLTAGE_PIN);
  currentData.batteryVoltage = (rawBattery * 5.0) / 1023.0 * 2; // Voltage divider adjustment
  
  // Read solar panel voltage
  int rawSolar = analogRead(SOLAR_VOLTAGE_PIN);
  currentData.solarVoltage = (rawSolar * 5.0) / 1023.0 * 2;
  
  // Print to Serial for debugging
  printSensorData();
}

void printSensorData() {
  Serial.println(F("=== Sensor Readings ==="));
  Serial.print(F("Temperature: "));
  Serial.print(currentData.temperature);
  Serial.println(F("°C"));
  Serial.print(F("Humidity: "));
  Serial.print(currentData.humidity);
  Serial.println(F("%"));
  Serial.print(F("Light Level: "));
  Serial.print(currentData.lightLevel);
  Serial.println(F("%"));
  Serial.print(F("Gas Level: "));
  Serial.println(currentData.gasLevel);
  Serial.print(F("Battery Voltage: "));
  Serial.print(currentData.batteryVoltage);
  Serial.println(F("V"));
  Serial.print(F("Solar Voltage: "));
  Serial.print(currentData.solarVoltage);
  Serial.println(F("V"));
  Serial.println(F("====================="));
}

void displayStartupMessage() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("Solar Weather"));
  lcd.setCursor(0, 1);
  lcd.print(F("Monitor v1.0"));
  delay(2000);
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("Initializing..."));
  delay(1000);
  
  // Show system status
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("DHT Sensor: OK"));
  lcd.setCursor(0, 1);
  lcd.print(F("LCD: Ready"));
  delay(1500);
  
  lastDisplayUpdate = millis();
}

void calibrateGasSensor() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("Calibrating Gas"));
  lcd.setCursor(0, 1);
  lcd.print(F("Sensor..."));
  
  Serial.println(F("Calibrating MQ-135 gas sensor..."));
  Serial.println(F("Please ensure clean air environment"));
  
  // Warm up period
  unsigned long startTime = millis();
  float sum = 0;
  int samples = 0;
  
  while (millis() - startTime < GAS_CALIBRATION_TIME) {
    int gasReading = analogRead(GAS_SENSOR_PIN);
    sum += gasReading;
    samples++;
    
    // Update progress on LCD
    int progress = map(millis() - startTime, 0, GAS_CALIBRATION_TIME, 0, 16);
    lcd.setCursor(0, 1);
    for (int i = 0; i < progress; i++) {
      lcd.print(F("="));
    }
    
    delay(100);
  }
  
  baselineGas = sum / samples;
  gasCalibrated = true;
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("Gas Calibrated"));
  lcd.setCursor(0, 1);
  lcd.print(F("Baseline: "));
  lcd.print((int)baselineGas);
  delay(2000);
  
  Serial.print(F("Gas sensor calibrated. Baseline: "));
  Serial.println(baselineGas);
}

void handleDisplay() {
  if (millis() - lastDisplayUpdate >= DISPLAY_TIME) {
    updateDisplay();
    lastDisplayUpdate = millis();
  }
}

void updateDisplay() {
  lcd.clear();
  
  switch (currentDisplayMode) {
    case 0: // Temperature and Humidity
      displayTemperatureHumidity();
      break;
    case 1: // Light and Gas levels
      displayLightGas();
      break;
    case 2: // Battery and Solar status
      displayPowerStatus();
      break;
    case 3: // System overview
      displaySystemOverview();
      break;
  }
  
  // Auto-advance display mode
  currentDisplayMode = (currentDisplayMode + 1) % DISPLAY_MODES;
}

void displayTemperatureHumidity() {
  lcd.setCursor(0, 0);
  lcd.print(F("Temp: "));
  if (currentData.temperature != -999) {
    lcd.print(currentData.temperature, 1);
    lcd.write(2); // Degree symbol
    lcd.print(F("C"));
  } else {
    lcd.print(F("Error"));
  }
  
  lcd.setCursor(0, 1);
  lcd.print(F("Humidity: "));
  if (currentData.humidity != -999) {
    lcd.print(currentData.humidity, 0);
    lcd.write(3); // Drop symbol
  } else {
    lcd.print(F("Err"));
  }
}

void displayLightGas() {
  lcd.setCursor(0, 0);
  lcd.print(F("Light: "));
  lcd.print(currentData.lightLevel);
  lcd.print(F("%"));
  
  lcd.setCursor(0, 1);
  lcd.print(F("Air Quality: "));
  if (currentData.gasLevel < 50) {
    lcd.print(F("Good"));
  } else if (currentData.gasLevel < 100) {
    lcd.print(F("Fair"));
  } else {
    lcd.print(F("Poor"));
  }
}

void displayPowerStatus() {
  lcd.setCursor(0, 0);
  lcd.write(0); // Battery symbol
  lcd.print(F("Batt: "));
  lcd.print(currentData.batteryVoltage, 1);
  lcd.print(F("V"));
  
  lcd.setCursor(0, 1);
  lcd.write(1); // Solar symbol
  lcd.print(F("Solar: "));
  lcd.print(currentData.solarVoltage, 1);
  lcd.print(F("V"));
}

void displaySystemOverview() {
  lcd.setCursor(0, 0);
  lcd.print(F("System: "));
  if (currentData.systemHealthy) {
    lcd.print(F("OK"));
  } else {
    lcd.print(F("WARN"));
  }
  
  lcd.setCursor(9, 0);
  lcd.print(millis() / 1000 / 60); // Uptime in minutes
  lcd.print(F("min"));
  
  lcd.setCursor(0, 1);
  lcd.print(F("Touch to cycle"));
}

void checkSystemHealth() {
  currentData.systemHealthy = true;
  
  // Check battery voltage
  if (currentData.batteryVoltage < BATTERY_LOW_THRESHOLD) {
    currentData.systemHealthy = false;
    Serial.println(F("WARNING: Low battery voltage!"));
  }
  
  // Check sensor readings
  if (currentData.temperature == -999 || currentData.humidity == -999) {
    currentData.systemHealthy = false;
    Serial.println(F("WARNING: DHT sensor error!"));
  }
  
  // Check if charging
  if (currentData.solarVoltage > currentData.batteryVoltage + 0.5) {
    Serial.println(F("INFO: Battery charging"));
  } else {
    Serial.println(F("INFO: Battery not charging"));
  }
}

void handleTouchInput() {
  Serial.println(F("Touch detected!"));
  
  // Cycle through display modes manually
  currentDisplayMode = (currentDisplayMode + 1) % DISPLAY_MODES;
  updateDisplay();
  lastDisplayUpdate = millis();
  
  // Brief feedback
  lcd.setCursor(15, 1);
  lcd.print(F("*"));
  delay(200);
  
  // Reset sleep timer
  lastDisplayUpdate = millis();
}

void touchInterrupt() {
  touchPressed = true;
}

void enterSleepMode() {
  Serial.println(F("Entering sleep mode..."));
  
  // Turn off LCD backlight if available
  // Note: This depends on your LCD module
  
  // Display sleep message
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("Sleep Mode"));
  lcd.setCursor(0, 1);
  lcd.print(F("Touch to wake"));
  
  delay(1000);
  lcd.noDisplay(); // Turn off display
  
  // Sleep for multiple cycles to save power
  for (int i = 0; i < SLEEP_INTERVAL; i++) {
    LowPower.powerDown(SLEEP_8S, ADC_OFF, BOD_OFF);
    
    // Check if woken by touch
    if (touchPressed) {
      break;
    }
  }
  
  // Wake up
  lcd.display(); // Turn on display
  Serial.println(F("Waking up from sleep..."));
  
  // Force sensor reading after wake up
  lastSensorRead = 0;
  lastDisplayUpdate = millis();
}

// Additional utility functions

void displayError(String errorMsg) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("ERROR:"));
  lcd.setCursor(0, 1);
  lcd.print(errorMsg);
  delay(3000);
}

void displayAlert(String alertMsg) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("ALERT:"));
  lcd.setCursor(0, 1);
  lcd.print(alertMsg);
  delay(2000);
}

// Data logging function (if SD card is added later)
void logDataToSerial() {
  Serial.print(millis());
  Serial.print(F(","));
  Serial.print(currentData.temperature);
  Serial.print(F(","));
  Serial.print(currentData.humidity);
  Serial.print(F(","));
  Serial.print(currentData.lightLevel);
  Serial.print(F(","));
  Serial.print(currentData.gasLevel);
  Serial.print(F(","));
  Serial.print(currentData.batteryVoltage);
  Serial.print(F(","));
  Serial.println(currentData.solarVoltage);
}

// Function to get air quality description
String getAirQualityDescription(int gasLevel) {
  if (gasLevel < 30) return F("Excellent");
  else if (gasLevel < 60) return F("Good");
  else if (gasLevel < 90) return F("Fair");
  else if (gasLevel < 120) return F("Poor");
  else return F("Hazardous");
}

// Function to estimate battery percentage
int getBatteryPercentage(float voltage) {
  if (voltage >= 4.1) return 100;
  else if (voltage >= 3.9) return 75;
  else if (voltage >= 3.7) return 50;
  else if (voltage >= 3.5) return 25;
  else return 0;
}
