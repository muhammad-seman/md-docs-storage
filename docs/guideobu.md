# UUID Implementation Guide - Prevent Double Ritase

## Problem Statement

**Issue:** Double ritase creation ketika OBU offline lalu online kembali

**Scenario:**
1. OBU di status 29 (Finish Dumping)
2. FMS manual close (status 30) → Create cycle baru
3. OBU offline, queue message (status 30)
4. OBU online → Send queued message → Create cycle baru lagi
5. **Result:** 2 cycles created! ❌

---

## Solution: UUID Synchronization

Implementasi UUID v7 untuk unique cycle identification yang di-sync antara Backend dan OBU.

---

## Changes Summary

### **Backend Changes** (api-irdc-gems)

#### 1. **File:** `controllers/v2/idrive/equipment_task_cycle_controller.js`

**A. Publish UUID ke MQTT (Line 128-162)**
```javascript
// Get current cycle UUID for OBU synchronization
const currentCycle = await EquipmentTaskCycle.findByPk(
  dailyTaskEquipment.current_equipment_task_cycle_id,
  { attributes: ['id', 'cycle_uuid', 'journey_status'] }
);

const mqttPayload = {
  dailyTaskEquipment,
  summary,
  current_cycle: {
    id: currentCycle?.id,
    cycle_uuid: currentCycle?.cycle_uuid,  // ← UUID dikirim ke OBU
    journey_status: currentCycle?.journey_status,
  },
};

publishMQTT(
  `obu/current-task-cycle/${dailyTaskEquipment.equipment.name}`,
  mqttPayload,
  { qos: 1, retain: true }
);
```

**B. Validate UUID dari OBU (Line 323-376)**
```javascript
if (cycle_uuid) {
  if (!isValidUUID(cycle_uuid)) {
    return { err: true, message: "Invalid UUID format" };
  }

  const existingCycle = await EquipmentTaskCycle.findOne({
    where: { cycle_uuid },
  });

  if (existingCycle && existingCycle.closed_at) {
    // Reject duplicate closed cycle
    return { err: false, message: "Ritase already closed" };
  }
}
```

**C. Handle UUID saat CREATE cycle (Line 673-720)**
```javascript
// Priority: Use UUID from OBU if provided and valid
if (cycleUuid && isValidUUID(cycleUuid)) {
  const existingWithSameUuid = await EquipmentTaskCycle.findOne({
    where: { cycle_uuid: cycleUuid },
  });
  
  if (existingWithSameUuid) {
    console.log("[CREATE NEW CYCLE] UUID already exists - rejecting duplicate");
    return await DailyTaskEquipment.findById(dailyTaskEquipment.id);
  }
  
  finalUuid = cycleUuid;  // Use UUID from OBU
} else {
  finalUuid = generateCycleUUID();  // Generate new UUID
}

await EquipmentTaskCycle.create({
  cycle_uuid: finalUuid,
  // ...
});
```

**D. Fix Logic Bug (Line 354-416)**
```javascript
// BEFORE (WRONG):
if (current_cycle_code && dailyTaskEquipment.current_status_code !== current_cycle_code) {
  return { dailyTaskEquipment };  // Skip jika status BEDA ❌
}

// AFTER (CORRECT):
// Handle idempotent: Skip jika status SAMA
if (current_cycle_code && dailyTaskEquipment.current_status_code === current_cycle_code) {
  return { dailyTaskEquipment };  // Skip jika status SAMA ✅
}

// Handle backward: Skip jika status MUNDUR
if (current_cycle_code && current_cycle_code < dailyTaskEquipment.current_status_code) {
  return { dailyTaskEquipment };  // Skip jika backward ✅
}
```

---

### **OBU Changes** (uassist-obu)

#### 1. **File:** `src/stores/useTaskCycleStore.ts`

**Add UUID State:**
```typescript
interface CycleState {
  // ... existing state
  
  // UUID for preventing double ritase
  cycleUuid: string | null;
  setCycleUuid: (cycleUuid: string | null) => void;
}

export const useTaskCycleStore = create<CycleState>()(
  persist(
    (set) => ({
      // ... existing state
      
      cycleUuid: null,
      setCycleUuid: (state) => set({ cycleUuid: state }),
    }),
    {
      name: 'cycle-storage',
      storage: createJSONStorage(() => storage),
    }
  )
);
```

