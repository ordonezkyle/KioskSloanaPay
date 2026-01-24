# Quick Integration Guide - Payment to Relay

## ✅ GOOD NEWS: Everything is Already Integrated!

Your kiosk is **ready to go**. When a customer makes a payment, the relay automatically activates. Here's exactly what happens:

---

## 3-Step Process

### Step 1: Customer Makes Payment
**Where:** Any payment screen (QR, Cash, or Solana)

```
Customer selects payment method
         ↓
System generates QR code OR waits for cash/wallet confirmation
         ↓
Customer completes payment
```

### Step 2: Payment Confirmation Received
**What happens in backend:**

```python
# File: app.py
# Line 615 (QR Payment webhook):
update_payment_status(reference, "PAID")  # Mark as PAID
session_id = save_sanitization_session(payment["id"])

# Line 885 - THIS IS THE TRIGGER:
trigger_sanitizer()  # 🎯 RELAYS ACTIVATE HERE!
```

### Step 3: Relays Activate Automatically

```
Relay Sequence Activates:
├─ PHASE 1: BRUSH (GPIO 17) → 20 seconds
├─ PHASE 2: SOLENOID (GPIO 27) → 5 seconds  
└─ PHASE 3: BLOWER (GPIO 22) + UV (GPIO 23) → 30 seconds

Total: 55 seconds of sanitization
```

---

## Payment Methods (All Trigger Relays)

### 1️⃣ QR Payment (GCash/Maya)
**URL:** `http://localhost:5000/pay/qr`
```
Customer scans QR
     ↓
Pays via GCash/Maya
     ↓
PayMongo sends webhook
     ↓
App verifies payment
     ↓
✅ RELAYS TRIGGERED
```
**Backend Code:** `app.py` line 885 in `process_webhook_payment()`

---

### 2️⃣ Cash Payment
**URL:** `http://localhost:5000/pay/cash`
```
Customer inserts cash
     ↓
Admin clicks "Process"
     ↓
POST /simulate_cash
     ↓
✅ RELAYS TRIGGERED IMMEDIATELY
```
**Backend Code:** `app.py` line 924 in `simulate_cash()`

---

### 3️⃣ Solana Payment
**URL:** `http://localhost:5000/solana_pay`
```
Customer scans QR with Solana wallet
     ↓
Confirms in wallet
     ↓
POST /confirm_solana_payment
     ↓
✅ RELAYS TRIGGERED
```
**Backend Code:** `app.py` line 1054 in `confirm_solana_payment()`

---

## Visual Timeline

```
User Interface Flow:
│
├─ Main Screen
│  ├─ [QR Payment] ──────────────────────┐
│  ├─ [Cash Payment] ─────────────────┐  │
│  └─ [Solana Pay] ────────────────┐  │  │
│                                   │  │  │
│  Browser-side Polling              │  │  │
│  (checks every 3 seconds)          │  │  │
│     ↓                              │  │  │
│  "Payment Confirmed" ← ← ← ← ← ← ←┘  │  │
│     ↓                              │  │
│  Backend Relay Sequence ← ← ← ← ← ←┘  │
│     ↓                              │
│  "Processing..." ← ← ← ← ← ← ← ← ←┘
│     ↓
└─ Rating Screen


Backend Processing:
│
├─ Payment received (webhook or API call)
│
├─ Verify payment amount & reference
│
├─ UPDATE database: status = "PAID"
│
├─ 🎯 CALL trigger_sanitizer()  ←─── THIS IS WHERE RELAYS ACTIVATE
│     ├─ set_relay("brush", 1)  → GPIO 17 ON
│     ├─ sleep(20)
│     ├─ set_relay("brush", 0)  → GPIO 17 OFF
│     ├─ set_relay("solenoid", 1)  → GPIO 27 ON
│     ├─ sleep(5)
│     ├─ set_relay("solenoid", 0)  → GPIO 27 OFF
│     ├─ set_relay("blower", 1)  → GPIO 22 ON
│     ├─ set_relay("uv", 1)  → GPIO 23 ON
│     ├─ sleep(30)
│     ├─ set_relay("blower", 0)  → GPIO 22 OFF
│     └─ set_relay("uv", 0)  → GPIO 23 OFF
│
├─ UPDATE database: sanitization_sessions → completed_at = now()
│
├─ Log stats to daily_stats table
│
└─ Return success to frontend
```

---

## Key Code Sections

