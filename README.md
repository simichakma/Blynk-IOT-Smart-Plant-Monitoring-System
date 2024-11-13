// Blynk IOT Smart Plant Monitoring System

/* Connections
Relay. D3
Btn.   D7
Soil.  A0
PIR.   D5
SDA.   D2
SCL.   D1
Temp.  D4
*/

// Blynk Configuration
#define BLYNK_TEMPLATE_ID "TMPL63qowfz9K"
#define BLYNK_TEMPLATE_NAME "Smart Plant"
#define BLYNK_AUTH_TOKEN "wk1b4HPu9-LEO_wU9u4uAPT5Lm4Wizit"

// Include necessary libraries
#include <LiquidCrystal_I2C.h>
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include <DHT.h>

// Initialize the LCD display
LiquidCrystal_I2C lcd(0x3F, 16, 2); // 16x2 LCD

char auth[] = "wk1b4HPu9-LEO_wU9u4uAPT5Lm4Wizit";  // Enter your Blynk Auth token
char ssid[] = "Noyon";  // Enter your WIFI SSID
char pass[] = "bsr@66+9_66#b";  // Enter your WIFI Password

DHT dht(D4, DHT11); // (DHT sensor pin, sensor type) D4 DHT11 Temperature Sensor
BlynkTimer timer;

// Define component pins
#define soil A0 // A0 Soil Moisture Sensor
#define PIR D5  // D5 PIR Motion Sensor
#define RELAY_PIN_1 D3 // Pin for relay
#define PUSH_BUTTON_1 D7 // Pin for button

int PIR_ToggleValue;
int relay1State = LOW; // Initial state for the relay
int pushButton1State = HIGH; // Initial state for button

void setup() {
  Serial.begin(9600);
  lcd.begin(16, 2); // Initialize LCD with 16 columns and 2 rows
  lcd.backlight();
  pinMode(PIR, INPUT);

  pinMode(RELAY_PIN_1, OUTPUT);
  digitalWrite(RELAY_PIN_1, LOW);
  pinMode(PUSH_BUTTON_1, INPUT_PULLUP);
  digitalWrite(RELAY_PIN_1, relay1State);

  Blynk.begin(auth, ssid, pass, "blynk.cloud", 80);
  dht.begin();

  lcd.setCursor(0, 0);
  lcd.print("  Initializing  ");
  for (int a = 5; a <= 10; a++) {
    lcd.setCursor(a, 1);
    lcd.print(".");
    delay(500);
  }
  lcd.clear();
  lcd.setCursor(11, 1);
  lcd.print("W:OFF");

  // Call the functions at intervals
  timer.setInterval(100L, soilMoistureSensor);
  timer.setInterval(100L, DHT11sensor);
  timer.setInterval(500L, checkPhysicalButton);
}

// Get the soil moisture values
void soilMoistureSensor() {
  int value = analogRead(soil);
  value = map(value, 0, 1024, 0, 100);
  value = (value - 100) * -1;

  Blynk.virtualWrite(V3, value);
  lcd.setCursor(0, 1);
  lcd.print("S:");
  lcd.print(value);
  lcd.print(" ");
}

// Get the DHT11 sensor values
void DHT11sensor() {
  float h = dht.readHumidity();
  float t = dht.readTemperature();

  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  Blynk.virtualWrite(V0, t);
  Blynk.virtualWrite(V1, h);

  lcd.setCursor(0, 0);
  lcd.print("T:");
  lcd.print(t);

  lcd.setCursor(8, 0);
  lcd.print("H:");
  lcd.print(h);
}

// Check physical button state
void checkPhysicalButton() {
  if (digitalRead(PUSH_BUTTON_1) == LOW) {
    if (pushButton1State != LOW) {
      relay1State = !relay1State;
      digitalWrite(RELAY_PIN_1, relay1State);
      Blynk.virtualWrite(V12, relay1State);
    }
    pushButton1State = LOW;
  } else {
    pushButton1State = HIGH;
  }
}

// Get the PIR sensor values
void PIRsensor() {
  bool value = digitalRead(PIR);
  if (value) {
    Blynk.logEvent("pirmotion", "WARNNG! Motion Detected!"); // Enter your Event Name
    WidgetLED LED(V5);
    LED.on();
  } else {
    WidgetLED LED(V5);
    LED.off();
  }
}

void loop() {
  if (PIR_ToggleValue == 1) {
    lcd.setCursor(5, 1);
    lcd.print("M:ON ");
    PIRsensor();
  } else {
    lcd.setCursor(5, 1);
    lcd.print("M:OFF");
    WidgetLED LED(V5);
    LED.off();
  }

  if (relay1State == HIGH) {
    lcd.setCursor(11, 1);
    lcd.print("W:ON ");
  } else if (relay1State == LOW) {
    lcd.setCursor(11, 1);
    lcd.print("W:OFF");
  }

  Blynk.run();  // Run the Blynk library
  timer.run();  // Run the Blynk timer
}
