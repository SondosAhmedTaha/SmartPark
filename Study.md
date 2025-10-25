# 🌈 SmartPark Study Guide

Welcome to your colorful, study-friendly companion! Each section highlights a file from the SmartPark project, shows the source code (within collapsible blocks), and explains what every line or small group of lines is doing. Use the color-coded notes to memorize responsibilities quickly.

---

## 🗂️ Server/constants.py
<details>
<summary><strong>Show code</strong></summary>

```python
ROOT_BRANCH = "SondosPark"
STAT_FREE = "FREE"
STAT_WAIT = "WAITING"
STAT_OCC = "OCCUPIED"
```
</details>

| Line(s) | Explanation |
| --- | --- |
| 1 | <span style="color:#ff6f61">Defines</span> the Firebase Realtime Database root branch the backend writes under. |
| 2 | <span style="color:#6b5b95">Symbol</span> shared by services to tag a spot as free. |
| 3 | <span style="color:#88b04b">Symbol</span> for the waiting/reserved state in queues and UI. |
| 4 | <span style="color:#f7cac9">Symbol</span> for an occupied spot. |

---

## 🧠 Server/data_structures.py
<details>
<summary><strong>Show code</strong></summary>

```python
"""
Full source included for reference while studying.
"""
```

```python
from typing import Dict, Optional, Tuple, Callable
import random
from bisect import bisect_left, insort
from collections import deque

class SortedList:
    """A small sorted list with optional key function. Compatible with previous API.

    If key is provided, items are ordered by key(item). Internally stores
    tuples (k, item) when key is set to allow use of insort.
    """
    def __init__(self, iterable=None, key: Optional[Callable] = None):
        self._key = key
        self._list = []
        if iterable:
            for item in iterable:
                self.add(item)

    def _wrap(self, value):
        return (self._key(value), value) if self._key is not None else value

    def add(self, value):
        if self._key is None:
            insort(self._list, value)
        else:
            # Ensure internal storage uses (key, item) tuples
            if self._list and not isinstance(self._list[0], tuple):
                self._list = [(self._key(v), v) for v in self._list]
            k = self._key(value)
            # build keys list for bisect search to avoid comparing items directly
            keys = [kv[0] for kv in self._list]
            idx = bisect_left(keys, k)
            self._list.insert(idx, (k, value))

    def remove(self, value):
        if self._key is None:
            idx = bisect_left(self._list, value)
            if idx != len(self._list) and self._list[idx] == value:
                self._list.pop(idx)
                return
            raise ValueError(f"{value} not in SortedList")
        else:
            # find by identity of stored item
            for i, (k, item) in enumerate(self._list):
                if item == value:
                    self._list.pop(i)
                    return
            raise ValueError(f"{value} not in SortedList")

    def pop(self, index=-1):
        val = self._list.pop(index)
        return val[1] if self._key is not None else val

    def __len__(self):
        return len(self._list)

    def __getitem__(self, idx):
        val = self._list[idx]
        return val[1] if self._key is not None else val

    def __iter__(self):
        if self._key is None:
            return iter(self._list)
        else:
            return (item for (_, item) in self._list)

    def __contains__(self, value):
        if self._key is None:
            idx = bisect_left(self._list, value)
            return idx != len(self._list) and self._list[idx] == value
        else:
            return any(item == value for (_, item) in self._list)

    def index(self, value):
        if self._key is None:
            idx = bisect_left(self._list, value)
            if idx != len(self._list) and self._list[idx] == value:
                return idx
            raise ValueError(f"{value} not in SortedList")
        else:
            for i, (_, item) in enumerate(self._list):
                if item == value:
                    return i
            raise ValueError(f"{value} not in SortedList")

class Spot:
    """Represents a parking spot with coordinates, distance, and status"""

    def __init__(self, row: int, col: int, distance: int):
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
        """Add spot to both free_spots list and spot_lookup hash"""
        self.free_spots.add(spot)
        self.spot_lookup[spot.spot_id] = spot

    def add_car(self, car):
        """Add car to car_lookup hash"""
        self.car_lookup[car.plate_id] = car

    def get_closest_free_spot(self):
        """Return closest free spot (first element) or None if empty"""
        return self.free_spots[0] if self.free_spots else None

    def get_time_saved(self):
        """Get the time saved based on the distance difference between the farthest and closest free spots."""
        if len(self.free_spots) < 2:
            return 0
        else:
            return self.free_spots[-1].distance_from_entry - self.free_spots[0].distance_from_entry

    def remove_spot_from_free(self, spot):
        """Remove spot from free_spots list"""
        # allow passing either Spot object or spot_id string
        if isinstance(spot, str):
            s = self.spot_lookup.get(spot)
            if s and s in self.free_spots:
                self.free_spots.remove(s)
            return
        if spot in self.free_spots:
            self.free_spots.remove(spot)

    def remove_spot_by_id(self, spot_id: str):
        """Remove a spot from free_spots by its spot_id string."""
        spot = self.spot_lookup.get(spot_id)
        if spot:
            try:
                self.remove_spot_from_free(spot)
            except Exception:
                pass

    def add_spot_to_free(self, spot):
        """Add spot back to free_spots list"""
        if spot not in self.free_spots:
            self.free_spots.add(spot)

    # Simple lookups
    def get_spot(self, spot_id):
        """Get spot by spot_id from hash table"""
        return self.spot_lookup.get(spot_id)

    def get_car(self, car_id):
        """Get car by car_id from hash table"""
        return self.car_lookup.get(car_id)

    # Pairing operations
    def set_waiting_pair(self, car_id, spot_id):
        """Set current waiting pair"""
        self.waiting_pair = {"car_id": car_id, "spot_id": spot_id}

    def get_waiting_pair(self):
        """Get current waiting pair"""
        return self.waiting_pair

    def clear_waiting_pair(self):
        """Clear current waiting pair"""
        self.waiting_pair = None

    # Occupied spots tracking methods
    def add_occupied_spot(self, spot_id, car_id):
        """Add a spot to occupied hash table - O(1)"""
        self.occupied_spots_with_cars[spot_id] = car_id

    def remove_occupied_spot(self, spot_id):
        """Remove a spot from occupied hash table - O(1)"""
        self.occupied_spots_with_cars.pop(spot_id, None)

    def get_occupied_spots(self):
        """Get list of occupied spot tuples [(spot_id, car_id), ...]"""
        return list(self.occupied_spots_with_cars.items())

    def get_random_occupied_spot(self):
        """Get a random occupied spot tuple (spot_id, car_id) or None if empty"""
        return random.choice(list(self.occupied_spots_with_cars.items())) if self.occupied_spots_with_cars else None

    # Grid/BFS utilities
    def _parse_spot_coords(self, spot_id: str) -> Tuple[int, int]:
        """Parse spot_id formatted as '(row,col)' or 'row,col' into (row, col) ints."""
        s = spot_id.strip()
        if s.startswith('(') and s.endswith(')'):
            s = s[1:-1]
        parts = s.split(',')
        return int(parts[0]), int(parts[1])

    def _format_coord_tuple(self, row: int, col: int, with_paren: bool = False) -> str:
        if with_paren:
            return f"({row},{col})"
        return f"{row},{col}"

    def find_closest(self, gate_row: int = 0, gate_col: int = 2) -> Optional[Tuple[int, int]]:
        """Find closest FREE spot to the gate using BFS on the grid of spots.

        Returns (row, col) or None if no free spots. The return shape matches the
        rest of the code and the UI which uses 'row,col' string keys.
        """
        # Build set of free positions from spot_lookup
        free_positions = set()
        for sid, spot in self.spot_lookup.items():
            if getattr(spot, 'status', None) == 'FREE':
                row, col = self._parse_spot_coords(sid)
                free_positions.add((row, col))

        if not free_positions:
            return None

        # BFS from gate
        q = deque()
        start = (gate_row, gate_col)
        q.append(start)
        seen = {start}

        # neighbor deltas: up/down/left/right
        deltas = [(-1, 0), (1, 0), (0, -1), (0, 1)]

        while q:
            r, c = q.popleft()
            if (r, c) in free_positions:
                # return (row, col) to match the 'row,col' key format used by the DB
                return (r, c)
            # Collect potential neighbors, then sort deterministically by (col, row)
            neighbors = []
            for dr, dc in deltas:
                nr, nc = r + dr, c + dc
                if (nr, nc) in seen:
                    continue
                # only explore coordinates that exist in the grid (either free or occupied)
                if (nr, nc) in free_positions or any((nr, nc) == self._parse_spot_coords(sid) for sid in self.spot_lookup):
                    neighbors.append((nr, nc))

            # sort neighbors so tie-breaker prefers lower column, then lower row
            neighbors.sort(key=lambda rc: (rc[1], rc[0]))
            for nr, nc in neighbors:
                seen.add((nr, nc))
                q.append((nr, nc))

        return None

    def allocate_closest_spot(self, car_id: str, gate_row: int = 0, gate_col: int = 2) -> Optional[str]:
        """Allocate the closest free spot (BFS) for car_id.

        Returns the allocated spot id as 'row,col' string (no parentheses) or None.
        Also sets waiting_pair and removes spot from free_spots.
        """
        coord = self.find_closest(gate_row, gate_col)
        if coord is None:
            return None
        # find_closest now returns (row, col)
        row, col = coord
        # our spot_lookup keys use '(row,col)'
        key_paren = self._format_coord_tuple(row, col, with_paren=True)
        key_plain = self._format_coord_tuple(row, col, with_paren=False)
        spot = self.spot_lookup.get(key_paren) or self.spot_lookup.get(f"({row},{col})")
        if spot is None:
            # try the plain key
            spot = self.spot_lookup.get(key_plain)
        if spot is None:
            return None

        # remove from free_spots if present and mark as waiting
        try:
            # mark in-memory
            spot.status = 'WAITING'
            self.remove_spot_from_free(spot)
        except Exception:
            pass

        # set waiting pair and return plain key 'row,col'
        self.set_waiting_pair(car_id, key_plain)
        print(f"[ParkingLot] Allocated spot {key_plain} to car {car_id}; free_spots_count={len(self.free_spots)}")
        return key_plain
```
</details>

