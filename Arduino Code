#include <Wire.h> 
#include <SPI.h> 
#include <SD.h> 
#include <Adafruit_GFX.h> 
#include <Adafruit_SSD1306.h> 
#include <SparkFun_VEML7700_Arduino_Library.h> 
#include "BluetoothSerial.h"  

 
// ========================================== 
// BLUETOOTH SETTINGS 
// ========================================== 

#if !defined(CONFIG_BT_ENABLED) || !defined(CONFIG_BLUEDROID_ENABLED) 
#error Bluetooth is not enabled! Please run `make menuconfig` to and enable it 
#endif 
BluetoothSerial SerialBT; 
unsigned long lastPing = 0; // Timer for the BT heartbeat 
 

// ========================================== 
// DISPLAY SETTINGS 
// ========================================== 

#define SCREEN_WIDTH 128 
#define SCREEN_HEIGHT 32 
#define OLED_RESET    -1 
#define SCREEN_ADDRESS 0x3C  
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET); 

// ========================================== 
// SENSOR & SD SETTINGS (Standard Pins) 
// ========================================== 

VEML7700 mySensor; 
const int chipSelect = 5;  
File logFile; 
String currentFileName = ""; 

// ========================================== 
// BUTTON PINS & TRACKING 
// ========================================== 

const int btnBack   = 27;  
const int btnSelect = 26;  
const int btnUp     = 25;  
const int btnDown   = 33;  

bool lastBack = HIGH; 
bool lastSelect = HIGH; 
bool lastUp = HIGH; 
bool lastDown = HIGH; 

// ========================================== 
// SYSTEM STATES 
// ========================================== 

enum SystemState {  
  STATE_OFF,  
  STATE_MENU,  
  STATE_LOGGING,  
  STATE_MESSAGE,  
  STATE_CONFIRM_WIPE, 
  STATE_FILE_BROWSER, 
  STATE_FILE_VIEWER, 
  STATE_CONFIRM_BT  
}; 

SystemState currentState = STATE_OFF; 

// ========================================== 
// MENU VARIABLES 
// ========================================== 

const int NUM_MENU_ITEMS = 4; 

String menuItems[NUM_MENU_ITEMS] = {"Record Data", "Data Sets", "Clear SD Card", "Export via BT"}; 

int menuIndex = 0; 

int confirmIndex = 0;   // 0 = No, 1 = Yes (For SD Wipe) 
int confirmBTIndex = 0; // 0 = No, 1 = Yes (For Bluetooth) 

// ========================================== 
// BROWSER & VIEWER VARIABLES 
// ========================================== 

enum BrowserMode { MODE_VIEW, MODE_BT_EXPORT }; 

BrowserMode currentBrowserMode = MODE_VIEW; 

int availableLogs[100];  
int browserFileCount = 0; 
int browserIndex = 0; 
int selectedBTLog = 0;  

String viewerCurrentFile = ""; 
int viewerTotalLines = 0; 
int viewerScrollPos = 0; 

// ========================================== 
// MATH & LADDER VARIABLES 
// ========================================== 

const int MAX_LINES = 3;  

struct SensorData { 

  int readNumber; 
  float luxValue; 

}; 
SensorData history[MAX_LINES];   

int totalReadingsCount = 0;  

double totalLuxSum = 0.0;     
float currentAverage = 0.0;   

unsigned long previousMillis = 0; 
const long interval = 1000;  

String tempMessage = ""; 
unsigned long messageStartTime = 0; 

// ========================================== 
// SETUP ROUTINE 
// ========================================== 

