# Relay Activation Function - Implementation Summary

## New Function: `activate_relays_on_payment()`

A dedicated function has been added to handle relay activation when payment is confirmed.

### Function Signature
```python
def activate_relays_on_payment(session_id, payment_ref):
    """Activate relays after payment is confirmed.
    
    This function is called when a payment is confirmed to:
    1. Start the sanitizer in a background thread
    2. Log the activation event
    3. Create session record
    
    Args:
        session_id: The sanitization session ID
        payment_ref: The payment reference
    """
```

### What It Does

1. **Logs payment confirmation** - Shows payment details with timestamp
2. **Checks hardware availability** - Displays if running in hardware mode or simulation
3. **Lists available relays** - Shows which relay channels are initialized
4. **Starts background thread** - Calls `trigger_sanitizer_background()` to run sanitizer
5. **Returns status** - Provides success/error response with details

### Return Value

**Success Response:**
```python
{
    "success": True,
    "session_id": 1,
    "payment_ref": "helmet-1705859400-abc123",
    "hardware_mode": True,  # True = hardware control, False = simulation
    "timestamp": "2026-01-21T10:30:45.123456",
    "message": "Relays activated after payment confirmation"
}
```

**Error Response:**
```python
{
    "success": False,
    "error": "Error message details",
    "session_id": 1,
    "timestamp": "2026-01-21T10:30:45.123456"
}
```

### Console Output Example

```
============================================================
💳 PAYMENT CONFIRMED - ACTIVATING RELAYS
============================================================
Payment Reference: helmet-1705859400-abc123
Session ID: 1
Timestamp: 2026-01-21T10:30:45.123456
============================================================
✅ Hardware mode enabled - relays will physically activate
✅ Available relays: ['brush', 'solenoid', 'blower', 'uv']

🚀 Starting sanitizer background thread...
🔄 Creating background thread for session 1...
   RPI_AVAILABLE=True
   relay_lines initialized: 4 relays

✅ Background thread started (ID: sanitizer-1) for session 1
✅ Relay activation initiated for session 1
```

## Integration Points

The function is now called in **all payment confirmation scenarios**:

### 1. **QRPh Webhook Processing**
- Location: `process_webhook_payment()` function
- When: PayMongo sends webhook for successful payment
- Example: Payment received via GCash/Maya

### 2. **Cash Payment**
- Location: `simulate_cash()` function
- When: Cash payment is confirmed
- Example: Manual cash payment entry

### 3. **Solana Payment Confirmation**
- Location: `confirm_solana_payment()` function
- When: Solana blockchain payment confirmed
- Example: Blockchain transaction successful

### 4. **Test Payment Endpoint**
- Location: `test_payment()` function
- When: Test endpoint marks payment as paid
- Example: Testing relay functionality

### 5. **Manual Payment Marking**
- Location: `mark_paid()` function
- When: Admin manually marks payment as paid
- Example: `POST /mark_paid/<reference>`

### 6. **Payment Confirmation Function**
- Location: `mark_payment_as_paid()` function
- When: Internal payment confirmation logic
- Example: Direct function call from anywhere

## Flow Diagram

```
Payment Confirmed
    ↓
activate_relays_on_payment()
    ↓
    ├─ Check if hardware available
    ├─ Log payment details
    ├─ trigger_sanitizer_background()
    │   ├─ Create background thread
    │   └─ Start relay cycle in thread
    ├─ Complete sanitization session
    └─ Return status (success/error)
```

## Key Features

### ✅ Comprehensive Logging
- Payment reference and session ID
- Timestamp for audit trail
- Hardware mode confirmation
- Available relay information

### ✅ Error Handling
- Try-catch wrapping
- Full traceback on errors
- Graceful failure reporting
- Session tracking on failure

### ✅ Hardware Detection
- Warns if running in simulation mode
- Shows available relays
- Confirms hardware initialization

### ✅ Background Execution
- Non-blocking relay activation
- Returns immediately to client
- Relays execute asynchronously

## Testing

### Test 1: Manual Relay Activation
```bash
# Turn on brush relay
curl http://localhost:5000/test_relay/brush/1

# Check console output for:
# ✅ Hardware mode enabled - relays will physically activate
# ⚡ Sending command to GPIO: brush=1
```

### Test 2: Payment Confirmation
1. Create a payment
2. Check `/health` endpoint for relay status
3. Confirm payment
4. Watch console for relay activation messages

### Test 3: Check Session Creation
```bash
# After payment, check database:
SELECT * FROM sanitization_sessions WHERE id = 1;
```

## Console Indicators

| Indicator | Meaning |
|-----------|---------|
| ✅ | Success/confirmation |
| 🧵 | Thread operation |
| 💳 | Payment event |
| ⚡ | GPIO command |
| 🚀 | Startup/launch |
| ❌ | Error |
| ⚠️ | Warning/caution |
| 🔄 | Background/async operation |

## Environment Variables

No new environment variables needed. Uses existing configuration:
- `RPI_AVAILABLE` - Automatically detected
- `relay_lines` - Populated at startup
- `GPIO_PINS` - Configured pins

## Backward Compatibility

✅ All existing payment flows still work
✅ No breaking changes
✅ Optional: Can still use `trigger_sanitizer_background()` directly if needed
✅ Session and database tracking unchanged

## Future Enhancements

Possible additions:
- Relay timing configuration per phase
- Custom relay sequences
- Retry logic for failed relays
- Email notifications on relay activation
- Relay state validation before/after
- Relay temperature monitoring
