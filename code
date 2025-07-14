#include <Arduino.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <EEPROM.h>
#include <WiFi.h>
#include <pgmspace.h>
#include <JQ6500_Serial.h>

// MP3 Setup
JQ6500_Serial mp3(Serial2);
const byte mp3Volume = 27;
const int mp3RxPin = 25;
const int mp3TxPin = 26;

// LCD Setup
const int sdaPin = 21;
const int sclPin = 22;
LiquidCrystal_I2C lcd(0x27, 16, 2);
bool lcdReady = false;

// Button Pins
const int modeButton = 15;
const int nextButton = 17;
const int prevButton = 18;
const int selectButton = 19;
const int finalizeButton = 16;

// Mode States
enum Mode { LINE_SELECT, ROUTE_SELECT, STATION_SELECT, FINALIZED };
enum Line { UP_LINE, DOWN_LINE };
Mode currentMode = LINE_SELECT;
int currentRoute = 0; // 0: Kalyan, 1: Kasara, 2: Khopoli
const char* routeNames[] = {"Kalyan", "Kasara", "Khopoli"};
int stationIndex = 0;
int haltIndex = 0;
Line currentLine = UP_LINE;
bool firstHalt = true;

// Halt Indices
uint16_t upKalyanHalts[30];
uint16_t upKasaraHalts[30];
uint16_t upKhopoliHalts[30];
uint16_t downKalyanHalts[30];
uint16_t downKasaraHalts[30];
uint16_t downKhopoliHalts[30];
int upKalyanCount = 0;
int upKasaraCount = 0;
int upKhopoliCount = 0;
int downKalyanCount = 0;
int downKasaraCount = 0;
int downKhopoliCount = 0;

// Station Lists
const char* mainStations[] = {"CSMT", "Byculla", "Parel", "Dadar", "Matunga", "Kurla", "Ghatkopar", "Vikhroli", "Bhandup", "Mulund", "Thane", "Kalva", "Mumbra", "Diva Jn", "Dombivli", "Kalyan Jn"};
const char* northeastStations[] = {"CSMT", "Byculla", "Parel", "Dadar", "Matunga", "Kurla", "Ghatkopar", "Vikhroli", "Bhandup", "Mulund", "Thane", "Kalva", "Mumbra", "Diva Jn", "Dombivli", "Kalyan Jn", "Shahad", "Ambivli", "Titwala", "Khadavli", "Vasind", "Asangaon", "Atgaon", "Thansit", "Khardi", "Umbermali", "Kasara"};
const char* southeastStations[] = {"CSMT", "Byculla", "Parel", "Dadar", "Matunga", "Kurla", "Ghatkopar", "Vikhroli", "Bhandup", "Mulund", "Thane", "Kalva", "Mumbra", "Diva Jn", "Dombivli", "Kalyan Jn", "Vithalwadi", "Ulhasnagar", "Ambernath", "Badlapur", "Vangani", "Shelu", "Neral Junction", "Bhivpuri Road", "Karjat", "Palasdari", "Kelavli", "Dolavli", "Lowjee", "Khopoli"};
const int mainCount = sizeof(mainStations) / sizeof(mainStations[0]);
const int northeastCount = sizeof(northeastStations) / sizeof(northeastStations[0]);
const int southeastCount = sizeof(southeastStations) / sizeof(southeastStations[0]);

// EEPROM Setup
const int maxHalts = 30;
const int eepromSize = 372; // Adjusted for 6 halt blocks
const int upKalyanAddr = 0;
const int upKasaraAddr = 62;
const int upKhopoliAddr = 124;
const int downKalyanAddr = 186;
const int downKasaraAddr = 248;
const int downKhopoliAddr = 310;