### Relay Activation (Line 332 - app.py)
```python
def trigger_sanitizer():
    """This function runs when payment is confirmed"""
    print("🧼 STARTING HELMET SANITIZATION CYCLE")
    
    # PHASE 1: BRUSH
    set_relay("brush", 1)
    print("🔄 BRUSH ON")
    time.sleep(20)
    set_relay("brush", 0)
    print("🔄 BRUSH OFF")
    
    # PHASE 2: SOLENOID
    set_relay("solenoid", 1)
    print("💧 SOLENOID ON")
    time.sleep(5)
    set_relay("solenoid", 0)
    print("💧 SOLENOID OFF")
    
    # PHASE 3: BLOWER + UV
    set_relay("blower", 1)
    set_relay("uv", 1)
    print("🌬️ BLOWER + UV ON")
    time.sleep(30)
    set_relay("blower", 0)
    set_relay("uv", 0)
    print("🌬️ BLOWER + UV OFF")
    
    print("✅ SANITIZATION CYCLE COMPLETE!")
```

### Payment Verification (Line 599 - app.py)
```python
def mark_payment_as_paid(ref, payment_db_id):
    """This is called by all payment methods"""
    update_payment_status(ref, "PAID")
    session_id = save_sanitization_session(payment_db_id)
    
    print(f"🧼 Triggering sanitizer for session {session_id}")
    
    # 🎯 HERE IS WHERE RELAYS GET TRIGGERED:
    trigger_sanitizer()  # ← All 4 relays activate here!
    
    complete_sanitization_session(session_id)
    update_daily_stats()
    
    return jsonify({"status": "PAID", "session_id": session_id})
```

---

## All Payment Entry Points that Trigger Relays

| Location | Line | Method | URL Endpoint |
|----------|------|--------|--------------|
| QR Webhook | 885 | `process_webhook_payment()` | `POST /paymongo_webhook` |
| Cash | 924 | `simulate_cash()` | `POST /simulate_cash` |
| Solana | 1054 | `confirm_solana_payment()` | `POST /confirm_solana_payment` |
| Test | 1137 | `test_payment()` | `GET /test_payment/<ref>` |
| Manual | 1175 | `mark_paid()` | `POST /mark_paid/<ref>` |
| Webhook Sim | 1210 | `payment_paid()` | `POST /payment_paid` |

**All of these call `trigger_sanitizer()` which activates the relays!**

---

## Testing It

### Test 1: Instant Test (No Payment Needed)
```
1. Go to: http://localhost:5000/pay/qr
2. Click "Test Payment" button
3. Watch relays activate in console output
```

### Test 2: Manual Payment Simulation
```bash
# Mark a payment as paid and trigger relays
curl -X POST http://localhost:5000/mark_paid/<reference>
```

### Test 3: Direct Relay Test
```bash
# Directly call test endpoint
curl http://localhost:5000/test_payment/<reference>
```

---

## Monitoring

### Check What's Happening
1. **Console Output** - Watch backend logs in terminal
   ```
   🧼 STARTING HELMET SANITIZATION CYCLE
   🔄 BRUSH ON
   🔄 BRUSH OFF
   💧 SOLENOID ON
   💧 SOLENOID OFF
   🌬️ BLOWER + UV ON
   🌬️ BLOWER + UV OFF
   ✅ SANITIZATION CYCLE COMPLETE!
   ```

2. **Database** - Check `helmet_sanitizer.db`
   ```
   SELECT * FROM payments WHERE status = 'PAID';
   SELECT * FROM sanitization_sessions;
   ```

3. **Admin Dashboard**
   ```
   http://localhost:5000/admin
   (Username: admin, Password: admin123)
   ```

---

## Troubleshooting

### Problem: Relays not activating?

**Check 1: Is payment getting marked as PAID?**
```
GET /check_payment/<reference>
Should return: {"status": "PAID"}
```

**Check 2: Is backend receiving the payment?**
Look for log: `"✅ Payment {reference} processed successfully"`

**Check 3: Are GPIO pins configured correctly?**
```
GET /health
Should show: "gpio_available": true
```

**Check 4: Running in simulation mode?**
If `"gpio_available": false` - you're in simulation mode (no real relays)
Still works for testing! Just prints to console instead.

---

## Summary

✅ **Everything is integrated!**

The flow is:
1. Customer pays → 
2. Payment confirmed → 
3. `trigger_sanitizer()` called → 
4. All 4 relays activate in sequence → 
5. Sanitization completes → 
6. Customer rates service

**No additional code needed!** Just run the app and test payments will activate relays automatically.