void setup() { 
  Serial.begin(115200); 

  // Hardware stabilization delay 
  delay(1000);  

  pinMode(btnBack, INPUT_PULLUP); 
  pinMode(btnSelect, INPUT_PULLUP); 
  pinMode(btnUp, INPUT_PULLUP); 
  pinMode(btnDown, INPUT_PULLUP); 

  // MISO Stability fix 
  pinMode(19, INPUT_PULLUP);  

  // Initialize Bluetooth 

  SerialBT.begin("ESP32_Logger");  

  // Initialize I2C and OLED 
  Wire.begin();  

  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {  
    while (1);  
  } 
   

  // Initialize Sensor 
  mySensor.begin(); 

  // STABILITY FIX: 400kHz Ultra-Safe Speed Limit + 5 Retries 

  bool sdInitialized = false; 

  for (int i = 0; i < 5; i++) { 
    if (SD.begin(chipSelect, SPI, 400000)) {  
      sdInitialized = true; 
      break;  

    } 
    delay(500);  
  } 

  if (!sdInitialized) { 
    displayMessage("SD Card Error!", 3000); 
  } 

  // Start with screen clear 
  display.clearDisplay(); 
  display.display(); 
} 
 

// ========================================== 
// MAIN LOOP & STATE MACHINE 
// ========================================== 

void loop() { 

  // Edge detection for buttons 
  bool pressBack   = checkButton(btnBack, lastBack); 
  bool pressSelect = checkButton(btnSelect, lastSelect); 
  bool pressUp     = checkButton(btnUp, lastUp); 
  bool pressDown   = checkButton(btnDown, lastDown); 

  switch (currentState) { 
     
    // ------------------------------------ 

    case STATE_OFF: 

      if (pressBack) { 
        currentState = STATE_MENU; 
        menuIndex = 0; 
        drawMenu(); 

      } 
      break; 

    // ------------------------------------ 

    case STATE_MENU: 
      // --- THE BT HEARTBEAT --- 

      // Sends a "ping" to the laptop every 3s to prevent auto-disconnect 
      if (millis() - lastPing > 3000) { 
        if (SerialBT.hasClient()) {  
          SerialBT.println("Device Active..."); 
        } 
        lastPing = millis(); 
      } 

      // Menu Navigation 
      if (pressBack) { 

        currentState = STATE_OFF; 
        display.clearDisplay(); 
        display.display(); 

      }  
      else if (pressUp) { 
        menuIndex--; 

        if (menuIndex < 0) { 
          menuIndex = NUM_MENU_ITEMS - 1;  

        } 
        drawMenu(); 

      }  
      else if (pressDown) { 

        menuIndex++; 
        if (menuIndex >= NUM_MENU_ITEMS) { 
          menuIndex = 0;  

        } 

        drawMenu(); 
      }  

      else if (pressSelect) { 
        executeMenuChoice(); 

      } 
      break; 
    // ------------------------------------ 

    case STATE_FILE_BROWSER: 

      if (pressBack) {  
        currentState = STATE_MENU;  
        drawMenu();  
      } 

      else if (pressUp) { 
        if (browserFileCount > 0) { 
          browserIndex--; 
          if (browserIndex < 0) { 

            browserIndex = browserFileCount - 1; 
          } 
          drawFileBrowser(); 

        } 
      } 

      else if (pressDown) { 
        if (browserFileCount > 0) { 
          browserIndex++; 

          if (browserIndex >= browserFileCount) { 
            browserIndex = 0; 

          } 
          drawFileBrowser(); 
        } 

      } 

      else if (pressSelect) { 

        if (browserFileCount > 0) { 

          if (currentBrowserMode == MODE_VIEW) { 
            char filename[16]; 
            sprintf(filename, "/LOG_%03d.TXT", availableLogs[browserIndex]); 
            viewerCurrentFile = String(filename); 
            openFileViewer(); 
          }  

          else if (currentBrowserMode == MODE_BT_EXPORT) { 

            selectedBTLog = availableLogs[browserIndex]; 
            confirmBTIndex = 0; // Default to NO for safety 
            currentState = STATE_CONFIRM_BT; 
            drawConfirmBT(); 

          } 
        } 
      } 

      break; 

    // ------------------------------------ 

    case STATE_CONFIRM_BT: 
      if (pressBack) {  

        currentState = STATE_FILE_BROWSER;  
        drawFileBrowser();  

      } 

      else if (pressUp || pressDown) {  

        confirmBTIndex = !confirmBTIndex; // Toggle Yes/No 

        drawConfirmBT();  

      } 

      else if (pressSelect) { 

        if (confirmBTIndex == 0) {  

          // Selected NO 
          currentState = STATE_FILE_BROWSER;  
          drawFileBrowser();  

        } 

        else {  
          // Selected YES 
          sendDataViaBT(selectedBTLog);  

        } 
      } 
      break; 

    // ------------------------------------ 

    case STATE_FILE_VIEWER: 

      if (pressBack) {  
        currentState = STATE_FILE_BROWSER;  
        drawFileBrowser();  
      } 

      else if (pressUp) { 

        if (viewerScrollPos > 0) { 
          viewerScrollPos--; 
          drawFileViewer(); 
        } 
      } 

      else if (pressDown) { 
        if (viewerScrollPos < viewerTotalLines - MAX_LINES) { 
          viewerScrollPos++; 
          drawFileViewer(); 
        } 
      } 

      break; 

    // ------------------------------------ 
    case STATE_CONFIRM_WIPE: 
      if (pressBack) {  
        currentState = STATE_MENU;  
        drawMenu();  
      } 

      else if (pressUp || pressDown) {  
        confirmIndex = !confirmIndex; // Toggle Yes/No 
        drawConfirmWipe();  

      } 

      else if (pressSelect) { 
        if (confirmIndex == 0) {  
          // Selected NO 
          currentState = STATE_MENU;  
          drawMenu();  

        } 

        else {  
          // Selected YES 
          clearSDCard();  
          displayMessage("SD Card Empty!", 2000);  
        } 
      } 
      break; 

    // ------------------------------------ 

    case STATE_LOGGING: 

      if (pressSelect || pressBack) { 

        if (logFile) { 

          logFile.close();  

        } 

        displayMessage("Saved to SD!", 2000);  

      }  
      else { 
        unsigned long currentMillis = millis(); 

        if (currentMillis - previousMillis >= interval) { 
          previousMillis = currentMillis; 
          takeReading(); 
        } 
      } 
      break;  

    // ------------------------------------ 
    case STATE_MESSAGE: 
      display.clearDisplay(); 
      display.setCursor(0, 10); 
      display.println(tempMessage); 
      display.display();    

      // Wait for duration to pass 
      if (millis() - messageStartTime > 2000) {  

        // If we just finished a BT transfer, go back to the browser 

        if (currentBrowserMode == MODE_BT_EXPORT && tempMessage == "Sent Successfully!") { 

           currentState = STATE_FILE_BROWSER; 
           drawFileBrowser(); 
        } else { 

           // Otherwise, go back to main menu 
           currentState = STATE_MENU; 
           drawMenu(); 

        } 
      } 

      break; 
  } 
} 

 

