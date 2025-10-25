# Smart Parking IoT — Fully Annotated Study Guide


## The guide includes

- 📖 Purpose explanations for each file/function
- 💡 Step-by-step code walkthroughs with inline comments
- 🎯 Real-world examples showing how code works
- 🔍 Technical details (time complexity, algorithms, etc.)
- 🚀 Visual diagrams in comments


This guide walks through your **entire project** and explains every major command, function, and file in a **study-friendly** way.  
You'll see, for each file:

- **What the file does (Overview)**
- **Key concepts & APIs used**
- **Annotated code** — the original code with added commentary blocks so you can quickly revise.

> ⚠️ Note: This is a learning document, not a build artifact. The code blocks include extra comments that you should remove before compiling.


## Comment Legend


- **Arduino/C++/JavaScript** comments start with `// ...`
- **Python** comments start with `# ...`
- **HTML** comments use `<!-- ... -->`
- **CSS** comments use `/* ... */`


## SpotNode.ino

#### File explanation
ESP32 firmware for a single spot: handles Wi‑Fi, sensor reading, LED control, and RTDB sync with debounce and stream/poll fallbacks.



**Overview.** ESP32 spot firmware: WiFiManager for provisioning, Firebase RTDB stream+poll, ultrasonic sensor reads,
RGB LED states (FREE/WAITING/OCCUPIED), debounce, and RTDB sync under `/SPOTS`.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



cpp// VISUAL DIAGRAM — High-Level Flow
// ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
//   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
// ESP32 spot firmware: WiFiManager for provisioning, Firebase RTDB stream+poll, ultrasonic sensor reads,
// RGB LED states (FREE/WAITING/OCCUPIED), debounce, and RTDB sync under `/SPOTS`.

#include <WiFi.h>
// Library import/include
#include <WiFiManager.h>          // by tzapu
// Library import/include
#include <Firebase_ESP_Client.h>
// Library import/include
#include "addons/TokenHelper.h"
// Library import/include
#include "addons/RTDBHelper.h"
// Library import/include

// ===================== Wi-Fi Status LED & BOOT button ======================
const int WIFI_LED_PIN     = 2;    // built-in LED on most ESP32 dev boards
// Constant/configuration
const bool LED_ACTIVE_LOW  = false;// set true if LOW turns LED ON on your board
// Constant/configuration
const int  BTN_PIN         = 0;    // BOOT button (GPIO0)
// Constant/configuration
const uint32_t LONGPRESS_MS = 3000;
// Constant/configuration

WiFiManager wm;
bool portalActive = false;

void wifiLedWrite(bool on) {
// Function start
  int level = on
      ? (LED_ACTIVE_LOW ? LOW : HIGH)
      : (LED_ACTIVE_LOW ? HIGH : LOW);
  pinMode(WIFI_LED_PIN, OUTPUT);
  digitalWrite(WIFI_LED_PIN, level);
}

void setWifiLedConnected(bool connected) {
// Function start
  wifiLedWrite(connected); // ON when connected, OFF otherwise
}

// ===================== Your original config ================================
#define API_KEY      "AIzaSyBCg3n1wtcmBncNHEfxjL7PT5hXlEJB4TE"
// Constant/configuration
#define DATABASE_URL "https://park-19f5b-default-rtdb.europe-west1.firebasedatabase.app"
// Constant/configuration
const char* SPOT_PATH = "/SondosPark/SPOTS/0,0";
// Constant/configuration

// LED pins (RGB)
const int LED_R = 23;
// Constant/configuration
const int LED_G = 22;
// Constant/configuration
const int LED_B = 21;
// Constant/configuration
const bool LED_COMMON_ANODE = false;
// Constant/configuration

// Ultrasonic pins
const int TRIG_PIN = 5;
// Constant/configuration
const int ECHO_PIN = 18;
// Constant/configuration

// Distance thresholds
const int THRESH_ENTER = 12;
// Constant/configuration
const int THRESH_EXIT  = 18;
// Constant/configuration

// Firebase globals
FirebaseData fbdo;
FirebaseData fbStream;
FirebaseAuth auth;
FirebaseConfig config;
String currentStatus = "UNKNOWN";  // Real status from DB

// ===================== LED (spot status) ===================================
void setRgb(bool r, bool g, bool b) {
// Function start
  auto drive = [&](int pin, bool on) {
    digitalWrite(pin, LED_COMMON_ANODE ? !on : on);
  };
  drive(LED_R, r);
  drive(LED_G, g);
  drive(LED_B, b);
}

void showStatusOnLed(const String& s) {
// Function start
  if (s == "FREE")          setRgb(false, false, true);   // Blue
// Control flow
  else if (s == "WAITING")  setRgb(true,  true,  false);  // Orange (R+G)
  else if (s == "OCCUPIED") setRgb(true,  false, false);  // Red
  else                      setRgb(false, false, false);  // Off
}

// ===================== Sensor ==============================================
long readDistanceCM() {
// Function start
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  if (duration == 0) return -1;
// Control flow
  return duration / 58;
}

// ===================== WiFi via WiFiManager ================================
void wifiInitWithPortal() {
// Function start
  WiFi.mode(WIFI_STA);
  setWifiLedConnected(false);

  Serial.println("* WiFi: trying autoConnect()");
  bool ok = wm.autoConnect("Sondos-Parking-Setup", "12345678"); // opens portal if no saved Wi-Fi
  if (!ok) {
// Control flow
    Serial.println("⚠️  Portal cancelled/failed. Rebooting…");
    delay(1000);
    ESP.restart();
  }

  // Connected
  Serial.print("✅ WiFi connected. IP: ");
  Serial.println(WiFi.localIP());
  setWifiLedConnected(true);
}

// Long-press BOOT at runtime → reset creds + reopen portal
void handleLongPressToReconfigure() {
// Function start
  static bool wasDown = false;
  static uint32_t downAt = 0;

  bool down = (digitalRead(BTN_PIN) == LOW);
  if (down && !wasDown) downAt = millis();
// Control flow

  if (down && (millis() - downAt > LONGPRESS_MS)) {
// Control flow
    Serial.println("🔁 Long-press → reset Wi-Fi & open portal");
    wm.resetSettings();
    WiFi.disconnect(true, true);
    delay(200);

    portalActive = true;
    setWifiLedConnected(false); // OFF while not connected

    // Blocking until user saves Wi-Fi in portal
    wm.startConfigPortal("Sondos-Parking-Setup", "12345678");

    // Reconnected
    portalActive = false;
    setWifiLedConnected(true);
    Serial.print("✅ Reconnected. IP: ");
    Serial.println(WiFi.localIP());
    Serial.println("👍 Saved again. Next reset will auto-connect.");

    while (digitalRead(BTN_PIN) == LOW) delay(10); // wait release
// Control flow
  }
  wasDown = down;

  // Blink Wi-Fi LED while portal is active
  if (portalActive) {
// Control flow
    static unsigned long t = 0;
    static bool on = false;
    if (millis() - t > 400) {
// Control flow
      t = millis();
      on = !on;
      wifiLedWrite(on);
    }
  }
}

// ===================== Firebase ============================================
void streamCallback(FirebaseStream data) {
// Function start
  if (data.dataTypeEnum() == fb_esp_rtdb_data_type_string) {
// Control flow
    currentStatus = data.stringData();
    currentStatus.trim();
    Serial.print("[STREAM] status -> "); Serial.println(currentStatus);
    showStatusOnLed(currentStatus);
  }
}

void streamTimeoutCallback(bool timeout) {
// Function start
  if (timeout) Serial.println("[STREAM] timeout, trying to resume...");
// Control flow
}

bool firebaseAuthInit() {
// Function start
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;
// RTDB path / configuration
  config.token_status_callback = tokenStatusCallback;

  bool ok = Firebase.signUp(&config, &auth, "", ""); // anonymous
  if (!ok) {
// Control flow
    Serial.print("Auth error: ");
    Serial.println(config.signer.signupError.message.c_str());
  } else {
    Serial.println("Anonymous sign-in success.");
  }

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
  return ok;
}

bool startStream() {
// Function start
  String statusPath = String(SPOT_PATH) + "/status";
  if (!Firebase.RTDB.beginStream(&fbStream, statusPath)) {
// Control flow
    Serial.print("Stream error: ");
    Serial.println(fbStream.errorReason());
    return false;
  }
  Firebase.RTDB.setStreamCallback(&fbStream, streamCallback, streamTimeoutCallback);
  return true;
}

void publishStatus(const String& s) {
// Function start
  if (!Firebase.ready()) return;
// Control flow
  String base = String(SPOT_PATH);
  if (!Firebase.RTDB.setString(&fbdo, base + "/status", s)) {
// Control flow
    Serial.print("FB set(status) error: ");
    Serial.println(fbdo.errorReason());
  }
  Firebase.RTDB.setInt(&fbdo, base + "/lastUpdateMs", millis());
}

// ========== Poll DB (fallback if stream fails) ==========
unsigned long lastPoll = 0;
const unsigned long POLL_INTERVAL = 3000;
// Constant/configuration

void pollStatusFromDB() {
// Function start
  if (!Firebase.ready()) return;
// Control flow
  String path = String(SPOT_PATH) + "/status";
  if (Firebase.RTDB.getString(&fbdo, path)) {
// Control flow
    String newStatus = fbdo.stringData();
    newStatus.trim();
    if (newStatus != currentStatus) {
// Control flow
      currentStatus = newStatus;
      Serial.print("[POLL] status -> "); Serial.println(currentStatus);
      showStatusOnLed(currentStatus);
    }
  }
}

// ===================== Arduino Setup =======================================
String lastDesired = "UNKNOWN";
unsigned long lastChangeTime = 0;
const unsigned long STABLE_TIME = 3000; // 3 seconds
// Constant/configuration

void setup() {
// Function start
  Serial.begin(115200);

  // IO
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LED_R, OUTPUT);
  pinMode(LED_G, OUTPUT);
  pinMode(LED_B, OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(WIFI_LED_PIN, OUTPUT);

  showStatusOnLed("UNKNOWN");
  setWifiLedConnected(false);

  // Wi-Fi via WiFiManager (no hardcoded SSID/PASS)
  wifiInitWithPortal();

  // Firebase
  firebaseAuthInit();
  while (!Firebase.ready()) delay(50);
// Control flow
  startStream();

  Serial.println("ℹ️  Hold BOOT button ~3s to re-open the Wi-Fi portal.");
}

// ===================== Loop ================================================
void loop() {
// Function start
  // BOOT long-press handler + portal blink
  handleLongPressToReconfigure();

  // 1. Sensor reading
  long d = readDistanceCM();
  Serial.print("Distance: "); Serial.println(d);

  // 2. Determine desired status
  String desired = currentStatus;

  if (currentStatus == "WAITING") {
// Control flow
    if (d > 0 && d < THRESH_ENTER) {
// Control flow
      desired = "OCCUPIED";
    } else {
      desired = currentStatus; // stay WAITING
    }
  } else {
    if (d > 0 && d < THRESH_ENTER) {
// Control flow
      desired = "OCCUPIED";
    } else if (d > THRESH_EXIT) {
      desired = "FREE";
    }
  }

  // 3. Debounce/stabilize for 3 seconds
  if (desired != lastDesired) {
// Control flow
    lastDesired = desired;
    lastChangeTime = millis();
  }

  // 4. Update Firebase if stable and changed
  if ((millis() - lastChangeTime > STABLE_TIME) && desired != currentStatus && Firebase.ready()) {
// Control flow
    currentStatus = desired;
    publishStatus(currentStatus);
    showStatusOnLed(currentStatus);
  }

  // 5. Poll DB every 3 sec (fallback)
  if (millis() - lastPoll > POLL_INTERVAL) {
// Control flow
    pollStatusFromDB();
    lastPoll = millis();
  }

  delay(400);
}
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## setup_simulation.py

#### File explanation
Bootstraps local simulation with credentials and connectivity checks.



**Overview.** Connectivity and bootstrap checks for RTDB access.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# Connectivity and bootstrap checks for RTDB access.

import threading
# Library import/include
import time
# Library import/include
from firebase_admin import db
# Library import/include
from firebase_init import db as _db_init
# Library import/include
from constants import ROOT_BRANCH, STAT_WAIT, STAT_OCC
# Library import/include
import argparse
# Library import/include
import sys
# Library import/include

BASE = db.reference(ROOT_BRANCH)
# Constant/configuration
SPOTS = BASE.child("SPOTS")
# Constant/configuration
CARS = BASE.child("CARS")
# Constant/configuration

def _on_spots(event):
# Function start
    print("[SPOTS EVENT]", event.event_type, event.path, "->", str(event.data)[:120])
# RTDB path / configuration

def _on_cars(event):
# Function start
    print("[CARS EVENT]", event.event_type, event.path, "->", str(event.data)[:120])
# RTDB path / configuration

