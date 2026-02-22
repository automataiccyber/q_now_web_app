# Android App: Quick Migration Guide (arrivalIndex Pattern)

## TL;DR - What Changed
**OLD**: Scan all counters for `awaitingArrival` (slow, large responses, complex)  
**NEW**: Read single `/arrivalIndex/{uid}` (~300 bytes, fast, simple)

---

## 3-Step Changes Required

### Step 1: Replace Authorization Check
**Find this pattern in your RFID scan handler:**
```
Loop through /establishments/{estId}/counters/{counterId}/awaitingArrival
Search for patient UID
```

**Replace with:**
```java
// Single read instead of scanning
db.child("arrivalIndex").child(patientId)
  .get()
  .addOnSuccessListener(snapshot -> {
    if (!snapshot.exists()) {
      showError("No authorization found");
      return;
    }
    
    Map<String, Object> auth = snapshot.getValue(Map.class);
    String counterId = (String) auth.get("counterId");
    String status = (String) auth.get("status");
    
    // Validate
    if (!counterId.equals(mCurrentCounterId)) {
      showError("Wrong counter: " + counterId);
      return;
    }
    
    if (!"awaiting_arrival".equals(status)) {
      showError("Invalid status");
      return;
    }
    
    // ✅ VALID - show PIN prompt
    showPinPrompt(patientId, auth);
  });
```

### Step 2: Update Arrival Record (Just Extract Data)
**When writing arrival, use `awaitingKey` and `requestKey` from arrivalIndex:**
```java
Map<String, Object> arrival = new HashMap<>();
arrival.put("uid", patientId);
arrival.put("awaitingKey", auth.get("awaitingKey"));   // ← From arrivalIndex
arrival.put("requestKey", auth.get("requestKey"));     // ← From arrivalIndex
arrival.put("counterId", mCurrentCounterId);
arrival.put("timestamp", System.currentTimeMillis());
arrival.put("method", "rfid");

db.child("establishments")
  .child(mEstablishmentId)
  .child("arrivals")
  .push()
  .setValue(arrival);
```

### Step 3: Update UI Display
**Show patient info from arrivalIndex:**
```java
// Get counter name
db.child("establishments").child(estId)
  .child("counters").child(counterId).child("name")
  .get()
  .addOnSuccessListener(snap -> 
    displayMessage("Proceed to: " + snap.getValue(String.class))
  );

// Show chain position if needed
if ((Long) auth.get("chainOrder") > 0) {
  displayMessage("⛓️ Queue position: " + (auth.get("chainOrder") + 1));
}
```

---

## Database Paths (Reference)

| Purpose | OLD Path | NEW Path |
|---------|----------|----------|
| Get authorization | `/establishments/{estId}/counters/{counterId}/awaitingArrival/{uid}` | `/arrivalIndex/{uid}` |
| Verify PIN | `/patients/{uid}/pin` | Same (no change) |
| Write arrival | `/establishments/{estId}/arrivals/{key}` | Same (no change) |

---

## What Stayed the Same
- ✅ PIN verification logic (no changes)
- ✅ Arrival record writing structure (still write to `/arrivals`)
- ✅ Error handling patterns (just update messages)
- ✅ Notification sending (same as before)

---

## Testing Checklist
- [ ] No arrivalIndex → "No authorization found" error
- [ ] Wrong counterId in arrivalIndex → "Wrong counter" error  
- [ ] Valid arrivalIndex → PIN prompt appears
- [ ] Valid PIN → Arrival written successfully
- [ ] Check Firebase Console: reads should be fast (~300 bytes)

---

## Example: Before & After

**BEFORE (Old Code):**
```java
// Loop through all counters - SLOW
db.child("establishments").child(estId).child("counters")
  .get()
  .addOnSuccessListener(snap -> {
    for (DataSnapshot counter : snap.getChildren()) {
      if (counter.hasChild("awaitingArrival")) {
        for (DataSnapshot awaiting : counter.child("awaitingArrival").getChildren()) {
          if (awaiting.child("uid").getValue().equals(patientId)) {
            // Found it! - took many reads
            showPinPrompt(patientId, awaiting.getValue());
          }
        }
      }
    }
  });
```

**AFTER (New Code):**
```java
// Direct read - FAST
db.child("arrivalIndex").child(patientId)
  .get()
  .addOnSuccessListener(snap -> {
    if (!snap.exists()) {
      showError("No authorization");
      return;
    }
    showPinPrompt(patientId, snap.getValue(Map.class));
  });
```

---

## Key Points
1. **arrivalIndex** replaces scanning all counters
2. Extract `awaitingKey` and `requestKey` from arrivalIndex when writing arrivals
3. Everything else stays the same (PIN logic, notification logic, arrival record structure)
4. This is ~10x faster and eliminates buffer issues