| Line(s) | Explanation |
| --- | --- |
| 1-4 | <span style="color:#ff6f61">Imports</span> modules needed for typing, randomness, maintaining sorted order, and BFS queues. |
| 6-33 | Defines the `SortedList` container. The constructor stores the optional key function, seeds initial items, and maintains sorted order when adding elements (lines 22-33). |
| 35-48 | Removal logic handles both keyed and unkeyed storage, raising descriptive errors if the value is absent. |
| 50-83 | Utility dunder methods (`pop`, `__len__`, `__getitem__`, `__iter__`, `__contains__`, `index`) provide list-like ergonomics while respecting keyed storage. |
| 86-100 | `Spot` class constructor sets RTDB-facing fields and the canonical ID format. |
| 102-115 | `Car` class tracks RTDB fields (status, allocated spot, timestamp) and local bookkeeping (`actual_spot`). |
| 117-137 | `ParkingLot` constructor initializes sorted collections, lookup tables, and flags indicating overall fullness. |
| 139-157 | Simple CRUD helpers for spots, cars, and derived metrics like the distance-based "time saved" heuristic. |
| 160-183 | Helpers that remove or re-add spots from the sorted free list, accepting either objects or spot IDs. |
| 185-205 | Lookup and waiting-pair management utilities wrap the dictionaries holding the latest car/spot assignments. |
| 207-223 | Occupancy tracking keeps a hash map of spot→car mappings and surfaces convenience accessors. |
| 224-237 | Coordinate conversion helpers ensure consistent `(row,col)` formatting between Firebase and in-memory code. |
| 238-284 | Breadth-first search that finds the closest free spot to the gate while exploring only valid coordinates. |
| 286-318 | Allocation routine that calls BFS, updates state to `WAITING`, removes the spot from the free pool, records the waiting pair, and prints a debug line. |

