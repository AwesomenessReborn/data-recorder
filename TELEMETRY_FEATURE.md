# Future Feature: Inter-Device Telemetry & Synchronization

## Concept

Enable the two Android phones to communicate during data collection to synchronize their recordings in real-time, reducing or eliminating the need for post-processing alignment.

## Benefits

1. **Reduced post-processing** - Less reliance on cross-correlation for alignment
2. **Live verification** - Confirm both devices are recording at expected rates before the session
3. **Synchronized start/stop** - Start/stop recording on both devices with one button press
4. **Drift tracking** - Monitor clock drift in real-time during the session
5. **Quality checks** - Detect if one device stops recording or drops samples

## Technical Approaches

### Option 1: Bluetooth Low Energy (BLE)

**Pros:**
- Low power consumption
- Built into all Android devices
- ~10m range (sufficient for body-worn devices)
- Can work without internet

**Cons:**
- Latency: ~10-50ms typical
- Not precise enough for sample-level sync (208Hz = 4.8ms period)
- Still requires post-processing alignment

**Use case:** Coordinating start/stop, exchanging metadata, sync tap coordination

```kotlin
// One phone acts as GATT server, the other as client
// Exchange timestamps, send "start recording" command, etc.
```

### Option 2: Wi-Fi Direct / Hotspot

**Pros:**
- Lower latency than BLE (~5-20ms)
- Higher bandwidth (can stream full sensor data if needed)
- Standard socket programming

**Cons:**
- Higher power consumption
- One phone must act as hotspot (may interfere with normal Wi-Fi)
- Still not precise enough for sample-level sync

**Use case:** Coordinating sessions + streaming data for live preview

```kotlin
// One phone creates hotspot, other connects
// Use TCP sockets to exchange timestamped messages
```

### Option 3: Time Synchronization Protocol (NTP-like)

Instead of synchronizing samples, synchronize the *clocks* between devices:

**Approach:**
1. Establish BLE or Wi-Fi connection
2. Implement two-way timestamp exchange (similar to NTP)
3. Calculate round-trip time and offset between device clocks
4. Adjust timestamps in post-processing using the measured offset

**Pros:**
- More accurate than relying on button-press timing
- Can measure and compensate for drift over time

**Cons:**
- Still affected by network latency jitter
- Requires careful implementation to be more accurate than current sync-tap method

### Option 4: Audio Beacon Sync

A creative alternative to network sync:

**Approach:**
1. Both phones record audio alongside IMU data
2. One phone emits an ultrasonic or audible "sync beep"
3. Detect the beep in both audio streams
4. Use the beep timestamps as alignment anchors

**Pros:**
- No network setup required
- Can verify sync quality by checking beep cross-correlation
- Works even if phones are far apart (within earshot)

**Cons:**
- Requires microphone access
- Adds audio processing overhead
- Beeps could be annoying during data collection

## Recommended Hybrid Approach

**Near-term (Phase 1):**
- Keep current post-processing alignment (it works well!)
- Add **BLE coordination** for:
  - Synchronized start/stop from primary phone
  - Exchange device metadata (model, sensor info)
  - "Sync tap" notification (both phones log annotation when one taps sync button)

**Long-term (Phase 2):**
- Add **clock offset measurement** using BLE timestamp exchanges
- Measure offset at start, end, and periodically during recording
- Log offset measurements in metadata
- Use in post-processing to improve alignment accuracy

## Implementation Sketch (Phase 1)

```kotlin
// Primary phone (coordinator)
class RecorderCoordinator {
    private val bleServer = BleGattServer()

    fun broadcastStartRecording() {
        bleServer.sendCommand("START_RECORDING")
        startLocalRecording()
    }

    fun broadcastSyncTap() {
        bleServer.sendCommand("SYNC_TAP")
        logSyncTap()
    }
}

// Secondary phone (follower)
class RecorderFollower {
    private val bleClient = BleGattClient()

    init {
        bleClient.onCommandReceived = { command ->
            when (command) {
                "START_RECORDING" -> startLocalRecording()
                "SYNC_TAP" -> logSyncTap()
                "STOP_RECORDING" -> stopLocalRecording()
            }
        }
    }
}
```

## Key Design Principles

1. **Telemetry should enhance, not replace post-processing alignment**
   - Even with sync, still log monotonic timestamps and do cross-correlation
   - Treat telemetry as a quality-of-life improvement

2. **Fail gracefully**
   - If BLE connection drops, both devices continue recording independently
   - App should work fine in offline/solo mode

3. **Keep it simple**
   - Don't try to stream full 208Hz sensor data over BLE
   - Just exchange control signals and periodic clock offset measurements

## Complexity Assessment

Adding telemetry is a **medium-complexity feature**:
- BLE API is well-documented but has setup overhead
- Need to handle connection lifecycle (pairing, reconnection, etc.)
- Testing requires two physical devices

**Recommendation:** Build the core recording app first, verify it achieves 208Hz and exports clean CSVs, *then* add telemetry as a v2 feature.

## Alternative: Keep It Manual

Your current approach (manual start + sync taps) has advantages:
- Simple, no network complexity
- Post-processing already handles alignment well
- Fewer failure modes

Consider whether telemetry actually solves a problem you're experiencing, or if it's optimization for its own sake.
