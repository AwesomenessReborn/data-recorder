# IMU Data Recorder for Android

## Purpose

A minimal Android app to record high-frequency IMU data (accelerometer + gyroscope) from Android devices at **208 Hz** and export to CSV. This app is purpose-built for a multi-device IMU mapping research project where two phones record simultaneously and the data is correlated in post-processing.

This app replaces the third-party "Sensor Logger" app, which failed to deliver the requested sampling rate (requested 100Hz, delivered 50-57Hz depending on device).

## Context: Why This App Exists

We are building a mapping function that maps IMU data from one body location to another. Two phones (Pixel 8 and Pixel 8a, both with STMicro LSM6DSO IMUs) record simultaneously, and the signals are aligned in post-processing using cross-correlation.

Key findings from our investigation that drive the requirements:

| Finding | Detail | Implication |
|---------|--------|-------------|
| Sensor Logger under-delivers | Requested 100Hz, got 50-57Hz | Must use `SENSOR_DELAY_FASTEST` and verify actual rate |
| Clock drift between devices | ~4.5 ms/minute measured | Must use monotonic timestamps (`event.timestamp`) |
| Different sampling rates per device | Pixel 8: 50Hz, Pixel 8a: 57Hz with Sensor Logger | Must log actual timestamps, not assume fixed rate |
| Inference engine requires 208Hz | Model expects 208Hz input | Hardware supports up to 416Hz (LSM6DSO), so 208Hz is achievable |
| NTP sync is unreliable | 20-100ms jitter | Do NOT use epoch time for sensor timestamps |

## Requirements

### Functional Requirements

1. **Record accelerometer and gyroscope data at 208Hz** (or as close as hardware allows)
2. **Use `event.timestamp`** (nanoseconds since boot, monotonic) as the primary timestamp
3. **Record an epoch offset once at recording start** so timestamps can be converted to wall-clock time in post-processing
4. **Export data to CSV files** (one per sensor)
5. **Include a "sync tap" button** that inserts a marker/annotation in the data stream (for physical synchronization between devices)
6. **Upload CSV files to Google Drive** (or save to local storage for manual transfer)
7. **Display real-time sampling rate** on screen so the user can verify 208Hz is being achieved
8. **Display recording duration and sample count** on screen

### Non-Functional Requirements

- Minimal UI — this is a research data collection tool, not a consumer app
- Recording must work with screen off (use a foreground service)
- Must not drop samples due to disk I/O (buffer in memory, flush in batches)
- CSV format must be simple and pandas-compatible

## Technical Specification

### Sensor Configuration

```kotlin
// Request SENSOR_DELAY_FASTEST to get hardware maximum rate
// The LSM6DSO supports: 12.5, 26, 52, 104, 208, 416 Hz
// SENSOR_DELAY_FASTEST should give 416Hz on Pixel 8/8a
// We will downsample to 208Hz in post-processing, OR
// use SensorManager.registerListener with specific period:
//   period = 1_000_000 / 208 = 4807 microseconds

val SAMPLING_PERIOD_US = 4807 // microseconds (≈208 Hz)

sensorManager.registerListener(
    listener,
    accelerometer,
    SAMPLING_PERIOD_US
)
```

**Important**: The `samplingPeriodUs` parameter is a *hint* to the system. The actual rate may differ. That's why we log raw `event.timestamp` values and verify the actual rate on screen.

### Timestamp Strategy

```kotlin
// At recording start, capture the offset between monotonic and wall-clock time:
val epochOffsetNs = System.currentTimeMillis() * 1_000_000L - SystemClock.elapsedRealtimeNanos()

// For each sensor event, use event.timestamp directly (nanoseconds since boot)
// This is monotonic and will not jump due to NTP syncs

// In post-processing, wall-clock time can be recovered:
// wall_clock_ns = event.timestamp + epochOffsetNs
```

**DO NOT** use `System.currentTimeMillis()` or `System.nanoTime()` for per-sample timestamps. These are subject to NTP adjustments and are not guaranteed monotonic.

### Data Buffering Strategy

At 208Hz with 2 sensors, that's ~416 data points per second. Writing each one to disk individually will cause I/O bottlenecks and dropped samples.

```
Recommended approach:
1. Accumulate samples in an ArrayList or ArrayDeque in memory
2. Every 5 seconds (or ~1000 samples), flush the buffer to disk on a background thread
3. Use BufferedWriter for CSV output
4. Pre-allocate buffer capacity to avoid GC pressure
```

### CSV Output Format

**Accelerometer.csv:**
```csv
timestamp_ns,x,y,z
783012345678900,0.0234,-9.7891,0.1456
783012350485700,0.0189,-9.7923,0.1501
...
```

**Gyroscope.csv:**
```csv
timestamp_ns,x,y,z
783012345681200,0.00123,-0.00456,0.00789
783012350488100,0.00134,-0.00423,0.00812
...
```