def start_listener(block_forever=True):
# Function start
    s_stream = SPOTS.listen(_on_spots)
# RTDB path / configuration
    c_stream = CARS.listen(_on_cars)
# RTDB path / configuration

    print("[Listener] Attached to:")
    print(f"  /{ROOT_BRANCH}/SPOTS")
# RTDB path / configuration
    print(f"  /{ROOT_BRANCH}/CARS")
# RTDB path / configuration

    try:
# Control flow
        if block_forever:
# Control flow
            threading.Event().wait()
    finally:
        s_stream.close()
        c_stream.close()

def check_firebase_connection(timeout=5):
# Function start
    """Attempt a single read of the ROOT_BRANCH with a timeout.
# RTDB path / configuration
    Returns (True, data) on success or (False, error_message) on failure/timeout.
    """
    evt = threading.Event()
    result = {"ok": False, "data": None, "error": None}

    def target():
# Function start
        try:
# Control flow
            data = BASE.get()
            result["ok"] = True
            result["data"] = data
        except Exception as e:
# Control flow
            result["error"] = str(e)
        finally:
            evt.set()

    t = threading.Thread(target=target, daemon=True)
    t.start()
    if not evt.wait(timeout):
# Control flow
        return False, f"Timeout after {timeout}s while reading /{ROOT_BRANCH}"
# RTDB path / configuration
    if result["ok"]:
# Control flow
        return True, result["data"]
    return False, result["error"]

if __name__ == "__main__":
# Control flow
    parser = argparse.ArgumentParser(description="Setup simulation listener / connectivity check")
    parser.add_argument("--test", action="store_true", help="Only run connectivity test and exit")
    parser.add_argument("--listen", action="store_true", help="Start the listener (will block)")
    args = parser.parse_args()

    ok, info = check_firebase_connection(timeout=5)
    if not ok:
# Control flow
        print("[Firebase Test] FAILED:", info)
        # If user only wanted to test, exit with non-zero
        if args.test or not args.listen:
# Control flow
            sys.exit(1)
        # otherwise proceed to try attaching listener (may still fail)
        print("[Firebase Test] Proceeding to attach listener despite test failure...")
    else:
        print("[Firebase Test] OK - root data preview:", str(info)[:200])

    if args.test and not args.listen:
# Control flow
        # test only requested
        sys.exit(0)

    # default behavior: start listener (after test)
    start_listener(block_forever=True)
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## constants.py

#### File explanation
Centralized constants for DB paths, statuses, and timing.



**Overview.** Central constants: RTDB branch, statuses, intervals, thresholds.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# Central constants: RTDB branch, statuses, intervals, thresholds.

ROOT_BRANCH = "SondosPark"
# Constant/configuration
STAT_FREE = "FREE"
# Constant/configuration
STAT_WAIT = "WAITING"
# Constant/configuration
STAT_OCC = "OCCUPIED"
# Constant/configuration
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## data_structures.py

#### File explanation
In-memory model and algorithms (sorted free list, BFS for closest).



**Overview.** Parking lot in-memory model: free list (sorted by distance), BFS for closest.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# Parking lot in-memory model: free list (sorted by distance), BFS for closest.

from typing import Dict, Optional, Tuple, Callable
# Library import/include
import random
# Library import/include
from bisect import bisect_left, insort
# Library import/include
from collections import deque
# Library import/include

class SortedList:
    """A small sorted list with optional key function. Compatible with previous API.

    If key is provided, items are ordered by key(item). Internally stores
    tuples (k, item) when key is set to allow use of insort.
    """
    def __init__(self, iterable=None, key: Optional[Callable] = None):
# Function start
        self._key = key
        self._list = []
        if iterable:
# Control flow
            for item in iterable:
# Control flow
                self.add(item)

    def _wrap(self, value):
# Function start
        return (self._key(value), value) if self._key is not None else value

    def add(self, value):
# Function start
        if self._key is None:
# Control flow
            insort(self._list, value)
        else:
            # Ensure internal storage uses (key, item) tuples
            if self._list and not isinstance(self._list[0], tuple):
# Control flow
                self._list = [(self._key(v), v) for v in self._list]
            k = self._key(value)
            # build keys list for bisect search to avoid comparing items directly
            keys = [kv[0] for kv in self._list]
            idx = bisect_left(keys, k)
            self._list.insert(idx, (k, value))

    def remove(self, value):
# Function start
        if self._key is None:
# Control flow
            idx = bisect_left(self._list, value)
            if idx != len(self._list) and self._list[idx] == value:
# Control flow
                self._list.pop(idx)
                return
            raise ValueError(f"{value} not in SortedList")
        else:
            # find by identity of stored item
            for i, (k, item) in enumerate(self._list):
# Control flow
                if item == value:
# Control flow
                    self._list.pop(i)
                    return
            raise ValueError(f"{value} not in SortedList")

    def pop(self, index=-1):
# Function start
        val = self._list.pop(index)
        return val[1] if self._key is not None else val

    def __len__(self):
# Function start
        return len(self._list)

    def __getitem__(self, idx):
# Function start
        val = self._list[idx]
        return val[1] if self._key is not None else val

    def __iter__(self):
# Function start
        if self._key is None:
# Control flow
            return iter(self._list)
        else:
            return (item for (_, item) in self._list)

    def __contains__(self, value):
# Function start
        if self._key is None:
# Control flow
            idx = bisect_left(self._list, value)
            return idx != len(self._list) and self._list[idx] == value
        else:
            return any(item == value for (_, item) in self._list)

    def index(self, value):
# Function start
        if self._key is None:
# Control flow
            idx = bisect_left(self._list, value)
            if idx != len(self._list) and self._list[idx] == value:
# Control flow
                return idx
            raise ValueError(f"{value} not in SortedList")
        else:
            for i, (_, item) in enumerate(self._list):
# Control flow
                if item == value:
# Control flow
                    return i
            raise ValueError(f"{value} not in SortedList")

class Spot:
    """Represents a parking spot with coordinates, distance, and status"""
    
    def __init__(self, row: int, col: int, distance: int):
# Function start
        # RTDB fields
        self.status = "FREE"
        self.waiting_car_id = "-"
        self.seen_car_id = "-"
        self.distance_from_entry = distance
        
        # Local-only field for efficiency (matches RTDB key format)
        # use plain 'row,col' key format to match event_generator and RTDB child naming
        self.spot_id = f"{row},{col}"
    
    # RTDB sync methods removed

class Car:
    """Represents a car with plate ID, status, and parking information"""
    
    def __init__(self, plate_id: str):
# Function start
        # RTDB fields
        self.plate_id = plate_id
        self.status = "waiting"  # waiting, parked, parked_illegally
        self.allocated_spot = "-"
        self.timestamp = ""
        
        # Local-only fields
        self.actual_spot = None  # Where car actually parked (if different from allocated)
    
    # RTDB sync methods removed

class ParkingLot:
    """Main class that manages all parking lot data structures and operations"""
    
    def __init__(self):
# Function start
        # AVL tree (SortedList) of free spots, ordered by distance from entry
        # support key to order by Spot.distance_from_entry
        self.free_spots = SortedList(key=lambda spot: spot.distance_from_entry)

        # Hash tables for O(1) lookups
        self.spot_lookup = {}  # spot_id -> Spot object
        self.car_lookup = {}   # car_plate -> Car object
        # Current waiting pair (car allocated to spot but not yet parked)
        self.waiting_pair = None  # {"car_id": str, "spot_id": str} or None

        # Hash table of occupied spots: spot_id -> car_id (O(1) operations)
        self.occupied_spots_with_cars = {}

        # Variable to hold the time we saved (e.g., last state save timestamp)
        self.saved_time = None
        self.isFull = False
    
    # Basic data operations
    def add_spot(self, spot):
# Function start
        """Add spot to both free_spots list and spot_lookup hash"""
        self.free_spots.add(spot)
        self.spot_lookup[spot.spot_id] = spot
    
    def add_car(self, car):
# Function start
        """Add car to car_lookup hash"""
        self.car_lookup[car.plate_id] = car
    
    def get_closest_free_spot(self):
# Function start
        """Return closest free spot (first element) or None if empty"""
        return self.free_spots[0] if self.free_spots else None
    
    def get_time_saved(self):
# Function start
        """Get the time saved based on the distance difference between the farthest and closest free spots."""
        if len(self.free_spots) < 2:
# Control flow
            return 0
        else:
            return self.free_spots[-1].distance_from_entry - self.free_spots[0].distance_from_entry
        
    
    def remove_spot_from_free(self, spot):
# Function start
        """Remove spot from free_spots list"""
        # allow passing either Spot object or spot_id string
        if isinstance(spot, str):
# Control flow
            s = self.spot_lookup.get(spot)
            if s and s in self.free_spots:
# Control flow
                self.free_spots.remove(s)
            return
        if spot in self.free_spots:
# Control flow
            self.free_spots.remove(spot)

    def remove_spot_by_id(self, spot_id: str):
# Function start
        """Remove a spot from free_spots by its spot_id string."""
        spot = self.spot_lookup.get(spot_id)
        if spot:
# Control flow
            try:
# Control flow
                self.remove_spot_from_free(spot)
            except Exception:
# Control flow
                pass
    
    def add_spot_to_free(self, spot):
# Function start
        """Add spot back to free_spots list"""
        if spot not in self.free_spots:
# Control flow
            self.free_spots.add(spot)
    
    # Simple lookups
    def get_spot(self, spot_id):
# Function start
        """Get spot by spot_id from hash table"""
        return self.spot_lookup.get(spot_id)
    
    def get_car(self, car_id):
# Function start
        """Get car by car_id from hash table"""
        return self.car_lookup.get(car_id)
    
    # Pairing operations
    def set_waiting_pair(self, car_id, spot_id):
# Function start
        """Set current waiting pair"""
        self.waiting_pair = {"car_id": car_id, "spot_id": spot_id}
    
    def get_waiting_pair(self):
# Function start
        """Get current waiting pair"""
        return self.waiting_pair
    
    def clear_waiting_pair(self):
# Function start
        """Clear current waiting pair"""
        self.waiting_pair = None
    
    # Occupied spots tracking methods
    def add_occupied_spot(self, spot_id, car_id):
# Function start
        """Add a spot to occupied hash table - O(1)"""
        self.occupied_spots_with_cars[spot_id] = car_id
    
    def remove_occupied_spot(self, spot_id):
# Function start
        """Remove a spot from occupied hash table - O(1)"""
        self.occupied_spots_with_cars.pop(spot_id, None)
    
    def get_occupied_spots(self):
# Function start
        """Get list of occupied spot tuples [(spot_id, car_id), ...]"""
        return list(self.occupied_spots_with_cars.items())
    
    def get_random_occupied_spot(self):
# Function start
        """Get a random occupied spot tuple (spot_id, car_id) or None if empty"""
        return random.choice(list(self.occupied_spots_with_cars.items())) if self.occupied_spots_with_cars else None

    # Grid/BFS utilities
    def _parse_spot_coords(self, spot_id: str) -> Tuple[int, int]:
# Function start
        """Parse spot_id formatted as '(row,col)' or 'row,col' into (row, col) ints."""
        s = spot_id.strip()
        if s.startswith('(') and s.endswith(')'):
# Control flow
            s = s[1:-1]
        parts = s.split(',')
        return int(parts[0]), int(parts[1])

    def _format_coord_tuple(self, row: int, col: int, with_paren: bool = False) -> str:
# Function start
        if with_paren:
# Control flow
            return f"({row},{col})"
        return f"{row},{col}"

    def find_closest(self, gate_row: int = 0, gate_col: int = 2) -> Optional[Tuple[int, int]]:
# Function start
        """Find closest FREE spot to the gate using BFS on the grid of spots.

        Returns (row, col) or None if no free spots. The return shape matches the
        rest of the code and the UI which uses 'row,col' string keys.
        """
        # Build set of free positions from spot_lookup
        free_positions = set()
        for sid, spot in self.spot_lookup.items():
# Control flow
            if getattr(spot, 'status', None) == 'FREE':
# Control flow
                row, col = self._parse_spot_coords(sid)
                free_positions.add((row, col))

        if not free_positions:
# Control flow
            return None

        # BFS from gate
        q = deque()
        start = (gate_row, gate_col)
        q.append(start)
        seen = {start}

        # neighbor deltas: up/down/left/right
        deltas = [(-1, 0), (1, 0), (0, -1), (0, 1)]

        while q:
# Control flow
            r, c = q.popleft()
            if (r, c) in free_positions:
# Control flow
                # return (row, col) to match the 'row,col' key format used by the DB
                return (r, c)
            # Collect potential neighbors, then sort deterministically by (col, row)
            neighbors = []
            for dr, dc in deltas:
# Control flow
                nr, nc = r + dr, c + dc
                if (nr, nc) in seen:
# Control flow
                    continue
                # only explore coordinates that exist in the grid (either free or occupied)
                if (nr, nc) in free_positions or any((nr, nc) == self._parse_spot_coords(sid) for sid in self.spot_lookup):
# Control flow
                    neighbors.append((nr, nc))

            # sort neighbors so tie-breaker prefers lower column, then lower row
            neighbors.sort(key=lambda rc: (rc[1], rc[0]))
            for nr, nc in neighbors:
# Control flow
                seen.add((nr, nc))
                q.append((nr, nc))

        return None

    def allocate_closest_spot(self, car_id: str, gate_row: int = 0, gate_col: int = 2) -> Optional[str]:
# Function start
        """Allocate the closest free spot (BFS) for car_id.

        Returns the allocated spot id as 'row,col' string (no parentheses) or None.
        Also sets waiting_pair and removes spot from free_spots.
        """
        coord = self.find_closest(gate_row, gate_col)
        if coord is None:
# Control flow
            return None
        # find_closest now returns (row, col)
        row, col = coord
        # our spot_lookup keys use '(row,col)'
        key_paren = self._format_coord_tuple(row, col, with_paren=True)
        key_plain = self._format_coord_tuple(row, col, with_paren=False)
        spot = self.spot_lookup.get(key_paren) or self.spot_lookup.get(f"({row},{col})")
        if spot is None:
# Control flow
            # try the plain key
            spot = self.spot_lookup.get(key_plain)
        if spot is None:
# Control flow
            return None

        # remove from free_spots if present and mark as waiting
        try:
# Control flow
            # mark in-memory
            spot.status = 'WAITING'
            self.remove_spot_from_free(spot)
        except Exception:
# Control flow
            pass

        # set waiting pair and return plain key 'row,col'
        self.set_waiting_pair(car_id, key_plain)
        print(f"[ParkingLot] Allocated spot {key_plain} to car {car_id}; free_spots_count={len(self.free_spots)}")
        return key_plain
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## Init_Park.py

#### File explanation
Seeds the RTDB with a clean grid and fixed distances.



**Overview.** Seed RTDB with a grid of FREE spots and precomputed distances.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# Seed RTDB with a grid of FREE spots and precomputed distances.

import time
# Library import/include
from firebase_admin import db
# Library import/include
from firebase_init import db as _db_init  # ensures app is initialized
# Library import/include
from constants import ROOT_BRANCH, STAT_FREE
# Library import/include


# Parking-lot size (adjust as needed)
ROWS = 10
# Constant/configuration
COLS = 5
# Constant/configuration

# configurable entry location (row, col) used to compute distanceFromEntry
ENTRY_ROW = 3
# Constant/configuration
ENTRY_COL = 0
# Constant/configuration


def _spot_id(r, c):
# Function start
    # use canonical key format 'row,col' to match other modules
    return f"{r},{c}"


def distance_from_entry(r, c):
# Function start
    """Manhattan distance from the configured entry point."""
    return abs(r - ENTRY_ROW) + abs(c - ENTRY_COL)


def main():
# Function start
    base = db.reference(ROOT_BRANCH)
# RTDB path / configuration
    spots_ref = base.child("SPOTS")
# RTDB path / configuration

    # create metadata
    meta = {
        "_meta": {
            "rows": ROWS,
            "cols": COLS,
            "lastInit": int(time.time()),
        },
       
    }
    base.update(meta)

    payload = {}
    now_ms = int(time.time() * 1000)
    for r in range(ROWS):
# Control flow
        for c in range(COLS):
# Control flow
            sid = _spot_id(r, c)
            payload[sid] = {
                "row": r,
                "col": c,
                "status": STAT_FREE,
                "distanceFromEntry": distance_from_entry(r, c),
                "lastUpdateMs": now_ms,
                "seenCarId": "-",      # initialized as null
                "waitingCarId": "-",   # initialized as null
            }

    spots_ref.set(payload)
    print(f"[WRITE] Initialized {ROWS*COLS} spots under /{ROOT_BRANCH}/SPOTS (keys as 'row,col')")
# RTDB path / configuration

    # sanity read
    data = spots_ref.get() or {}
    print(f"[READ] Spots in DB: {len(data)}")



if __name__ == "__main__":
# Control flow
    main()
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## RTDB_listener.py

#### File explanation
Reactive listeners for `/SPOTS` and `/CARS` to debug and verify flows.



**Overview.** Attach listeners to `/SPOTS` and `/CARS` for debugging flows.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# Attach listeners to `/SPOTS` and `/CARS` for debugging flows.

import threading
# Library import/include
import time
# Library import/include
from firebase_admin import db
# Library import/include
from firebase_init import db as _db_init  # ensures firebase is initialized
# Library import/include
from constants import ROOT_BRANCH, STAT_WAIT, STAT_OCC
# Library import/include

BASE = db.reference(ROOT_BRANCH)
# Constant/configuration
SPOTS = BASE.child("SPOTS")
# Constant/configuration
CARS = BASE.child("CARS")
# Constant/configuration


def _on_spots(event):
# Function start
    # event: {event_type, path, data}
    try:
# Control flow
        print(f"[SPOTS EVENT] {event.event_type} {event.path} -> {str(event.data)[:120]}")
# RTDB path / configuration
    except Exception:
# Control flow
        print("[SPOTS EVENT] (print failed)")
# RTDB path / configuration

    # Example: auto-confirm WAITING -> OCCUPIED when 'arrivalConfirmed' flag appears
    # (You can delete this if your flow is different.)
    try:
# Control flow
        if isinstance(event.data, dict) and event.data.get("arrivalConfirmed"):
# Control flow
            # path like '/(r,c)'
            spot_id = event.path.strip("/")
            if spot_id:
# Control flow
                SPOTS.child(spot_id).update({
# RTDB path / configuration
                    "status": STAT_OCC,
                    "arrivalConfirmed": None,
                    "lastUpdate": int(time.time()),
                })
                print(f"[AUTO] {spot_id}: WAITING→OCCUPIED by listener")
    except Exception as e:
# Control flow
        print("[SPOTS HANDLER ERROR]", e)
# RTDB path / configuration


def _on_cars(event):
# Function start
    try:
# Control flow
        print(f"[CARS EVENT] {event.event_type} {event.path} -> {str(event.data)[:120]}")
# RTDB path / configuration
    except Exception:
# Control flow
        print("[CARS EVENT] (print failed)")
# RTDB path / configuration


def start_listener(block_forever: bool = True):
# Function start
    s_stream = SPOTS.listen(_on_spots)
# RTDB path / configuration
    c_stream = CARS.listen(_on_cars)
# RTDB path / configuration

    print("[Listener] Attached to:")
    print(f" /{ROOT_BRANCH}/SPOTS")
# RTDB path / configuration
    print(f" /{ROOT_BRANCH}/CARS")
# RTDB path / configuration

    try:
# Control flow
        if block_forever:
# Control flow
            threading.Event().wait()
    finally:
        try:
# Control flow
            s_stream.close()
        except Exception:
# Control flow
            pass
        try:
# Control flow
            c_stream.close()
        except Exception:
# Control flow
            pass


if __name__ == "__main__":
# Control flow
    start_listener()
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## firebase_init.py

#### File explanation
Initializes Firebase Admin and discovers the correct RTDB URL.



**Overview.** Initialize Firebase Admin SDK and resolve database URL.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# Initialize Firebase Admin SDK and resolve database URL.

import os
# Library import/include
import json
# Library import/include
import time
# Library import/include
import firebase_admin
# Library import/include
from firebase_admin import credentials
# Library import/include
from firebase_admin import db
# Library import/include

# Try env override first (recommended)
ENV_DB_URL = os.environ.get("RTDB_URL") or os.environ.get("FIREBASE_DATABASE_URL")
# Constant/configuration

def _probe_url(url):
# Function start
    try:
# Control flow
        import requests
# Library import/include
        resp = requests.get(url.rstrip("/") + "/.json", timeout=3)
        # treat 404 as non-existing endpoint; 200/401/403/etc means endpoint exists
        return resp.status_code != 404
    except Exception:
# Control flow
        return False

def _derive_database_url(cred_path):
# Function start
    if ENV_DB_URL:
# Control flow
        return ENV_DB_URL

    try:
# Control flow
        info = json.load(open(cred_path))
        pid = info.get("project_id")
    except Exception:
# Control flow
        return None

    if not pid:
# Control flow
        return None

    candidates = [
        f"https://{pid}-default-rtdb.firebaseio.com",
        f"https://{pid}-default-rtdb.europe-west1.firebasedatabase.app",
        f"https://{pid}-default-rtdb.us-central1.firebasedatabase.app",
        f"https://{pid}-default-rtdb.europe-west3.firebasedatabase.app",
    ]

    for u in candidates:
# Control flow
        if _probe_url(u):
# Control flow
            return u

    # fallback to first candidate
    return candidates[0]

# credential path (use GOOGLE_APPLICATION_CREDENTIALS if set, else 'secret.json' next to this file)
# If the process CWD is the repo root (common when running scripts), a plain
# "secret.json" won't be found. Point the default to the Server/secret.json
# location (next to this module) so both `cd Server && python ...` and
# `python Server/simulation_sondos.py` work.
default_secret = os.path.join(os.path.dirname(__file__), "secret.json")
cred_path = os.environ.get("GOOGLE_APPLICATION_CREDENTIALS", default_secret)

# Prefer certificate file when available. If missing, fall back to Application Default
# Credentials so importing this module doesn't crash immediately. This makes local
# development easier but we emit a clear message explaining how to provide creds.
if os.path.exists(cred_path):
# Control flow
    try:
# Control flow
        cred = credentials.Certificate(cred_path)
    except Exception as e:
# Control flow
        # Unexpected parsing error from certificate file
        raise RuntimeError(f"Failed to load Firebase certificate from {cred_path}: {e}") from e
else:
    if "GOOGLE_APPLICATION_CREDENTIALS" in os.environ:
# Control flow
        # User explicitly set the env var but file does not exist -> fail fast with clear message
        raise FileNotFoundError(
            f"GOOGLE_APPLICATION_CREDENTIALS is set to '{cred_path}' but the file was not found.\n"
            "Please set GOOGLE_APPLICATION_CREDENTIALS to a valid service account JSON path or place 'secret.json' in the Server folder."
        )
    # No certificate file found; fall back to Application Default Credentials (ADC).
    # ADC will work if the environment has been configured (e.g. gcloud auth application-default login)
    # or when running on a Google Cloud environment. We keep `cred` as None so
    # `firebase_admin.initialize_app` uses the default credential flow.
    print("[Firebase] Warning: credential file not found; falling back to Application Default Credentials.\n"
          "If you expect to use a service account file locally, set GOOGLE_APPLICATION_CREDENTIALS or add 'secret.json'.")
    cred = None

database_url = _derive_database_url(cred_path)
if not database_url:
# Control flow
    database_url = os.environ.get("RTDB_URL")  # final fallback

# If we still don't have a database URL, that's unrecoverable for parts of the
# application that expect a Realtime Database (e.g. simulation scripts). Fail
# fast with a helpful message so the user knows how to fix their environment.
if not database_url:
# Control flow
    raise RuntimeError(
        """Firebase Realtime Database URL could not be determined.
Provide one of the following to continue:
 1) Set the RTDB_URL or FIREBASE_DATABASE_URL environment variable to your RTDB URL.
# RTDB path / configuration
    Example: export RTDB_URL='https://<project>-default-rtdb.europe-west1.firebasedatabase.app'
 2) Place a service account JSON named 'secret.json' in the Server folder or set
    GOOGLE_APPLICATION_CREDENTIALS to its absolute path so the code can derive the project_id.

Note: If you intend to run without a Realtime Database (read-only or offline), adjust imports that
call db.reference or mock the database in tests.
"""
    )

# Initialize Firebase app idempotently. If `cred` is None, firebase_admin will use
# Application Default Credentials or other configured defaults.
try:
# Control flow
    firebase_admin.get_app()
except ValueError:
# Control flow
    init_opts = {"databaseURL": database_url} if database_url else None
    if init_opts:
# Control flow
        firebase_admin.initialize_app(cred, init_opts)
    else:
        # initialize without options if we don't have a databaseURL
        firebase_admin.initialize_app(cred)

print(f"[Firebase] Using DB: {database_url}")
# export db (other modules import `from firebase_init import db as _db_init`)
# Note: `db` is already imported from firebase_admin above
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## event_generator.py

#### File explanation
Simulates realistic parking scenarios to test logic end-to-end.



**Overview.** Simulate arrivals, allocation, parking, and departures to stress logic.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# Simulate arrivals, allocation, parking, and departures to stress logic.

# Event Generator - Simulates car arrivals and departures
# This file generates events that write to Firebase RTDB, triggering the listener