// MP3 Mapping (Simplified subset for brevity)
const char stationFiles[][15] PROGMEM = {"Ambernath", "Ambivli", "Asangaon", "Atgaon", "Badlapur", "Bhandup", "Bhivpuri Road", "Byculla", "CSMT", "Dadar", "Diva Jn", "Dolavli", "Dombivli", "Ghatkopar", "Kalva", "Kalyan Jn", "Karjat", "Kasara", "Kelavli", "Khadavli", "Khardi", "Khopoli", "Kurla", "Lowjee", "Matunga", "Mulund", "Mumbra", "Neral Junction", "Palasdari", "Parel", "Shahad", "Shelu", "Thane", "Thansit", "Titwala", "Ulhasnagar", "Umbermali", "Vangani", "Vasind", "Vikhroli", "Vithalwadi"};
const int fileNumbers[] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41};
const int numFiles = sizeof(stationFiles) / sizeof(stationFiles[0]);

// Wi-Fi Variables
const char* targetSSID = "RailWire";
unsigned long wifiStartTime = 0;
bool wifiLongEnough = false;
bool wifiDetected = false;
unsigned long lastScanTime = 0;
const unsigned long scanInterval = 2000;
const unsigned long holdTime = 35000;
const unsigned long gracePeriod = 4000;
bool wifiMsgActive = false;
unsigned long wifiMsgTime = 0;
const unsigned long wifiMsgDur = 1000;
bool pendingAnnounce = false;
unsigned long announceDelay = 0;
const unsigned long announceDelayTime = 4000;
String nextStation = "";

// Button Variables
const unsigned long debounceTime = 5;
const unsigned long longPressTime = 2000;
unsigned long lastPressTime = 0;
int lastModeState = HIGH;
int lastNextState = HIGH;
int lastPrevState = HIGH;
int lastSelectState = HIGH;
int lastFinalizeState = HIGH;

// Message Variables
bool msgActive = false;
unsigned long msgTime = 0;
String currentMsg = "";
const unsigned long msgDur = 1000;

// Announcement Variables
unsigned long lastAnnounceTime = 0;
String playingStation = "";
const unsigned long announceInterval = 60000;
bool mp3Ready = false;

String getStationName(int fileIdx) {
  for (int i = 0; i < numFiles; i++) if (fileNumbers[i] == fileIdx) {
    char buffer[15];
    strcpy_P(buffer, (char*)pgm_read_word(&(stationFiles[i])));
    return String(buffer);
  }
  return "";
}

int getFileIndex(String station) {
  for (int i = 0; i < numFiles; i++) {
    char buffer[15];
    strcpy_P(buffer, (char*)pgm_read_word(&(stationFiles[i])));
    if (String(buffer) == station) return fileNumbers[i];
  }
  return 0;
}

String getCurrentStation() {
  const char* stations[] = {mainStations, northeastStations, southeastStations};
  int counts[] = {mainCount, northeastCount, southeastCount};
  int idx = currentRoute;
  int maxIdx = counts[idx] - 1;
  return (currentLine == UP_LINE) ? String(stations[idx][stationIndex]) : String(stations[idx][maxIdx - stationIndex]);
}

int getStationIndex(String station, int route) {
  const char* stations[] = {mainStations, northeastStations, southeastStations};
  int counts[] = {mainCount, northeastCount, southeastCount};
  for (int i = 0; i < counts[route]; i++){
    if (String(stations[route][i]) == station) return i;
  }
  return -1;
}

uint16_t* getHaltList() {
  if (currentLine == UP_LINE){
    return (currentRoute == 0) ? upKalyanHalts : (currentRoute == 1) ? upKasaraHalts : upKhopoliHalts;
  } 
  else return (currentRoute == 0) ? downKalyanHalts : (currentRoute == 1) ? downKasaraHalts : downKhopoliHalts;
}

int& getHaltCount() {
  if (currentLine == UP_LINE) {
    return (currentRoute == 0) ? upKalyanCount : (currentRoute == 1) ? upKasaraCount : upKhopoliCount;
  }
  else return (currentRoute == 0) ? downKalyanCount : (currentRoute == 1) ? downKasaraCount : downKhopoliCount;
}

