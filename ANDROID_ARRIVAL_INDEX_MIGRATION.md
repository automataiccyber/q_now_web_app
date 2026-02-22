# Android App Migration Guide: arrivalIndex Pattern Implementation

## Overview
The Q-Now system has been updated to use a new **arrivalIndex pattern** for queue management. This document explains what changed and what your Android app needs to update.

---

## What Changed: The Problem & Solution

### Old Architecture (DEPRECATED)
- Android scanned RFID card and looked up patient in: `/establishments/{estId}/counters/{counterId}/awaitingArrival`
- Web app managed multiple queue requests with `order` field and `waitingForTurn` flag
- Chain queueing required scanning all counters for next request
- **Problem**: Complex, error-prone, doesn't scale well

### New Architecture (CURRENT)
- **arrivalIndex** is now the single source of truth: `/arrivalIndex/{uid}`
- Android reads ONLY ONE small entry (~300 bytes) instead of large counter objects
- Web app manages queue logic and `chainOrder` field exclusively
- **Benefits**: Simpler, more efficient, eliminates ghosts/orphans, prevents buffer issues

---

## Firebase Database Structure (New)

### Key Node: `/arrivalIndex/{uid}`
This is what your Android app will read for patient authorization:

```json
{
  "uid": "kwQbAxs1NGcTLZnFuLlLpxUnMgE2",
  "counterId": "counter1",
  "estId": "-OaLChP8gA0v7MLD5RG-",
  "requestKey": "-OkReq123xyz",
  "awaitingKey": "-OkAwait456abc",
  "chainOrder": 0,
  "status": "awaiting_arrival",
  "createdAt": 1706880000000
}
```

### Supporting Nodes (Read-Only for Android)
- `/patients/{uid}/pin` - Patient's PIN for verification
- `/patients/{uid}/name` - Patient name for display
- `/establishments/{estId}` - Establishment details
- `/establishments/{estId}/counters/{counterId}` - Counter details

---

## What Your Android App Should Do

### 1. **RFID Scan Flow (Main Change)**

**OLD CODE:**
```
Scan RFID Card
  → Get patient ID from /credentials/rfid/{scannedUID}/patientId
  → Search /establishments/{estId}/counters/{counterId}/awaitingArrival for patient
  → Prompt for PIN
  → Write arrival record
```

**NEW CODE:**
```
Scan RFID Card
  → Get patient ID from /credentials/rfid/{scannedUID}/patientId
  → READ /arrivalIndex/{patientId} (SINGLE READ, tiny response ~300 bytes)
  → Validate entry exists AND counterId matches AND status="awaiting_arrival"
  → Prompt for PIN
  → Write arrival record (same as before)
```

### 2. **Authorization Validation**

**What to Check:**
```javascript
// Pseudo-code for your Android logic
const uid = patientIdFromRfid;
const counterIdFromDevice = getCurrentCounterId();

const arrivalIndex = db.child("arrivalIndex").child(uid).get();

if (!arrivalIndex.exists()) {
  showError("No authorization found. Not awaiting arrival at any counter.");
  return;
}

const authData = arrivalIndex.getValue();

// Validate counter match
if (authData.counterId != counterIdFromDevice) {
  showError(`Wrong counter. You're authorized for ${authData.counterId}`);
  return;
}

// Validate status
if (authData.status != "awaiting_arrival") {
  showError(`Invalid status: ${authData.status}`);
  return;
}

// VALID: Proceed to PIN verification
```

### 3. **PIN Verification (No Changes)**

**Keep existing logic:**
- Prompt user for PIN (0-9, timeout 15s, # to submit, * to clear)
- Verify against `/patients/{uid}/pin`
- Same validation and retry logic

### 4. **Arrival Record Writing (No Changes)**

**Keep existing logic:**
```javascript
// Write arrival record (immutable event log)
const now = Date.now();
const arrivalRecord = {
  uid: patientId,
  awaitingKey: authData.awaitingKey,      // from arrivalIndex
  requestKey: authData.requestKey,        // from arrivalIndex
  counterId: currentCounterId,
  establishment: establishmentId,
  timestamp: now,
  method: "rfid"                          // indicates RFID scan
};

db.child("establishments")
  .child(establishmentId)
  .child("arrivals")
  .push()
  .setValue(arrivalRecord);
```

### 5. **UI/Display Changes**

**Update display to show:**
```javascript
// Patient's current queue position
const arrivalIndex = db.child("arrivalIndex").child(uid).get();
displayMessage(`${patientName} - Proceed to ${arrivalIndex.counterId}`);

