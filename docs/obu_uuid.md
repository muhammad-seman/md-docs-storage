Perubahan yang Diperlukan di OBU:**

### **1. Subscribe ke Topic MQTT untuk Terima UUID dari Backend**

OBU harus subscribe ke:
```
obu/current-task-cycle/{equipment_name}
```

**Payload yang diterima OBU sekarang include `cycle_uuid`:**
```json
{
  "dailyTaskEquipment": { ... },
  "summary": { ... },
  "current_cycle": {
    "id": 12505,
    "cycle_uuid": "019aa1a1-8df3-745b-9ac8-7ff0e450a722",
    "journey_status": 22
  }
}
```

---

### **2. Simpan `cycle_uuid` di Local Storage OBU**

**Ketika terima response dari backend:**
```javascript
// Di OBU code
onMqttMessage('obu/current-task-cycle/{equipment_name}', (message) => {
  const { current_cycle } = message;
  
  // Save to local storage
  localStorage.setItem('current_cycle_uuid', current_cycle.cycle_uuid);
  localStorage.setItem('current_cycle_id', current_cycle.id);
  localStorage.setItem('current_journey_status', current_cycle.journey_status);
  
  console.log('[OBU] Received cycle UUID from backend:', current_cycle.cycle_uuid);
});
```

---

### **3. Kirim `cycle_uuid` Setiap Kali Update Status**

**Sebelum (LAMA - TANPA UUID):**
```json
{
  "daily_task_equipment_id": 746092,
  "time": "2025-11-20T14:20:00.000Z",
  "user_id": 100,
  "ip_address": "192.168.1.16",
  "current_cycle_code": 23
}
```

**Sekarang (BARU - DENGAN UUID):**
```json
{
  "daily_task_equipment_id": 746092,
  "cycle_uuid": "019aa1a1-8df3-745b-9ac8-7ff0e450a722",  ← TAMBAHKAN INI!
  "time": "2025-11-20T14:20:00.000Z",
  "user_id": 100,
  "ip_address": "192.168.1.16",
  "current_cycle_code": 23
}
```

---

### **4. Logic di OBU untuk Handle Offline Queue**

**Pseudocode untuk OBU:**

```javascript
// OBU Code - Handle status update
function sendStatusUpdate(statusCode) {
  // Get cycle UUID dari local storage
  const cycleUuid = localStorage.getItem('current_cycle_uuid');
  
  const payload = {
    daily_task_equipment_id: EQUIPMENT_ID,
    cycle_uuid: cycleUuid,  // ← Kirim UUID jika ada
    time: new Date().toISOString(),
    user_id: USER_ID,
    ip_address: IP_ADDRESS,
    current_cycle_code: statusCode,
  };
  
  console.log('[OBU] Sending status update with UUID:', cycleUuid);
  
  if (isOnline()) {
    // Send immediately
    publishMQTT('drvatt/task-cycle/', payload);
  } else {
    // Queue offline - simpan dengan UUID
    queueOfflineMessage(payload);
  }
}

// Ketika online kembali
function onBackOnline() {
  console.log('[OBU] Back online, processing queued messages...');
  
  const queuedMessages = getQueuedMessages();
  
  queuedMessages.forEach((message) => {
    // Backend akan reject duplicate berdasarkan UUID
    publishMQTT('drvatt/task-cycle/', message);
  });
}
```

---

## **Flow Scenario dengan UUID:**

### **Scenario 1: Normal Flow (Online)**
```
OBU (Status 22) → Backend 
                  ↓ Create cycle
                  ↓ Generate UUID: 019aa1a1-xxx
                  ↓ Save to DB
                  ↓ Publish to MQTT
OBU ← Receive UUID (019aa1a1-xxx)
    ↓ Save to localStorage
    
OBU (Status 23 + UUID) → Backend
                         ↓ Check UUID exists ✅
                         ↓ Update same cycle (no duplicate)
```

### **Scenario 2: Offline → Online (FIXED!)**
```
Status 29 (Finish Dumping)
├─ FMS: Close (30) → Create new (22) → UUID: 019aa1a1-xxx
│                                        ↓ Publish to MQTT
├─ OBU: OFFLINE (tidak terima UUID)
│       Queue: (status 30 WITHOUT UUID)
│
└─ OBU: ONLINE
        ↓ Send queued (status 30 WITHOUT UUID)
        ↓ Backend: Generate UUID: 019aa1a2-yyy
        ↓ Publish UUID: 019aa1a2-yyy to MQTT
        ↓ OBU receive UUID: 019aa1a2-yyy
        
✅ Result: 2 cycles (tapi berbeda equipment atau different timing)
   ATAU Backend detect duplicate dan reject!
```

**Lebih baik:**
```
Status 29
├─ FMS: Close (30) → Backend reject (OBU belum kirim 30)
│                    OR Backend create → UUID: 019aa1a1-xxx
│                                         ↓ Publish to MQTT
│
├─ OBU: OFFLINE (tidak dapat UUID 019aa1a1-xxx)
│       Queue: (status 30 WITHOUT UUID)
│
└─ OBU: ONLINE 
        ↓ Fetch latest cycle dari backend via API
        ↓ Get UUID: 019aa1a1-xxx
        ↓ Save to localStorage
        ↓ Clear queue (cycle sudah closed)
        
✅ Result: No duplicate!
```

---

## **Rekomendasi Implementasi di OBU:**

### **A. Subscribe MQTT Topic**
```javascript
mqttClient.subscribe('obu/current-task-cycle/' + equipmentName);

mqttClient.on('message', (topic, message) => {
  if (topic.includes('obu/current-task-cycle/')) {
    const data = JSON.parse(message);
    
    if (data.current_cycle) {
      localStorage.setItem('cycle_uuid', data.current_cycle.cycle_uuid);
      console.log('[OBU] UUID saved:', data.current_cycle.cycle_uuid);
    }
  }
});
```

### **B. Kirim UUID saat Update**
```javascript
function sendStatusUpdate(statusCode) {
  const cycleUuid = localStorage.getItem('cycle_uuid');
  
  const payload = {
    daily_task_equipment_id: 746092,
    cycle_uuid: cycleUuid || null,  // null jika belum ada
    time: new Date().toISOString(),
    user_id: 100,
    ip_address: getIPAddress(),
    current_cycle_code: statusCode,
  };
  
  mqtt.publish('drvatt/task-cycle/', JSON.stringify(payload));
}
```

### **C. Handle Back Online**
```javascript
function onReconnect() {
  console.log('[OBU] Reconnected to server');
  
  // Sync latest cycle dari backend
  fetch('/api/v2/idrive/task-equipment/current-cycle?daily_task_equipment_id=746092')
    .then(res => res.json())
    .then(data => {
      if (data.current_cycle) {
        localStorage.setItem('cycle_uuid', data.current_cycle.cycle_uuid);
        console.log('[OBU] Synced UUID:', data.current_cycle.cycle_uuid);
      }
      
      // Kirim queued messages
      processQueuedMessages();
    });
}
```

---

## **Summary Perubahan OBU:**

✅ **Subscribe** ke `obu/current-task-cycle/{equipment_name}`  
✅ **Save** `cycle_uuid` ke localStorage saat terima dari backend  
✅ **Kirim** `cycle_uuid` setiap kali update status  
✅ **Sync** cycle UUID saat back online  
✅ **Backend** akan otomatis reject duplicate berdasarkan UUID