// ========================================== 
// CORE UI & UTILITY FUNCTIONS 
// ========================================== 

 

bool checkButton(int pin, bool &lastState) { 
  bool isPressed = false; 
  bool current = digitalRead(pin); 

  if (current == LOW && lastState == HIGH) { 
    delay(25); // Debounce 
    isPressed = true; 
  } 

  lastState = current; 
  return isPressed; 

} 

void drawMenu() { 

  display.clearDisplay(); 

  display.setTextSize(1); 
  // Scrolling logic to fit 4 items on a 3 line screen 
  int startIdx = menuIndex - 1; 
  if (startIdx < 0) startIdx = 0; 
  if (startIdx > NUM_MENU_ITEMS - 3) startIdx = NUM_MENU_ITEMS - 3; 

   

  for (int i = 0; i < 3; i++) { 
    int idx = startIdx + i; 

    if (idx < NUM_MENU_ITEMS) { 
      display.setCursor(idx == menuIndex ? 0 : 8, i * 10); 

      if (idx == menuIndex) { 
        display.setTextColor(SSD1306_BLACK, SSD1306_WHITE); 
        display.print(">"); 

      } else { 
        display.setTextColor(SSD1306_WHITE, SSD1306_BLACK); 

      } 
      display.print(menuItems[idx]); 
    } 
  } 

  display.display(); 
}  