// Show counter-specific info
const counter = db.child("establishments")
  .child(arrivalIndex.estId)
  .child("counters")
  .child(arrivalIndex.counterId)
  .get();
displayMessage(`Counter: ${counter.name}`);

// Chain queueing indicator (if applicable)
if (arrivalIndex.chainOrder > 0) {
  displayMessage(`⛓️ Queue #${arrivalIndex.chainOrder + 1} (chain)`);
}
```

---

## Migration Checklist

### Phase 1: Update Arrival Validation Logic
- [ ] Replace counter scanning loop with single arrivalIndex read
- [ ] Update validation to check: exists, counterId match, status check
- [ ] Add logging for debugging (which index was checked, result)
- [ ] Test with sample patient data

### Phase 2: Update UI/Display
- [ ] Show patient name from `/patients/{uid}/name`
- [ ] Display counter info from arrivalIndex
- [ ] Show chain order indicator if present
- [ ] Update error messages for new logic

### Phase 3: Testing
- [ ] Test: Patient with no arrivalIndex → Error
- [ ] Test: Patient with wrong counterId → Error
- [ ] Test: Patient with valid arrivalIndex → Success → PIN prompt
- [ ] Test: Valid PIN → Arrival written
- [ ] Test: Chain queueing scenario (chainOrder > 0)
- [ ] Test: Invalid PIN → Retry (existing logic)

### Phase 4: Performance Validation
- [ ] Verify Firebase reads are ~300 bytes (10-20x smaller than before)
- [ ] Check network latency improvements
- [ ] Monitor for any buffer/memory issues (should be eliminated)

---

## Key Differences from OLD System

| Aspect | OLD | NEW |
|--------|-----|-----|
| **Authorization Source** | Scan all counters' awaitingArrival | Single `/arrivalIndex/{uid}` read |
| **Response Size** | 5-10 KB per full counter object | ~300 bytes per patient |
| **Speed** | Slow (multiple reads) | Fast (single indexed read) |
| **Chain Info** | `waitingForTurn` flag + `order` field | `chainOrder` field only |
| **Source of Truth** | Multiple awaitingArrival entries | One arrivalIndex entry |
| **Ghost Entries** | Possible (manual cleanup needed) | Eliminated by design |

---

## Error Handling

### Update Your Error Cases

**New Error Scenarios:**
```javascript
1. arrivalIndex doesn't exist
   → Message: "No authorization. Ask counter staff to accept your request."
   → Action: Return to home, don't attempt PIN entry

2. counterId mismatch
   → Message: "Wrong counter. Go to Counter: {authData.counterId}"
   → Action: Guide user to correct counter

3. Status not "awaiting_arrival"
   → Message: "Invalid state: {authData.status}"
   → Action: Contact support

4. arrivalIndex.awaitingKey is null
   → Message: "Missing authorization details. Try scanning again."
   → Action: Return to home
```

---

## Code Example: Java/Kotlin Implementation

### Java Example
```java
public void handleRfidScan(String scannedUID) {
    // Step 1: Get patient ID from RFID mapping
    String patientId = getPatientIdFromRfid(scannedUID);
    
    // Step 2: READ arrivalIndex (NEW - single read)
    DatabaseReference arrivalIndexRef = mDatabase
        .child("arrivalIndex")
        .child(patientId);
    
    arrivalIndexRef.addListenerForSingleValueEvent(new ValueEventListener() {
        @Override
        public void onDataChange(@NonNull DataSnapshot snapshot) {
            if (!snapshot.exists()) {
                showError("No authorization found");
                return;
            }
            
            Map<String, Object> authData = (Map<String, Object>) snapshot.getValue();
            
            // Validate counter match
            String authorizedCounter = (String) authData.get("counterId");
            if (!authorizedCounter.equals(mCurrentCounterId)) {
                showError("Wrong counter. Go to: " + authorizedCounter);
                return;
            }
            
            // Validate status
            String status = (String) authData.get("status");
            if (!"awaiting_arrival".equals(status)) {
                showError("Invalid status: " + status);
                return;
            }
            
            // VALID: Show PIN prompt
            showPinPrompt(patientId, authData);
        }
        
        @Override
        public void onCancelled(@NonNull DatabaseError error) {
            showError("Error checking authorization: " + error.getMessage());
        }
    });
}

private void showPinPrompt(String patientId, Map<String, Object> authData) {
    // Existing PIN verification logic
    // ...
    
    // On successful PIN:
    writeArrivalRecord(patientId, authData);
}

