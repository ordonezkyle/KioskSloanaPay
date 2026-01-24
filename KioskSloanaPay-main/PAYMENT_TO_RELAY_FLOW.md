# Payment to Relay Activation Flow

## Current Integration Status: ✅ COMPLETE

The helmet sanitizer kiosk is already fully integrated to activate relays upon successful payment. Here's how it works:

---

## Payment Methods & Relay Trigger Points

### 1. **QR Code Payment (PayMongo QRPh - GCash/Maya)**
**File:** `templates/qr_payment.html`
**Endpoint:** `/check_payment/<reference>`

**Flow:**
1. Customer selects "QR Payment" on main screen
2. QR code is generated via `/create_payment` → PayMongo API
3. Customer scans and pays via GCash/Maya
4. Frontend polls `/check_payment/<reference>` every 3 seconds
5. ✅ **When payment confirmed** → Relays triggered automatically:
   - BRUSH runs for 20 seconds
   - SOLENOID runs for 5 seconds
   - BLOWER + UV run for 30 seconds (55 seconds total)
6. Customer rates service and goes home

### 2. **Cash Payment**
**File:** `templates/cash_payment.html`
**Endpoint:** `/simulate_cash`

**Flow:**
1. Customer inserts cash
2. Administrator or system clicks "Process Payment" button
3. POST request sent to `/simulate_cash`
4. ✅ **Payment processed immediately** → Relays triggered
5. Sanitization cycle begins (55 seconds)

### 3. **Solana Payment**
**File:** `templates/solana_payment.html`
**Endpoint:** `/confirm_solana_payment`

**Flow:**
1. Customer selects "Solana Pay"
2. QR code for Solana transaction generated
3. Customer scans with Solana wallet app
4. Customer confirms transaction in wallet
5. POST request to `/confirm_solana_payment`
6. ✅ **When confirmed** → Relays triggered automatically

### 4. **Manual Test Payment**
**Endpoint:** `/test_payment/<reference>`

**Flow:**
1. For testing/debugging purposes
2. Click "Test Payment" button on payment screen
3. ✅ **Instantly triggers relays** without actual payment

---

## Relay Cycle Details

**Total Duration:** 55 seconds

```
Phase 1: BRUSH (20 seconds)
├─ GPIO 17 activated
├─ Motor rotates to clean helmet
└─ Log: "🔄 BRUSH ON/OFF"

Phase 2: SOLENOID (5 seconds)
├─ GPIO 27 activated
├─ Water/liquid spray dispenser
└─ Log: "💧 SOLENOID ON/OFF"

Phase 3: BLOWER + UV (30 seconds)
├─ GPIO 22 activated (Blower)
├─ GPIO 23 activated (UV Light)
├─ Dries helmet and sterilizes
└─ Log: "🌬️ BLOWER + UV ON/OFF"
```

---

## How Relay Activation Works

### Backend (Python/Flask - `app.py`)

```python
# When payment is confirmed, this function is called:
def trigger_sanitizer():
    """
    Activates all 4-channel relay devices in sequence
    """
    set_relay("brush", 1)      # Turn ON (HIGH)
    time.sleep(20)
    set_relay("brush", 0)      # Turn OFF (LOW)
    
    set_relay("solenoid", 1)   # Turn ON
    time.sleep(5)
    set_relay("solenoid", 0)   # Turn OFF
    
    set_relay("blower", 1)     # Turn ON
    set_relay("uv", 1)         # Turn ON
    time.sleep(30)
    set_relay("blower", 0)     # Turn OFF
    set_relay("uv", 0)         # Turn OFF

# GPIO Control Function
def set_relay(relay_name, state):
    """
    state = 1 → Relay ON
    state = 0 → Relay OFF
    """
    relay_lines[relay_name].set_value(state)
```

### Frontend (JavaScript)

**QR Payment Example:**
```javascript
// In qr_payment.html, after payment confirmed:
async function handlePaymentSuccess(sessionId) {
    // Show success animation
    // Call backend to start sanitization
    // Frontend displays countdown (3 seconds)
    // Then shows sanitization progress (55 seconds)
    // Automatically navigates to rating page
}
```

---

## GPIO Pin Configuration

| Component | GPIO Pin | Channel | Function |
|-----------|----------|---------|----------|
| Brush | 17 | CH1 | Rotating brush motor |
| Solenoid | 27 | CH2 | Water/spray dispenser |
| Blower | 22 | CH3 | Air drying fan |
| UV Light | 23 | CH4 | UV sterilization |

---

## Database Tracking

When payment is successful:

1. **Payment Record Updated:**
   - Status changed from "PENDING" → "PAID"
   - `paid_at` timestamp recorded
   - Session ID created

2. **Sanitization Session Created:**
   - Records when sanitization started
   - Records when sanitization completed
   - Links to payment record

3. **Daily Statistics Updated:**
   - Total payments count
   - Total revenue
   - Successful sanitizations count
   - Average rating

---

## Verification & Monitoring

### Test Endpoints

1. **Test Quick Payment:**
   ```
   GET /test_payment/<reference>
   ```
   Instantly triggers relay without payment

2. **Check Payment Status:**
   ```
   GET /check_payment/<reference>
   ```
   Returns: PENDING, PAID, NOT_FOUND

3. **Manual Mark Paid:**
   ```
   POST /mark_paid/<reference>
   ```
   Manually mark payment as paid and trigger relay

4. **Health Check:**
   ```
   GET /health
   ```
   Returns GPIO status and system info

---

## Frontend User Experience Flow

```
Main Menu
    ↓
Select Payment Method (QR / Cash / Solana)
    ↓
[QR Mode]              [Cash Mode]            [Solana Mode]
Generate QR    →  OR  →  Insert Cash    →  OR  →  Show QR Code
Display QR              Verify Amount         Scan with Wallet
Customer Scans          Click Process          Confirm Payment
    ↓                      ↓                        ↓
Poll Payment Status → Poll Payment Status → POST /confirm_solana_payment
    ↓                      ↓                        ↓
Payment Confirmed ← Relays Activated → Relays Activated
    ↓
Show Success Message (3s countdown)
    ↓
Sanitization Progress (55s animation)
    ↓
Sanitization Complete
    ↓
Rate Service (1-5 stars)
    ↓
Thank You / Return Home
```

---

## Troubleshooting

### Relay Not Triggering?

1. **Check GPIO Status:**
   ```
   GET /health
   ```
   Should show: `"gpio_available": true`

2. **Check Payment Status:**
   ```
   GET /check_payment/<reference>
   ```
   Should show: `"status": "PAID"`

3. **Check Database:**
   Look at `helmet_sanitizer.db` → `payments` table
   Verify `status` = 'PAID'

4. **Check Logs:**
   Look for message: `"🧼 Triggering sanitizer for session [ID]"`

### Common Issues

| Issue | Solution |
|-------|----------|
| Relay not moving | Check GPIO pin connections |
| Payment not confirmed | Check webhook URL in PayMongo dashboard |
| Polling stuck | Refresh page or try test payment |
| GPIO not available | Running in simulation mode (for testing) |

---

## Admin Dashboard

Access at: `http://localhost:5000/admin`

View:
- Real-time payment history
- Sanitization statistics
- Customer ratings
- Revenue analytics
- Payment method breakdown

---

## Summary

✅ **The system is already fully functional!**

When a customer:
1. Makes a successful payment (QR, Cash, or Solana)
2. System receives payment confirmation
3. **Relays activate automatically**
4. Sanitization cycle runs (55 seconds)
5. Customer gets confirmation and rating prompt

No additional configuration needed - everything is integrated!