import firebase_admin
# Library import/include
from firebase_admin import credentials, db
# Library import/include
import random
# Library import/include
import time
# Library import/include
import datetime
# Library import/include
from data_structures import ParkingLot
# Library import/include
import typing
# Library import/include
from constants import ROOT_BRANCH
# Library import/include


def refresh_spot_from_db(parking_lot: typing.Optional[ParkingLot], spot_id: str):
# Function start
    """Refresh a single spot's status from RTDB into the in-memory ParkingLot.

    This reads /{ROOT_BRANCH}/SPOTS/{spot_id} and updates spot_lookup and free_spots
# RTDB path / configuration
    so allocations use the latest external sensor updates (useful for sensor nodes).
    """
    if not parking_lot or not spot_id:
# Control flow
        return
    try:
# Control flow
        spots_ref = db.reference(f"/{ROOT_BRANCH}/SPOTS")
# RTDB path / configuration
        node = spots_ref.child(str(spot_id)).get() or {}
        status = node.get('status')
        # ensure Spot object exists in parking_lot.spot_lookup
        sp = parking_lot.get_spot(spot_id)
        if not sp:
# Control flow
            # attempt to parse row,col and use distance if available
            try:
# Control flow
                row, col = spot_id.split(',')
                dist = node.get('distanceFromEntry') or node.get('distanceFromEntry', 0) or 0
                sp = type('SpotProxy', (), {})()
                sp.spot_id = spot_id
                sp.status = status or 'FREE'
                sp.distance_from_entry = int(dist) if dist is not None else 0
                # register in parking_lot
                parking_lot.spot_lookup[spot_id] = sp
                try:
# Control flow
                    parking_lot.add_spot(sp)
                except Exception:
# Control flow
                    try:
# Control flow
                        parking_lot.add_spot_to_free(sp)
                    except Exception:
# Control flow
                        pass
            except Exception:
# Control flow
                return
        else:
            # update existing Spot object status and free/occupied containers
            prev = getattr(sp, 'status', None)
            sp.status = status or prev or 'FREE'
            # reflect change in free_spots container
            try:
# Control flow
                if sp.status == 'FREE':
# Control flow
                    parking_lot.add_spot_to_free(sp)
                else:
                    parking_lot.remove_spot_from_free(sp)
            except Exception:
# Control flow
                pass
    except Exception as e:
# Control flow
        print(f"[WARN] refresh_spot_from_db({spot_id}) failed: {e}")

def generate_plate_id():
# Function start
    """Generate a random 8-digit car plate ID"""
    return f"{random.randint(10000000, 99999999)}"

def simulate_car_arrival(parking_lot: typing.Optional[ParkingLot] = None):
# Function start
    """Simulate a new car arriving - writes to Firebase RTDB and tries to allocate a spot using parking_lot"""
    plate_id = generate_plate_id()
    timestamp = datetime.datetime.now().isoformat()
    
    car_data = {
        'Id': plate_id,
        'ClosestSpot': '-',
        'SpotIn': {'Arrievied': False},
        'status': 'waiting',
        'allocatedSpot': '-',
        'timestamp': timestamp
    }
    
    cars_ref = db.reference(f"/{ROOT_BRANCH}/CARS")
# RTDB path / configuration
    cars_ref.child(plate_id).set(car_data)

    # if this is a sensor-controlled spot (like 0,0) make sure we refresh it from DB
    try:
# Control flow
        refresh_spot_from_db(parking_lot, '0,0')
    except Exception:
# Control flow
        pass

    allocated_spot = None

    if parking_lot:
# Control flow
        # Try common allocation APIs on the provided ParkingLot
        try:
# Control flow
            # Prefer BFS-based allocator when available. Use explicit gate coords
            # so allocations are consistent with the UI. Gate defaults to (0,2)
            import os
# Library import/include
            gate_row = int(os.environ.get('GATE_ROW', '0'))
            gate_col = int(os.environ.get('GATE_COL', '2'))
            if hasattr(parking_lot, 'allocate_closest_spot'):
# Control flow
                # compute BFS closest before allocator mutates free_spots
                try:
# Control flow
                    bfs_coord = parking_lot.find_closest(gate_row=gate_row, gate_col=gate_col)
                    bfs_key = f"{bfs_coord[0]},{bfs_coord[1]}" if bfs_coord else None
                except Exception:
# Control flow
                    bfs_key = None

                allocated_spot = parking_lot.allocate_closest_spot(plate_id, gate_row=gate_row, gate_col=gate_col)

                # defensive: if allocator result differs from precomputed BFS, prefer the BFS key
                if bfs_key and allocated_spot != bfs_key:
# Control flow
                    print(f"[WARN] allocator returned {allocated_spot} but BFS closest was {bfs_key} -> preferring BFS result")
                    # try to return allocated_spot to free_spots (if it was removed)
                    try:
# Control flow
                        if hasattr(parking_lot, 'get_spot') and allocated_spot:
# Control flow
                            spobj = parking_lot.get_spot(allocated_spot)
                            if spobj:
# Control flow
                                spobj.status = 'FREE'
                                if hasattr(parking_lot, 'add_spot_to_free'):
# Control flow
                                    parking_lot.add_spot_to_free(spobj)
                                else:
                                    try:
# Control flow
                                        parking_lot.free_spots.add(spobj)
                                    except Exception:
# Control flow
                                        pass
                    except Exception:
# Control flow
                        pass

                    # remove bfs_key from free list and set as waiting for this plate
                    allocated_spot = bfs_key
                    try:
# Control flow
                        if hasattr(parking_lot, 'get_spot'):
# Control flow
                            bsp = parking_lot.get_spot(bfs_key)
                            if bsp:
# Control flow
                                try:
# Control flow
                                    parking_lot.remove_spot_from_free(bsp)
                                except Exception:
# Control flow
                                    try:
# Control flow
                                        parking_lot.free_spots.remove(bsp)
                                    except Exception:
# Control flow
                                        pass
                                parking_lot.set_waiting_pair(plate_id, bfs_key)
                    except Exception:
# Control flow
                        pass
            else:
                # if no BFS allocator, fall back to a simple free_spots pop
                if hasattr(parking_lot, 'free_spots'):
# Control flow
                    try:
# Control flow
                        sp = parking_lot.free_spots.pop()
                        # if SortedList, pop() returns a Spot; if plain container, use it directly
                        allocated_spot = sp.spot_id if hasattr(sp, 'spot_id') else sp
                    except Exception:
# Control flow
                        allocated_spot = None
        except Exception as e:
# Control flow
            # allocation failed; leave as waiting
            print(f"⚠️ ParkingLot allocation error: {e}")

    if allocated_spot:
# Control flow
        # update RTDB to reflect allocation: assign closest spot and mark as waiting
        cars_ref.child(plate_id).update({'allocatedSpot': allocated_spot, 'ClosestSpot': allocated_spot, 'SpotIn': {'Arrievied': False}, 'status': 'waiting'})

        # write to UI branch so console reflects the waiting state
        spots_ref = db.reference(f"{ROOT_BRANCH}/SPOTS")
# RTDB path / configuration
        spots_ref.child(str(allocated_spot)).update({'status': 'WAITING', 'waitingCarId': plate_id, 'seenCarId': '-'})
        print(f"🔔 Car {plate_id} assigned to spot {allocated_spot} (waiting)")
    else:
        print(f"⏳ Car {plate_id} added to queue (no spot allocated)")

    return plate_id

def simulate_car_departure(parking_lot: typing.Optional[ParkingLot]):
# Function start
    """Simulate a car leaving using the parking lot structure and update RTDB"""
    if not parking_lot:
# Control flow
        print("❌ No ParkingLot provided!")
        return None

    spot_car_pair = None
    try:
# Control flow
        # Build a list of occupied (spot_id, car_id) tuples from available APIs
        occ_list = None
        if hasattr(parking_lot, 'get_occupied_spots'):
# Control flow
            occ_list = parking_lot.get_occupied_spots()
        elif hasattr(parking_lot, 'occupied_spots_with_cars'):
            occ_list = list(getattr(parking_lot, 'occupied_spots_with_cars').items())
        elif hasattr(parking_lot, 'occupied_spots'):
            occ_dict = getattr(parking_lot, 'occupied_spots') or {}
            occ_list = list(occ_dict.items())

        if occ_list:
# Control flow
            # Exclude the sensor-controlled spot '0,0' (and variant '(0,0)') per user request
            filtered = [(s, c) for (s, c) in occ_list if str(s) not in ('0,0', '(0,0)')]
            if not filtered:
# Control flow
                # No occupied spots available except the sensor-controlled one -> don't depart
                print("❌ No occupied spots found (excluding 0,0). Skipping departure.")
                return None
            spot_car_pair = random.choice(filtered)
        else:
            spot_car_pair = None
    except Exception as e:
# Control flow
        print(f"⚠️ Error querying ParkingLot: {e}")

    if not spot_car_pair:
# Control flow
        print("❌ No occupied spots found!")
        return None

    spot_id, departing_car_id = spot_car_pair
    print(f"🚗 Car {departing_car_id} leaving spot {spot_id}")

    # Update parking lot internal structures if possible so freed spot is visible to allocators
    try:
# Control flow
        if parking_lot:
# Control flow
            # Prefer modern API to remove occupied spot
            if hasattr(parking_lot, 'remove_occupied_spot'):
# Control flow
                try:
# Control flow
                    parking_lot.remove_occupied_spot(spot_id)
                except Exception:
# Control flow
                    # fallback to older dicts
                    if hasattr(parking_lot, 'occupied_spots_with_cars'):
# Control flow
                        parking_lot.occupied_spots_with_cars.pop(spot_id, None)
                    elif hasattr(parking_lot, 'occupied_spots'):
                        getattr(parking_lot, 'occupied_spots').pop(spot_id, None)
            else:
                # old API support
                try:
# Control flow
                    if hasattr(parking_lot, 'occupied_spots_with_cars'):
# Control flow
                        parking_lot.occupied_spots_with_cars.pop(spot_id, None)
                    elif hasattr(parking_lot, 'occupied_spots'):
                        getattr(parking_lot, 'occupied_spots').pop(spot_id, None)
                except Exception:
# Control flow
                    pass

            # set Spot.status back to FREE and add to free_spots container if possible
            try:
# Control flow
                if hasattr(parking_lot, 'get_spot'):
# Control flow
                    spot_obj = parking_lot.get_spot(spot_id)
                    if spot_obj:
# Control flow
                        spot_obj.status = 'FREE'
                        if hasattr(parking_lot, 'add_spot_to_free'):
# Control flow
                            parking_lot.add_spot_to_free(spot_obj)
                        else:
                            fs = getattr(parking_lot, 'free_spots', None)
                            if fs is not None:
# Control flow
                                try:
# Control flow
                                    fs.add(spot_obj)
                                except Exception:
# Control flow
                                    try:
# Control flow
                                        fs.append(spot_obj)
                                    except Exception:
# Control flow
                                        pass
                else:
                    fs = getattr(parking_lot, 'free_spots', None)
                    if fs is not None:
# Control flow
                        try:
# Control flow
                            fs.add(spot_id)
                        except Exception:
# Control flow
                            try:
# Control flow
                                fs.append(spot_id)
                            except Exception:
# Control flow
                                pass
            except Exception as e:
# Control flow
                print(f"⚠️ Error updating ParkingLot on departure: {e}")
    except Exception as e:
# Control flow
        print(f"⚠️ Error updating ParkingLot on departure: {e}")

    # Update RTDB - mark spot free and mark car as departed
    # update UI branch for spots (reset seen/waiting)
    spots_ref = db.reference(f"/{ROOT_BRANCH}/SPOTS")
# RTDB path / configuration
    spots_ref.child(str(spot_id)).update({'status': 'FREE', 'carId': None, 'seenCarId': '-', 'waitingCarId': '-'})
    # Remove the car record from the RTDB so departed cars don't linger.
    # Attempt both the namespaced branch and the legacy top-level /CARS to be safe.
# RTDB path / configuration
    try:
# Control flow
        namespaced_ref = db.reference(f"/{ROOT_BRANCH}/CARS").child(str(departing_car_id))
# RTDB path / configuration
        try:
# Control flow
            # Best-effort update to mark departed before deletion (useful for listeners)
            namespaced_ref.update({'status': 'departed', 'allocatedSpot': '-'})
        except Exception:
# Control flow
            pass
        try:
# Control flow
            namespaced_ref.delete()
            print(f"[DB] Deleted car {departing_car_id} from /{ROOT_BRANCH}/CARS")
# RTDB path / configuration
        except Exception:
# Control flow
            print(f"[DB] Failed to delete car {departing_car_id} from /{ROOT_BRANCH}/CARS")
# RTDB path / configuration
    except Exception:
# Control flow
        print(f"[DB] Error handling namespaced car delete for {departing_car_id}")

    # Also attempt to remove legacy top-level /CARS entries if present
# RTDB path / configuration
    try:
