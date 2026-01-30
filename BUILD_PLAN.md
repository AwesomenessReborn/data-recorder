# Build Plan: IMU Data Recorder

## Objective
Build a robust Android application to record high-frequency IMU data (208Hz) from Pixel 8/8a devices for multi-device mapping research.

## Status Analysis
- **Core Service**: `SensorRecorderService.kt` is implemented with basic recording, buffering, and CSV writing logic.
- **UI**: `MainActivity.kt` is currently boilerplate "Hello World".
- **Configuration**: `AndroidManifest.xml` has necessary permissions.
- **Missing Features**:
    - Real-time sampling rate calculation (logic described in README but missing in Service).
    - User Interface (Start/Stop, Sync Tap, Status Dashboard).
    - Runtime Permission handling (Notifications).
    - Connection between Activity and Service.

## Phase 1: Core Recording Functionality (Current Focus)

### 1. Service Enhancements (`SensorRecorderService.kt`)
- [ ] **Implement Rate Tracking**: Add `CircularBuffer` and logic to calculate instantaneous Hz for Accelerometer and Gyroscope.
- [ ] **Expose State**: Ensure properties (`isRecording`, `sampleCounts`, `rates`) are accessible to the bound Activity.

### 2. UI Implementation (`MainActivity.kt`)
- [ ] **Dashboard Layout**: Create Composable UI matching the design in README.
    - Status Indicator (Recording vs Idle).
    - Duration Timer.
    - Sample Counters.
    - Real-time Rate Display.
- [ ] **Controls**:
    - Large "Start/Stop" button.
    - "Sync Tap" button (enabled only when recording).
- [ ] **Service Binding**: Implement `ServiceConnection` to bind to `SensorRecorderService`.
- [ ] **Live Updates**: Use a coroutine/timer in the UI to poll the Service every ~500ms for updates.

### 3. Permissions & System Integration
- [ ] **Runtime Permissions**: Request `POST_NOTIFICATIONS` on app launch (Android 13+).
- [ ] **Wake Lock**: Verify WakeLock in Service handles long recordings correctly.

## Phase 2: Refinement & Validation (Next Steps)
- [ ] **Field Test**: Run on Pixel 8 and 8a simultaneously.
- [ ] **Data Verification**: Use `quick_sampling_analysis.py` to verify 208Hz target is met.
- [ ] **Telemetry**: (Future) Implement BLE coordination as per `TELEMETRY_FEATURE.md`.

## Execution Steps for Today
1.  Update `SensorRecorderService.kt` to include sampling rate calculation.
2.  Rewrite `MainActivity.kt` to implement the functional UI and service binding.