---

## 🔌 Server/firebase_init.py
<details>
<summary><strong>Show code</strong></summary>

```python
import json
import os
from typing import Optional

import firebase_admin
from firebase_admin import credentials, db
import requests

DEFAULT_DB_SUFFIX = "-default-rtdb"
DEFAULT_REGION = "asia-southeast1"


class FirebaseInitializationError(Exception):
    """Raised when Firebase cannot be initialized due to configuration issues."""


def get_default_db_url(project_id: str, region: str = DEFAULT_REGION) -> str:
    """Construct the default RTDB URL for the given project and region."""
    return f"https://{project_id}-{region}.{project_id}.firebaseio.com"


def _probe_database_url(db_url: str) -> bool:
    """Probe the RTDB URL to confirm it exists/accepts requests."""
    try:
        response = requests.get(db_url + "/.settings/rules.json", timeout=5)
        return response.ok
    except requests.RequestException:
        return False


def initialize_firebase(cred_path: Optional[str] = None, db_url: Optional[str] = None):
    """Initialize Firebase admin SDK and return the RTDB reference."""
    if firebase_admin._apps:
        return db.reference('/')

    if cred_path is None:
        cred_path = os.environ.get('GOOGLE_APPLICATION_CREDENTIALS')

    if cred_path and os.path.exists(cred_path):
        cred = credentials.Certificate(cred_path)
    else:
        raise FirebaseInitializationError("Firebase credential file not found. Set GOOGLE_APPLICATION_CREDENTIALS.")

    if db_url is None:
        db_url = os.environ.get('FIREBASE_DATABASE_URL')

    if not db_url:
        with open(cred_path, 'r', encoding='utf-8') as f:
            service_account = json.load(f)
        project_id = service_account.get('project_id')
        if not project_id:
            raise FirebaseInitializationError("Service account file missing project_id.")
        db_url = get_default_db_url(project_id)
        if not _probe_database_url(db_url):
            alt_url = f"https://{project_id}.firebaseio.com"
            if _probe_database_url(alt_url):
                db_url = alt_url
            else:
                raise FirebaseInitializationError("Could not determine a valid Firebase RTDB URL.")

    firebase_admin.initialize_app(credentials.Certificate(cred_path), {
        'databaseURL': db_url
    })
    return db.reference('/')
```
</details>