private void writeArrivalRecord(String patientId, Map<String, Object> authData) {
    Map<String, Object> arrival = new HashMap<>();
    arrival.put("uid", patientId);
    arrival.put("awaitingKey", authData.get("awaitingKey"));
    arrival.put("requestKey", authData.get("requestKey"));
    arrival.put("counterId", mCurrentCounterId);
    arrival.put("timestamp", System.currentTimeMillis());
    arrival.put("method", "rfid");
    
    mDatabase
        .child("establishments")
        .child(mEstablishmentId)
        .child("arrivals")
        .push()
        .setValue(arrival);
}
```

### Kotlin Example
```kotlin
fun handleRfidScan(scannedUID: String) {
    val patientId = getPatientIdFromRfid(scannedUID)
    
    // NEW: Single arrivalIndex read
    database.child("arrivalIndex").child(patientId)
        .get()
        .addOnSuccessListener { snapshot ->
            if (!snapshot.exists()) {
                showError("No authorization found")
                return@addOnSuccessListener
            }
            
            val authData = snapshot.value as? Map<String, Any> ?: return@addOnSuccessListener
            
            // Validate counter
            val authorizedCounter = authData["counterId"] as? String
            if (authorizedCounter != currentCounterId) {
                showError("Wrong counter. Go to: $authorizedCounter")
                return@addOnSuccessListener
            }
            
            // Validate status
            val status = authData["status"] as? String
            if (status != "awaiting_arrival") {
                showError("Invalid status: $status")
                return@addOnSuccessListener
            }
            
            // VALID: Show PIN prompt
            showPinPrompt(patientId, authData)
        }
        .addOnFailureListener { error ->
            showError("Error checking authorization: ${error.message}")
        }
}

private fun writeArrivalRecord(patientId: String, authData: Map<String, Any>) {
    val arrival = mapOf(
        "uid" to patientId,
        "awaitingKey" to authData["awaitingKey"],
        "requestKey" to authData["requestKey"],
        "counterId" to currentCounterId,
        "timestamp" to System.currentTimeMillis(),
        "method" to "rfid"
    )
    
    database.child("establishments")
        .child(establishmentId)
        .child("arrivals")
        .push()
        .setValue(arrival)
}
```

---

## Debugging Tips

### Enable Logging
```java
// Add debug logging for all arrivalIndex reads
Log.d("RFID_DEBUG", "Scanning patient: " + patientId);
Log.d("RFID_DEBUG", "Reading: /arrivalIndex/" + patientId);

// Log the response
Log.d("RFID_DEBUG", "AuthData received: " + authData.toString());
Log.d("RFID_DEBUG", "Authorized counter: " + authData.get("counterId"));
Log.d("RFID_DEBUG", "Expected counter: " + currentCounterId);
Log.d("RFID_DEBUG", "Status: " + authData.get("status"));
```

### Common Issues & Solutions

**Issue**: "No authorization found" but patient scanned card
- **Cause**: arrivalIndex not created yet (counter staff hasn't accepted request)
- **Solution**: Ensure patient request was accepted via Counter.html first

**Issue**: "Wrong counter" error
- **Cause**: Patient's arrivalIndex points to different counter (chain queueing)
- **Solution**: Show user which counter to go to next

**Issue**: Multiple slow reads instead of one
- **Cause**: Code still using old counter scanning logic
- **Solution**: Replace with single `db.child("arrivalIndex").child(uid)` read

---

## Deployment Checklist

- [ ] All old counter-scanning code removed
- [ ] Single arrivalIndex read implemented
- [ ] Validation logic updated (counterId, status checks)
- [ ] Error messages updated for new scenarios
- [ ] PIN verification logic unchanged
- [ ] Arrival record writing unchanged
- [ ] UI updated to show patient info from arrivalIndex
- [ ] Logging added for debugging
- [ ] Tested with multiple scenarios
- [ ] Performance verified (reads should be fast)
- [ ] Version bumped (minor version increment)

---

## Questions or Issues?

If you encounter issues during migration:

1. **Check the Web Updates**: Counter.html and Search.html were updated first - see those changes for context
2. **Verify Database**: Use Firebase Console to check `/arrivalIndex/{testPatientUid}` structure
3. **Enable Verbose Logging**: Add extensive logging to track data flow
4. **Test Incrementally**: Test each step (read, validate, PIN, write) separately

---

## Summary

**The core change**: Replace complex counter scanning with a single arrivalIndex lookup.

**Before**:
```
Scan → Loop through all counters → Check awaitingArrival → Slow, large responses
```

**After**:
```
Scan → Read /arrivalIndex/{uid} → Fast, ~300 bytes, single source of truth
```

Everything else (PIN verification, arrival writing, notifications) stays the same!
