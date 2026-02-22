# Firestore Notification Pushes Implementation - Counter.html

## Overview
Added comprehensive Firestore notification push functionality to Counter.html to align with Android NotificationAdapter expectations. All queue lifecycle events and chain queue steps now push notifications to `patients/{uid}/inbox/{docId}`.

## Implementation Details

### 1. Firestore Initialization (Lines 591, 744)

**Added SDK Import:**
```html
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-firestore-compat.js"></script>
```

**Initialized Firestore:**
```javascript
const firestore = firebase.firestore();
```

### 2. Notification Helper Functions (Lines 657-703)

Three helper functions added to manage notification pushes:

#### A. pushArrivalNotification() - Lines 657-681
Pushes arrival update notifications to Firestore
- **Purpose**: Handle all queue lifecycle events (awaiting_arrival, waiting, in_service, completed, rejected, chain steps)
- **Parameters**:
  ```javascript
  uid              // Patient UID
  data = {
    estId,         // Establishment ID
    estName,       // Establishment name
    counterId,     // Counter ID
    counterName,   // Counter name
    status,        // "awaiting_arrival", "waiting", "in_service", "completed", "rejected"
    message,       // Notification message
    chainOrder     // For chain queue steps (0 for non-chained)
  }
  ```
- **Writes to**: `patients/{uid}/inbox/{autoId}`
- **Error handling**: Logs to console if push fails

#### B. pushAppointmentNotification() - Lines 687-703
Pushes appointment-related notifications (ready for future implementation)
- **Purpose**: Handle appointment acceptance, rejection, rescheduling
- **Parameters**:
  ```javascript
  uid
  data = {
    estId, estName,
    type: "appointment",
    status,          // "accepted", "rejected", "rescheduled", "completed"
    service,
    message
  }
  ```

#### C. pushInfoNotification() - Lines 709-727
Generic fallback notifications
- **Purpose**: Other notification types
- **Parameters**:
  ```javascript
  uid
  data = {
    estId, estName,
    type,    // Custom type
    message
  }
  ```

### 3. Queue Lifecycle Notification Pushes

#### A. Request Accepted (Lines 971-998)
**Trigger**: Counter staff clicks ACCEPT on a queue request

**For New Arrivals (No existing arrivalIndex):**
```javascript
await pushArrivalNotification(reqData.uid, {
    estId: estId,
    estName: estName,
    counterId: counterId,
    counterName: counterName,
    status: "awaiting_arrival",
    message: "Your request has been accepted. Please proceed to check in.",
    chainOrder: 0
});
```

**For Chain Queue Steps (Patient already has arrivalIndex):**
```javascript
await pushArrivalNotification(reqData.uid, {
    estId: estId,
    estName: estName,
    counterId: counterId,
    counterName: counterName,
    status: "waiting",
    message: "You've been added to a chained queue. Please wait for the next available counter.",
    chainOrder: nextChainOrder
});
```

**Android Behavior**:
- status="awaiting_arrival" → Shows "ARRIVAL" pill, message tells patient to proceed to counter
- status="waiting" with chainOrder>0 → Shows "⚓ Queue #N" pill for chained queue

#### B. Request Rejected (Lines 1000-1048)
**Trigger**: Counter staff clicks REJECT on a queue request

```javascript
await pushArrivalNotification(reqData.uid, {
    estId: estId,
    estName: estName,
    counterId: counterId,
    counterName: counterName,
    status: "rejected",
    message: `Your request has been rejected at ${counterName}.`,
    chainOrder: 0
});
```

#### C. Patient Arrival (Waiting in Queue) (Lines 2185-2196)
**Trigger**: Patient arrives via RFID scan or PIN verification, moved from "awaiting_arrival" to active queue

```javascript
await pushArrivalNotification(patientId, {
    estId: estId,
    estName: estName,
    counterId: targetCounterId,
    counterName: counterName,
    status: "waiting",
    message: `You are now in the queue at ${counterName}.`,
    chainOrder: arrivalIndexData.chainOrder || 0
});
```

**Android Behavior**: Shows patient in Active Queues list

#### D. Service Started (In Service) (Lines 1664-1673)
**Trigger**: Counter staff clicks "Admit Next" (btnNext), patient moves from "waiting" to "in_service"

```javascript
await pushArrivalNotification(data.uid, {
    estId: estId,
    estName: estNameVal,
    counterId: counterId,
    counterName: counterNameVal,
    status: "in_service",
    message: `You are now being served at ${counterNameVal}.`,
    chainOrder: data.chainOrder || 0
});
```

**Android Behavior**: Active queue updates to show patient is being served

#### E. Chain Queue Step Ready (Lines 1850-1859)
**Trigger**: Patient completes service at one counter, next counter in chain is ready

```javascript
await pushArrivalNotification(qData.uid, {
    estId: nextEstId,
    estName: nextEstName,
    counterId: nextCounterId,
    counterName: nextCounterName,
    status: "waiting",
    message: `Please proceed to ${nextCounterName} for the next step of your service.`,
    chainOrder: lowestChainOrder
});
```

**Android Behavior**: Shows "⚓ Queue #N: You're next in a chained queue!"

