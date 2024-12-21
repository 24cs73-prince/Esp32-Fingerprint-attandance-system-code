# Esp32-Fingerprint-attandance-system-code


 FINGERPRINT ATTENDANCE SYSTEM
 
* CODE:

  
#include <WiFi.h>
#include <HTTPClient.h>
#include <Adafruit_Fingerprint.h>
// Wi-Fi credentials
const char* ssid = "OnePlus Nord CE4";
const char* password = "123454321";
// Google Sheets Web App URL
const String googleScriptURL =
"https://script.google.com/macros/s/AKfycbxoFXmGm0FaE21no1NunkTRfr
HF9x3PA1QW-Ie4xUJg1Zs8TGMzmhDydigc9EkaR6-4lA/exec"; // Replace
with your Google Script URL
// Define UART2 for ESP32
HardwareSerial mySerial(2); // Using UART2 for ESP32 (RX=16, TX=17)
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);
// Define buzzer pin
#define BUZZER_PIN 27
void setup() {
 Serial.begin(115200); // Debugging Serial0
 WiFi.begin(ssid, password);

 while (WiFi.status() != WL_CONNECTED) {
 delay(1000);
 Serial.println("Connecting to WiFi...");
 }
 Serial.println("Connected to WiFi!");
 mySerial.begin(57600, SERIAL_8N1, 16, 17); // UART2: RX=16, TX=17
 pinMode(BUZZER_PIN, OUTPUT); // Set buzzer pin as output
 digitalWrite(BUZZER_PIN, LOW); // Turn buzzer off initially
 // Initialize the fingerprint sensor
 if (finger.verifyPassword()) {
 Serial.println("Fingerprint sensor found and verified!");
 } else {
 Serial.println("Did not find fingerprint sensor :(");
 while (1); // Halt if sensor is not found
 }
 Serial.println("Welcome! Please select an option:");
 Serial.println("1: Enroll Fingerprint");
 Serial.println("2: Verify Fingerprint");
 Serial.println("3: Update Existing Fingerprint");
}
void loop() {
 Serial.println("\nChoose an option:");
 while (!Serial.available()); // Wait for input from the user
 int choice = Serial.parseInt();
 // Clear the Serial buffer to avoid repeated inputs
 while (Serial.available()) {
 Serial.read();
 }
 if (choice == 1) {
 Serial.println("Fingerprint Enrollment Selected.");
 enrollFingerprints();
 } else if (choice == 2) {
 Serial.println("Fingerprint Verification Selected.");
 verifyFingerprints();
 } else if (choice == 3) {
 Serial.println("Fingerprint Update Selected.");
 updateFingerprint();
 } else {
 Serial.println("Invalid choice. Try again.");
 }
}
// Function to enroll fingerprints
void enrollFingerprints() {
 while (true) {
 Serial.println("Enter ID # (1-127) to enroll fingerprint, or type 0 to finish
enrollment:");
 while (!Serial.available());
 int id = Serial.parseInt();
 // Clear the Serial buffer
 while (Serial.available()) {
 Serial.read();
 }
 if (id == 0) {
 Serial.println("Enrollment complete!");
 break;
 }
 if (id < 1 || id > 127) {
 Serial.println("Invalid ID. Please enter a number between 1 and 127.");
 continue;
 }
 enrollFingerprint(id);
 }
}
// Function to verify fingerprints and send data to Google Sheets
void verifyFingerprints() {
 while (true) {
 Serial.println("Place your finger on the sensor for verification, or type 0 to
exit:");
 bool waitingForFinger = true;
 while (waitingForFinger) {
 int p = finger.getImage();
 if (p == FINGERPRINT_NOFINGER) {
 delay(100);
 continue;
 }
 if (Serial.available() && Serial.parseInt() == 0) {
 while (Serial.available()) Serial.read();
 Serial.println("Exiting fingerprint verification.");
 return;
 }
 if (p == FINGERPRINT_OK) {
 p = finger.image2Tz();
 if (p == FINGERPRINT_OK) {
 p = finger.fingerFastSearch();
 if (p == FINGERPRINT_OK) {
 Serial.print("Match found! ID: ");
 Serial.println(finger.fingerID);
 // Send data to Google Sheets
 String studentID = String(finger.fingerID);
 String subject = "Subject Name"; // Replace dynamically if needed
 String date = getCurrentDate();
 String time = getCurrentTime();
 sendDataToGoogleSheet(studentID, subject, date, time);
 playNoisyBuzzer();
 } else {
 Serial.println("No match found.");
 playGentleBuzzer();
 }
 } else {
 Serial.println("Error processing fingerprint.");
 }
 waitingForFinger = false;
 } else if (p == FINGERPRINT_PACKETRECIEVEERR) {
 Serial.println("Communication error.");
 } else if (p == FINGERPRINT_IMAGEFAIL) {
 Serial.println("Imaging error.");
 }
 }
 }
}
// Function to enroll a single fingerprint
void enrollFingerprint(int id) {
 int p = -1;
 Serial.print("Enrolling ID #");
 Serial.println(id);
 Serial.println("Place your finger on the sensor.");
 while (p != FINGERPRINT_OK) {
 p = finger.getImage();
 if (p == FINGERPRINT_NOFINGER) continue;
 if (p == FINGERPRINT_OK) Serial.println("Image taken.");
 else return;
 }
 p = finger.image2Tz(1);
 if (p != FINGERPRINT_OK) return;
 Serial.println("Remove your finger and place it again.");
 delay(2000);
 while (p != FINGERPRINT_NOFINGER) p = finger.getImage();
 Serial.println("Place your finger on the sensor again.");
 p = -1;
 while (p != FINGERPRINT_OK) {
 p = finger.getImage();
 if (p == FINGERPRINT_NOFINGER) continue;
 if (p == FINGERPRINT_OK) Serial.println("Image taken.");
 else return;
 }
 p = finger.image2Tz(2);
 if (p != FINGERPRINT_OK) return;
 p = finger.createModel();
 if (p != FINGERPRINT_OK) return;
 p = finger.storeModel(id);
 if (p == FINGERPRINT_OK) Serial.println("Fingerprint enrolled
successfully!");
}
// Function to update an existing fingerprint
void updateFingerprint() {
 Serial.println("Enter ID # (1-127) to update fingerprint:");
 while (!Serial.available());
 int id = Serial.parseInt();
 while (Serial.available()) {
 Serial.read();
 }
 if (id < 1 || id > 127) {
 Serial.println("Invalid ID. Please enter a number between 1 and 127.");
 return;
 }
 enrollFingerprint(id);
 Serial.println("Fingerprint updated successfully!");
}
// Function to send data to Google Sheets
void sendDataToGoogleSheet(String id, String subject, String date, String
time) {
 HTTPClient http;
 String url = googleScriptURL;
 http.begin(url);
 http.addHeader("Content-Type", "application/json");
 String jsonData = "{\"id\":\"" + id + "\", \"subject\":\"" + subject + "\",
\"date\":\"" + date + "\", \"time\":\"" + time + "\"}";
 int httpResponseCode = http.POST(jsonData);
 if (httpResponseCode > 0) {
 Serial.println("Data sent successfully!");
 } else {
 Serial.println("Error sending data.");
 }
 http.end();
}
// Function to get current date
String getCurrentDate() {
 return __DATE__;
}
// Function to get current time
String getCurrentTime() {
 return __TIME__;
}
// Buzzer sound for a match
void playNoisyBuzzer() {
 for (int i = 0; i < 5; i++) {
 digitalWrite(BUZZER_PIN, HIGH);
 delay(100);
 digitalWrite(BUZZER_PIN, LOW);
 delay(100);
 }
}
// Buzzer sound for no match
void playGentleBuzzer() {
 for (int i = 0; i < 2; i++) {
 digitalWrite(BUZZER_PIN, HIGH);
 delay(200);
 digitalWrite(BUZZER_PIN, LOW);
 delay(200);
 }
} 