int getEepromAddr() {
  if (currentLine == UP_LINE) {
    return (currentRoute == 0) ? upKalyanAddr : (currentRoute == 1) ? upKasaraAddr : upKhopoliAddr;
  }
  else return (currentRoute == 0) ? downKalyanAddr : (currentRoute == 1) ? downKasaraAddr : downKhopoliAddr;
}

bool initMp3() {
  Serial2.begin(9600, SERIAL_8N1, mp3RxPin, mp3TxPin);
  for (int i = 0; i < 3; i++) {
    mp3.reset(); delay(100);
    mp3.setVolume(mp3Volume); mp3.setSource(MP3_SRC_BUILTIN);
    mp3.setLoopMode(MP3_LOOP_ONE_STOP); mp3.pause();
    if (mp3.getStatus() != MP3_STATUS_STOPPED) return true;
    delay(200);
  }
  return false;
}

void playAnnouncement(String station, bool pending) {
  if (!mp3Ready || station == "" || station == "Unknown") { mp3.pause(); playingStation = ""; return; }
  int fileNum = getFileIndex(station);
  if (fileNum < 1 || fileNum > numFiles) { displayMessage("No Audio"); mp3.pause(); playingStation = ""; return; }
  mp3.pause(); playingStation = station; mp3.playFileByIndexNumber(fileNum);
  delay(200); if (mp3.getStatus() != MP3_STATUS_PLAYING) { playingStation = ""; return; }
  lastAnnounceTime = millis();
}

int findHaltIndex(int idx) {
  uint16_t* list = getHaltList(); int count = getHaltCount();
  for (int i = 0; i < count; i++) if (list[i] == idx) return i;
  return -1;
}

void toggleHalt(String station) {
  int idx = getStationIndex(station, currentRoute);
  if (idx == -1) return;
  uint16_t* list = getHaltList(); int& count = getHaltCount();
  int pos = findHaltIndex(idx);
  if (pos == -1 && count < maxHalts) {
    int insertPos = 0;
    for (int i = 0; i < count; i++) {
      if ((currentLine == UP_LINE ? list[i] > idx : list[i] < idx)) insertPos = i + 1;
    }
    for (int i = count; i > insertPos; i--) list[i] = list[i - 1];
    list[insertPos] = idx; count++; displayMessage("Added: " + station);
  } else if (pos != -1) {
    for (int i = pos; i < count - 1; i++) list[i] = list[i + 1];
    list[count - 1] = 0; count--; displayMessage("Removed: " + station);
  }
}

void saveHalts() {
  int addr = getEepromAddr(); int count = getHaltCount(); uint16_t* list = getHaltList();
  EEPROM.write(addr, count); addr += 2;
  for (int i = 0; i < maxHalts; i++) {
    int val = (i < count) ? list[i] : 0;
    EEPROM.write(addr++, val & 0xFF); EEPROM.write(addr++, (val >> 8) & 0xFF);
  }
  if (EEPROM.commit()) Serial.println("Halt saved");
  else displayMessage("Save Failed");
}

void loadHalts() {
  int addr = getEepromAddr(); int& count = getHaltCount(); uint16_t* list = getHaltList();
  count = EEPROM.read(addr); addr += 2;
  if (count < 0 || count > maxHalts) count = 0;
  for (int i = 0; i < count; i++) {
    int val = EEPROM.read(addr++) | (EEPROM.read(addr++) << 8);
    if (val < (currentRoute == 0 ? mainCount : currentRoute == 1 ? northeastCount : southeastCount)) list[i] = val;
  }
  for (int i = count; i < maxHalts; i++) list[i] = 0;
}

void displayMsg(String msg) {
  if (!lcdReady) return;
  msgActive = true; msgTime = millis(); currentMsg = msg;
  lcd.clear(); lcd.setCursor(0, 0); lcd.print(msg.length() > 16 ? msg.substring(0, 16) : msg);
}