void drawConfirmWipe() { 

  display.clearDisplay(); 
  display.setTextColor(SSD1306_WHITE); 
  display.setCursor(0, 0); 
  display.print("Confirm Wipe?"); 

  display.setCursor(10, 12); 

  if (confirmIndex == 0) { 
    display.setTextColor(SSD1306_BLACK, SSD1306_WHITE); 
  } else { 
    display.setTextColor(SSD1306_WHITE, SSD1306_BLACK); 
  } 
  display.print(" No ");  

  display.setCursor(10, 22); 
  if (confirmIndex == 1) { 
    display.setTextColor(SSD1306_BLACK, SSD1306_WHITE); 

  } else { 
    display.setTextColor(SSD1306_WHITE, SSD1306_BLACK); 
  } 

  display.print(" Yes "); 
  display.display(); 
} 

void drawConfirmBT() { 

  display.clearDisplay(); 
  display.setTextColor(SSD1306_WHITE); 
  display.setCursor(0, 0); 
  display.print("Send LOG_"); 

  if (selectedBTLog < 10) display.print("00"); 
  else if (selectedBTLog < 100) display.print("0"); 
  display.print(selectedBTLog); 
  display.print("?"); 

  display.setCursor(10, 12); 

  if (confirmBTIndex == 0) { 
    display.setTextColor(SSD1306_BLACK, SSD1306_WHITE); 

  } else { 
    display.setTextColor(SSD1306_WHITE, SSD1306_BLACK); 
  } 

  display.print(" No "); 

  display.setCursor(10, 22); 
  if (confirmBTIndex == 1) { 
    display.setTextColor(SSD1306_BLACK, SSD1306_WHITE); 
  } else { 
    display.setTextColor(SSD1306_WHITE, SSD1306_BLACK); 

  } 
  display.print(" Yes ");    

  display.display(); 
}  

void executeMenuChoice() { 

  if (menuIndex == 0) { 
    startNewLog(); 
  } 
  else if (menuIndex == 1 || menuIndex == 3) { 

    // Both Data Sets and BT Export open the browser 
    if (menuIndex == 1) { 
      currentBrowserMode = MODE_VIEW; 

    } else { 
      currentBrowserMode = MODE_BT_EXPORT; 
    }   

    display.clearDisplay(); 
    display.setCursor(0, 10); 
    display.print("Reading Disk..."); 
    display.display();     

    loadAvailableLogs(); 
    browserIndex = 0; 
    currentState = STATE_FILE_BROWSER; 
    drawFileBrowser(); 

  }  

  else if (menuIndex == 2) { 
    confirmIndex = 0; // Default to NO 
    currentState = STATE_CONFIRM_WIPE; 
    drawConfirmWipe(); 

  } 
}  

// ========================================== 
// MEMORY-SAFE BLUETOOTH EXPORT 
// ========================================== 

void sendDataViaBT(int logNumber) { 
  display.clearDisplay(); 
  display.setCursor(0, 10); 

  display.print("Transmitting..."); 
  display.display(); 

  char filename[16]; 
  sprintf(filename, "/LOG_%03d.TXT", logNumber); 
  File f = SD.open(filename);   

  if (f) { 
    SerialBT.println("--- LOG DATA TRANSFER START ---"); 
    SerialBT.println("Filename: " + String(filename));    

    // NATIVE BYTE BLASTING: Prevents memory fragmentation 
    uint8_t buffer[64]; 
    while (int bytesRead = f.read(buffer, sizeof(buffer))) { 
      SerialBT.write(buffer, bytesRead); 

    } 
    SerialBT.println("\n--- TRANSFER COMPLETE ---"); 

    f.close(); 

    displayMessage("Sent Successfully!", 2000); 
  } else { 
    displayMessage("Read Error!", 2000); 
  } 
} 

 

// ========================================== 
// FAST FILE SYSTEM & BROWSER 
// ========================================== 

