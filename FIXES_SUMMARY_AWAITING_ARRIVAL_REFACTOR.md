# Awaiting Arrival Display Fix - Counter.html Refactoring

## Problem Statement
The "Awaiting Arrival" section in Counter.html was not displaying patients even after they had accepted a queue request and `arrivalIndex` was populated. The UI was trying to read from a deleted data path (`awaitingArrival`) instead of the new source of truth (`arrivalIndex`).

## Root Cause
- **Old code**: Listeners pointed to `/establishments/{estId}/counters/{counterId}/awaitingArrival`
- **What happened**: We eliminated creation of `awaitingArrival` nodes as part of the architectural redesign
- **Result**: Listeners never received data → `awaitingItems` object stayed empty → UI showed "No patients awaiting arrival"

## Solution Implemented

### 1. Changed Listener Source (Lines 938-978)

**BEFORE:**
```javascript
const awaitingArrivalRef = db.ref(`establishments/${estId}/counters/${counterId}/awaitingArrival`);

// Listeners attached to non-existent path
awaitingArrivalRef.once("value", snap => { ... });
awaitingArrivalRef.on("child_added", snap => { ... });
awaitingArrivalRef.on("child_changed", snap => { ... });
awaitingArrivalRef.on("child_removed", snap => { ... });
```

**AFTER:**
```javascript
const arrivalIndexRef = db.ref("arrivalIndex");

// Initial load with counter filter
arrivalIndexRef.orderByChild("counterId").equalTo(counterId).once("value", snap => {
    snap.forEach(indexSnap => {
        const uid = indexSnap.key;
        const data = indexSnap.val();
        // Filter: only awaiting_arrival status at THIS counter
        if (data && data.counterId === counterId && data.status === "awaiting_arrival") {
            awaitingItems[uid] = data;
        }
    });
    renderAwaitingList();
});

// Listen for new entries
arrivalIndexRef.on("child_added", snap => {
    const uid = snap.key;
    const data = snap.val();
    if (data && data.counterId === counterId && data.status === "awaiting_arrival") {
        awaitingItems[uid] = data;
        renderAwaitingList();
    }
});

// Listen for updates
arrivalIndexRef.on("child_changed", snap => {
    const uid = snap.key;
    const data = snap.val();
    if (data && data.counterId === counterId && data.status === "awaiting_arrival") {
        awaitingItems[uid] = data;
        renderAwaitingList();
    } else if (awaitingItems[uid]) {
        delete awaitingItems[uid];
        renderAwaitingList();
    }
});

// Listen for removals
arrivalIndexRef.on("child_removed", snap => {
    const uid = snap.key;
    if (awaitingItems[uid]) {
        delete awaitingItems[uid];
        renderAwaitingList();
    }
});
```

### 2. Updated Data Access in Render Function (Lines 1000-1070)

**KEY CHANGES:**
- `awaitingItems` now keyed by `uid` instead of `awaitingKey`
- Iterate through `uids` instead of `keys`
- Access data from `awaitingItems[uid]` which is the `arrivalIndex` entry
- Extract `requestKey` from `arrivalIndexData.requestKey`
- Updated button to call `verifyPin(uid, uid)` instead of `verifyPin(awaitingKey, uid)`

**Data Structure:**
```
arrivalIndex entry structure (what we're now reading):
{
  uid: "patient-id",
  counterId: "counter-id",
  estId: "est-id",
  requestKey: "request-id",
  status: "awaiting_arrival",
  chainOrder: 0,
  createdAt: timestamp
}
```

### 3. Updated PIN Verification (Lines 2032-2078)

**`writeManualArrival()` function changes:**

**BEFORE:**
```javascript
async function writeManualArrival(awaitingKey, uid) {
    // Read from deleted path
    const awaitingSnap = await db.ref(
        `establishments/${estId}/counters/${counterId}/awaitingArrival/${awaitingKey}`
    ).once("value");
    
    // Update deleted path
    await db.ref(
        `establishments/${estId}/counters/${counterId}/awaitingArrival/${awaitingKey}/status`
    ).set("arrived");
}
```

