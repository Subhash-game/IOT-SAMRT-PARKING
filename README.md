
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal.h>
#include <SoftwareSerial.h>

#define IR_SENSOR_1 2
#define IR_SENSOR_2 3
#define IR_SENSOR_3 4
#define SERVO_PIN 5
#define BUZZER_PIN 6

// Initialize Servo and LCD
Servo parkingGate;
LiquidCrystal lcd(A0, A1, A2, A3, A4, A5);  // Connect LCD to Arduino
SoftwareSerial RFID_Serial(9, 10);  // RX, TX pins for RFID
String tag = "";

// Authorized RFID Tags
String authorizedTags[] = {"4B00E27AC91A", "5000DBBCE0D7", "3800AA27D267"};  // Add more authorized tags if needed

// Parking Slot Data
bool slot1Occupied = false, slot2Occupied = false, slot3Occupied = false;
unsigned long entryTimeSlot1 = 0, entryTimeSlot2 = 0, entryTimeSlot3 = 0;
float hourlyRate = 5.0;  // Example rate: 5 units per hour
float conversionRate = 80.0; // Conversion rate from units to INR (e.g., 1 unit = 80 INR)
String lastRFID = "";  // To track the last RFID scanned
String string, str1, str2, str3;
// Setup
void setup() {
  Serial.begin(9600);
  RFID_Serial.begin(9600);
  
  // Initialize pin modes
  pinMode(IR_SENSOR_1, INPUT);
  pinMode(IR_SENSOR_2, INPUT);
  pinMode(IR_SENSOR_3, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Attach Servo to the pin
  parkingGate.attach(SERVO_PIN);

  // Initialize LCD
  lcd.begin(16, 2);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Smart Parking");
  delay(1000);
  lcd.clear();
}

// Loop to check parking status and RFID tags
void loop() {
  // Read the IR sensors to detect the presence of a car
  while(Serial.available())
  {
    char a=Serial.read();
    Serial.println("Received:");
    Serial.println(a);
    delay(1000);
    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("received:");
    lcd.print(a);
    delay(1000);

    if(a=='b')
    {
       parkingGate.write(90);  // Open gate (90 degrees)
       delay(3000);
        parkingGate.write(0);  // Open gate (90 degrees)
        lcd.clear();
        lcd.setCursor(0,0);
        lcd.print("valid vehicle");
        delay(1000);
    }
    else if(a=='c')
    {
       parkingGate.write(90);  // Open gate (90 degrees)
       delay(3000);
        parkingGate.write(0);  // Open gate (90 degrees)
        lcd.clear();
        lcd.setCursor(0,0);
        lcd.print("valid vehicle");
        delay(1000);
    }
  }
  // Read the IR sensors to detect the presence of a car
  int a1 = digitalRead(IR_SENSOR_1);  // Slot 1 sensor
lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Slot 1: ");
  lcd.print(a1 == LOW ? "Occupied" : "Free");
  delay(1000);
str1 = "@" + String((a1));
Serial.println(str1);
delay(2000);

  int a2 = digitalRead(IR_SENSOR_2);  // Slot 2 sensor
lcd.setCursor(0, 1);
  lcd.print("Slot 2: ");
  lcd.print(a2 == LOW ? "Occupied" : "Free");
  delay(1000);
  str1 = "#" + String(a2);
Serial.println(str1);
delay(2000);

  int a3 = digitalRead(IR_SENSOR_3);  // Slot 3 sensor
lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("Slot 3: ");
  lcd.print(a3 == LOW ? "Occupied" : "Free");
  str1 = "$" + String(a3);
Serial.println(str1);
delay(2000);

  // Read RFID tag
  tag = "";
  while (RFID_Serial.available()) {
    char c = RFID_Serial.read();
    tag += c;
    delay(100);  // Allow enough time to read the full tag
  }

  // Check if an RFID tag was detected
  if (tag.length() > 0) {
    tag.trim();  // Remove any unnecessary whitespace or extra characters
    Serial.println("RFID Tag Detected: " + tag);

    // Authenticate RFID tag
    if (isAuthorized(tag)) {
      if (lastRFID == tag) {
        // Same card detected again, deduct payment
        deductPayment();
        lastRFID = "";  // Reset last RFID after payment
      } else {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Valid Tag - Open");
        delay(1000);

        // Check available slots and open gate for the first free slot
        if (!slot1Occupied && a1 == HIGH) {  // Slot 1 available
          openGate(1);
          entryTimeSlot1 = millis();
          slot1Occupied = true;
          displayParkingDetails(1);
        } else if (!slot2Occupied && a2 == HIGH) {  // Slot 2 available
          openGate(2);
          entryTimeSlot2 = millis();
          slot2Occupied = true;
          displayParkingDetails(2);
        } else if (!slot3Occupied && a3 == HIGH) {  // Slot 3 available
          openGate(3);
          entryTimeSlot3 = millis();
          slot3Occupied = true;
          displayParkingDetails(3);
        } else {
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("No free slots!");
          delay(2000);  // Wait before clearing the message
        }

        lastRFID = tag;  // Set last RFID tag after first scan
      }
    } else {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Unauthorized tag!");
      delay(2000);  // Wait before clearing the message
    }
  }

  // Handle exit and payment for each slot
  if (slot1Occupied && a1 == LOW) {  // Car has exited Slot 1
    calculateTimeAndCharge(1);
    slot1Occupied = false;
  } else if (slot2Occupied && a2 == LOW) {  // Car has exited Slot 2
    calculateTimeAndCharge(2);
    slot2Occupied = false;
  } else if (slot3Occupied && a3 == LOW) {  // Car has exited Slot 3
    calculateTimeAndCharge(3);
    slot3Occupied = false;
  }

  delay(1000);
}

// Function to check if the RFID tag is authorized
bool isAuthorized(String tag) {
  for (int i = 0; i < 3; i++) {
    if (tag == authorizedTags[i]) {
      return true;
    }
  }
  return false;
}

// Function to open parking gate
void openGate(int slot) {
  Serial.println("Opening Gate for Slot " + String(slot));
  parkingGate.write(90);  // Open gate (90 degrees)
  delay(3000);
  parkingGate.write(0);   // Close gate
  digitalWrite(BUZZER_PIN, HIGH);  // Sound buzzer
  delay(1000);
  digitalWrite(BUZZER_PIN, LOW);  // Turn off buzzer
  delay(5000);  // Keep the gate open for 5 seconds
}

// Function to display parking details
void displayParkingDetails(int slot) {
  unsigned long timeSpent;
  float charge;

  if (slot == 1) {
    timeSpent = (millis() - entryTimeSlot1) / 1000;  // Time in seconds
  } else if (slot == 2) {
    timeSpent = (millis() - entryTimeSlot2) / 1000;
  } else {
    timeSpent = (millis() - entryTimeSlot3) / 1000;
  }

  // Convert time spent from seconds to hours for charging
  float hours = timeSpent / 3600.0;
  charge = hours * hourlyRate;

  // Convert charge to INR
  float chargeINR = charge * conversionRate;

  // Display time spent and charge
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Time Spent: ");
  lcd.print(hours, 2);
  lcd.setCursor(0, 1);
  lcd.print("Charge: ₹");
  lcd.print(chargeINR, 2);
  delay(5000);  // Show the charge for 5 seconds

  // Indicate payment completion
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Payment Completed");
  delay(3000);
}

// Function to calculate parking time and charge
void calculateTimeAndCharge(int slot) {
  unsigned long timeSpent;
  float charge;

  if (slot == 1) {
    timeSpent = (millis() - entryTimeSlot1) / 1000;  // Time in seconds
  } else if (slot == 2) {
    timeSpent = (millis() - entryTimeSlot2) / 1000;
  } else {
    timeSpent = (millis() - entryTimeSlot3) / 1000;
  }

  // Convert time spent from seconds to hours for charging
  float hours = timeSpent / 3600.0;
  charge = hours * hourlyRate;

  // Convert charge to INR
  float chargeINR = charge * conversionRate;

  // Display time spent and charge
  Serial.print("Time Spent: ");
  Serial.print(hours, 2);
  Serial.print(" hours - Charge: ₹");
  Serial.println(chargeINR);

  // Show the charge on LCD
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Time Spent: ");
  lcd.print(hours, 2);
  lcd.setCursor(0, 1);
  lcd.print("Charge: ₹");
  lcd.print(chargeINR, 2);
  delay(5000);  // Show the charge for 5 seconds

  // Indicate payment completed
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Payment Completed");
  delay(3000);  // Wait before clearing the message
}

// Simulate payment deduction
void deductPayment() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Payment Deducted");
  delay(3000);  // Wait before clearing the message
}