void displayStation() {
  if (!lcdReady) return;
  lcd.clear();
  if (currentMode == FINALIZED) {
    uint16_t* list = getHaltList(); int count = getHaltCount();
    if (count > 0) {
      lcd.setCursor(0, 0); lcd.print("Halt " + String(haltIndex + 1) + "/" + String(count));
      lcd.setCursor(0, 1); lcd.print(getStationFromIndex(list[haltIndex]));
    } else { lcd.setCursor(0, 0); lcd.print("No Halts"); }
  } else if (currentMode == STATION_SELECT) {
    String station = getCurrentStation();
    lcd.setCursor(0, 0); lcd.print(station);
    if (findHaltIndex(stationIndex) != -1) { lcd.setCursor(15, 0); lcd.print("*"); }
    lcd.setCursor(0, 1); lcd.print("Halts: " + String(getHaltCount()));
  }
}

void showLineSelect() { if (lcdReady) { lcd.clear(); lcd.print("Choose Line"); lcd.setCursor(0, 1); lcd.print("Next=UP Prev=DN"); } }
void showRouteSelect() { if (lcdReady) { lcd.clear(); lcd.print("Route: " + String(routeNames[currentRoute])); } }

void scanWifi() {
  if (currentMode != FINALIZED || millis() - lastScanTime < scanInterval) return;
  WiFi.mode(WIFI_STA); WiFi.disconnect(); delay(50);
  WiFi.scanNetworks(true, true); lastScanTime = millis();
}

void processWifi() {
  int result = WiFi.scanComplete();
  if (result <= 0) { WiFi.scanDelete(); return; }
  bool found = false;
  for (int i = 0; i < result; i++) if (WiFi.SSID(i) == targetSSID) found = true;
  WiFi.scanDelete();
  unsigned long now = millis();
  if (found) {
    if (!wifiDetected) { wifiStartTime = now; wifiDetected = true; }
    else if (now - wifiStartTime >= holdTime && !wifiLongEnough) {
      wifiLongEnough = true; wifiMsgActive = true; wifiMsgTime = now;
      lcd.clear(); lcd.print("At Station"); mp3.pause(); playingStation = "";
    }
  } else if (wifiDetected && wifiLongEnough) {
    wifiDetected = false; wifiLongEnough = false; wifiStartTime = 0;
    wifiMsgActive = true; wifiMsgTime = now;
    uint16_t* list = getHaltList(); int count = getHaltCount();
    if (count > 0) { haltIndex = (haltIndex + 1) % count; lcd.clear(); lcd.print("To Next Halt"); }
    else { lcd.clear(); lcd.print("No Halts"); }
    pendingAnnounce = true; announceDelay = now; nextStation = getStationFromIndex(list[haltIndex]);
  }
}

bool scanI2C() {
  Wire.begin(sdaPin, sclPin);
  for (uint8_t addr = 0x20; addr <= 0x3F; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0 && (addr == 0x27 || addr == 0x3F)) return true;
  }
  return false;
}

void setup() {
  Serial.begin(115200); delay(1000);
  pinMode(modeButton, INPUT_PULLUP); pinMode(nextButton, INPUT_PULLUP);
  pinMode(prevButton, INPUT_PULLUP); pinMode(selectButton, INPUT_PULLUP);
  pinMode(finalizeButton, INPUT_PULLUP);
  for (int i = 0; i < 3; i++) if (scanI2C()) { lcd.init(); lcd.backlight(); lcdReady = true; break; delay(1000); }
  if (lcdReady) { lcd.print("WELCOME"); delay(1000); }
  EEPROM.begin(eepromSize);
  mp3Ready = initMp3();
  WiFi.mode(WIFI_STA); WiFi.disconnect(); delay(100);
  showLineSelect();
}