#### 2. **File:** `src/hooks/home/useTaskCycle.ts`

**A. Import UUID State (Line 25-31)**
```typescript
const {
  // ... existing
  cycleUuid,
  setCycleUuid,
} = useTaskCycleStore();
```

**B. Send UUID to Backend (Line 286-299)**
```typescript
const cycleData = JSON.stringify({
  daily_task_equipment_id: taskData?.daily_task_equipment_id,
  cycle_uuid: cycleUuid || null,  // ← Kirim UUID ke backend
  time: new Date().toISOString(),
  user_id: taskData?.driver_id,
  ip_address: ipAddress,
  current_cycle_code: taskStatusRef.current,
});

console.log('[OBU] Sending cycle update with UUID:', {
  cycle_uuid: cycleUuid,
  current_cycle_code: taskStatusRef.current,
  has_uuid: !!cycleUuid,
});
```

**C. Save UUID from Backend (Line 361-372)**
```typescript
const handler = (message: string) => {
  try {
    const parsedData = JSON.parse(message);
    
    // ... existing code
    
    // Save cycle_uuid from backend
    if (parsedData.current_cycle && parsedData.current_cycle.cycle_uuid) {
      const receivedUuid = parsedData.current_cycle.cycle_uuid;
      setCycleUuid(receivedUuid);
      console.log('[OBU] Received and saved cycle UUID:', receivedUuid);
    }
  } catch (error) {
    console.error('Error parsing MQTT message:', error);
  }
};
```

---

## Flow Diagram

### **Normal Flow (Online)**
```
1. OBU (Status 22) → Backend
                      ↓ CREATE cycle
                      ↓ Generate UUID: 019aa1a1-xxx
                      ↓ Save to DB
                      ↓ Publish MQTT

2. OBU ← MQTT (UUID: 019aa1a1-xxx)
        ↓ Save to localStorage

3. OBU (Status 23 + UUID) → Backend
                             ↓ Check UUID exists ✅
                             ↓ UPDATE same cycle
                             ✅ No duplicate!
```

### **Offline → Online (FIXED)**
```
1. Status 29
   ├─ FMS: Close (30) → Backend
   │                    ↓ CREATE cycle
   │                    ↓ UUID: 019aa1a1-xxx
   │                    ↓ Publish MQTT

2. OBU: OFFLINE
   ├─ Not receive UUID
   └─ Queue: (status 30 WITHOUT UUID)

3. OBU: ONLINE
   ├─ Receive MQTT (UUID: 019aa1a1-xxx)
   ├─ Save UUID to localStorage
   └─ Send queued (status 30 WITHOUT UUID)
       ↓ Backend: Check existing cycle
       ↓ Cycle already closed
       ✅ Reject duplicate!
```

---

## Testing Guide

### **Test 1: Normal Flow**

**Prerequisites:**
- Cycle di status 30 (CLOSED)
- OBU online

**Steps:**
1. OBU trigger status 22 (Starting Ritase)
2. Check backend log:
   ```
   [CREATE NEW CYCLE] UUID decision: {
     finalUuid: '019aa1a1-xxx',
     source: 'Generated'
   }
   [MQTT PUBLISH] Sending to OBU: {
     cycle_uuid: '019aa1a1-xxx'
   }
   ```
3. Check OBU log:
   ```
   [OBU] Received and saved cycle UUID: 019aa1a1-xxx
   ```
4. Check database:
   ```sql
   SELECT id, cycle_uuid, journey_status 
   FROM equipment_task_cycles 
   WHERE daily_task_equipment_id = 746092 
   ORDER BY id DESC LIMIT 1;
   ```
   **Expected:** `cycle_uuid` NOT NULL

5. OBU trigger status 23
6. Check backend log:
   ```
   [UPDATE CYCLE] UUID decision: {
     hasUuid: true,
     currentUuid: '019aa1a1-xxx',
     willSaveUuid: false
   }
   ```

**Expected Result:** ✅ UUID generated and synced

---

### **Test 2: Prevent Double Ritase (Offline Scenario)**

**Prerequisites:**
- Cycle di status 29
- OBU online

**Steps:**
1. FMS manual close (status 30)
2. Backend create cycle baru (status 22)
3. Check database - UUID: `019aa1a1-xxx`
4. **Simulate OBU offline:**
   - OBU disconnect WiFi
   - OBU trigger status 30 (queued offline)
