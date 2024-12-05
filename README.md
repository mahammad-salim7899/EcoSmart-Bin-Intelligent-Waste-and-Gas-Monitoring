# EcoSmart-Bin-Intelligent-Waste-and-Gas-Monitoring
Smart Dustbin with IoT Integration

This project implements a Smart Dustbin system designed to enhance waste management efficiency. The smart dustbin features:

Automatic Lid Control: Operates based on object proximity using ultrasonic sensors.
Bin Level Monitoring: Tracks the fill level of the dustbin in real-time.
Gas Detection: Monitors toxic gas levels using an MQ2 gas sensor for safety.
OLED Display: Displays bin level and gas status visually.
IoT Connectivity: Integrated with the Blynk platform for remote monitoring and notifications.
This system aims to promote smarter waste disposal practices by automating lid operation, sending alerts when the bin is full, and ensuring safety by detecting hazardous gases.
here is the code:
// Define Blynk credentials and template information
#define BLYNK_TEMPLATE_ID "template_id"
#define BLYNK_TEMPLATE_NAME "template_name"
#define BLYNK_AUTH_TOKEN "authontication _token"

#define BLYNK_PRINT Serial  // Enable serial debugging for Blynk

#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>
#include <ESP32Servo.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// Wi-Fi credentials
char auth[] = "authontication _token";  // Blynk authentication token
char ssid[] = "******";  // Wi-Fi SSID
char pass[] = "*****";  // Wi-Fi password

// OLED display settings
#define SCREEN_WIDTH 128  // Width of the OLED display in pixels
#define SCREEN_HEIGHT 64  // Height of the OLED display in pixels
#define OLED_RESET -1     // OLED reset pin (-1 if using Arduino reset pin)
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

BlynkTimer timer;  // Timer for managing periodic tasks

// Pin definitions
#define echoPin 32          // Ultrasonic sensor echo pin for bin level
#define trigPin 33          // Ultrasonic sensor trigger pin for bin level
#define echoPin2 25         // Ultrasonic sensor echo pin for lid control
#define trigPin2 26         // Ultrasonic sensor trigger pin for lid control
#define gasSensorPin 35     // MQ2 gas sensor pin (ADC pin)

Servo servo;  // Servo motor for lid control

// Variables for ultrasonic sensors
long duration, duration2;
int distance, distance2;
int binLevel = 0;  // Bin level percentage

// Notification flags
bool binNotificationSent = false;  // Flag for bin level notification
bool gasNotificationSent = false; // Flag for gas sensor notification

// Function to handle sensors and notifications
void SMESensor() {
  ultrasonic();          // Measure bin level using first ultrasonic sensor
  ultrasonic2();         // Detect object using second ultrasonic sensor

  // Control the lid based on proximity of an object detected by the second ultrasonic sensor
  if (distance2 < 15) {  // Open lid if object is within 15 cm range
    servo.write(90);     // Rotate servo to 90 degrees (open position)
    Blynk.virtualWrite(V2, 90);  // Update Blynk app
    delay(500);          // Allow time for servo to move
  } else {
    servo.write(0);      // Close lid (0 degrees)
    Blynk.virtualWrite(V2, 0);  // Update Blynk app
    delay(1000);         // Allow time for servo to move
  }

  // Bin level notification logic
  if (binLevel >= 80 && !binNotificationSent) {
    Blynk.logEvent("bin_alert", "Warning: Bin level has reached full");
    binNotificationSent = true;  // Prevent repeated notifications
  } else if (binLevel < 80) {
    binNotificationSent = false;  // Reset notification flag if bin level decreases
  }

  // Gas sensor reading and notifications
  int gasLevel = analogRead(gasSensorPin);  // Read gas sensor value
  Blynk.virtualWrite(V3, gasLevel);        // Send gas level to Blynk app

  if (gasLevel > 600 && !gasNotificationSent) {  // If gas level exceeds threshold
    Blynk.logEvent("gas_alert", "Toxic waste detected");
    gasNotificationSent = true;  // Prevent repeated notifications
  } else if (gasLevel <= 600) {
    gasNotificationSent = false;  // Reset notification flag if gas level decreases
  }

  // Display bin level and gas status on the OLED screen
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(WHITE);
  display.setCursor(0, 0);
  display.print("Bin Level: ");
  display.print(binLevel);
  display.println("%");

  if (gasLevel > 600) {
    display.setCursor(0, 20);
    display.print("Toxic gas detected!");
  }

  display.display();

  delay(1000);  // Delay 1 second before the next cycle
}

// Function to measure bin level using the first ultrasonic sensor
void ultrasonic() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  duration = pulseIn(echoPin, HIGH);  // Measure pulse duration
  distance = duration * 0.034 / 2;   // Calculate distance in cm

  // Map distance to bin level percentage (0% to 100%)
  binLevel = map(distance, 21, 0, 0, 100);

  // Send distance and bin level to Blynk app
  Blynk.virtualWrite(V0, distance);
  Blynk.virtualWrite(V1, binLevel);
}

// Function to detect object proximity using the second ultrasonic sensor
void ultrasonic2() {
  digitalWrite(trigPin2, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin2, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin2, LOW);

  duration2 = pulseIn(echoPin2, HIGH);  // Measure pulse duration
  distance2 = duration2 * 0.034 / 2;   // Calculate distance in cm

  // Send distance to Blynk app
  Blynk.virtualWrite(V0, distance2);
}

void setup() {
  Serial.begin(9600);  // Initialize serial communication

  servo.attach(13);    // Attach servo motor to GPIO pin 13 on ESP32

  // Configure pin modes for ultrasonic sensors and gas sensor
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(trigPin2, OUTPUT);
  pinMode(echoPin2, INPUT);
  pinMode(gasSensorPin, INPUT);

  // Initialize OLED display
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {  // Address 0x3C for OLED
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }
  display.display();  // Show splash screen
  delay(2000);        // Delay for splash screen

  // Connect to Blynk server
  Blynk.begin(auth, ssid, pass);
}

void loop() {
  Blynk.run();  // Run Blynk tasks
  SMESensor();  // Run sensor monitoring and notifications
}