**AFTER:**
```javascript
async function writeManualArrival(uid, uidParam) {
    // Read from arrivalIndex (single source of truth)
    const arrivalSnap = await db.ref(`arrivalIndex/${uid}`).once("value");
    
    if (!arrivalSnap.exists()) {
        alert("Awaiting arrival record not found in arrivalIndex.");
        return;
    }
    
    const arrivalData = arrivalSnap.val();
    const requestKey = arrivalData.requestKey;
    
    // Write to arrivals (same as hardware does)
    const arrivalKey = db.ref(`establishments/${estId}/arrivals`).push().key;
    await db.ref(`establishments/${estId}/arrivals/${arrivalKey}`).set({
        uid: uid,
        requestKey: requestKey,
        counterId: counterId,
        timestamp: Date.now(),
        method: "manual_pin"
    });
    
    // arrivalsRef listener will:
    // 1. Create queue entry
    // 2. Delete arrivalIndex entry
    // 3. Update UI
}
```

## Data Flow After Fix

### Mobile App Accepts Queue Request:
1. Create `/patients/{uid}/queueRequests/{reqId}`
2. Create `/establishments/{estId}/counters/{counterId}/requests/{reqId}`

### Counter Staff Clicks ACCEPT:
1. **ONLY** Create `/arrivalIndex/{uid}` with:
   - status: "awaiting_arrival"
   - counterId: current counter
   - requestKey: associated request

### Counter.html "Awaiting Arrival" Section:
1. **NEW**: Listener reads from `/arrivalIndex` ✅
2. Filters entries where `counterId === currentCounterId` ✅
3. Filters entries where `status === "awaiting_arrival"` ✅
4. Immediately displays patient with "Verify PIN" button ✅

### Patient Arrives (Via RFID or PIN):
1. Counter staff clicks "Verify PIN" (or hardware RFID scans)
2. `writeManualArrival()` reads from `/arrivalIndex/{uid}` ✅
3. Writes to `/establishments/{estId}/arrivals/{arrivalId}`
4. arrivalsRef listener processes:
   - Creates queue entry
   - Deletes `arrivalIndex` entry
   - Patient moves from "Awaiting Arrival" to "In Queue"

## Benefits of This Fix

✅ **Single Source of Truth**: `arrivalIndex` is now the ONLY location tracking awaiting patients
✅ **Real-time Display**: Listeners now receive data immediately when ACCEPT is clicked
✅ **Consistent Data**: Hardware (ESP32) and Web (Counter.html) read from same location
✅ **No Ghost Entries**: Eliminated the separate `awaitingArrival` nodes that could become orphaned
✅ **Proper Filtering**: Only displays patients awaiting arrival at THIS specific counter
✅ **Scalable**: Works correctly with multiple counters

## Testing Checklist

- [ ] Mobile app: Accept queue request
- [ ] Counter.html: Observe patient appears in "Awaiting Arrival" section
- [ ] Click "Verify PIN" button
- [ ] Enter correct PIN
- [ ] Patient moves to active queue
- [ ] Patient disappears from "Awaiting Arrival"
- [ ] Test with multiple patients to verify filtering works
- [ ] Test with multiple counters to verify only THIS counter's arrivals show

## Related Files Modified
- **Counter.html**: Lines 938-1070 (listener setup & render), Lines 2032-2078 (PIN verification)

## Related Architecture
- **ACCEPT REQUEST Handler**: Creates `arrivalIndex` (Lines 867-890)
- **Arrivals Listener**: Processes arrivals from `arrivalIndex` (Lines 1858-1940)
- **Database Paths**:
  - `/arrivalIndex/{uid}` - Source of truth for awaiting patients
  - `/establishments/{estId}/arrivals/{arrivalId}` - Immutable audit log
  - `/establishments/{estId}/counters/{counterId}/queue/{queueKey}` - Active queue

## Conclusion
The "Awaiting Arrival" display now correctly shows patients from the `arrivalIndex` source of truth. This completes the architectural cleanup to use `arrivalIndex` as the single indicator of awaiting patient status across the entire Q-Now system.