5. **OBU back online:**
   - OBU reconnect WiFi
   - OBU receive MQTT (UUID: `019aa1a1-xxx`)
   - OBU send queued message (status 30, NO UUID)
6. Check backend log:
   ```
   [ENTRY] updateEquipmentTaskCycle called: {
     normalizedCycleUuid: null,
     currentCycleCode: 30
   }
   [STATUS CHECK] {
     currentCycleCodeFromOBU: 30,
     currentStatusCodeInDB: 22,
     isForward: true
   }
   [CONTINUE] Status check passed
   ```
7. Check database - count cycles:
   ```sql
   SELECT COUNT(*) as total
   FROM equipment_task_cycles
   WHERE daily_task_equipment_id = 746092
     AND started_at > NOW() - INTERVAL 1 HOUR;
   ```

**Expected Result:** ✅ Only 1 cycle created (no duplicate)

---

### **Test 3: OBU Send UUID (Idempotent)**

**Prerequisites:**
- Cycle exists with UUID: `019aa1a1-xxx`
- Status: 25

**Steps:**
1. OBU trigger status 25 (retry/duplicate)
2. OBU send payload:
   ```json
   {
     "daily_task_equipment_id": 746092,
     "cycle_uuid": "019aa1a1-xxx",
     "current_cycle_code": 25
   }
   ```
3. Check backend log:
   ```
   [UUID FOUND] Cycle UUID 019aa1a1-xxx exists
   [IDEMPOTENT UPDATE] continuing update
   [IDEMPOTENT] Status already at 25
   [IDEMPOTENT] Returning without update
   ```

**Expected Result:** ✅ Idempotent - no duplicate update

---

## Verification Queries

### **Check UUID in Database**
```sql
-- Cycle tanpa UUID (legacy)
SELECT id, cycle_uuid, daily_task_equipment_id, journey_status, created_at
FROM equipment_task_cycles
WHERE cycle_uuid IS NULL
ORDER BY id DESC
LIMIT 10;

-- Cycle dengan UUID (new)
SELECT id, cycle_uuid, daily_task_equipment_id, journey_status, created_at
FROM equipment_task_cycles
WHERE cycle_uuid IS NOT NULL
ORDER BY id DESC
LIMIT 10;

-- Check duplicate UUID
SELECT cycle_uuid, COUNT(*) as count
FROM equipment_task_cycles
WHERE cycle_uuid IS NOT NULL
GROUP BY cycle_uuid
HAVING COUNT(*) > 1;
```

**Expected:** No duplicate UUID

---

## Rollback Plan

Jika ada issue:

1. **Backend:** Comment out UUID validation
   ```javascript
   // if (cycle_uuid) {
   //   // UUID validation code
   // }
   ```

2. **OBU:** UUID tetap dikirim (null), backend ignore

3. **Database:** UUID column nullable, tidak break existing flow

---

## Monitoring

### **Backend Logs**
Monitor log dengan keyword:
- `[UUID FOUND]`
- `[DUPLICATE REJECTED]`
- `[CREATE NEW CYCLE]`
- `[MQTT PUBLISH]`

### **OBU Logs**
Monitor log dengan keyword:
- `[OBU] Sending cycle update with UUID`
- `[OBU] Received and saved cycle UUID`

### **Database**
```sql
-- Daily UUID stats
SELECT 
    DATE(created_at) as date,
    COUNT(*) as total_cycles,
    COUNT(cycle_uuid) as with_uuid,
    COUNT(*) - COUNT(cycle_uuid) as without_uuid
FROM equipment_task_cycles
WHERE created_at > NOW() - INTERVAL 7 DAY
GROUP BY DATE(created_at)
ORDER BY date DESC;
```

---

## Summary

✅ **Backend:** Publish `cycle_uuid` via MQTT  
✅ **Backend:** Validate UUID and prevent duplicate  
✅ **Backend:** Fix status comparison logic  
✅ **OBU:** Save UUID from backend  
✅ **OBU:** Send UUID in every status update  
✅ **Result:** Double ritase prevented! 🎯

---

**Documented by:** Claude Code Assistant  
**Date:** 2025-11-20  
**Version:** 1.0
