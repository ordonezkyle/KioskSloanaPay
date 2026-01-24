# Relay Trigger Troubleshooting Guide

## Why Relays Don't Trigger - Common Issues

### 1. **Running on Windows Instead of Raspberry Pi**
- `gpiod` library only works on Linux/Raspberry Pi
- On Windows, code runs in **SIMULATION MODE**

**Check:**
```bash
# Look at startup output
python app.py
# You should see: "✅ gpiod Module Loaded" OR "⚠️ gpiod Not Available"
```

### 2. **GPIO Initialization Failed**
- Even on RPi, GPIO setup can fail silently
- May be wrong GPIO chip path, permission issues, or hardware conflicts

**Check:**
```bash
# Visit diagnostic endpoint
http://localhost:5000/gpio_status
```

### 3. **Background Thread Not Running**
- Thread might be started but not executing `trigger_sanitizer()`
- Check if thread is actually calling relay functions

## Diagnostic Endpoints

### 1. **Health Check**
```
GET http://localhost:5000/health
```
Shows basic system status including relay count and names.

### 2. **GPIO Status** (PRIMARY DIAGNOSTIC)
```
GET http://localhost:5000/gpio_status
```
Returns detailed GPIO status:
- `rpi_available`: Is hardware control enabled?
- `gpio_chip`: GPIO chip object
- `configured_pins`: What pins are configured
- `initialized_relays`: Which relays were successfully initialized
- `relay_count`: Total initialized relays
- `diagnostics`: Individual relay status for each pin
- `problems`: List of detected issues

**Example Response (Good):**
```json
{
  "rpi_available": true,
  "gpio_chip": "<gpiod.Chip object at 0x...>",
  "configured_pins": {
    "brush": 17,
    "solenoid": 27,
    "blower": 22,
    "uv": 23
  },
  "initialized_relays": ["brush", "solenoid", "blower", "uv"],
  "relay_count": 4,
  "problems": ["✅ All systems operational"]
}
```

**Example Response (Bad):**
```json
{
  "rpi_available": false,
  "gpio_chip": null,
  "initialized_relays": [],
  "problems": [
    "❌ gpiod not imported - hardware control disabled"
  ]
}
```

### 3. **Test Individual Relay**
```
GET http://localhost:5000/test_relay/{relay_name}/{state}
```

**Examples:**
```bash
# Turn ON brush relay
http://localhost:5000/test_relay/brush/1

# Turn OFF brush relay  
http://localhost:5000/test_relay/brush/0

# Turn ON solenoid
http://localhost:5000/test_relay/solenoid/1

# Turn OFF solenoid
http://localhost:5000/test_relay/solenoid/0
```

**Check the console output** - you'll see if relay command was actually sent to hardware or just simulated.

## Reading Console Output

### Startup Output

**On Raspberry Pi (GOOD):**
```
🔧 Attempting to initialize GPIO...
✅ GPIO Chip opened: <gpiod.Chip object at 0x...>
✅ GPIO Pin 17 (brush): CONFIGURED
✅ GPIO Pin 27 (solenoid): CONFIGURED
✅ GPIO Pin 22 (blower): CONFIGURED
✅ GPIO Pin 23 (uv): CONFIGURED
✅ All 4 relay pins configured successfully
✅ relay_lines dict: ['brush', 'solenoid', 'blower', 'uv']

🚀 HELMET SANITIZER KIOSK - COMPLETE FIXED VERSION
========================================================
📊 SYSTEM STATUS:
   ✅ Hardware Mode: ENABLED
   ✅ Relays: ['brush', 'solenoid', 'blower', 'uv']
   ✅ GPIO Chip: <gpiod.Chip object at 0x...>
```

**On Windows (SIMULATION MODE):**
```
⚠️ gpiod Not Available - Running in Simulation Mode

🚀 HELMET SANITIZER KIOSK - COMPLETE FIXED VERSION
========================================================
📊 SYSTEM STATUS:
   ⚠️ Hardware Mode: DISABLED (simulation only)
```

### During Relay Trigger