| Line(s) | Explanation |
| --- | --- |
| 1-7 | Load standard libs plus Firebase Admin SDK and `requests` for network probing. |
| 9-10 | Default constants for building RTDB URLs. |
| 13-14 | Custom exception clarifies initialization failures. |
| 17-19 | Helper builds the default RTDB URL from a service account project ID. |
| 22-28 | `_probe_database_url` hits the RTDB rules endpoint to check reachability, swallowing network errors gracefully. |
| 31-64 | `initialize_firebase` lazily initializes the SDK, locating credentials, deriving a database URL when not provided, probing for fallback URLs, and finally bootstrapping the app before returning a root database reference. |

---

## 🌱 Server/Init_Park.py
<details>
<summary><strong>Show code</strong></summary>

```python
from firebase_admin import db

from .constants import ROOT_BRANCH, STAT_FREE


def initialize_parking_lot(rows: int = 10, cols: int = 5, default_distance: int = 3):
    """Seed the RTDB with a grid of empty parking spots."""
    ref = db.reference(f"/{ROOT_BRANCH}/SPOTS")
    spots = {}
    for row in range(rows):
        for col in range(cols):
            spot_id = f"{row},{col}"
            spots[spot_id] = {
                "status": STAT_FREE,
                "waitingCarId": "-",
                "seenCarId": "-",
                "distanceFromEntry": default_distance,
                "lastUpdated": ""
            }
    ref.set(spots)
```
</details>

| Line(s) | Explanation |
| --- | --- |
| 1 | Imports RTDB reference builder from the Firebase Admin SDK. |
| 3 | Shares constants for consistent branch names and spot statuses. |
| 6-18 | Creates a rectangular grid of default spot documents and writes them to `/SondosPark/SPOTS`, seeding the database for demos/tests. |

---

## 🚗 Server/event_generator.py
<details>
<summary><strong>Show imports</strong></summary>

```python
import random
from typing import Optional

from firebase_admin import db

from .constants import ROOT_BRANCH, STAT_FREE, STAT_OCC, STAT_WAIT
from .data_structures import ParkingLot, Spot
```
</details>

| Line(s) | Explanation |
| --- | --- |
| 1-2 | Randomness powers simulated arrivals/departures; typing clarifies optional returns. |
| 4 | Gains access to the RTDB. |
| 6-7 | Pulls shared constants and the in-memory data structures for managing allocations. |

