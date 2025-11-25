Implementation Steps** →

---

## 1. SOURCE TABLES (Input)

### **tc_events** (Traccar GPS Events)
```
Source: GPS Tracker (Traccar system)
Data: geofence enter/exit events
```
**Columns:**
- `id` (PK) → event_id
- `deviceid` → device_id
- `type` → 'geofenceEnter' | 'geofenceExit'
- `eventtime` → UTC timestamp
- `positionid` → GPS position reference
- `geofenceid` → ROM/PORT/COP/KMH/TIA/PORTKOSONG

**Action:** INSERT real-time dari GPS tracker

---

### **tc_devices**
```
Source: Master data unit/truck
Data: device metadata
```
**Columns:**
- `id` (PK)
- `name` → hull/unit name

**Action:** SELECT untuk lookup hull name

---

### **ucan_staging_old** (UCAN/ITWS Timbangan)
```
Source: ITWS timbangan system
Data: closing weight transactions
```
**Columns:**
- `id` (PK)
- `device_id`
- `sk_id_itws` → transaction ID
- `closing_created` → waktu timbangan
- `closing_weight` → berat (ton)
- `status` → 'NEW' | 'ATTACHED' | 'FAILED'
- `linked_id` → post_integration.id (setelah attach)
- `attach_method` → 'COP' | 'PORTKOSONG' | 'PORTEXIT'
- `diff_min` → selisih waktu (menit)

**Action:** INSERT dari sistem ITWS

---

### **fsm_replay_stage** (Staging/Batch)
```
Source: Manual upload atau batch replay
Data: historical events untuk reprocessing
```
**Columns:**
- `id` (PK)
- `event_id`, `device_id`, `geofence_id`, `position_id`
- `kind` → ROM/PORT/COP/etc
- `event_type` → enter/exit
- `devicetime` → waktu event (lokal)
- `hull`, `equipment_id`
- `used_flag` → 0|1 (sudah diproses atau belum)

**Action:** INSERT manual/batch → SELECT WHERE used_flag=0

---

## 2. PROCESSING LAYER (SP)

### **sp_build_ritase** (Main Processor)
```
Input: tc_events (per event)
Output: post_integration (trip formed)
```

**Actions per Event Type:**

| Kind | Type | Action | Tables Affected |
|------|------|--------|----------------|
| ROM | ENTER | INSERT trip baru ATAU UPDATE trip open | post_integration (I/U), anomaly_log (I) |
| ROM | EXIT | UPDATE trip (rom_exit_time) | post_integration (U), anomaly_log (I) |
| PORT | ENTER | UPDATE trip ATAU INSERT baru | post_integration (I/U), anomaly_log (I) |
| PORT | EXIT | UPDATE trip (port_exit_time, closing) | post_integration (U), anomaly_log (I) |
| COP | ENTER/EXIT | UPDATE trip (cop_confirm_time, closing) | post_integration (U), anomaly_log (I) |
| PORTKOSONG | ENTER | UPDATE trip (portkosong_time, closing) | post_integration (U), anomaly_log (I) |
| KMH | ENTER | UPDATE trip ATAU INSERT placeholder | post_integration (I/U), anomaly_log (I) |
| TIA | ENTER/EXIT | UPDATE flag is_tia=1 | post_integration (U), anomaly_log (I) |

**Internal Actions:**
1. **SELECT** tc_devices → get hull
2. **SELECT** post_integration → find trip kandidat (open/nearest)
3. **INSERT/UPDATE** post_integration → build/update trip
4. **Same-kind dedupe:** UPDATE multi-row → set is_suspect=1 (loser)
5. **ROM-CLOSING merge:** UPDATE 2 row → merge + flag loser
6. **Final dedupe:** UPDATE multi-row → cluster closing, flag duplicate
7. **INSERT** _buckets_to_touch (temp) → track affected buckets
8. **CALL** _pis_reseq_bucket → resequence ritase
9. **INSERT** anomaly_log → log anomalies
10. **INSERT** fsm_debug_log → log debug info

---

### **_pis_reseq_bucket** (Resequencer)
```
Input: (device_id, task_date, shift_id)
Output: post_integration.ritase updated
```