# Control flow
        legacy_ref = db.reference(f"/CARS").child(str(departing_car_id))
# RTDB path / configuration
        try:
# Control flow
            legacy_ref.delete()
            print(f"[DB] Deleted car {departing_car_id} from /CARS (legacy)")
# RTDB path / configuration
        except Exception:
# Control flow
            # not fatal; just log
            print(f"[DB] Failed to delete car {departing_car_id} from /CARS (legacy)")
# RTDB path / configuration
    except Exception:
# Control flow
        pass

    print(f"✅ Triggered departure for spot {spot_id}")
    return departing_car_id


def simulate_car_parked(parking_lot: typing.Optional[ParkingLot], plate_id: str):
# Function start
    """Mark a previously-assigned car as physically arrived and update RTDB.

    - Sets CARS/{plate}.SpotIn.Arrievied = True and status = 'parked'
# RTDB path / configuration
    - Updates SondosPark/SPOTS/{spot}: status -> 'OCCUPIED', carId/seenCarId -> plate, waitingCarId -> '-'
# RTDB path / configuration
    - Updates parking_lot internal structures where possible
    Returns the allocated spot id or None on failure.
    """
    if not plate_id:
# Control flow
        print("❌ No plate id provided to simulate_car_parked")
        return None

    cars_ref = db.reference(f"/{ROOT_BRANCH}/CARS")
# RTDB path / configuration
    # Find allocated spot: prefer parking_lot mapping, fallback to DB stored allocatedSpot
    allocated_spot = None
    try:
# Control flow
        if parking_lot and hasattr(parking_lot, 'occupied_spots'):
# Control flow
            # find spot by matching plate id
            for s, c in getattr(parking_lot, 'occupied_spots').items():
# Control flow
                if c == plate_id:
# Control flow
                    allocated_spot = s
                    break
        if not allocated_spot:
# Control flow
            car_record = cars_ref.child(plate_id).get() or {}
            allocated_spot = car_record.get('allocatedSpot')
    except Exception as e:
# Control flow
        print(f"⚠️ Error determining allocated spot for {plate_id}: {e}")

    if not allocated_spot:
# Control flow
        print(f"❌ No allocated spot found for car {plate_id}")
        return None

    # Update car record: arrived
    try:
# Control flow
        cars_ref.child(plate_id).update({'SpotIn': {'Arrievied': True}, 'status': 'parked', 'allocatedSpot': allocated_spot})
    except Exception as e:
# Control flow
        print(f"⚠️ Failed to update car record for {plate_id}: {e}")

    # Update spot record to OCCUPIED and set seen/waiting fields
    spots_ref = db.reference(f"{ROOT_BRANCH}/SPOTS")
# RTDB path / configuration
    ts = int(time.time() * 1000)
    try:
# Control flow
        spots_ref.child(str(allocated_spot)).update({'status': 'OCCUPIED', 'carId': plate_id, 'seenCarId': plate_id, 'waitingCarId': '-', 'lastUpdateMs': ts})
    except Exception as e:
# Control flow
        print(f"⚠️ Failed to update spot {allocated_spot} for parked car {plate_id}: {e}")

    # Update parking_lot internal structures if APIs available
    try:
# Control flow
        if parking_lot:
# Control flow
            if hasattr(parking_lot, 'add_occupied_spot'):
# Control flow
                parking_lot.add_occupied_spot(allocated_spot, plate_id)
            elif hasattr(parking_lot, 'occupied_spots'):
                parking_lot.occupied_spots[allocated_spot] = plate_id
            # remove from free_spots if present
            if hasattr(parking_lot, 'free_spots'):
# Control flow
                fs = getattr(parking_lot, 'free_spots')
                try:
# Control flow
                    if isinstance(fs, set):
# Control flow
                        fs.discard(allocated_spot)
                    else:
                        # try remove for list-like
                        try:
# Control flow
                            fs.remove(allocated_spot)
                        except Exception:
# Control flow
                            pass
                except Exception:
# Control flow
                    pass
    except Exception as e:
# Control flow
        print(f"⚠️ Error updating parking_lot internals for parked car {plate_id}: {e}")

    print(f"✅ Car {plate_id} parked at spot {allocated_spot}")
    return allocated_spot
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## dashboard.py

#### File explanation
Flask backend that assembles state and serves `/api/status` to the UI.



**Overview.** Flask backend exposing `/api/status` for the tablet UI.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# Flask backend exposing `/api/status` for the tablet UI.

from flask import Flask, jsonify, render_template
# Library import/include
import firebase_init  # ensures firebase_admin is initialized
# Library import/include
from firebase_admin import db
# Library import/include
from data_structures import ParkingLot, Spot
# Library import/include
import time
# Library import/include

# Note: the repository contains a `template/` directory (singular). Keep the
# value in sync so Jinja can find `index.html`.
app = Flask(__name__, static_folder='static', template_folder='template')
ROOT = '/SondosPark/SPOTS'
# Constant/configuration


def build_parkinglot_from_db(snapshot):
# Function start
    pl = ParkingLot()
    for sid, s in (snapshot or {}).items():
# Control flow
        if not isinstance(s, dict):
# Control flow
            continue
        try:
# Control flow
            row_str, col_str = sid.replace('(', '').replace(')', '').split(',')
            row, col = int(row_str), int(col_str)
        except Exception:
# Control flow
            # if malformed key, skip
            continue
        dist = s.get('distanceFromEntry', 0) or 0
        spot = Spot(row, col, dist)
        spot.status = s.get('status', 'FREE')
        spot.waiting_car_id = s.get('waitingCarId', '-')
        spot.seen_car_id = s.get('seenCarId', '-')
        # use plain id format 'row,col'
        spot.spot_id = f"{row},{col}"
        pl.spot_lookup[spot.spot_id] = spot
        if spot.status == 'FREE':
# Control flow
            pl.free_spots.add(spot)
    return pl


@app.route('/')
def index():
# Function start
    # pass a timestamp to template so static assets can be cache-busted
    import time
# Library import/include
    return render_template('index.html', ts=int(time.time()))


@app.route('/api/status')
def api_status():
# Function start
    ref = db.reference(ROOT)
    data = ref.get() or {}

    # compute closest free using ParkingLot BFS
    pl = build_parkinglot_from_db(data)
    # gate configuration - default gate coordinates (row=0, col=2)
    import os
# Library import/include
    gate_row = int(os.environ.get('GATE_ROW', '0'))
    gate_col = int(os.environ.get('GATE_COL', '2'))
    closest = pl.find_closest(gate_row, gate_col)
    closest_str = f"{closest[0]},{closest[1]}" if closest else None

    # Determine waiting car at the gate (if any)
    # gate position may be present in pl.spot_lookup as 'row,col'
    gate_key = f"{gate_row},{gate_col}"
    gate_spot = pl.get_spot(gate_key)
    waiting_car = None
    if gate_spot:
# Control flow
        waiting_car = getattr(gate_spot, 'waiting_car_id', None)

    # If no car is waiting exactly at the gate, fall back to any WAITING spot
    # and prefer the one nearest the gate (Manhattan distance). This ensures
    # the arriving box shows the car assigned even if it's not placed exactly
    # on the gate cell.
    if not waiting_car or waiting_car == '-':
# Control flow
        best = None
        best_dist = None
        for sid, s in (data or {}).items():
# Control flow
            if not isinstance(s, dict):
# Control flow
                continue
            if s.get('status') == 'WAITING':
# Control flow
                try:
# Control flow
                    rstr, cstr = sid.replace('(', '').replace(')', '').split(',')
                    r, c = int(rstr), int(cstr)
                except Exception:
# Control flow
                    continue
                dist = abs(r - gate_row) + abs(c - gate_col)
                if best is None or dist < best_dist:
# Control flow
                    best = s.get('waitingCarId')
                    best_dist = dist
        if best:
# Control flow
            waiting_car = best

    # normalize keys for the client: strip parentheses so keys are 'row,col'
    normalized = {}
    for sid, s in (data or {}).items():
# Control flow
        if not isinstance(s, dict):
# Control flow
            continue
        # strip parentheses and whitespace
        key = sid.replace('(', '').replace(')', '').strip()
        normalized[key] = s

    # compute free count for UI
    free_count = sum(1 for s in normalized.values() if isinstance(s, dict) and (s.get('status') or '').upper() == 'FREE')
    is_full = free_count == 0

    # include timestamp
    now = int(time.time() * 1000)
    return jsonify({
        'spots': normalized,
        'closest_free': closest_str,
        'ts': now,
        'gate': {'row': gate_row, 'col': gate_col},
        'gate_waiting_car': waiting_car or '-',
        'free_count': free_count,
        'is_full': is_full,
    })


if __name__ == '__main__':
# Control flow
    # Listen on all interfaces so tablet can connect; use port 8000
    app.run(host='0.0.0.0', port=8000, debug=False)
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## index.html

#### File explanation
Skeleton of the dashboard page structure.



**Overview.** Dashboard HTML skeleton.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



html<!-- VISUAL DIAGRAM — High-Level Flow
ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
  Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot -->
<!-- Dashboard HTML skeleton. -->

<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>Parking Dashboard</title>
  <link rel="stylesheet" href="/static/css/styles.css?ts={{ ts }}">
  </head>
  <body>
    <div class="container">
      <h1>Parking Dashboard</h1>
      <div class="layout">
        <div class="left">
          <div id="grid"></div>
        </div>
        <div class="right">
          <!-- arriving and banner now live inside right column -->
          <div class="arriving-wrap">
            <div id="arriving" class="arriving">
              <span class="arriving-label">Arriving Car ID:</span>
              <span id="arriving-id" class="arriving-id-box">-</span>
            </div>
            <div id="side-banner" class="side-banner" style="display:none">THERE ARE FREE SPOTS IN THE PARK</div>
<!-- RTDB path / configuration -->
          </div>

          <h2>The closest free spot to the gate is :</h2>
          <div id="closest-pill" class="closest-pill">-</div>
          <div class="meta">Last update: <span id="ts">-</span></div>
        </div>
      </div>
    </div>

  <script src="/static/js/app.js?ts={{ ts }}"></script>
  </body>
</html>
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## app.js

#### File explanation
Client-side logic: polling, grid rendering, banners, and closest-spot display.



**Overview.** Front-end polling + grid rendering + banner/closest spot display.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



javascript// VISUAL DIAGRAM — High-Level Flow
// ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
//   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
// Front-end polling + grid rendering + banner/closest spot display.

const gridEl = document.getElementById('grid')
// Constant/configuration
const closestEl = document.getElementById('closest')
// Constant/configuration
const tsEl = document.getElementById('ts')
// Constant/configuration
const arrivingIdEl = document.getElementById('arriving-id')
// Constant/configuration

function render(spots, closest, freeCount){
// Function start
  // Fixed 5 columns, rows 0..9
  const cols = [[],[],[],[],[]]
// Constant/configuration
  for(let col=0; col<5; col++){
// Control flow
    for(let row=0; row<10; row++){
// Control flow
      const key = `${row},${col}`
// Constant/configuration
      cols[col].push({row,col,key,info:spots[key]})
    }
  }

  gridEl.innerHTML=''
  cols.forEach(c=>{
    const colEl = document.createElement('div')
// Constant/configuration
    colEl.className='col'
    // gate marker
    if(window._gate && window._gate.col === c[0].col){
// Control flow
      const g = document.createElement('div')
// Constant/configuration
      g.className='gate'
      g.textContent = 'gate'
      colEl.appendChild(g)
    }else{
      // placeholder for alignment
      const ph = document.createElement('div')
// Constant/configuration
      ph.style.height='30px'
      colEl.appendChild(ph)
    }

    c.forEach(s=>{
      const sp = document.createElement('div')
// Constant/configuration
      const label = document.createElement('div')
// Constant/configuration
      label.className = 'spot-label'
      sp.className='spot'
      // normalize status (be case-insensitive and robust to missing fields)
      let st = 'FREE'
      if(s.info){
// Control flow
        if(s.info.status) st = s.info.status
// Control flow
        else if(s.info.Status) st = s.info.Status
        else if(s.info.state) st = s.info.state
      }
  st = ('' + st).toUpperCase()
  if(st === 'FREE') sp.classList.add('free')
// Control flow
  else if(st === 'WAITING' || st === 'PENDING') sp.classList.add('waiting')
  else if(st === 'WRONG_PARK') sp.classList.add('wrong')
  else sp.classList.add('occupied')
      sp.textContent = `(${s.row},${s.col})`
      // small textual status under the tile
      // If the whole lot is full (freeCount===0) we hide the 'OCCUPIED' label to reduce clutter
      if(typeof freeCount !== 'undefined' && freeCount === 0 && st === 'OCCUPIED'){
// Control flow
        label.textContent = ''
      }else{
        label.textContent = st
      }
      label.style.marginTop = '6px'
      label.style.fontSize = '12px'
      label.style.fontWeight = '700'
      label.style.color = '#444'
      if(closest && closest===s.key){
// Control flow
        sp.style.outline='4px solid rgba(0,0,0,0.25)'
      }
      const wrapper = document.createElement('div')
// Constant/configuration
      wrapper.style.display = 'flex'
      wrapper.style.flexDirection = 'column'
      wrapper.style.alignItems = 'center'
      wrapper.appendChild(sp)
      wrapper.appendChild(label)
      colEl.appendChild(wrapper)
    })
    gridEl.appendChild(colEl)
  })
}