<details>
<summary><strong>Event generator functions</strong></summary>

```python
def _spot_ref() -> db.Reference:
    return db.reference(f"/{ROOT_BRANCH}/SPOTS")


def _car_ref() -> db.Reference:
    return db.reference(f"/{ROOT_BRANCH}/CARS")
```

```python
def load_parking_lot() -> ParkingLot:
    """Hydrate a ParkingLot model from Firebase."""
    lot = ParkingLot()
    spots_snapshot = _spot_ref().get() or {}
    for spot_id, data in spots_snapshot.items():
        row, col = lot._parse_spot_coords(spot_id)
        spot = Spot(row, col, data.get('distanceFromEntry', 0))
        spot.status = data.get('status', STAT_FREE)
        spot.waiting_car_id = data.get('waitingCarId', '-')
        spot.seen_car_id = data.get('seenCarId', '-')
        lot.add_spot(spot)
        if spot.status == STAT_FREE:
            lot.add_spot_to_free(spot)
        else:
            lot.add_occupied_spot(spot.spot_id, data.get('waitingCarId', '-'))
    return lot
```

```python
def simulate_car_arrival(lot: ParkingLot, car_id: str) -> Optional[str]:
    allocated = lot.allocate_closest_spot(car_id)
    if allocated is None:
        return None
    _car_ref().child(car_id).update({
        "status": "waiting",
        "allocatedSpot": allocated,
    })
    _spot_ref().child(allocated).update({
        "status": STAT_WAIT,
        "waitingCarId": car_id,
    })
    return allocated
```

```python
def confirm_parking(lot: ParkingLot, car_id: str, spot_id: str):
    lot.clear_waiting_pair()
    lot.add_occupied_spot(spot_id, car_id)
    _car_ref().child(car_id).update({
        "status": "parked",
        "allocatedSpot": spot_id,
    })
    _spot_ref().child(spot_id).update({
        "status": STAT_OCC,
        "waitingCarId": "-",
    })
```

```python
def simulate_departure(lot: ParkingLot, car_id: str, spot_id: str):
    lot.remove_occupied_spot(spot_id)
    spot = lot.get_spot(spot_id)
    if spot:
        spot.status = STAT_FREE
        lot.add_spot_to_free(spot)
    _car_ref().child(car_id).delete()
    _spot_ref().child(spot_id).update({
        "status": STAT_FREE,
        "waitingCarId": "-",
    })
```
</details>

| Line(s) | Explanation |
| --- | --- |
| 9-15 | `_spot_ref` and `_car_ref` centralize Firebase paths. |
| 18-33 | `load_parking_lot` rebuilds the `ParkingLot` object from RTDB snapshots, ensuring free/occupied sets match remote data. |
| 36-48 | `simulate_car_arrival` reserves a spot, updates car + spot nodes, and returns the assigned `row,col`. |
| 51-61 | `confirm_parking` finalizes WAITING → OCCUPIED transitions for a car. |
| 64-76 | `simulate_departure` releases a spot, deletes the car node, and returns the spot to the free pool. |

---

## 🖥️ Server/dashboard.py
This Flask module imports Flask/Jinja helpers, initializes the app, loads the current lot snapshot on startup, and exposes two endpoints:

| Line(s) | Explanation |
| --- | --- |
| 1-7 | Import Flask, `render_template`, `jsonify`, datetime utilities, Firebase initialization, and the parking lot data structures. |
| 10-24 | `create_app` configures static/template folders and initializes Firebase before returning an app instance. |
| 27-57 | `/` route renders `index.html`, injecting timestamps for cache-busting the CSS/JS assets. |
| 60-115 | `/api/status` builds a JSON payload summarizing each spot, waiting car, closest free spot (via BFS), and overall counts/flags used by the front-end poller. |

---

## 🔁 Server/RTDB_listener.py
| Line(s) | Explanation |
| --- | --- |
| 1-12 | Import Firebase Admin, logging helpers, constants, and the parking lot model. |
| 15-47 | `start_listeners` ensures Firebase is initialized, creates a `ParkingLot`, and attaches stream listeners to `/SPOTS` and `/CARS`. |
| 50-95 | Spot listener reacts to state changes, updating the in-memory model and optionally auto-promoting WAITING → OCCUPIED. |
| 98-140 | Car listener watches arrival confirmation flags and cleans up car nodes when departures are detected. |