**Actions:**
1. **UPDATE** post_integration SET ritase=NULL (reset)
2. **SELECT** post_integration → ROW_NUMBER() by closing_time_norm, closing_created, ROM, PORT, COP, id
3. **UPDATE** post_integration SET ritase=1,2,3,...
4. **INSERT** fsm_debug_log

**Called by:**
- sp_build_ritase (otomatis di akhir)
- sp_ucan_attach (setelah attach UCAN)

---

### **sp_ucan_attach** (UCAN Integrator)
```
Input: ucan_staging_old (batch)
Output: post_integration + UCAN data
```

**Actions:**
1. **SELECT** ucan_staging_old WHERE status='NEW' LIMIT batch_size
2. **LOOP** per UCAN row:
   - **SELECT** fn_shift_of_local → get task_date, shift_id
   - **SELECT** post_integration → find trip match (COP/PK/PX ±60 min)
   - **IF FOUND:**
     - **UPDATE** post_integration SET transactionID, closing_created, closing_weight, task_date, shift_id, ritase=NULL
     - **UPDATE** ucan_staging_old SET status='ATTACHED', linked_id
     - **CALL** _pis_reseq_bucket (bucket lama)
     - **CALL** _pis_reseq_bucket (bucket baru)
   - **IF NOT FOUND:**
     - **UPDATE** ucan_staging_old SET status='FAILED'
     - **INSERT** anomaly_log_ucan

---

### **fn_shift_of_local** (Helper Function)
```
Input: datetime lokal
Output: 'YYYY-MM-DD|shift_id'
```

**Logic:**
- 06:00-17:59 → Shift 1, tanggal sama
- 18:00-05:59 → Shift 2, <06:00 mundur 1 hari

**Called by:**
- sp_build_ritase (setiap event)
- sp_ucan_attach (setiap UCAN row)

---

## 3. OUTPUT TABLE (Result)

### **post_integration** (Main Output)
```
Purpose: Final ritase data (GPS + UCAN)
```

**Data Flow:**

| Column | Populated By | Source |
|--------|--------------|--------|
| id (PK) | AUTO_INCREMENT | - |
| device_id, hull | sp_build_ritase | tc_devices |
| task_date, shift_id | fn_shift_of_local | calculated |
| rom_enter_time, rom_exit_time | sp_build_ritase | tc_events (ROM) |
| port_enter_time, port_exit_time | sp_build_ritase | tc_events (PORT) |
| cop_confirm_time | sp_build_ritase | tc_events (COP) |
| portkosong_time | sp_build_ritase | tc_events (PK) |
| rom_geofence_id, port_geofence_id, cop_geofence_id | sp_build_ritase | tc_events.geofenceid |
| closing_kind, closing_ts, closing_score | sp_build_ritase | calculated |
| transactionID | sp_ucan_attach | ucan_staging_old.sk_id_itws |
| closing_created | sp_ucan_attach | ucan_staging_old.closing_created |
| closing_weight | sp_ucan_attach | ucan_staging_old.closing_weight |
| attach_method, diff_min | sp_ucan_attach | calculated |
| ritase | _pis_reseq_bucket | calculated (ROW_NUMBER) |
| event_id, event_type | sp_build_ritase | tc_events |
| is_suspect, suspect_reason | sp_build_ritase | calculated (dedupe) |
| rom_missing_flag | sp_build_ritase | 1 (KMH tanpa ROM) |
| is_tia, via_tia | sp_build_ritase | 1 (TIA event) |
| source | sp_build_ritase | 'GPS' |
| created_at, updated_at | sp_build_ritase | NOW(3) |

**Unique Index:**
- `ux_ritase_bucket` (device_id, task_date, shift_id, ritase)
  - Memaksa ritase unik per bucket
  - NULL allowed (multiple NULL OK)

---

## 4. LOGGING TABLES (Audit Trail)

### **anomaly_log**
```
Purpose: Anomaly detection & warnings
```

**Inserted by:** sp_build_ritase

**Examples:**
- ROM_BOUNCE (ROM enter dekat ROM sebelumnya)
- ROM_SPIKE (ROM dwell <3 menit)
- ROM_EXIT_AFTER_FSM (ROM exit setelah PORT/COP aktif)
- DUPLICATE_SAME_KIND (dedupe multi closing)
- PK_IGNORED_NO_TRIP (PK diabaikan)
- OVERSHIFT_INFO (trip lintas shift)

---

