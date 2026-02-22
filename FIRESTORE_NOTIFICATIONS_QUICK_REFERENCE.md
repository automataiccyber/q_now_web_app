# Firestore Notifications - Quick Reference

## TL;DR: What Was Added to Counter.html

### 1. **Firestore SDK + Initialization**
```html
<!-- Line 591 -->
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-firestore-compat.js"></script>

<!-- Line 744 -->
<script>
  const firestore = firebase.firestore();
</script>
```

### 2. **Three Helper Functions** (Lines 657-727)
```javascript
// Push queue notifications
async function pushArrivalNotification(uid, data)

// Push appointment notifications (ready for future)
async function pushAppointmentNotification(uid, data)

// Push generic notifications
async function pushInfoNotification(uid, data)
```

### 3. **Notifications at 7 Queue Lifecycle Points**

| Event | Line | Trigger | Notification |
|-------|------|---------|--------------|
| Accept Request | 975 | Counter clicks ACCEPT | status: "awaiting_arrival" |
| Reject Request | 1042 | Counter clicks REJECT | status: "rejected" |
| Admit Next | 1664 | Counter clicks "Admit Next" | status: "in_service" |
| Patient Arrival | 2195 | RFID scan or PIN verified | status: "waiting" |
| Chain Queue | 1850 | Auto-queue to next counter | status: "waiting" + chainOrder |
| End Service (Final) | 1943 | Service complete, no chain | status: "completed" |
| End Service (Chain) | 1954 | Service complete, chain exists | status: "waiting" + chainOrder |

---

## How It Works

### Data Flow
```
Counter Staff Action
       ↓
Update RTDB (atomic)
       ↓
Push Firestore Notification (async)
       ↓
Android App Receives
       ↓
NotificationAdapter Updates UI
```

### Firestore Document Example
```json
{
  "path": "patients/kwQbAxs1NGcTLZnFuLlLpxUnMgE2/inbox/randomDocId",
  "data": {
    "estId": "-OaLChP8gA0v7MLD5RG-",
    "estName": "City Hospital",
    "counterId": "counter1",
    "counterName": "Registration",
    "type": "arrival_update",
    "status": "awaiting_arrival",
    "message": "Your request has been accepted. Please proceed to check in.",
    "chainOrder": 0,
    "timestamp": "2026-02-02T10:30:45.123Z"
  }
}
```

---

## Android Behavior by Status

### status = "awaiting_arrival"
- **Display**: Pill with "ARRIVAL"
- **Color**: Warning (orange/yellow)
- **Message**: "Proceed to Counter: {counterName}"
- **Show In**: Inbox notifications

### status = "waiting" (chainOrder = 0)
- **Display**: In Active Queue list
- **Icon**: Queue position indicator
- **Message**: "You're next in queue at {counterName}"
- **Show In**: Active Queues section

### status = "waiting" (chainOrder > 0)
- **Display**: Pill with "⚓ Queue #N"
- **Color**: Primary (blue)
- **Message**: "You're next in a chained queue"
- **Show In**: Inbox notifications

### status = "in_service"
- **Display**: Pill with "ARRIVAL"
- **Color**: Warning (orange/yellow)
- **Message**: "You are now being served at {counterName}"
- **Show In**: Active Queue (highlighted)

### status = "completed"
- **Display**: Recently Visited
- **Icon**: Checkmark
- **Message**: "Service completed"
- **Show In**: History section

### status = "rejected"
- **Display**: Hidden
- **Show In**: Not shown to user

---

## Code Locations

### Notification Helpers
- **Lines 650-727**: All three helper functions

### Integration Points
```
Line 975  → pushArrivalNotification() called in ACCEPT REQUEST
Line 1042 → pushArrivalNotification() called in REJECT REQUEST
Line 1664 → pushArrivalNotification() called in ADMIT NEXT
Line 1850 → pushArrivalNotification() called in CHAIN QUEUE STEP
Line 1943 → pushArrivalNotification() called in SERVICE COMPLETION (final)
Line 1954 → pushArrivalNotification() called in SERVICE COMPLETION (chain)
Line 2195 → pushArrivalNotification() called in PATIENT ARRIVAL
```

---

## Minimal Example: How to Push a Notification

```javascript
// Simple usage
await pushArrivalNotification(patientUID, {
    estId: "establishment123",
    estName: "Hospital Name",
    counterId: "counter1",
    counterName: "Counter 1",
    status: "awaiting_arrival",
    message: "Your request has been accepted",
    chainOrder: 0
});

// What happens:
// 1. Creates doc in: patients/{patientUID}/inbox/{autoId}
// 2. Sets all fields above
// 3. Adds serverTimestamp
// 4. Logs success/error to console
// 5. Does NOT block further code execution
```