---

## 🌀 Server/simulation_sondos.py
| Line(s) | Explanation |
| --- | --- |
| 1-35 | Imports Firebase helpers, time utilities, CLI prompts, and the event generator/data structures. |
| 38-114 | Utility functions to wipe/reset the database, migrate legacy nodes, and backup/restore spot data. |
| 117-205 | Core simulation logic: enqueue arrivals, auto-allocate spots, simulate wrong-spot overrides, and manage departure timing. |
| 208-260 | Command-line driven main loop: parse args, initialize Firebase, and run continuous or finite simulations with graceful exit handling. |

---

## 🧰 Tools/setup_simulation.py
| Line(s) | Explanation |
| --- | --- |
| 1-22 | Imports `argparse`, Firebase init helpers, listener starter, and logging utilities. |
| 25-83 | CLI command that validates Firebase credentials, optionally initializes the RTDB, and spins up listeners for quick smoke testing before running full simulations. |

---

## 🤖 ESP32/SpotNode/SpotNode.ino
| Section | Explanation |
| --- | --- |
| Includes | Pulls WiFiManager, Firebase Arduino Client, ultrasonic sensor handling, and RGB LED support libraries. |
| Global constants | Configure pins, Firebase credentials placeholders, sensor thresholds (`THRESH_ENTER/EXIT`), and status color mappings. |
| `setup()` | Initializes serial logging, configures the RGB LED, launches WiFiManager for credential capture, connects to Firebase, and registers stream callbacks. |
| `loop()` | Repeatedly measures ultrasonic distance, debounces transitions with hysteresis, updates Firebase when state changes, and animates the LED + on-board logging. |

---

## 🎨 Front-end Assets
| File | Highlights |
| --- | --- |
| `Server/static/js/app.js` | Polls `/api/status`, updates the DOM grid, animates arrivals, shows timestamps, and handles the parking-full banner. Each function (`fetchStatus`, `renderGrid`, `updateClosest`) is annotated with colorful callouts in the full guide. |
| `Server/static/css/styles.css` | Defines the dashboard layout, responsive grid sizing, color palette for FREE/WAITING/OCCUPIED, banner animations, and typography. Comments in the guide explain each section. |
| `Server/template/index.html` | Provides the dashboard structure—header, parking grid, status cards, and script/style includes with cache-busting query params. |

---

## 🧪 UNIT TESTS
| File | Focus |
| --- | --- |
| `UNIT TESTS/Server_Tests/test_parkinglot_bfs.py` | Validates BFS allocation order, verifying that the nearest free spot is chosen and state tables update correctly. |
| `UNIT TESTS/Server_Tests/test_event_generator.py` | Exercises arrival/departure helpers with Firebase mocks. |
| `UNIT TESTS/Server_Tests/test_arrival_then_parked.py` | Ensures WAITING → OCCUPIED transitions update both RTDB and in-memory data consistently. |
| `UNIT TESTS/Server_Tests/test_spot_lifecycle.py` | Simulates a full spot lifecycle (free → waiting → occupied → free). |
| `UNIT TESTS/Server_Tests/test_integration_firebase.py` | Smoke test for Firebase initialization + RTDB reads. |
| `UNIT TESTS/Server_Tests/simulate_car_departure.py` | CLI-style script verifying departure cleanup logic. |
| `UNIT TESTS/Server_Tests/conftest.py` | Houses reusable fixtures that mock Firebase interactions. |
| `UNIT TESTS/ESP32_Tests/*.ino` | Standalone sketches to verify WiFi, Firebase, ultrasonic, and RGB LED behavior individually. |

---

## 📘 Documentation & Assets
All Markdown guides (`Documentation/*.md`) explain setup, calibration, troubleshooting, and error handling. The study guide summarizes the purpose of each section so you can answer process-related questions.

---

## 📄 Study.pdf
A matching PDF (`Study.pdf`) mirrors this markdown content so you can read offline or annotate a printed copy.

Happy studying! 🌟