### **fsm_debug_log**
```
Purpose: Technical debug log
```

**Inserted by:**
- sp_build_ritase (enter/leave, actions)
- _pis_reseq_bucket (reseq start/done)

**Examples:**
- SP_ANCHOR_ENTER, SP_ANCHOR_LEAVE
- _pis_reseq_bucket (start/done)

---

### **anomaly_log_ucan**
```
Purpose: UCAN attachment failures
```

**Inserted by:** sp_ucan_attach

**Examples:**
- UCAN_ATTACH: 'No matching trip within tolerance'

---

## 5. COMPLETE DATA FLOW DIAGRAM

```
┌─────────────────┐
│  tc_events      │ ← GPS Tracker (real-time)
│  (GPS Events)   │
└────────┬────────┘
         │
         ↓ trigger/scheduler
┌────────────────────────────────────┐
│  sp_build_ritase                   │
│  ├─ SELECT tc_devices (hull)       │
│  ├─ SELECT post_integration (find) │
│  ├─ INSERT/UPDATE post_integration │
│  ├─ Dedupe (UPDATE is_suspect=1)   │
│  ├─ Merge ROM-CLOSING              │
│  ├─ Final dedupe closing           │
│  ├─ INSERT anomaly_log             │
│  ├─ INSERT fsm_debug_log           │
│  └─ CALL _pis_reseq_bucket         │
└────────┬───────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  _pis_reseq_bucket                  │
│  ├─ UPDATE post_integration         │
│  │   SET ritase=NULL (reset)        │
│  ├─ SELECT ROW_NUMBER()             │
│  └─ UPDATE post_integration         │
│      SET ritase=1,2,3,...           │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  post_integration                   │
│  (Trip with GPS data + ritase)      │
└────────┬────────────────────────────┘
         │
         ↓ scheduled batch (setiap 5-15 menit)
┌─────────────────┐
│ ucan_staging_old│ ← ITWS Timbangan
│ (UCAN data)     │
└────────┬────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  sp_ucan_attach                     │
│  ├─ SELECT ucan_staging_old (NEW)   │
│  ├─ LOOP per UCAN:                  │
│  │   ├─ SELECT fn_shift_of_local    │
│  │   ├─ SELECT post_integration     │
│  │   │   (match COP/PK/PX ±60min)   │
│  │   ├─ UPDATE post_integration     │
│  │   │   (attach UCAN, ritase=NULL) │
│  │   ├─ UPDATE ucan_staging_old     │
│  │   │   (status=ATTACHED)          │
│  │   ├─ CALL _pis_reseq_bucket      │
│  │   │   (bucket lama)              │
│  │   └─ CALL _pis_reseq_bucket      │
│  │       (bucket baru)              │
│  └─ INSERT anomaly_log_ucan (fail)  │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  post_integration                   │
│  (Final: GPS + UCAN + ritase)       │
└─────────────────────────────────────┘
```

---

## 6. ACTION SUMMARY PER TABLE

| Table | INSERT | SELECT | UPDATE | DELETE |
|-------|--------|--------|--------|--------|
| tc_events | Traccar | sp_build_ritase | - | - |
| tc_devices | Manual | sp_build_ritase | - | - |
| ucan_staging_old | ITWS | sp_ucan_attach | sp_ucan_attach | - |
| fsm_replay_stage | Manual/Batch | sp_replay (optional) | - | - |
| post_integration | sp_build_ritase | sp_build_ritase, sp_ucan_attach | sp_build_ritase, sp_ucan_attach, _pis_reseq_bucket | - |
| anomaly_log | sp_build_ritase | - | - | - |
| fsm_debug_log | sp_build_ritase, _pis_reseq_bucket | - | - | - |
| anomaly_log_ucan | sp_ucan_attach | - | - | - |

---

## 7. TIMING & FREQUENCY

| Process | Trigger | Frequency |
|---------|---------|-----------|
| tc_events → sp_build_ritase | Event-driven (real-time) | Per GPS event (~detik/menit) |
| sp_build_ritase → _pis_reseq_bucket | Otomatis (internal call) | Setiap trip dibuat/update |
| ucan_staging_old → sp_ucan_attach | Scheduled job | Batch 5-15 menit |
| sp_ucan_attach → _pis_reseq_bucket | Otomatis (internal call) | Per UCAN attach