async function poll(){
  try{
// Control flow
    const r = await fetch('/api/status')
// Constant/configuration
    const j = await r.json()
// Constant/configuration
  const spots = j.spots || {}
// Constant/configuration
  const closest = j.closest_free
// Constant/configuration
  const freeCount = typeof j.free_count !== 'undefined' ? j.free_count : null
// Constant/configuration
  if(j.gate) window._gate = j.gate
// Control flow
  render(spots, closest, freeCount)
  // show parking full banner when DB reports no free spots
  const banner = document.getElementById('side-banner')
// Constant/configuration
  if(banner){
// Control flow
    // Show a user-friendly message for both states
    if(typeof freeCount === 'number'){
// Control flow
      if(freeCount === 0){
// Control flow
        banner.textContent = 'THERE ARE NO FREE SPOTS'
// RTDB path / configuration
        banner.style.background = '#ff3b2f' // bright red
        banner.style.color = '#000' // black text for contrast
        banner.style.display = 'flex'
        document.querySelector('.arriving-wrap').classList.add('square')
        banner.classList.add('square')
      }else{
        banner.textContent = 'THERE ARE FREE SPOTS'
// RTDB path / configuration
        banner.style.background = '#7be36a' // bright green
        banner.style.color = '#000'
        banner.style.display = 'flex'
        document.querySelector('.arriving-wrap').classList.add('square')
        banner.classList.add('square')
      }
    }else{
      // fallback: use is_full if free_count not provided
      if(j.is_full){
// Control flow
        banner.textContent = 'THERE ARE NO FREE SPOTS'
// RTDB path / configuration
        banner.style.background = '#ff3b2f'
        banner.style.color = '#000'
        banner.style.display = 'flex'
        document.querySelector('.arriving-wrap').classList.add('square')
        banner.classList.add('square')
      }else{
        banner.style.display = 'none'
        document.querySelector('.arriving-wrap').classList.remove('square')
        banner.classList.remove('square')
      }
    }
  }
  // display free_count in the right panel (optional)
  const metaEl = document.querySelector('.meta')
// Constant/configuration
  if(metaEl && typeof j.free_count !== 'undefined'){
// Control flow
    metaEl.textContent = `Last update: ${new Date(j.ts).toLocaleTimeString()} — Free spots: ${j.free_count}`
  }
  const pill = document.getElementById('closest-pill')
// Constant/configuration
  pill.textContent = closest ? `(${closest})` : '-'
      if(arrivingIdEl){
// Control flow
        const gid = j.gate_waiting_car || '-'
// Constant/configuration
        // always set text (ensures UI shows current value even if we missed a transient)
        const prev = arrivingIdEl.textContent
// Constant/configuration
        arrivingIdEl.textContent = gid
        // animate when id changes
        if(prev !== gid){
// Control flow
          arrivingIdEl.classList.remove('pulse')
          void arrivingIdEl.offsetWidth
          arrivingIdEl.classList.add('pulse')
        }
      }
    tsEl.textContent = new Date(j.ts).toLocaleTimeString()
  }catch(e){
    console.error(e)
  }
}
// poll faster so UI catches transient waiting states
setInterval(poll, 400)
poll()
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## styles.css

#### File explanation
Visual appearance and color codes for each status.



**Overview.** Visual styling and color coding for statuses.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



css/* VISUAL DIAGRAM — High-Level Flow
ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
  Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot */
/* Visual styling and color coding for statuses. */