---

## Android Listening Code (Reference)

Android app listens to notifications with:
```kotlin
firestore.collection("patients")
    .document(currentUser.uid)
    .collection("inbox")
    .addSnapshotListener { snapshot ->
        snapshot?.documents?.forEach { doc ->
            val notification = doc.toObject(Notification::class.java)
            // NotificationAdapter displays it based on type + status
        }
    }
```

---

## Error Handling

All notification pushes include error handling:
```javascript
try {
    await firestore.collection("patients").doc(uid).collection("inbox").add(notifData);
    console.log(`✅ Pushed notification to ${uid}`);
} catch (err) {
    console.error(`❌ Error pushing notification:`, err);
    // Queue processing continues - notifications don't block
}
```

---

## Testing: Quick Smoke Test

1. **Open Counter.html in browser**
2. **Open browser DevTools Console** (F12)
3. **Accept a queue request**
4. **Look for console log**: ✅ Pushed arrival notification
5. **Open Android app**
6. **Check Notifications**: Should see "awaiting_arrival" notification
7. **Click "Admit Next"**
8. **Check Android**: Should see "in_service" notification

---

## Troubleshooting

### Issue: No notifications appearing in Android
- [ ] Check browser console for errors (❌ Error pushing notification)
- [ ] Verify Firestore SDK loaded (check Network tab)
- [ ] Verify Firestore initialized (check console for "const firestore")
- [ ] Check Android app has Firestore listener attached
- [ ] Verify Firestore security rules allow writes

### Issue: Notifications appear but wrong status
- [ ] Check browser console message has correct `status` field
- [ ] Check NotificationAdapter handles that status value
- [ ] Verify chainOrder is correct (0 for non-chained)

### Issue: Notifications slow/delayed
- [ ] Check browser network (DevTools Network tab)
- [ ] Check Firestore quota (usually not an issue)
- [ ] Verify Android app is listening to right collection path
- [ ] Check timestamps in Firestore (should be server time)

---

## Database Paths Used

### RTDB (Existing)
```
/patients/{uid}/notifications/{notifKey}        ← RTDB notifications (legacy)
/patients/{uid}/queueRequests/{requestKey}      ← Request status
/patients/{uid}/activeQueues/{queueKey}         ← Current queue
/establishments/{estId}/counters/{counterId}/requests/{requestKey}
```

### Firestore (New)
```
/patients/{uid}/inbox/{autoGeneratedId}         ← Notifications (NEW)
```

---

## Key Implementation Details

### Dual Writes
- RTDB notifications still created (backward compatibility)
- Firestore notifications added in parallel
- If one fails, other still works
- Android app uses Firestore for primary notifications

### Non-Blocking
- All `await pushArrivalNotification()` calls are `async`
- Don't block RTDB update (`await db.ref().update()`)
- If notification push fails, queue processing continues

### Atomic Writes
- RTDB update is atomic (all or nothing)
- Firestore write is non-atomic (fire and forget)
- Guarantees RTDB data consistency even if Firestore unavailable

### Timestamp
- Uses `firebase.firestore.FieldValue.serverTimestamp()`
- Firestore auto-sets on server (not client time)
- Ensures accurate timeline across all devices

---

## Summary of Changes

| Change | Lines | Purpose |
|--------|-------|---------|
| Firestore SDK Import | 591 | Load Firestore library |
| Firestore Init | 744 | Initialize connection |
| Helper Functions | 650-727 | Reusable notification logic |
| Request Accept | 975 | Push awaiting_arrival |
| Request Reject | 1042 | Push rejected |
| Admit Next | 1664 | Push in_service |
| Chain Queue | 1850 | Push waiting + chainOrder |
| Service Complete (Final) | 1943 | Push completed |
| Service Complete (Chain) | 1954 | Push waiting + chainOrder |
| Patient Arrival | 2195 | Push waiting |

**Total**: ~200 lines of code added across ~9 locations

---

## Next Steps

1. **Test**: Deploy and test with Android app
2. **Verify**: Confirm all 7 lifecycle notifications appear
3. **Monitor**: Check browser console for any `❌ Error pushing notification`
4. **Iterate**: Adjust messages if needed
5. **Scale**: Deploy to production

---

**Status**: ✅ Implementation Complete - Ready for Testing