#### F. Service Completion - Final (Lines 1943-1952)
**Trigger**: Counter staff clicks "End Service" (btnEndService), patient completes final service with NO more chain steps

```javascript
await pushArrivalNotification(qData.uid, {
    estId: estId,
    estName: estNameVal,
    counterId: counterId,
    counterName: counterNameVal,
    status: "completed",
    message: "Your service has been completed. Thank you for visiting!",
    chainOrder: 0
});
```

**Android Behavior**: Removes from Active Queues, appears in Recently Visited

#### G. Service Completion - With Chain Progression (Lines 1954-1962)
**Trigger**: Counter staff clicks "End Service", patient continues to next chain step

```javascript
await pushArrivalNotification(qData.uid, {
    estId: estId,
    estName: estNameVal,
    counterId: counterId,
    counterName: counterNameVal,
    status: "waiting",
    message: "Service complete at this counter. Proceeding to next step...",
    chainOrder: qData.chainOrder || 0
});
```

## Firestore Document Structure

All notifications are written to:
```
patients/{uid}/inbox/{autoGenId}
```

With fields:
```javascript
{
  estId: string,           // Establishment ID
  estName: string,         // Establishment name
  counterId: string,       // Counter ID (optional)
  counterName: string,     // Counter name (optional)
  type: "arrival_update",  // Notification type
  status: string,          // Queue status (awaiting_arrival, waiting, in_service, completed, rejected)
  message: string,         // Notification message
  chainOrder: number,      // For chained queues (0 = single queue)
  timestamp: ServerTimestamp  // Auto-set by Firestore
}
```

## Android NotificationAdapter Integration

The implementation aligns with Android's NotificationAdapter expectations:

1. **Arrival Update Pill** (type="arrival_update"):
   - status="awaiting_arrival" → "ARRIVAL" pill with warning color
   - status="waiting" (chainOrder=0) → Shows in active queue
   - status="waiting" (chainOrder>0) → "⚓ Queue #N" pill for chained queue
   - status="in_service" → "ARRIVAL" pill, message shows patient is being served
   - status="completed" → Not shown (removed from active queues)
   - status="rejected" → Not shown

2. **Message Display**:
   - Each notification has custom message text
   - Includes counter name context
   - Supports chainOrder for chain queue numbering

3. **Automatic UI Updates**:
   - HomeActivity listens to `patients/{uid}/inbox`
   - NotificationAdapter filters and displays based on type and status
   - Active Queues list updates automatically based on arrivalIndex status

## Queue State Transitions

```
Request → Accepted → Awaiting Arrival → Arrival → Waiting → In Service → Completed/Rejected

Notification Flow:
1. Accept Request → "awaiting_arrival" notification
2. Patient Arrives (RFID/PIN) → "waiting" notification
3. Admit Next → "in_service" notification
4. End Service → "completed" notification (or "waiting" if chain)
5. Chain Step → "waiting" notification with chainOrder

At Each Step:
- Firestore notification pushed to patients/{uid}/inbox
- RTDB notifications created (legacy)
- arrivalIndex updated to reflect current status
- Android ActiveQueueItem automatically updates
```

## Error Handling

All notification pushes are **non-blocking** and include error handling:
- Failures log to console but don't interrupt queue processing
- If Firestore is unavailable, RTDB notifications still function
- Allows graceful degradation if Firestore connectivity issues occur

## Testing Checklist

- [ ] Accept queue request → Firestore notification appears in inbox
- [ ] Reject queue request → "rejected" notification appears
- [ ] Patient arrives via RFID/PIN → "waiting" notification appears
- [ ] Click "Admit Next" → "in_service" notification appears
- [ ] Accept chained request → "waiting" notification with chainOrder>0
- [ ] Complete service (no chain) → "completed" notification
- [ ] Complete service (with chain) → "waiting" notification for next step
- [ ] Verify Android shows notifications in NotificationAdapter
- [ ] Verify Android Active Queues updates automatically
- [ ] Test multiple patients in parallel queues

## Related Components

- **Database paths**:
  - `patients/{uid}/inbox/{docId}` - Firestore notifications
  - `patients/{uid}/notifications/{notifKey}` - RTDB notifications (legacy)
  - `arrivalIndex/{uid}` - Queue state indicator
  
- **Android components**:
  - NotificationAdapter - Displays notifications
  - HomeActivity - Shows active queues (reads arrivalIndex)
  - MyRequestsActivity - Shows all requests
  
- **Web components**:
  - Counter.html - Queue management (this file)
  - All handlers now push notifications simultaneously with RTDB updates

## Summary

✅ **Firestore Integration**: Added complete Firestore support for real-time notifications
✅ **All Queue Events**: Covered all major queue lifecycle events
✅ **Chain Queue Support**: Notifications for multi-step service chains
✅ **Android Alignment**: Notifications match expected NotificationAdapter format
✅ **Non-blocking**: All pushes are async and don't block queue processing
✅ **Error Handling**: Graceful failures with console logging
✅ **Dual Writes**: Maintains both RTDB and Firestore notifications for compatibility

The implementation ensures seamless real-time notification delivery to Android app users at every stage of their queue journey.