void loop() {
  unsigned long now = millis();
  if (msgActive && now - msgTime >= msgDur) { msgActive = false; displayStation(); }
  if (wifiMsgActive && now - wifiMsgTime >= wifiMsgDur) { wifiMsgActive = false; displayStation(); }

  int modeState = digitalRead(modeButton);
  int nextState = digitalRead(nextButton);
  int prevState = digitalRead(prevButton);
  int selectState = digitalRead(selectButton);
  int finalizeState = digitalRead(finalizeButton);

  if (modeState == LOW && now - lastModeTime > debounceTime && currentMode == ROUTE_SELECT) {
    currentRoute = (currentRoute + 1) % 3; showRouteSelect(); lastModeTime = now;
  }
  if (selectState == LOW && now - lastSelectTime > debounceTime) {
    if (currentMode == ROUTE_SELECT) { currentMode = STATION_SELECT; loadHalts(); stationIndex = (currentLine == UP_LINE) ? 0 : (currentRoute == 0 ? mainCount - 1 : currentRoute == 1 ? northeastCount - 1 : southeastCount - 1); }
    else if (currentMode == STATION_SELECT) toggleHalt(getCurrentStation());
    lastSelectTime = now; displayStation();
  }
  if (nextState == LOW && now - lastNextTime > debounceTime) {
    if (currentMode == LINE_SELECT) { currentLine = UP_LINE; currentMode = ROUTE_SELECT; currentRoute = 0; showRouteSelect(); }
    else if (currentMode == STATION_SELECT) {
      int maxIdx = (currentRoute == 0) ? mainCount - 1 : (currentRoute == 1) ? northeastCount - 1 : southeastCount - 1;
      stationIndex = (currentLine == UP_LINE) ? (stationIndex + 1) % maxIdx : (stationIndex > 0) ? stationIndex - 1 : maxIdx;
    }
    lastNextTime = now; displayStation();
  }
  if (prevState == LOW && now - lastPrevTime > debounceTime) {
    if (currentMode == LINE_SELECT) { currentLine = DOWN_LINE; currentMode = ROUTE_SELECT; currentRoute = 0; showRouteSelect(); }
    else if (currentMode == STATION_SELECT) {
      int maxIdx = (currentRoute == 0) ? mainCount - 1 : (currentRoute == 1) ? northeastCount - 1 : southeastCount - 1;
      stationIndex = (currentLine == UP_LINE) ? (stationIndex > 0) ? stationIndex - 1 : maxIdx : (stationIndex + 1) % maxIdx;
    }
    lastPrevTime = now; displayStation();
  }
  if (finalizeState == LOW && now - lastFinalizeTime > debounceTime) {
    if (now - finalizePressStart >= longPressTime) {
      currentMode = LINE_SELECT; currentRoute = 0; stationIndex = 0; haltIndex = 0; firstHalt = true;
      upKalyanCount = upKasaraCount = upKhopoliCount = downKalyanCount = downKasaraCount = downKhopoliCount = 0;
      for (int i = 0; i < maxHalts; i++) upKalyanHalts[i] = upKasaraHalts[i] = upKhopoliHalts[i] = downKalyanHalts[i] = downKasaraHalts[i] = downKhopoliHalts[i] = 0;
      mp3.pause(); playingStation = ""; showLineSelect();
    } else if (finalizeState == HIGH && !haltFinalized) {
      saveHalts(); currentMode = FINALIZED; haltIndex = 0; firstHalt = true; loadHalts(); displayMsg("Route Set!");
    }
    lastFinalizeTime = now;
  }

  if (pendingAnnounce && now - announceDelay >= announceDelayTime && playingStation == "") {
    if (!firstHalt) playAnnouncement(nextStation, true); pendingAnnounce = false;
  }
  if (currentMode == FINALIZED && playingStation == "" && !wifiLongEnough) {
    scanWifi(); processWifi();
    if (now - lastAnnounceTime >= announceInterval) {
      uint16_t* list = getHaltList(); int count = getHaltCount();
      if (count > 0) playAnnouncement(getStationFromIndex(list[haltIndex]), false);
    }
  }
  delay(10);
}
