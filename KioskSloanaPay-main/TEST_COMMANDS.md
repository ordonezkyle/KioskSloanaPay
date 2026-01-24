# Quick Test Commands

## How to Test Payment → Relay Activation

### Test 1: Generate a Test Payment and Verify Relay Activation

**Step 1: Create a payment**
```bash
# Terminal 1 - Run the app
cd c:\Users\Earl Javier\Downloads\KioskSloanaPay-main\KioskSloanaPay-main
python app.py
# This will start on http://localhost:5000
```

**Step 2: Generate QR payment**
```bash
# Terminal 2 - Create payment
curl -X POST http://localhost:5000/create_payment -H "Content-Type: application/json"

# Response will include:
# "reference": "helmet-xxxxx-xxxxx"  ← Note this reference
```

**Step 3: Trigger relay with test payment**
```bash
# Use the reference from Step 2
curl http://localhost:5000/test_payment/helmet-xxxxx-xxxxx

# You should see in Terminal 1 output:
# 🧼 STARTING HELMET SANITIZATION CYCLE
# 🔄 BRUSH ON
# ... relay activation logs ...
# ✅ SANITIZATION CYCLE COMPLETE!
```

---

### Test 2: Simulate Cash Payment (Instant Relay)

```bash
# Send cash payment - triggers relay immediately
curl -X POST http://localhost:5000/simulate_cash

# Response: Session ID
# Terminal output shows relay activating
```

---

### Test 3: Manual Mark Payment as Paid

```bash
# First create a payment to get reference
curl -X POST http://localhost:5000/create_payment

# Then mark it as paid (triggers relay)
curl -X POST http://localhost:5000/mark_paid/helmet-xxxxx-xxxxx
```

---

### Test 4: Check Payment Status

```bash
# Create payment
PAYMENT_REF=$(curl -s -X POST http://localhost:5000/create_payment | grep -oP '"reference":"?\K[^"]*')

# Check status (before payment)
curl http://localhost:5000/check_payment/$PAYMENT_REF
# Returns: {"status": "PENDING"}

# Trigger relay with test payment
curl http://localhost:5000/test_payment/$PAYMENT_REF

# Check status again (after payment)
curl http://localhost:5000/check_payment/$PAYMENT_REF
# Returns: {"status": "PAID", "session_id": 1}
```

---

### Test 5: System Health Check

```bash
# Verify GPIO and system status
curl http://localhost:5000/health

# Response:
# {
#   "status": "OK",
#   "gpio_available": true,  # or false if in simulation mode
#   "database": "connected",
#   "payment_gateway": "PayMongo QRPh",
#   "webhook_enabled": true,
#   "timestamp": "2024-01-20T..."
# }
```

---

### Test 6: View Recent Payments

```bash
# List all payments (admin login required)
curl http://localhost:5000/list_payments

# Returns array of all payments in database
```

---

## Complete Test Flow (Step by Step)

### Using PowerShell

```powershell
# 1. Start the app (keep this window open)
cd "C:\Users\Earl Javier\Downloads\KioskSloanaPay-main\KioskSloanaPay-main"
python app.py

# 2. In another PowerShell window, create a payment
$response = Invoke-WebRequest -Uri "http://localhost:5000/create_payment" -Method POST -ContentType "application/json"
$data = $response.Content | ConvertFrom-Json
$reference = $data.reference
Write-Host "Payment Reference: $reference"

# 3. Trigger relay with test payment
Invoke-WebRequest -Uri "http://localhost:5000/test_payment/$reference" -Method GET

# 4. Check payment status
Invoke-WebRequest -Uri "http://localhost:5000/check_payment/$reference" -Method GET | Select-Object -ExpandProperty Content

# 5. View in browser (see relay cycle in real-time)
Start-Process "http://localhost:5000/pay/qr"
```

---

## What to Expect in Terminal Output

### When Relay Activates Successfully:

```
==================================================
🧼 STARTING HELMET SANITIZATION CYCLE
==================================================

📍 PHASE 1: BRUSH (20 seconds)
⚡ BRUSH: ON
[20 second wait]
⚡ BRUSH: OFF

📍 PHASE 2: SOLENOID (5 seconds)
⚡ SOLENOID: ON
[5 second wait]
⚡ SOLENOID: OFF

📍 PHASE 3: BLOWER + UV (30 seconds)
⚡ BLOWER: ON
⚡ UV: ON
[30 second wait]
⚡ BLOWER: OFF
⚡ UV: OFF

✅ SANITIZATION CYCLE COMPLETE!
==================================================
```

### If GPIO Not Available (Simulation Mode):

```
💡 [SIMULATION] BRUSH: ON
💡 [SIMULATION] BRUSH: OFF
💡 [SIMULATION] SOLENOID: ON
💡 [SIMULATION] SOLENOID: OFF
💡 [SIMULATION] BLOWER: ON
💡 [SIMULATION] UV: ON
💡 [SIMULATION] BLOWER: OFF
💡 [SIMULATION] UV: OFF
```

---

## Web Browser Testing

### Test via Website UI

1. **QR Payment Test:**
   - Go to: `http://localhost:5000/pay/qr`
   - Wait for QR code to generate
   - Click "Test Payment (Dev Mode)" button
   - Watch console for relay activation logs
   - See progress bar showing 55-second sanitization

2. **Cash Payment Test:**
   - Go to: `http://localhost:5000/pay/cash`
   - Click "Simulate Cash Payment"
   - See sanitization progress

3. **Solana Payment Test:**
   - Go to: `http://localhost:5000/solana_pay`
   - Click "Test Solana Payment"
   - See relay activation

---

## Database Testing

### Check Payment Records

```bash
# Using SQLite from PowerShell
# First, install sqlite if needed: pip install sqlite3

python -c "
import sqlite3
conn = sqlite3.connect('helmet_sanitizer.db')
c = conn.cursor()

# Show recent payments
c.execute('SELECT reference, payment_method, status, created_at FROM payments ORDER BY created_at DESC LIMIT 5')
print('Recent Payments:')
for row in c.fetchall():
    print(f'  {row[0]} | {row[1]} | {row[2]} | {row[3]}')

# Show sanitization sessions
c.execute('SELECT id, started_at, completed_at FROM sanitization_sessions ORDER BY id DESC LIMIT 5')
print('\nRecent Sanitization Sessions:')
for row in c.fetchall():
    print(f'  Session {row[0]} | Started: {row[1]} | Completed: {row[2]}')

conn.close()
"
```

---

## Admin Dashboard Monitoring

Access: `http://localhost:5000/admin`
- Username: `admin`
- Password: `admin123`

From dashboard you can:
- View all payments in real-time
- See successful sanitizations
- Monitor customer ratings
- View revenue analytics
- Check payment method breakdown

---

## Expected Results

### Successful Test Indicators ✅

1. **Terminal shows relay activation logs**
2. **Payment marked as PAID in database**
3. **Sanitization session created with completion time**
4. **55-second progress bar displays in browser**
5. **Rating page appears after sanitization**

### Common Issues ❌

| Issue | Cause | Fix |
|-------|-------|-----|
| No output in console | App crashed | Check error messages, reinstall dependencies |
| Relay doesn't activate | GPIO config wrong | Check `gpiod` installation on Raspberry Pi |
| Payment stays PENDING | Webhook not received | Check PayMongo webhook URL configuration |
| Database error | File locked | Close other connections to database |
| 404 error on endpoint | Flask app crashed | Restart with `python app.py` |

---

## How to Verify Relay is Working

### If on Raspberry Pi 5:

```bash
# Test GPIO directly
sudo gpioget gpiochip4 17  # Should show 0 when OFF, 1 when ON

# Monitor in real-time while payment processes
watch -n 0.1 "sudo gpioget gpiochip4 17 27 22 23"
```

### Hardware Check:

1. Verify relay module connected to Raspberry Pi
2. Check GPIO pins match configuration (17, 27, 22, 23)
3. Test relay manually with jumpers
4. Check power supply to relay module

---

## Summary

**The system is ready to test!**

Quick test:
1. Run: `python app.py`
2. Visit: `http://localhost:5000/pay/qr`
3. Click: "Test Payment" button
4. See: Relays activate in terminal output!

That's it! The entire flow is working.