**Metadata.json:**
```json
{
  "device_name": "Pixel 8",
  "device_model": "shiba",
  "android_version": "15",
  "recording_start_epoch_ms": 1769631573508,
  "epoch_offset_ns": 1769631573508000000,
  "requested_sampling_period_us": 4807,
  "sensors": {
    "accelerometer": {
      "name": "LSM6DSO Accelerometer",
      "vendor": "STMicro",
      "resolution": 0.0023956299,
      "max_range": 78.4532
    },
    "gyroscope": {
      "name": "LSM6DSO Gyroscope",
      "vendor": "STMicro",
      "resolution": 0.001065264,
      "max_range": 34.906586
    }
  }
}
```

**Annotations.csv** (for sync tap events):
```csv
timestamp_ns,label
783012345678900,sync_tap
783098765432100,sync_tap
```

### UI Layout

Minimal single-screen layout:

```
┌──────────────────────────────┐
│  IMU Recorder                │
│                              │
│  Status: ● RECORDING         │
│  Duration: 02:34             │
│  Samples: 31,824            │
│                              │
│  Accel Rate: 207.8 Hz        │
│  Gyro Rate:  208.1 Hz        │
│                              │
│  ┌──────────────────────┐    │
│  │    START / STOP       │    │
│  └──────────────────────┘    │
│                              │
│  ┌──────────────────────┐    │
│  │    SYNC TAP ⚡        │    │
│  └──────────────────────┘    │
│                              │
│  ┌──────────────────────┐    │
│  │    SAVE / UPLOAD      │    │
│  └──────────────────────┘    │
└──────────────────────────────┘
```

### App Architecture

```
app/
├── MainActivity.kt              # UI, button handlers
├── SensorRecorderService.kt     # Foreground service for recording
├── SensorDataBuffer.kt          # Thread-safe buffer with flush logic
├── CsvWriter.kt                 # CSV file writing utilities
├── DriveUploader.kt             # Google Drive upload (optional, can be manual)
└── MetadataCollector.kt         # Collects device/sensor info for metadata.json
```

### Android Manifest Requirements

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<uses-feature android:name="android.hardware.sensor.accelerometer" android:required="true" />
<uses-feature android:name="android.hardware.sensor.gyroscope" android:required="true" />
```

### Foreground Service

Recording MUST happen in a foreground service so it continues when the screen turns off. The user will be walking during data collection and may not be looking at the screen.

```kotlin
// Start as foreground service with persistent notification
val notification = NotificationCompat.Builder(this, CHANNEL_ID)
    .setContentTitle("IMU Recording")
    .setContentText("Recording sensor data...")
    .setSmallIcon(R.drawable.ic_sensor)
    .setOngoing(true)
    .build()

startForeground(NOTIFICATION_ID, notification)
```

### Verifying Actual Sampling Rate

Display a rolling average of the actual sampling rate so the user can confirm 208Hz before starting a real recording session:

```kotlin
// Track last N intervals
private val intervalBuffer = CircularBuffer(100)

override fun onSensorChanged(event: SensorEvent) {
    val interval = event.timestamp - lastTimestamp
    intervalBuffer.add(interval)
    lastTimestamp = event.timestamp

    // Update UI every second
    val avgIntervalNs = intervalBuffer.average()
    val actualHz = 1_000_000_000.0 / avgIntervalNs
    updateRateDisplay(actualHz)
}
```

## Post-Processing Pipeline

After recording, the CSV files are processed in Python:

```python
# 1. Load both device CSVs
# 2. Resample both to exactly 208Hz using cubic interpolation
# 3. Find sync tap annotations OR cross-correlate to find start offset
# 4. Align signals using the offset
# 5. If recording > 5 min, apply linear drift correction using start+end sync events
# 6. Feed aligned, resampled data into inference engine
```

## Recording Protocol for Multi-Device Sessions

1. Place both phones in their respective positions (e.g., ankle + waist)
2. Start recording on both phones (order doesn't matter — cross-correlation handles offset)
3. **Sync tap**: Hold both phones together firmly and tap sharply on a hard surface 3 times
4. Separate phones to their positions
5. Perform the activity (walking, running, etc.)
6. **End sync tap**: Bring phones together again and tap 3 times
7. Stop recording on both phones
8. Save/upload CSVs

The start and end sync taps provide two anchor points. The time offset at each anchor is measured via cross-correlation, and any drift between them is corrected with linear interpolation.

## Target Devices

- **Pixel 8** (shiba) — LSM6DSO IMU
- **Pixel 8a** (akita) — LSM6DSO IMU

Both use the same STMicro LSM6DSO 6-axis IMU which natively supports output data rates of 12.5, 26, 52, 104, 208, and 416 Hz. The 208Hz target is a native ODR for this sensor, so it should be achievable without interpolation artifacts.

## Build Configuration

- **Min SDK**: 29 (Android 10)
- **Target SDK**: 35 (Android 15)
- **Language**: Kotlin
- **Build system**: Gradle with Kotlin DSL
- **No external dependencies required** for core functionality (sensor recording + CSV export)
- **Optional**: Google Drive API client library for upload feature