**When Payment Processed (Console):**
```
🔄 Creating background thread for session 1...
   RPI_AVAILABLE=True
   relay_lines initialized: 4 relays

✅ Background thread started (ID: sanitizer-1) for session 1

🧵 Background thread started for session 1

============================================================
🧼 STARTING HELMET SANITIZATION CYCLE
============================================================
   RPI_AVAILABLE: True
   Relay Lines: ['brush', 'solenoid', 'blower', 'uv']
============================================================

🔌 Turning OFF all relays...
🔍 set_relay called: relay_name='brush', state=0, status=OFF
   RPI_AVAILABLE=True, relay_lines keys=['brush', 'solenoid', 'blower', 'uv']
⚡ Sending command to GPIO: brush=0
⚡ ✅ BRUSH: OFF (HARDWARE COMMAND SENT)

📍 PHASE 1: BRUSH (20 seconds)
   Sending: BRUSH ON
🔍 set_relay called: relay_name='brush', state=1, status=ON
   RPI_AVAILABLE=True, relay_lines keys=['brush', 'solenoid', 'blower', 'uv']
⚡ Sending command to GPIO: brush=1
⚡ ✅ BRUSH: ON (HARDWARE COMMAND SENT)
   Waiting 20 seconds...
```

## Debugging Steps

### Step 1: Check System Status
```
http://localhost:5000/gpio_status
```
Look at `problems` list. If all systems operational, GPIO is ready.

### Step 2: Test Relay Manually
```
http://localhost:5000/test_relay/brush/1
```
Watch console for output. Should see:
- `⚡ Sending command to GPIO: brush=1` (if on RPi)
- `💡 [SIMULATION] BRUSH: ON` (if on Windows)

### Step 3: Check Startup Logs
Look at console when app starts. Should see all 4 relays configured.

### Step 4: Create Test Payment and Check Logs
1. Create a payment (e.g., cash payment)
2. Watch console for background thread messages
3. Check if relay commands are being sent:
   - `⚡ Sending command to GPIO` = Hardware mode
   - `💡 [SIMULATION] BRUSH: ON` = Simulation mode

## What Each Console Message Means

| Message | Meaning |
|---------|---------|
| `⚡ Sending command to GPIO` | Actual GPIO command being sent (HARDWARE) |
| `💡 [SIMULATION]` | Simulation mode - no hardware control |
| `❌ RPI_AVAILABLE=False` | gpiod not imported |
| `❌ Relay not found in relay_lines` | GPIO not initialized for that relay |
| `🔄 Background thread started` | Sanitizer running in background |
| `🧼 STARTING SANITIZATION CYCLE` | Relay sequence beginning |
| `✅ PHASE X COMPLETE` | One phase of sanitization finished |
| `✅ SANITIZATION CYCLE COMPLETE!` | All phases done, relays OFF |

## Possible Causes & Solutions

### Cause 1: Running on Windows
**Symptom:** All messages show `[SIMULATION]`, GPIO shows `rpi_available: false`

**Solution:** 
- This is expected on Windows
- Deploy to Raspberry Pi for real hardware control
- Or use relay extension board on Windows if available

### Cause 2: gpiod Library Not Installed on RPi
**Symptom:** Startup shows `⚠️ GPIO Setup Error`

**Solution:**
```bash
# Install on Raspberry Pi
sudo apt-get install libgpiod2 python3-libgpiod
pip install libgpiod python-libgpiod
```

### Cause 3: Wrong GPIO Chip Path
**Symptom:** Error during GPIO setup mentioning gpiochip

**Solution:**
Check what GPIO chips are available:
```bash
ls /dev/gpiochip*
```
Then update `gpio_chip = gpiod.Chip('gpiochip4')` if needed.

### Cause 4: GPIO Permissions Issue
**Symptom:** Permission denied when accessing GPIO

**Solution:**
```bash
# Run with sudo
sudo python app.py
# OR add user to gpio group
sudo usermod -aG gpio pi
```

### Cause 5: Pin Already in Use
**Symptom:** "Line 17 is already in use" type error

**Solution:**
```bash
# Find what's using the pin
ps aux | grep gpio
# Kill the process using it
```

## Testing Complete Flow

### 1. Start App
```bash
python app.py
```

### 2. Check GPIO Status
```bash
curl http://localhost:5000/gpio_status | python -m json.tool
```

### 3. Test Brush Relay
```bash
curl http://localhost:5000/test_relay/brush/1
# Check console - should show relay command
sleep 2
curl http://localhost:5000/test_relay/brush/0
```

### 4. Test Full Sanitization
Create a payment manually:
- QR Payment → mark as paid
- Cash Payment
- Solana Payment → confirm

Watch console for full cycle:
- Background thread created
- All 4 phases run
- All relays turn OFF at end

## Quick Diagnostic Checklist

- [ ] App starts without errors
- [ ] Check `http://localhost:5000/gpio_status` shows relay count > 0
- [ ] Test `http://localhost:5000/test_relay/brush/1` and check console
- [ ] Create test payment and verify background thread starts
- [ ] Console shows `⚡ Sending command to GPIO` (not simulation)
- [ ] All 4 phases complete: BRUSH → SOLENOID → BLOWER+UV → OFF

If all pass: ✅ **Relays should trigger on payment!**