void loadAvailableLogs() { 
  browserFileCount = 0; 
  File root = SD.open("/"); 

  while (true) { 

    File entry = root.openNextFile(); 

    if (!entry) { 

      break;  
    } 

    String name = entry.name(); 

    if (name.startsWith("LOG_") || name.startsWith("/LOG_")) { 

      int u = name.indexOf('_'); 
      int d = name.indexOf('.'); 
      if (u != -1 && d != -1) { 
        availableLogs[browserFileCount] = name.substring(u + 1, d).toInt(); 

        browserFileCount++; 
        if (browserFileCount >= 100) {  
          entry.close();  
          break;  

        } 

      } 
    } 
    entry.close(); 

  } 
  root.close(); 

   

  // Bubble Sort to keep files in exact numerical order 

  for (int i = 0; i < browserFileCount - 1; i++) { 
    for (int j = 0; j < browserFileCount - i - 1; j++) { 

      if (availableLogs[j] > availableLogs[j + 1]) { 
        int temp = availableLogs[j];
        availableLogs[j] = availableLogs[j + 1]; 
        availableLogs[j + 1] = temp; 

      } 
    } 
  } 

} 
 

void drawFileBrowser() { 
  display.clearDisplay(); 

  if (browserFileCount == 0) { 

    display.setCursor(0, 10); 

    display.print("No logs found."); 

  } else { 

    int startIdx = browserIndex - 1; 
    if (startIdx < 0) startIdx = 0; 
    if (startIdx > browserFileCount - 3 && browserFileCount > 3) startIdx = browserFileCount - 3; 

    for (int i = 0; i < 3; i++) { 
      int idx = startIdx + i; 
      if (idx < browserFileCount) { 
        display.setCursor(idx == browserIndex ? 0 : 8, i * 10); 

        if (idx == browserIndex) {  
          display.setTextColor(SSD1306_BLACK, SSD1306_WHITE);  
          display.print(">");  
        } else {  
          display.setTextColor(SSD1306_WHITE, SSD1306_BLACK);  
        } 

        char filename[16]; 
        sprintf(filename, "LOG_%03d.TXT", availableLogs[idx]); 
        display.print(filename); 
      } 
    } 
  } 
  display.display(); 
} 


// ========================================== 
// MEMORY-SAFE FILE VIEWER 
// ========================================== 

void openFileViewer() { 

  display.clearDisplay(); 
  display.setCursor(0, 10); 
  display.print("Opening..."); 
  display.display(); 
  File f = SD.open(viewerCurrentFile); 

  viewerTotalLines = 0; 

  if (f) { 

    // FAST LINE COUNTING: Reads raw bytes to prevent crashes 

    uint8_t buf[64]; 

    while (int r = f.read(buf, sizeof(buf))) { 

      for (int i = 0; i < r; i++) { 

        if (buf[i] == '\n') { 

          viewerTotalLines++; 

        } 

      } 

    } 

    f.close(); 

    viewerTotalLines--; // Exclude the header row 

    if (viewerTotalLines < 0) viewerTotalLines = 0; 

  } 

   

  viewerScrollPos = 0; 

  currentState = STATE_FILE_VIEWER; 

  drawFileViewer(); 

} 

 

void drawFileViewer() { 

  display.clearDisplay(); 

  display.setTextColor(SSD1306_WHITE, SSD1306_BLACK); 

   

  // Print Title Bar 

  display.setCursor(0, 0); 

  display.print(viewerCurrentFile.substring(1));  

  display.print(" (");  

  display.print(viewerTotalLines);  

  display.print(")"); 

   

  File f = SD.open(viewerCurrentFile); 

  if (f) { 

    int targetLine = viewerScrollPos + 1; // +1 to skip header 

    int currentLine = 0; 

     

    // Fast skip to current scrolling position 

    while(f.available() && currentLine < targetLine) { 

      if (f.read() == '\n') { 

        currentLine++; 

      } 

    } 

     

    // Render the 3 visible lines with Exact Formatting 

    for (int i = 0; i < MAX_LINES; i++) { 

      if (f.available()) { 

        String line = f.readStringUntil('\n');  

        line.trim(); 

         

        int comma1 = line.indexOf(','); 

        int comma2 = line.indexOf(',', comma1 + 1); 

         

        if (comma1 != -1 && comma2 != -1) { 

          // Exact Requested Format: R#:# lx %d:#% 

          display.setCursor(0, (i + 1) * 8); 

           

          display.print("R");  

          display.print(line.substring(0, comma1)); 

          display.print(":");  

          display.print(line.substring(comma1 + 1, comma2).toFloat(), 1); 

          display.print(" lx %d:");  

           

          float devVal = line.substring(comma2 + 1).toFloat(); 

          if (devVal >= 0) { 

            display.print("+"); 

          } 

          display.print(devVal, 1);  

          display.print("%"); 

        } 

      } 

    } 

    f.close(); 

  } 

  display.display(); 

} 

 