:root{
  --free: #6fa8dc; /* blue */
  --waiting: #f6b26b; /* orange */
  --occupied: #e06666; /* red */
  --wrong: #7a4bd6; /* purple for wrong-park */
  --bg: #f7f7f7;
}
body{font-family: Arial, sans-serif; background:var(--bg); margin:0; padding:12px}
.container{max-width:1100px;margin:0 auto;position:relative}
h1{margin:8px 0}
#controls{display:flex;gap:20px;align-items:center;margin-bottom:12px}
.badge{display:inline-block;padding:8px 14px;border-radius:20px;background:#d0d0d0}
.layout{display:flex;gap:30px}
.left{flex:1}
.right{width:360px;display:flex;flex-direction:column;align-items:center;justify-content:center}

.gate-status{position:absolute;right:24px;top:16px;background:#fff;padding:10px 14px;border-radius:8px;border:1px solid #ddd;font-weight:700}

.arriving{position:absolute;right:36px;top:20px;display:flex;align-items:center;gap:12px}
.arriving-label{background:#000;color:#1e90ff;padding:8px 18px;border-radius:4px;font-weight:700;font-size:20px;box-shadow:0 0 0 6px #000}
.arriving-id-box{width:140px;min-width:140px;height:40px;border:3px solid #111;background:#fff;display:inline-flex;align-items:center;justify-content:center;font-weight:800;font-size:18px}

/* wrapper to hold arriving box and the inline banner beneath it */
.arriving-wrap{position:static;display:flex;flex-direction:column;align-items:center;gap:8px;margin-bottom:18px}
.arriving-wrap .arriving{position:static}
.arriving-wrap .side-banner{margin-top:0;min-width:220px;max-width:420px;width:420px;border-radius:12px;padding:10px 14px;font-size:20px}

/* Make the banner a compact square/box in the right side */
.arriving-wrap.square .side-banner,
.arriving-wrap .side-banner.square {
  /* large rectangular framed card */
  width:420px;
  height:260px;
  min-width:0;
  max-width:100%;
  padding:0 18px;
  display:flex;
  align-items:center;
  justify-content:center;
  border-radius:18px;
  border:12px solid #000; /* thick black frame */
  box-shadow:0 10px 30px rgba(0,0,0,0.18);
  font-size:36px;
  font-weight:900;
  text-align:center;
  line-height:1.05;
  color:#000; /* ensure readable text */
}

@keyframes pulse {
  0% { box-shadow: 0 0 0 0 rgba(30,144,255,0.0); }
  50% { box-shadow: 0 0 12px 4px rgba(30,144,255,0.25); }
  100% { box-shadow: 0 0 0 0 rgba(30,144,255,0.0); }
}
.arriving-id-box.pulse{ animation: pulse 900ms ease-in-out; }

#grid{display:grid;grid-template-columns:repeat(5, 120px);gap:20px}
.col{display:flex;flex-direction:column;gap:18px;align-items:center}
.col .gate{width:60px;height:30px;border-radius:8px;background:#222;color:#fff;display:flex;align-items:center;justify-content:center;margin-bottom:6px}

.closest-pill{margin-top:28px;background:var(--free);padding:18px 36px;border-radius:36px;font-size:20px;font-weight:700;color:#03386a;border:4px solid rgba(0,0,0,0.1)}
.meta{margin-top:12px;color:#666}
.spot{width:60px;height:40px;border-radius:6px;display:flex;align-items:center;justify-content:center;color:#fff;font-weight:bold}
.spot.free{background:var(--free);color:#03386a}
.spot.waiting{background:var(--waiting);color:#6b2e00}
.spot.occupied{background:var(--occupied);color:#641414}
.spot.wrong{background:var(--wrong);color:#fff}

.spot-label{height:18px;line-height:18px}

/* Banner: default is an inline card (not fixed). The arriving-wrap places it under the Arriving ID. */
.side-banner{
  display:flex;
  align-items:center;
  justify-content:center;
  background: linear-gradient(180deg,#9be86a,#7ecf4a);
  color:#e53935; /* red text */
  font-weight:800;
  padding:18px 22px;
  border:8px solid #000; /* heavy black frame as in design */
  border-radius:6px;
  box-shadow:0 6px 18px rgba(0,0,0,0.12);
  text-align:center;
  font-size:22px;
  line-height:1.1;
  width:auto !important;
  max-width:100% !important;
}

@media (max-width:480px){
  .side-banner{
    width:100%;
    box-sizing:border-box;
    font-size:16px;
    padding:12px 14px;
    border-width:6px;
  }
}

/* Inline banner variant when placed inside the right column */
.right .side-banner{
  /* when used in the right column, keep it compact */
  width:420px;
  max-width:100%;
  padding:14px 16px;
  font-size:18px;
}

/* make sure the fixed-top style doesn't interfere when used inline */
.side-banner[style*="display:none"]{display:none}

@media (max-width:600px){
  .spot{width:48px;height:34px;font-size:12px}
  #grid{gap:8px}
}

/* responsive adjustments for the side banner */
@media (max-width:900px){
  .arriving-wrap.square .side-banner,
  .arriving-wrap .side-banner.square {
    width:300px;
    height:180px;
    font-size:22px;
    border-width:10px;
  }
  .right{width:320px}
}
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.



## simulation_sondos.py

#### File explanation
Top-level orchestrator to reset, simulate arrivals, wrong-parks, and departures.



**Overview.** End-to-end simulation orchestrator with resets and wrong-park injections.

### Key concepts


- Study-friendly notes and APIs.

### Annotated code

#### Block purpose
This code block shows the **entire file** with inline comments. Use it to understand what each statement does and how data flows across modules.



python# VISUAL DIAGRAM — High-Level Flow
# ESP32 Spot  →  Firebase RTDB  →  Flask API  →  Tablet Dashboard
#   Sensor     →   /SPOTS path   →   /api/status →  Grid + Closest spot
# End-to-end simulation orchestrator with resets and wrong-park injections.

import random
# Library import/include
import time
# Library import/include
from firebase_admin import db
# Library import/include
from firebase_init import db as _db_init  # ensures app is initialized
# Library import/include
from constants import ROOT_BRANCH
# Library import/include
from data_structures import ParkingLot, Spot
# Library import/include
from event_generator import simulate_car_arrival, simulate_car_parked, simulate_car_departure, generate_plate_id
# Library import/include
import os
# Library import/include


# obtain the SPOTS reference lazily to avoid using a reference created before firebase app init
# RTDB path / configuration
def get_spots_ref():
# Function start
    return db.reference(f"/{ROOT_BRANCH}/SPOTS")
# RTDB path / configuration


def get_cars_ref():
# Function start
    """Return the CARS reference under the configured ROOT_BRANCH.
# RTDB path / configuration

    Other modules use db.reference(ROOT_BRANCH).child('CARS'), so ensure
# RTDB path / configuration
    we operate on the same path instead of the top-level '/CARS'.
# RTDB path / configuration
    """
    return db.reference(f"/{ROOT_BRANCH}/CARS")
# RTDB path / configuration


def migrate_top_level_cars(copy_only: bool = True):
# Function start
    """Copy car records referenced by SPOTS from top-level /CARS into /{ROOT_BRANCH}/CARS.
# RTDB path / configuration

    - If copy_only is True (default) the function copies records and leaves originals.
    - If copy_only is False, after copying the referenced records it will delete them from /CARS.
# RTDB path / configuration

    Returns a dict with counts: {'found': n_found, 'copied': n_copied, 'deleted': n_deleted}
    """
    spots = get_spots_ref().get() or {}
    occupied = {k: v for k, v in spots.items() if isinstance(v, dict) and (v.get('status') or '').upper() != 'FREE'}
    if not occupied:
# Control flow
        print("[SIM] No occupied spots found; nothing to migrate.")
        return {'found': 0, 'copied': 0, 'deleted': 0}

    top_ref = db.reference('/CARS')
# RTDB path / configuration
    ns_ref = get_cars_ref()

    top_data = top_ref.get() or {}
    ns_data = ns_ref.get() or {}

    found_ids = set(v.get('carId') for v in occupied.values() if v.get('carId'))
    found_ids = {fid for fid in found_ids if fid}
    print(f"[SIM] Found {len(found_ids)} carIds referenced by occupied spots")

    copied = 0
    deleted = 0
    for cid in found_ids:
# Control flow
        if cid in ns_data:
# Control flow
            # already present in namespaced node
            continue
        if cid in top_data:
# Control flow
            try:
# Control flow
                ns_ref.child(cid).set(top_data[cid])
                copied += 1
            except Exception as e:
# Control flow
                print(f"[SIM] Failed to copy car {cid}:", e)

    if not copy_only:
# Control flow
        for cid in found_ids:
# Control flow
            if cid in top_data:
# Control flow
                try:
# Control flow
                    top_ref.child(cid).delete()
                    deleted += 1
                except Exception as e:
# Control flow
                    print(f"[SIM] Failed to delete top-level car {cid}:", e)

    print(f"[SIM] Migration summary: found={len(found_ids)} copied={copied} deleted={deleted} (copy_only={copy_only})")
    return {'found': len(found_ids), 'copied': copied, 'deleted': deleted}


def clear_cars_and_reset_spots():
# Function start
    """Remove all car records from the DB and set every spot to FREE.

    This is a stronger reset than `set_all_spots_free()` because it also
    deletes the `CARS` node entirely so there are no leftover car records.
# RTDB path / configuration
    """
    print(f"[SIM] Clearing all car records from /{ROOT_BRANCH}/CARS and resetting /{ROOT_BRANCH}/SPOTS to FREE...")
# RTDB path / configuration
    cars_ref = get_cars_ref()

    # attempt to delete the whole node, retry a few times if necessary
    for attempt in range(3):
# Control flow
        try:
# Control flow
            cars_ref.delete()
            print(f"[SIM] /{ROOT_BRANCH}/CARS deleted.")
# RTDB path / configuration
            break
        except Exception as e:
# Control flow
            print(f"[SIM] Attempt {attempt+1} to delete /{ROOT_BRANCH}/CARS failed:", e)
# RTDB path / configuration
            # fallback to per-child delete on failure
            try:
# Control flow
                data = cars_ref.get() or {}
                for k in list(data.keys()):
# Control flow
                    try:
# Control flow
                        cars_ref.child(k).delete()
                    except Exception:
# Control flow
                        pass
                # re-check
                remaining = cars_ref.get() or {}
                if not remaining:
# Control flow
                    print(f"[SIM] /{ROOT_BRANCH}/CARS children deleted via fallback.")
# RTDB path / configuration
                    break
            except Exception as e2:
# Control flow
                print(f"[SIM] Failed to enumerate /{ROOT_BRANCH}/CARS for deletion:", e2)
# RTDB path / configuration
        time.sleep(0.5)
    else:
        print(f"[SIM] Warning: /{ROOT_BRANCH}/CARS could not be fully deleted after retries.")
# RTDB path / configuration

    # now set all spots to FREE (reuse helper) and verify
    set_all_spots_free()

    # verify spots and cars state; retry a couple times if DB hasn't reflected changes
    for attempt in range(5):
# Control flow
        try:
# Control flow
            cars_now = get_cars_ref().get()
            spots_now = get_spots_ref().get() or {}
            all_free = True
            for v in spots_now.values():
# Control flow
                if isinstance(v, dict) and (v.get('status') or '').upper() != 'FREE':
# Control flow
                    all_free = False
                    break
            if (not cars_now or cars_now == {}) and all_free:
# Control flow
                print(f"[SIM] Verification succeeded: /{ROOT_BRANCH}/CARS empty and all spots FREE.")
# RTDB path / configuration
                break
            else:
                print(f"[SIM] Verification attempt {attempt+1}: CARS present? {bool(cars_now)}; all_free? {all_free}. Retrying...")
# RTDB path / configuration
                # try resetting spots again if needed
                if not all_free:
# Control flow
                    set_all_spots_free()
        except Exception as e:
# Control flow
            print("[SIM] Verification error:", e)
        time.sleep(0.5)
    else:
        print("[SIM] Warning: verification failed — DB may not be fully reset.")


def load_parking_lot_from_db():
# Function start
    data = get_spots_ref().get() or {}
    if not data:
# Control flow
        print("[SIM] No spots found — did you run the initializer?")
        return None, {}

    pl = ParkingLot()
    # Keep a backup to restore later
    backup = {}

    for sid, s in data.items():
# Control flow
        if not isinstance(s, dict):
# Control flow
            continue
        # sid expected in form 'row,col' or '(row,col)'
        backup[sid] = s
        try:
# Control flow
            row_str, col_str = sid.strip('()').split(',')
            row, col = int(row_str), int(col_str)
        except Exception:
# Control flow
            # try splitting on other delimiters
            parts = sid.replace('(', '').replace(')', '').split(',')
            if len(parts) >= 2:
# Control flow
                row, col = int(parts[0]), int(parts[1])
            else:
                continue

        dist = s.get('distanceFromEntry', 0) or 0
        spot = Spot(row, col, dist)
        # mirror status from DB
        spot.status = s.get('status', 'FREE')
        spot.waiting_car_id = s.get('waitingCarId', '-')
        spot.seen_car_id = s.get('seenCarId', '-')
        pl.spot_lookup[spot.spot_id] = spot
        if spot.status == 'FREE':
# Control flow
            pl.free_spots.add(spot)

    # debug: print free spots and distances
    print(f"[SIM] Loaded parking lot: free_spots_count={len(pl.free_spots)}")
    sample = [(sp.spot_id, sp.distance_from_entry) for sp in pl.free_spots]
    print("[SIM] Free spots (id,dist) sample:", sample[:10])

    return pl, backup


def set_all_spots_free():
# Function start
    """Set every spot under ROOT_BRANCH/SPOTS to FREE in the RTDB.
# RTDB path / configuration

    This writes a minimal FREE state (status, carId, seenCarId, waitingCarId, lastUpdateMs)
    for every spot key found under the SPOTS node.
# Control flow
    """
    data = get_spots_ref().get() or {}
    if not data:
# Control flow
        print("[SIM] No spots found to reset.")
        return

    ts = int(time.time() * 1000)
    print(f"[SIM] Setting all {len(data)} spots to FREE...")
    for sid in data.keys():
# Control flow
        payload = {
            'status': 'FREE',
            'carId': None,
            'seenCarId': '-',
            'waitingCarId': '-',
            'lastUpdateMs': ts,
        }
        # attempt a few times per spot in case of transient errors
        for attempt in range(3):
# Control flow
            try:
# Control flow
                # use set() to ensure the entire spot entry is written to the expected shape
                get_spots_ref().child(sid).set({**(data.get(sid) if isinstance(data.get(sid), dict) else {}), **payload})
                break
            except Exception as e:
# Control flow
                print(f"⚠️ Failed to reset spot {sid} (attempt {attempt+1}):", e)
                time.sleep(0.2)
        else:
            print(f"⚠️ Giving up resetting spot {sid} after retries.")
    print("[SIM] All spots set to FREE (requests issued).")


def inject_wrong_park(parking_lot: ParkingLot):
# Function start
    """Cause a car to park in a random free spot that is NOT the BFS-closest.

    Behavior:
    - Choose the current BFS closest via parking_lot.find_closest
    - From the free_spots choose a different random free spot
    - Write a temporary 'WRONG_PARK' state to that spot (used to show purple) for 2s
    - After 2s, set spot to 'OCCUPIED' and create a CARS entry for the parked car
# RTDB path / configuration
    - Update parking_lot internal structures accordingly
    Returns the spot_id chosen or None on failure.
    """
    try:
# Control flow
        if not parking_lot:
# Control flow
            return None
        # determine BFS closest
        try:
# Control flow
            bfs = parking_lot.find_closest()
            bfs_key = f"{bfs[0]},{bfs[1]}" if bfs else None
        except Exception:
# Control flow
            bfs_key = None

        # list free spots available
        free_list = []
        if hasattr(parking_lot, 'free_spots'):
# Control flow
            try:
# Control flow
                free_list = [sp for sp in parking_lot.free_spots if getattr(sp, 'spot_id', None) != bfs_key]
            except Exception:
# Control flow
                # fallback: build from spot_lookup
                free_list = [s for s in parking_lot.spot_lookup.values() if getattr(s, 'status', None) == 'FREE' and s.spot_id != bfs_key]

        if not free_list:
# Control flow
            print("[SIM] No alternative free spot available for wrong-park")
            return None

        chosen = random.choice(free_list)
        chosen_id = chosen.spot_id if hasattr(chosen, 'spot_id') else str(chosen)

        spots_ref = get_spots_ref()
        cars_ref = get_cars_ref()

        # set temporary wrong-park visual state (use status 'WRONG_PARK' so UI can color purple)
        ts = int(time.time() * 1000)
        try:
# Control flow
            spots_ref.child(str(chosen_id)).update({'status': 'WRONG_PARK', 'carId': None, 'seenCarId': '-', 'waitingCarId': '-', 'lastUpdateMs': ts})
        except Exception as e:
# Control flow
            print("⚠️ Failed to write WRONG_PARK state for", chosen_id, e)

        # mark the BFS-closest spot as WAITING (orange) to reflect that the system had intended
        # the car to go there. Remove it from free_spots so UI shows orange.
        if bfs_key:
# Control flow
            try:
# Control flow
                spots_ref.child(str(bfs_key)).update({'status': 'WAITING', 'waitingCarId': '-', 'lastUpdateMs': ts})
                spobj = parking_lot.get_spot(bfs_key) if hasattr(parking_lot, 'get_spot') else None
                if spobj:
# Control flow
                    spobj.status = 'WAITING'
                    try:
# Control flow
                        parking_lot.remove_spot_from_free(spobj)
                    except Exception:
# Control flow
                        try:
# Control flow
                            parking_lot.remove_spot_from_free(bfs_key)
                        except Exception:
# Control flow
                            pass
            except Exception:
# Control flow
                pass

        print(f"[SIM] Injected wrong-park at {chosen_id} (purple) and set closest {bfs_key} to WAITING (orange).")
        # keep the closest WAITING state briefly (1s), but keep the wrong-park purple for 2s total
        time.sleep(1)

        # restore the BFS-closest spot to FREE (after 1s)
        if bfs_key:
# Control flow
            try:
# Control flow
                spots_ref.child(str(bfs_key)).update({'status': 'FREE', 'waitingCarId': '-', 'carId': None, 'seenCarId': '-', 'lastUpdateMs': int(time.time() * 1000)})
                spobj = parking_lot.get_spot(bfs_key) if hasattr(parking_lot, 'get_spot') else None
                if spobj:
# Control flow
                    spobj.status = 'FREE'
                    try:
# Control flow
                        parking_lot.add_spot_to_free(spobj)
                    except Exception:
# Control flow
                        try:
# Control flow
                            parking_lot.free_spots.add(spobj)
                        except Exception:
# Control flow
                            pass
            except Exception:
# Control flow
                pass

        # keep the wrong-park purple for one more second (total 2s)
        time.sleep(1)

        # mark wrong spot as occupied and create car record
        plate = generate_plate_id()
        car_payload = {'Id': plate, 'allocatedSpot': chosen_id, 'status': 'parked', 'SpotIn': {'Arrievied': True}, 'timestamp': time.time()}
        try:
# Control flow
            cars_ref.child(plate).set(car_payload)
            spots_ref.child(str(chosen_id)).update({'status': 'OCCUPIED', 'carId': plate, 'seenCarId': plate, 'waitingCarId': '-', 'lastUpdateMs': int(time.time() * 1000)})
        except Exception as e:
# Control flow
            print("⚠️ Failed to finalize wrong-park for", chosen_id, e)

        # update parking_lot internals: remove chosen from free and mark occupied
        try:
# Control flow
            try:
# Control flow
                parking_lot.remove_spot_from_free(chosen)
            except Exception:
# Control flow
                try:
# Control flow
                    parking_lot.remove_spot_from_free(chosen_id)
                except Exception:
# Control flow
                    pass
            if hasattr(parking_lot, 'add_occupied_spot'):
# Control flow
                parking_lot.add_occupied_spot(chosen_id, plate)
            else:
                parking_lot.occupied_spots_with_cars[chosen_id] = plate
        except Exception:
# Control flow
            pass

        print(f"[SIM] Wrong-park finalized: car {plate} at {chosen_id}; closest {bfs_key} was restored to FREE earlier")
        return chosen_id
    except Exception as e:
# Control flow
        print("[SIM] inject_wrong_park failed:", e)
        return None



def refresh_parking_lot(parking_lot: ParkingLot):
# Function start
    """Refresh the in-memory parking lot state from the database without losing structure.
    
    This updates the status and occupancy of spots to reflect external changes.
    """
    try:
# Control flow
        data = get_spots_ref().get() or {}
        if not data:
# Control flow
            return parking_lot
        
        # Update existing spots
        for sid, s in data.items():
# Control flow
            if not isinstance(s, dict):
# Control flow
                continue
            
            spot = parking_lot.spot_lookup.get(sid)
            if not spot:
# Control flow
                continue
            
            old_status = spot.status
            new_status = s.get('status', 'FREE')
            
            # Update spot attributes
            spot.status = new_status
            spot.waiting_car_id = s.get('waitingCarId', '-')
            spot.seen_car_id = s.get('seenCarId', '-')
            
            # Synchronize free_spots set
            if new_status == 'FREE' and spot not in parking_lot.free_spots:
# Control flow
                parking_lot.free_spots.add(spot)
            elif new_status != 'FREE' and spot in parking_lot.free_spots:
                parking_lot.free_spots.discard(spot)
            
            # Update occupied tracking
            car_id = s.get('carId')
            if new_status == 'OCCUPIED' and car_id:
# Control flow
                parking_lot.occupied_spots_with_cars[sid] = car_id
            elif sid in parking_lot.occupied_spots_with_cars and new_status != 'OCCUPIED':
                parking_lot.occupied_spots_with_cars.pop(sid, None)
        
        print(f"[SIM] Refreshed parking lot: free_spots={len(parking_lot.free_spots)}, occupied={len(parking_lot.occupied_spots_with_cars)}")
        return parking_lot
    except Exception as e:
# Control flow
        print(f"[SIM] Error refreshing parking lot: {e}")
        return parking_lot


def simulate_n_arrivals(n: int = 5, keep_changes: bool = False, wait_between: float = 1.0, arrival_interval: float = 7.0):
# Function start
    # ensure we start from a clean state in the DB: remove all cars and set spots FREE
    clear_cars_and_reset_spots()
    # reload parking lot from DB so in-memory model matches what we just written
    pl, backup = load_parking_lot_from_db()
    if pl is None:
# Control flow
        return

    created_plates = []
    # periodic departure timer
    depart_interval = float(os.environ.get('DEPART_INTERVAL_SECONDS', '30'))
    # when the lot is full we want an accelerated departure cadence
    depart_when_full = float(os.environ.get('DEPART_WHEN_FULL_SECONDS', '10'))
    last_depart_time = time.time()
    # wrong-park injector (every X seconds a car parks in a non-closest free spot)
    wrong_park_interval = float(os.environ.get('WRONG_PARK_SECONDS', '45'))
    last_wrong_time = time.time()
    # Add refresh interval
    refresh_interval = float(os.environ.get('REFRESH_INTERVAL_SECONDS', '3'))
    last_refresh_time = time.time()
    
    try:
# Control flow
        # sequential arrival -> parked for each car, ensuring allocation moves spot to occupied
        print(f"[SIM] Arrival interval set to {arrival_interval} seconds (env ARRIVAL_INTERVAL_SECONDS)")
        for i in range(n):
# Control flow
            # Periodically refresh parking lot state from DB
            if time.time() - last_refresh_time >= refresh_interval:
# Control flow
                pl = refresh_parking_lot(pl)
                last_refresh_time = time.time()
            
            # if in-memory shows no free spots, we still trigger departures every depart_when_full seconds
            try:
# Control flow
                if (not getattr(pl, 'free_spots', None)) or (hasattr(pl, 'free_spots') and len(pl.free_spots) == 0):
# Control flow
                    if time.time() - last_depart_time >= depart_when_full:
# Control flow
                        simulate_car_departure(pl)
                        last_depart_time = time.time()
                        # give a short moment for DB writes to propagate
                        time.sleep(min(1.0, wait_between))
                        # re-load parking lot state to reflect departure
                        pl, _ = load_parking_lot_from_db()
            except Exception:
# Control flow
                pass
            start_ts = time.time()
            plate = simulate_car_arrival(pl)
            created_plates.append(plate)
            # allow a short delay for the WAITING state to be written/read
            time.sleep(wait_between)
            # print debug snapshot after allocation
            print(f"[SIM] After arrival: free_spots_count={len(pl.free_spots)}; waiting_pair={pl.get_waiting_pair()}")
            # simulate the car physically parking
            simulate_car_parked(pl, plate)
            print(f"[SIM] After parked: free_spots_count={len(pl.free_spots)}; occupied_count={len(pl.occupied_spots_with_cars)}")
            # extra small sleep to let parked writes propagate
            time.sleep(wait_between)

            # pace arrivals so that each arrival starts approximately arrival_interval seconds apart
            elapsed = time.time() - start_ts
            remaining = arrival_interval - elapsed
            if remaining > 0:
# Control flow
                print(f"[SIM] Sleeping {remaining:.2f}s until next arrival (to respect arrival_interval)")
                time.sleep(remaining)

            # periodically inject a wrong-park event (a car parks in a random free spot that is NOT the closest)
            try:
# Control flow
                if time.time() - last_wrong_time >= wrong_park_interval:
# Control flow
                    _ = inject_wrong_park(pl)
# Constant/configuration
                    last_wrong_time = time.time()
            except Exception:
# Control flow
                pass

            # after all have parked, explicitly free each allocated spot and remove the car record
        cars_ref = get_cars_ref()
        spots_ref = get_spots_ref()
        parked_info = []
        for plate in created_plates:
# Control flow
            car_rec = cars_ref.child(plate).get() or {}
            allocated = car_rec.get('allocatedSpot')
            if allocated:
# Control flow
                parked_info.append((plate, allocated))

        for plate, spot in parked_info:
# Control flow
            ts = int(time.time() * 1000)
            try:
# Control flow
                spots_ref.child(str(spot)).update({'status': 'FREE', 'carId': None, 'seenCarId': '-', 'waitingCarId': '-', 'lastUpdateMs': ts})
            except Exception as e:
# Control flow
                print("⚠️ Failed to set spot FREE in DB:", spot, e)

            # update parking lot internals
            try:
# Control flow
                if hasattr(pl, 'remove_occupied_spot'):
# Control flow
                    pl.remove_occupied_spot(spot)
                elif hasattr(pl, 'remove_car'):
                    pl.remove_car(spot)
            except Exception:
# Control flow
                pass

            # remove car record from DB
            try:
# Control flow
                cars_ref.child(plate).delete()
            except Exception:
# Control flow
                pass

            print(f"🚗 Car {plate} departed from spot {spot} and spot set to FREE")
            time.sleep(wait_between)
            # trigger a periodic departure if enough time passed
            try:
# Control flow
                if time.time() - last_depart_time >= depart_interval:
# Control flow
                    simulate_car_departure(pl)
                    last_depart_time = time.time()
            except Exception:
# Control flow
                pass

        print(f"[SIM] Simulated {n} arrivals and parked them.")

        if not keep_changes:
# Control flow
            print("[SIM] Restoring original SPOTS from backup...")
# RTDB path / configuration
            for sid, s in backup.items():
# Control flow
                try:
# Control flow
                    get_spots_ref().child(sid).set(s)
                except Exception as e:
# Control flow
                    print("⚠️ Failed to restore spot", sid, e)
            # remove created cars
            cars_ref = get_cars_ref()
            for p in created_plates:
# Control flow
                try:
# Control flow
                    cars_ref.child(p).delete()
                except Exception:
# Control flow
                    pass
            print("[SIM] Restore complete.")
    except KeyboardInterrupt:
# Control flow
        print("[SIM] Interrupted by user — leaving current DB state as-is.")


def simulate_continuous_arrivals(keep_changes: bool = False, wait_between: float = 1.0, arrival_interval: float = 5.0):
# Function start
    """Run arrivals forever (until KeyboardInterrupt). Each arrival is paced by arrival_interval seconds.

    If the parking lot is full, a message is printed to the console each interval.
    On KeyboardInterrupt, if keep_changes is False the original SPOTS backup is restored and created car records removed.
# RTDB path / configuration
    """
    # prepare DB and in-memory lot (remove cars and reset spots to FREE)
    clear_cars_and_reset_spots()
    pl, backup = load_parking_lot_from_db()
    if pl is None:
# Control flow
        return

    created_plates = []
    depart_interval = float(os.environ.get('DEPART_INTERVAL_SECONDS', '30'))
    depart_when_full = float(os.environ.get('DEPART_WHEN_FULL_SECONDS', '10'))
    last_depart_time = time.time()
    wrong_park_interval = float(os.environ.get('WRONG_PARK_SECONDS', '45'))
    last_wrong_time = time.time()
    # Add refresh interval to periodically sync with DB
    refresh_interval = float(os.environ.get('REFRESH_INTERVAL_SECONDS', '3'))
    last_refresh_time = time.time()
    
    try:
# Control flow
        print(f"[SIM] Starting continuous arrivals every {arrival_interval}s (press Ctrl+C to stop)")
        while True:
# Control flow
            # Periodically refresh parking lot state from DB to catch external changes
            if time.time() - last_refresh_time >= refresh_interval:
# Control flow
                pl = refresh_parking_lot(pl)
                last_refresh_time = time.time()
            
            start_ts = time.time()
            # if no free spots, print a message and wait until next interval
            if not getattr(pl, 'free_spots', None) or len(pl.free_spots) == 0:
# Control flow
                # double-check the real DB in case the in-memory model drifted
                try:
# Control flow
                    data = get_spots_ref().get() or {}
                    db_free = sum(1 for v in data.values() if isinstance(v, dict) and (v.get('status') or '').upper() == 'FREE')
                except Exception:
# Control flow
                    db_free = 0

                if db_free == 0:
# Control flow
                    # parking full: trigger departures every depart_when_full seconds
                    try:
# Control flow
                        if time.time() - last_depart_time >= depart_when_full:
# Control flow
                            simulate_car_departure(pl)
                            last_depart_time = time.time()
                            # short wait to allow DB propagation before re-evaluating
                            time.sleep(min(arrival_interval, 1.0))
                            continue
                    except Exception:
# Control flow
                        pass
                    print("[SIM] PARKING FULL — no free spots available. Waiting until next check...")
                    time.sleep(arrival_interval)
                    continue
                else:
                    # DB reports free spots; resync the in-memory parking lot and continue
                    print(f"[SIM] In-memory free_spots empty but DB shows {db_free} FREE spots -> resyncing in-memory model")
                    try:
# Control flow
                        pl, backup = load_parking_lot_from_db()
                        if pl is None:
# Control flow
                            print("[SIM] Failed to reload parking lot from DB during resync")
                            time.sleep(arrival_interval)
                            continue
                    except Exception as e:
# Control flow
                        print(f"[SIM] Error resyncing parking lot: {e}")
                        time.sleep(arrival_interval)
                        continue

            plate = simulate_car_arrival(pl)
            if not plate:
# Control flow
                print("[SIM] simulate_car_arrival returned no plate — skipping")
                time.sleep(arrival_interval)
                continue

            created_plates.append(plate)
            # short delay so WAITING state persists briefly
            time.sleep(wait_between)
            print(f"[SIM] Arrival: free_spots_count={len(pl.free_spots)}; waiting_pair={pl.get_waiting_pair()}")
            simulate_car_parked(pl, plate)
            # (optionally) record or log parked cars; departures are timed periodically
            print(f"[SIM] Parked: free_spots_count={len(pl.free_spots)}; occupied_count={len(pl.occupied_spots_with_cars)}")
            time.sleep(wait_between)

            # pace to arrival_interval
            elapsed = time.time() - start_ts
            remaining = arrival_interval - elapsed
            if remaining > 0:
# Control flow
                time.sleep(remaining)

            # periodically inject wrong-park events
            try:
# Control flow
                if time.time() - last_wrong_time >= wrong_park_interval:
# Control flow
                    _ = inject_wrong_park(pl)
# Constant/configuration
                    last_wrong_time = time.time()
            except Exception:
# Control flow
                pass

            # trigger periodic departure
            try:
# Control flow
                if time.time() - last_depart_time >= depart_interval:
# Control flow
                    simulate_car_departure(pl)
                    last_depart_time = time.time()
            except Exception:
# Control flow
                pass

    except KeyboardInterrupt:
# Control flow
        print("[SIM] Continuous simulation interrupted by user — cleaning up...")
        if not keep_changes:
# Control flow
            print("[SIM] Restoring original SPOTS from backup...")
# RTDB path / configuration
            for sid, s in backup.items():
# Control flow
                try:
# Control flow
                    get_spots_ref().child(sid).set(s)
                except Exception as e:
# Control flow
                    print("⚠️ Failed to restore spot", sid, e)
            cars_ref = get_cars_ref()
            for p in created_plates:
# Control flow
                try:
# Control flow
                    cars_ref.child(p).delete()
                except Exception:
# Control flow
                    pass
            print("[SIM] Restore complete.")
        else:
            print("[SIM] Leaving changes in RTDB (KEEP_CHANGES=1).")


def main():
# Function start
    # Ensure DB is cleared and all spots set to FREE at program start
    clear_cars_and_reset_spots()

    keep = os.environ.get('KEEP_CHANGES', '0') == '1'
    wait = float(os.environ.get('WAIT_SECONDS', '1'))
    n = int(os.environ.get('N_ARRIVALS', '0'))
    arrival_interval = float(os.environ.get('ARRIVAL_INTERVAL_SECONDS', '5'))
    # If N_ARRIVALS is 0 -> run continuous arrivals until interrupted
    if n == 0:
# Control flow
        simulate_continuous_arrivals(keep_changes=keep, wait_between=wait, arrival_interval=arrival_interval)
    else:
        simulate_n_arrivals(n, keep_changes=keep, wait_between=wait, arrival_interval=arrival_interval)


if __name__ == "__main__":
# Control flow
    main()
#### Code block explanation
Read line-by-line comments to understand purpose, control flow, and data paths. Check RTDB paths, timing constants, and retries for robustness.


