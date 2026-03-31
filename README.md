# Java Satellite Simulator — Back End ("Back in Blackout")

> **COMP2511 Individual Project · Grade: 40/40 (Perfect) · Duration: 1 Month**

A full-stack, physics-based simulation of satellites orbiting Jupiter and communicating with ground-based devices. The back end exposes a RESTful HTTP API that drives a single-page front-end visualiser. The simulation engine enforces orbital mechanics, line-of-sight visibility, bandwidth constraints, relay chains, terrain-aware device movement, and file-transfer state machines — all managed in-memory with per-client session isolation.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Technology Stack](#3-technology-stack)
4. [Repository Structure](#4-repository-structure)
5. [Getting Started](#5-getting-started)
   - [Prerequisites](#prerequisites)
   - [Build](#build)
   - [Run](#run)
   - [Configuration](#configuration)
6. [API Reference](#6-api-reference)
   - [Device Endpoints](#device-endpoints)
   - [Satellite Endpoints](#satellite-endpoints)
   - [Entity Endpoints](#entity-endpoints)
   - [File Endpoints](#file-endpoints)
   - [Slope Endpoints](#slope-endpoints)
   - [Simulation Endpoint](#simulation-endpoint)
   - [Error Handling](#error-handling)
7. [Data Models & Schemas](#7-data-models--schemas)
   - [EntityInfoResponse](#entityinforesponse)
   - [FileInfoResponse](#fileinforesponse)
8. [Entity Types & Specifications](#8-entity-types--specifications)
   - [Devices](#devices)
   - [Satellites](#satellites)
9. [Simulation Physics](#9-simulation-physics)
   - [Orbital Motion](#orbital-motion)
   - [Visibility & Range](#visibility--range)
   - [Device Movement & Slopes](#device-movement--slopes)
   - [Relay Communication (DFS)](#relay-communication-dfs)
10. [File Transfer System](#10-file-transfer-system)
    - [Transfer Lifecycle](#transfer-lifecycle)
    - [Bandwidth Sharing](#bandwidth-sharing)
    - [Storage Constraints](#storage-constraints)
    - [Teleporting Satellite Special Case](#teleporting-satellite-special-case)
    - [Transfer Exceptions](#transfer-exceptions)
11. [Design Patterns & OOP Principles](#11-design-patterns--oop-principles)
12. [Testing](#12-testing)
13. [CI/CD Pipeline](#13-cicd-pipeline)
14. [Session & State Management](#14-session--state-management)
15. [Role & Key Contributions](#15-role--key-contributions)
16. [Project Outcomes](#16-project-outcomes)

---

## 1. Project Overview

**Back in Blackout** simulates a simplified model of orbital communications around Jupiter. Satellites orbit at constant angular velocities while ground-based devices sit on Jupiter's surface. The system computes, tick-by-tick:

- Which entities can communicate (range + line-of-sight + relay chains).
- How files propagate across communicating entities, respecting per-entity bandwidth and storage limits.
- How device positions change when moving across terrain slopes.
- Special satellite behaviours such as teleportation and relay bouncing.

The back end is a **Spark Java HTTP server** that maintains one `BlackoutController` per browser session. The front end — a bundled SPA served from `src/main/resources/app/` — visualises Jupiter, the satellites, devices, files, and communication arcs in real time.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Browser / SPA Client                      │
│        (React/Vue bundle — main.js + main.css)               │
└────────────────────────┬────────────────────────────────────┘
                         │  HTTP (JSON over REST-style endpoints)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Scintilla / Spark Web Server                │
│              unsw.App  (route definitions + CORS)            │
│                     port 4567  (default)                     │
└────────────────────────┬────────────────────────────────────┘
                         │  per-session BlackoutController
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   BlackoutController                         │
│   entityList : List<Entity>   slopeList : List<Slope>        │
│                                                              │
│  ┌──────────────┐   ┌────────────────────────────────────┐  │
│  │   Device     │   │             Satellite               │  │
│  │  (abstract)  │   │            (abstract)               │  │
│  ├──────────────┤   ├────────────────────────────────────┤  │
│  │ LaptopDevice │   │ StandardSatellite                   │  │
│  │ HandheldDevice   │ TeleportingSatellite                 │  │
│  │ DesktopDevice│   │ RelaySatellite                      │  │
│  └──────────────┘   └────────────────────────────────────┘  │
│                                                              │
│  ┌──────────┐  ┌────────────────────────┐  ┌────────────┐   │
│  │  File    │  │ FileTransferRestrictions│  │   Slope    │   │
│  └──────────┘  └────────────────────────┘  └────────────┘   │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────┐
         │   unsw.utils                  │
         │   Angle  ·  MathsHelper       │
         └───────────────────────────────┘
```

**Key architectural decisions:**

| Concern | Decision |
|---------|----------|
| State isolation | One `BlackoutController` instance per Spark session |
| Concurrency | `volatile Map<String, BlackoutController>` for session map |
| Persistence | None — fully in-memory |
| Transport | REST-style HTTP; all params passed as query strings |
| Serialisation | GSON (Google JSON library) |
| Front end | Pre-bundled SPA shipped as classpath resources |

---

## 3. Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | Java | 11 |
| Web Framework | Spark Java (`spark-core`) | 2.9.3 |
| JSON | GSON | 2.8.8 |
| Logging | SLF4J Simple | 1.7.x |
| Build | Gradle | (wrapper included) |
| Testing | JUnit 5 (Jupiter) | 5.8.0 |
| Code Quality | Checkstyle | 10.3.3 |
| CI/CD | GitLab CI + Docker | — |
| Front End | Pre-bundled JS SPA | bundled |

---

## 4. Repository Structure

```
satellite-sim-back-end/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   ├── scintilla/                  # Thin Spark wrapper (web server bootstrap)
│   │   │   │   ├── Scintilla.java
│   │   │   │   ├── WebServer.java
│   │   │   │   ├── Environment.java        # Reads env vars for port/address/mode
│   │   │   │   └── PlatformUtils.java
│   │   │   └── unsw/
│   │   │       ├── App.java                # Entry point; all HTTP route definitions
│   │   │       ├── blackout/               # Core simulation domain
│   │   │       │   ├── BlackoutController.java   # Orchestrator / service layer
│   │   │       │   ├── Entity.java               # Abstract base (all entities)
│   │   │       │   ├── Device.java               # Abstract base (devices)
│   │   │       │   ├── Satellite.java            # Abstract base (satellites)
│   │   │       │   ├── StandardSatellite.java
│   │   │       │   ├── TeleportingSatellite.java
│   │   │       │   ├── RelaySatellite.java
│   │   │       │   ├── LaptopDevice.java
│   │   │       │   ├── HandheldDevice.java
│   │   │       │   ├── DesktopDevice.java
│   │   │       │   ├── File.java                 # File & transfer-progress model
│   │   │       │   ├── Slope.java                # Terrain slope model
│   │   │       │   ├── FileTransferRestrictions.java
│   │   │       │   └── FileTransferException.java  # Exception hierarchy
│   │   │       ├── response/models/
│   │   │       │   ├── EntityInfoResponse.java   # JSON response DTO
│   │   │       │   └── FileInfoResponse.java     # JSON response DTO
│   │   │       └── utils/
│   │   │           ├── Angle.java                # Radian/degree angle utility
│   │   │           └── MathsHelper.java          # Geometry / visibility maths
│   │   └── resources/
│   │       └── app/                             # Bundled front-end SPA
│   │           ├── index.html
│   │           ├── main.js                      # ~1.1 MB minified bundle
│   │           ├── main.css
│   │           └── assets/                      # Sprite textures (PNG/JPEG)
│   └── test/
│       └── blackout/
│           ├── Task1Tests.java
│           ├── Task2ATests.java
│           ├── Task2BTests.java
│           ├── Task2CTests.java
│           ├── Task3Tests.java
│           └── TestHelpers.java
├── build.gradle                 # Gradle build + dependency config
├── checkstyle.xml               # Checkstyle rule set
├── .gitlab-ci.yml               # CI/CD pipeline definition
├── blog.md                      # Links to Jira development log
├── design.pdf                   # UML / design specification
└── .vscode/
    ├── settings.json
    └── java-formatter.xml
```

---

## 5. Getting Started

### Prerequisites

- **Java 11+** (JDK)
- **Gradle** is bundled via the Gradle wrapper (`gradlew`); no separate install needed.

### Build

```bash
# Compile all sources and run Checkstyle + tests
./gradlew build

# Compile only (skip tests)
./gradlew assemble

# Check code style
./gradlew checkstyleMain checkstyleTest
```

### Run

```bash
./gradlew run
```

On startup the server:
1. Binds to `0.0.0.0:4567` (configurable — see below).
2. Automatically opens `http://localhost:4567/app/` in the default browser (unless `HEADLESS` mode is enabled).
3. Serves the bundled SPA from classpath resources.

### Configuration

All configuration is controlled through environment variables read by `scintilla.Environment`:

| Environment Variable | Default | Description |
|---|---|---|
| `scintilla:ADDRESS` | `0.0.0.0` | Server bind address |
| `scintilla:port` | `4567` | Server bind port |
| `scintilla:HEADLESS` | _(unset)_ | Set to any value to suppress auto-open |
| `scintilla:SECURE` | _(unset)_ | Set to enable HTTPS (stub) |

**Example — custom port, headless:**

```bash
export "scintilla:port"=8080
export "scintilla:HEADLESS"=1
./gradlew run
```

**Run the packaged JAR directly:**

```bash
./gradlew jar
java -jar build/libs/satellite-sim-back-end-1.2.1.jar
```

---

## 6. API Reference

All endpoints are served under the root path. Query parameters carry all input; bodies are used only for file content. Responses are JSON. CORS is fully open (`*`) for all origins.

### Base URL

```
http://localhost:4567
```

---

### Device Endpoints

#### `PUT /api/device/` — Create a Device

| Query Param | Type | Required | Description |
|---|---|---|---|
| `deviceId` | `String` | ✅ | Unique device identifier |
| `type` | `String` | ✅ | `LaptopDevice` \| `HandheldDevice` \| `DesktopDevice` |
| `position` | `double` (radians) | ✅ | Angular position on Jupiter's surface |
| `isMoving` | `boolean` | ❌ | Whether device can traverse slopes (default `false`) |

**Response:** `""` (empty string, HTTP 200)

**Example:**
```http
PUT /api/device/?deviceId=laptop1&type=LaptopDevice&position=1.047&isMoving=false
```

---

#### `DELETE /api/device/` — Remove a Device

| Query Param | Type | Required | Description |
|---|---|---|---|
| `deviceId` | `String` | ✅ | ID of device to remove |

**Response:** `""` (empty string, HTTP 200)

---

#### `GET /api/device/all/` — List All Devices

**Response:** `Map<String, EntityInfoResponse>` — keyed by device ID.

```json
{
  "laptop1": {
    "id": "laptop1",
    "type": "LaptopDevice",
    "position": 1.047,
    "height": 69911000.0,
    "files": {}
  }
}
```

---

### Satellite Endpoints

#### `PUT /api/satellite/` — Create a Satellite

| Query Param | Type | Required | Description |
|---|---|---|---|
| `satelliteId` | `String` | ✅ | Unique satellite identifier |
| `type` | `String` | ✅ | `StandardSatellite` \| `TeleportingSatellite` \| `RelaySatellite` |
| `position` | `double` (radians) | ✅ | Initial angular position |
| `height` | `double` (metres) | ✅ | Orbital altitude above Jupiter's surface |

**Response:** `""` (empty string, HTTP 200)

**Example:**
```http
PUT /api/satellite/?satelliteId=sat1&type=StandardSatellite&position=5.934&height=10000
```

---

#### `DELETE /api/satellite/` — Remove a Satellite

| Query Param | Type | Required | Description |
|---|---|---|---|
| `satelliteId` | `String` | ✅ | ID of satellite to remove |

**Response:** `""` (empty string, HTTP 200)

---

#### `GET /api/satellite/all/` — List All Satellites

**Response:** `Map<String, EntityInfoResponse>` — keyed by satellite ID.

---

### Entity Endpoints

#### `GET /api/entity/info/` — Get Entity Info

Retrieves the full state (position, height, files) of either a device or a satellite.

| Query Param | Type | Required | Description |
|---|---|---|---|
| `id` | `String` | ✅ | Entity ID (device or satellite) |

**Response:** `EntityInfoResponse` (see [Data Models](#7-data-models--schemas))

---

#### `GET /api/entity/entitiesInRange/` — Get Communicable Entities

Returns all entities reachable from the given entity, including those reachable via relay chains.

| Query Param | Type | Required | Description |
|---|---|---|---|
| `id` | `String` | ✅ | Source entity ID |

**Response:** `List<EntityInfoResponse>`

---

### File Endpoints

#### `POST /api/device/file/` — Add File to Device

Directly creates a file on a device (simulates a locally stored file before any transmission).

| Query Param | Type | Required | Description |
|---|---|---|---|
| `deviceId` | `String` | ✅ | Target device ID |
| `fileName` | `String` | ✅ | File name |

| Request Body | Type | Description |
|---|---|---|
| _(raw body)_ | `String` | File content (UTF-8 text) |

**Response:** `""` (empty string, HTTP 200)

---

#### `POST /api/sendFile/` — Initiate File Transfer

Starts a file transfer from one entity to another. The transfer progresses each simulation tick.

| Query Param | Type | Required | Description |
|---|---|---|---|
| `fileName` | `String` | ✅ | Name of file to send |
| `fromId` | `String` | ✅ | Sender entity ID |
| `toId` | `String` | ✅ | Receiver entity ID |

**Response (success):** `""` (empty string, HTTP 200)

**Response (error):** `"ExceptionClassName:message"` — e.g. `"VirtualFileNoBandwidthException:sat1"`

---

### Slope Endpoints

#### `POST /api/createSlope/` — Create a Terrain Slope

Defines a slope on Jupiter's surface. Moving devices adjust their height when passing through the slope's angular range.

| Query Param | Type | Required | Description |
|---|---|---|---|
| `startAngle` | `int` (degrees) | ✅ | Slope start (degrees) |
| `endAngle` | `int` (degrees) | ✅ | Slope end (degrees) |
| `gradient` | `int` | ✅ | Height change factor per tick |

**Response:** `""` (empty string, HTTP 200)

---

### Simulation Endpoint

#### `POST /api/simulate/` — Advance Simulation

Runs the simulation for one or more ticks. Each tick: entities move, out-of-range transfers are cancelled, in-progress transfers advance by bandwidth bytes.

| Query Param | Type | Required | Description |
|---|---|---|---|
| `n` | `int` | ❌ | Number of ticks to simulate (default: `1`) |

**Response:** `List<Map<String, EntityInfoResponse>>` — a snapshot of all entity states after each tick.

```json
[
  {
    "sat1": { "id": "sat1", "type": "StandardSatellite", "position": 5.935, "height": 10000.0, "files": {} }
  }
]
```

---

### Error Handling

File transfer errors are returned as a plain string in the format `ExceptionClassName:message`:

| Exception | Cause |
|---|---|
| `VirtualFileNotFoundException` | Source entity does not have a complete copy of the file |
| `VirtualFileNoBandwidthException` | Sender's upload or receiver's download bandwidth is saturated |
| `VirtualFileAlreadyExistsException` | Destination already has a file with that name |
| `VirtualFileNoStorageSpaceException` | Destination has reached its maximum file count or byte limit |

---

## 7. Data Models & Schemas

### EntityInfoResponse

Returned by all entity-info and simulation endpoints.

```json
{
  "id":       "string  — entity identifier",
  "type":     "string  — e.g. StandardSatellite, LaptopDevice",
  "position": "double  — angular position in radians",
  "height":   "double  — metres above Jupiter centre",
  "files": {
    "<filename>": {
      "filename":       "string",
      "data":           "string  — bytes received so far",
      "fileSize":       "integer — total size in bytes",
      "isFileComplete": "boolean — true when fully transferred"
    }
  }
}
```

### FileInfoResponse

Embedded inside `EntityInfoResponse.files`.

| Field | Type | Description |
|---|---|---|
| `filename` | `String` | File name |
| `data` | `String` | Content received so far (partial on in-flight transfers) |
| `fileSize` | `int` | Total expected size in bytes |
| `isFileComplete` | `boolean` | `true` when `data.length() == fileSize` |

---

## 8. Entity Types & Specifications

### Entity Class Hierarchy

```
Entity  (abstract)
├── Device  (abstract)
│   ├── LaptopDevice
│   ├── HandheldDevice
│   └── DesktopDevice
└── Satellite  (abstract)
    ├── StandardSatellite
    ├── TeleportingSatellite
    └── RelaySatellite
```

All entities share:

| Field | Type | Description |
|---|---|---|
| `id` | `String` | Unique identifier |
| `type` | `String` | Type tag |
| `height` | `double` | Height above Jupiter (metres) |
| `position` | `Angle` | Angular position (radians internally) |
| `range` | `int` | Communication range (metres) |
| `linearV` | `double` | Linear velocity (m/s) |
| `direction` | `int` | `+1` anti-clockwise, `-1` clockwise |
| `filesList` | `List<File>` | Owned / received files |
| `sendingFiles` | `List<File>` | Files currently being uploaded |
| `restrictions` | `FileTransferRestrictions` | Bandwidth & storage limits |

---

### Devices

Devices sit on Jupiter's surface (`height = RADIUS_OF_JUPITER ≈ 69,911 km`) and can optionally move along terrain slopes.

| Device Type | Range | Linear Velocity | Notes |
|---|---|---|---|
| `LaptopDevice` | 100,000 m | 30 m/s | — |
| `HandheldDevice` | 50,000 m | 50 m/s | — |
| `DesktopDevice` | 200,000 m | 20 m/s | — |

All devices have **unlimited** file storage and bandwidth (no restrictions).

---

### Satellites

| Satellite Type | Linear Velocity | Range | Max Files | Max Bytes | Recv BW (B/tick) | Send BW (B/tick) | Direction | Special Behaviour |
|---|---|---|---|---|---|---|---|---|
| `StandardSatellite` | 2,500 m/s | 150,000 m | 3 | 80 | 1 | 1 | Clockwise | — |
| `TeleportingSatellite` | 1,000 m/s | 200,000 m | ∞ | 200 | 15 | 10 | Anti-clockwise | Teleports to 0° when crossing 180°; strips `'t'` characters from in-flight files |
| `RelaySatellite` | 1,500 m/s | 300,000 m | ∞ | ∞ | ∞ | ∞ | Clockwise* | Bounces between 140°–190°; acts as a communication relay; cannot store files itself |

\* RelaySatellite reverses direction when it reaches 140° or 190°, oscillating within that band.

---

## 9. Simulation Physics

### Orbital Motion

Each simulation tick (1 minute), every satellite advances by:

```
angularVelocity = linearVelocity / height   (radians / tick)
newPosition     = currentPosition + (direction × angularVelocity)
```

The `Angle` utility class wraps the position arithmetic, normalising values into `[0, 2π)`.

### Visibility & Range

Two entities can communicate only when **both** conditions hold:

1. **Range check** — Euclidean distance between entities ≤ the smaller of their two ranges.
2. **Visibility check** — Jupiter does not occlude the line of sight (ray-circle intersection against Jupiter's radius).

`MathsHelper` provides `isVisible(double height1, Angle pos1, double height2, Angle pos2)` and `getDistance(...)` for these calculations.

### Device Movement & Slopes

Moving devices (`isMoving = true`) advance their angular position each tick using their linear velocity and the current orbital radius. When the device's next position falls within a `Slope`'s `[startAngle, endAngle]` range, the device's height adjusts according to the slope's `gradient` field, enabling simulation of hilly terrain.

### Relay Communication (DFS)

`BlackoutController.communicableEntitiesInRange()` performs a **depth-first search** starting from the source entity. When a `RelaySatellite` is encountered, the DFS continues outward from the relay, accumulating all transitively reachable entities. This models multi-hop relay chains of arbitrary length.

```java
// Simplified DFS pseudocode
communicablesList(origin, current):
    for each entity in world:
        if inRange(current, entity) AND visible(current, entity)
           AND supported(origin, entity) AND notVisited(entity):
            add entity to list
            if entity is RelaySatellite:
                communicablesList(origin, entity)   // recurse through relay
```

---

## 10. File Transfer System

### Transfer Lifecycle

```
sendFile(fileName, fromId, toId)
    │
    ├── Validate: fromEntity has a complete copy
    ├── Validate: sender bandwidth not saturated
    ├── Validate: receiver bandwidth not saturated
    ├── Validate: fileName not already on receiver
    ├── Validate: receiver file count < maxFiles
    ├── Validate: receiver byte budget not exceeded
    │
    └── Create File(progress=0) on receiver's filesList
        + add to sender's sendingFiles
            │
            ▼
        simulate() tick:
            ├── Check communicability — cancel if out of range
            └── updateFileProgress() — advance progress by
                  floor(sendingBandwidth / concurrentUploads) bytes/tick
                  until progress == fileSize (isFileComplete = true)
```

### Bandwidth Sharing

When a satellite is simultaneously sending multiple files, the sending bandwidth is divided **equally** across all active uploads:

```
bytesThisTick = floor(sendingBandwidth / numberOfActiveUploads)
```

Receiving bandwidth works symmetrically. This means high-concurrency transfers slow each individual transfer proportionally.

### Storage Constraints

Before a transfer is accepted, two storage checks are made on the receiver:

| Check | Condition to reject |
|---|---|
| Max file count | `currentFiles >= maxFiles` |
| Max byte storage | `currentUsedBytes + incomingFileSize > maxBytes` |

### Teleporting Satellite Special Case

When a `TeleportingSatellite` crosses the 180° mark:

1. It **teleports** its position to 0° (rather than continuing past 180°).
2. It **reverses** its direction of travel.
3. Any file currently being transferred **to** the satellite has all `'t'` characters stripped from the already-transferred portion. Any file currently being transferred **from** the satellite similarly has `'t'` characters stripped from remaining content, and the transfer completes instantly.

### Transfer Exceptions

```java
FileTransferException  (checked)
├── VirtualFileNotFoundException       // source doesn't have complete file
├── VirtualFileNoBandwidthException    // bandwidth saturated
├── VirtualFileAlreadyExistsException  // filename collision on destination
└── VirtualFileNoStorageSpaceException // file count or byte limit hit
```

---

## 11. Design Patterns & OOP Principles

| Pattern / Principle | Where Applied |
|---|---|
| **Inheritance + Polymorphism** | `Entity → Device/Satellite → concrete types`; `move()`, `isSupportedEntity()`, etc. overridden per type |
| **Abstract Classes** | `Entity`, `Device`, `Satellite` define contracts and share common state |
| **Factory-style Instantiation** | `BlackoutController.createDevice()` / `createSatellite()` use `switch` on type string to instantiate concrete classes |
| **Strategy (implicit)** | Each entity type encapsulates its own movement, range, and bandwidth strategy |
| **DTO (Data Transfer Object)** | `EntityInfoResponse` and `FileInfoResponse` decouple domain objects from API layer |
| **Session Pattern** | `Map<sessionId, BlackoutController>` isolates state per client |
| **DFS Graph Traversal** | `getCommunicablesList()` for relay chain reachability |
| **Encapsulation** | All domain state private; accessed via getters/setters and domain methods |
| **Composition** | `Entity` composes `FileTransferRestrictions` and `List<File>` |
| **Single Responsibility** | `MathsHelper` owns all geometry; `BlackoutController` owns orchestration; `App` owns routing |

---

## 12. Testing

Tests are located in `src/test/blackout/` and use **JUnit 5**.

### Running Tests

```bash
# Run all tests
./gradlew test

# Run a specific test class
./gradlew test --tests "blackout.Task1Tests"
./gradlew test --tests "blackout.Task2ATests"
```

### Test Coverage

| File | Focus |
|---|---|
| `Task1Tests.java` | Entity creation/deletion, basic file support on devices |
| `Task2ATests.java` | File transfer between communicable entities |
| `Task2BTests.java` | Bandwidth restrictions and concurrent transfer throttling |
| `Task2CTests.java` | Complex multi-entity transfer scenarios |
| `Task3Tests.java` | Advanced scenarios: slopes, teleporting satellites, relay chains |
| `TestHelpers.java` | Assertion utilities (`assertListAreEqualIgnoringOrder`, etc.) |

### Example Test Pattern

```java
BlackoutController controller = new BlackoutController();

// Set up entities
controller.createSatellite("Sat1", "StandardSatellite",
    100 + RADIUS_OF_JUPITER, Angle.fromDegrees(340));
controller.createDevice("DevA", "HandheldDevice", Angle.fromDegrees(30));

// Add file and initiate transfer
controller.addFileToDevice("DevA", "hello.txt", "Hello World");
controller.sendFile("hello.txt", "DevA", "Sat1");

// Advance simulation and assert state
controller.simulate(10);
assertEquals("Hello World",
    controller.getInfo("Sat1").getFiles().get("hello.txt").getData());
```

---

## 13. CI/CD Pipeline

Defined in `.gitlab-ci.yml` using the Docker image `sneakypatriki/cs2511-gradle:latest`.

```
┌──────────┐        ┌──────────┐
│   lint   │ ──▶──  │  tests   │
└──────────┘        └──────────┘
 (soft fail)        (must pass)
```

| Stage | Command | Failure Mode |
|---|---|---|
| `lint` | `gradle checkstyleMain && gradle checkstyleTest` | `allow_failure: true` |
| `tests` | `gradle test` | Blocks merge on failure |

Artefacts (test reports, compiled JARs) are produced in `build/` and automatically excluded from version control via `.gitignore`.

---

## 14. Session & State Management

The application maintains **one `BlackoutController` instance per HTTP session**:

```java
private static volatile Map<String, BlackoutController> sessionStates;

// On each request:
String sessionId = request.session().id();
BlackoutController controller = sessionStates.computeIfAbsent(
    sessionId, k -> new BlackoutController());
```

- State is **ephemeral** — lost on server restart.
- **No database** — all entities, files, and slopes live in JVM heap memory.
- Multiple browser tabs / users each receive isolated simulation worlds.
- `volatile` keyword ensures visibility of the map reference across threads.
- CORS headers (`Access-Control-Allow-Origin: *`) are applied globally, permitting the front-end dev server to call the API on a different port during development.

---

## 15. Role & Key Contributions

As the **sole developer**, responsibilities spanned the full software engineering lifecycle:

- **System Design** — Authored the UML class diagram (`design.pdf`), identified the inheritance hierarchy, and planned the communication and file-transfer contracts before writing a single line of code.
- **Core Simulation Engine** — Implemented `BlackoutController`, all entity classes, the orbital motion model, line-of-sight visibility checker, the relay DFS, and the bandwidth-aware file transfer state machine.
- **REST API Layer** — Designed and implemented all HTTP endpoints in `App.java`, including session management, GSON serialisation, and structured error responses.
- **Terrain System** — Implemented `Slope` and the device slope-movement logic, including height adjustment and slope-detection per tick.
- **Special Satellite Behaviours** — Implemented `TeleportingSatellite` (teleport + `'t'`-stripping) and `RelaySatellite` (oscillating bounce + DFS relay logic).
- **Testing** — Wrote comprehensive JUnit 5 test suites across five task-level test classes covering normal paths, edge cases, and constraint violations.
- **CI/CD** — Configured the GitLab CI pipeline with lint and test stages.
- **Documentation** — Maintained JavaDoc, inline comments, and development logs throughout.

---

## 16. Project Outcomes

- ✅ **Perfect grade: 40/40** — all autograded test suites and manual review criteria passed.
- ✅ Simulation correctly models orbital motion, line-of-sight occlusion, relay chains, bandwidth throttling, storage limits, slope-aware device movement, and all special satellite behaviours.
- ✅ Clean object-oriented architecture — zero code smell warnings from Checkstyle, all style rules enforced.
- ✅ Full test coverage across five independent task test suites.
- ✅ RESTful API successfully drives a real-time 3D front-end visualiser.
- ✅ Demonstrated independent delivery of a complex, multi-faceted software system within a one-month timeline.