// ========================================== 

// ACTIVE LOGGING ENGINE 

// ========================================== 

void startNewLog() { 

  totalReadingsCount = 0;  

  totalLuxSum = 0.0;  

  currentAverage = 0.0; 

   

  for (int i = 0; i < MAX_LINES; i++) { 

    history[i].readNumber = 0; 

  } 

   

  // Find next available file slot 

  for (int i = 1; i <= 999; i++) { 

    char filename[16];  

    sprintf(filename, "/LOG_%03d.TXT", i); 

    if (!SD.exists(filename)) {  

      currentFileName = String(filename);  

      break;  

    } 

  } 

   

  logFile = SD.open(currentFileName, FILE_WRITE); 

   

  if (logFile) { 

    logFile.println("Read_Num, Lux_Value, Pct_Deviation"); 

    currentState = STATE_LOGGING;  

    display.clearDisplay();  

    display.display(); 

  } else {  

    displayMessage("SD Error!", 2000);  

  } 

} 

 

void takeReading() { 

  float currentLux = mySensor.getLux(); 

  totalReadingsCount++;  

   

  // Live Calibration 

  totalLuxSum += currentLux;  

  currentAverage = (float)(totalLuxSum / totalReadingsCount); 

   

  // Calculate static deviation for SD card 

  float snapDev = 0.0; 

  if (currentAverage > 0) { 

    snapDev = ((currentLux - currentAverage) / currentAverage) * 100.0; 

  } 

   

  // Write to SD 

  if (logFile) { 
    logFile.print(totalReadingsCount);  
    logFile.print(","); 
    logFile.print(currentLux, 2);  
    logFile.print(","); 
    logFile.println(snapDev, 2);  
    logFile.flush();  

  } 

   

  // Shift Ladder History 

  for (int i = 0; i < MAX_LINES - 1; i++) { 
    history[i] = history[i + 1]; 
  } 

  history[MAX_LINES - 1].readNumber = totalReadingsCount; 
  history[MAX_LINES - 1].luxValue = currentLux; 

   

  // Update UI Screen 
  display.clearDisplay();  
  display.setTextColor(SSD1306_WHITE); 

   

  // Line 1: Calibration 

  display.setCursor(0, 0);  
  display.print("CAL AVG: ");  
  display.print(currentAverage, 1);  
  display.print(" lx"); 

   

  // Lines 2-4: The History (Live Recalculation) 
  for (int i = 0; i < MAX_LINES; i++) { 
    if (history[i].readNumber > 0) { 
      display.setCursor(0, (i + 1) * 8);  

      // Exact Requested Format: R#:# lx %d:#% 

      display.print("R");  
      display.print(history[i].readNumber); 
      display.print(":");  
      display.print(history[i].luxValue, 1); 
      display.print(" lx %d:");  

       

      // Live Recalculation Engine 
      float liveDev = ((history[i].luxValue - currentAverage) / currentAverage) * 100.0; 
      if (liveDev >= 0) { 

        display.print("+"); 

      } 
      display.print(liveDev, 1);  
      display.print("%"); 
    } 
  } 
  display.display();  
}  

// ========================================== 
// FAST SD WIPE 
// ========================================== 

void clearSDCard() { 
  display.clearDisplay();  
  display.setCursor(0, 10);  
  display.print("Wiping Logs...");  
  display.display(); 

  // Use directory reader instead of guessing 
  loadAvailableLogs(); 

  for (int i = 0; i < browserFileCount; i++) { 
    char filename[16];  
    sprintf(filename, "/LOG_%03d.TXT", availableLogs[i]); 
    SD.remove(filename); 
  } 
  browserFileCount = 0; 
} 

// ========================================== 
// MESSAGE POPUP 
// ========================================== 
void displayMessage(String msg, int duration) { 
  tempMessage = msg;  
  messageStartTime = millis();  
  currentState = STATE_MESSAGE; 
} 

 
