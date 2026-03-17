# Android Emulator Debugger Agent

## Purpose

Build, run, and debug Android projects on emulators using Gradle and ADB.

Provides a comprehensive workflow for building Android apps, launching them on emulators, interacting with the UI, capturing logs, and diagnosing runtime behaviour. Handles emulator discovery, app installation, and log filtering.

## Use When

- You need to build and run an Android app on an emulator
- You want to interact with the app UI programmatically (tap, type, swipe)
- You need to capture and filter logcat output to diagnose a bug
- You want to take screenshots or inspect the UI hierarchy
- You need to reproduce a crash or runtime error

---

## Step 1 — Discover Available Emulators

```bash
# List all AVDs defined on this machine
emulator -list-avds

# List currently running emulators / connected devices
adb devices

# Example output:
# List of devices attached
# emulator-5554   device
# emulator-5556   device
```

If no emulator is running, start one:

```bash
# Start a named AVD (replace with your AVD name from -list-avds)
emulator -avd Pixel_8_API_35 &

# Wait for it to fully boot
adb wait-for-device
adb shell getprop sys.boot_completed   # returns "1" when ready
```

For multiple emulators, target a specific one by prefixing commands with `-s <serial>`:

```bash
adb -s emulator-5554 install app.apk
```

---

## Step 2 — Build and Install

### Debug build

```bash
# Assemble debug APK
./gradlew assembleDebug

# Install on the connected/running emulator
adb install -r app/build/outputs/apk/debug/app-debug.apk

# Or use the Gradle task which builds + installs in one step
./gradlew installDebug
```

### Specific variant or flavour

```bash
# Replace 'staging' with your build flavour
./gradlew installStagingDebug
```

### Check installed packages

```bash
adb shell pm list packages | grep <your.package.name>
```

---

## Step 3 — Launch the App

```bash
# Launch the main activity
adb shell monkey -p <your.package.name> -c android.intent.category.LAUNCHER 1

# Or use am start with explicit component
adb shell am start -n <your.package.name>/<your.package.name>.MainActivity

# Example:
adb shell am start -n com.example.myapp/com.example.myapp.MainActivity
```

---

## Step 4 — Capture Logs

Always clear logcat before reproducing a bug to avoid noise from previous sessions.

```bash
# Clear existing log buffer
adb logcat -c

# Stream all logs (very noisy — apply a filter)
adb logcat

# Filter by your app's package (API 31+)
adb logcat --pid=$(adb shell pidof -s <your.package.name>)

# Filter by tag
adb logcat -s MyTag:D

# Filter by priority (V=Verbose, D=Debug, I=Info, W=Warn, E=Error)
adb logcat *:E   # errors only

# Save logs to a file
adb logcat > logcat.txt

# Useful combined filter: errors + your app's tag
adb logcat MyApp:D *:E
```

### Finding crash stack traces

```bash
# Show only fatal crashes
adb logcat -s AndroidRuntime:E

# Pipe through grep for your package
adb logcat | grep -A 20 "FATAL EXCEPTION"
```

---

## Step 5 — UI Interaction

Use `adb shell input` to interact with the emulator UI without touching it manually.

```bash
# Tap at screen coordinates (x y)
adb shell input tap 540 1200

# Long press
adb shell input swipe 540 1200 540 1200 1000   # same start/end = long press, duration ms

# Swipe (scroll down)
adb shell input swipe 540 1400 540 400 300

# Type text into focused field
adb shell input text "hello@example.com"

# Key events
adb shell input keyevent KEYCODE_BACK
adb shell input keyevent KEYCODE_HOME
adb shell input keyevent KEYCODE_ENTER
adb shell input keyevent KEYCODE_DEL        # backspace
```

### Common keycodes

| Action | Keycode |
|--------|---------|
| Back | `KEYCODE_BACK` |
| Home | `KEYCODE_HOME` |
| Recents | `KEYCODE_APP_SWITCH` |
| Enter / confirm | `KEYCODE_ENTER` |
| Volume up | `KEYCODE_VOLUME_UP` |

---

## Step 6 — Screenshots and UI Inspection

```bash
# Take a screenshot and pull to local machine
adb shell screencap /sdcard/screenshot.png
adb pull /sdcard/screenshot.png ./screenshot.png

# Record screen (stops on Ctrl+C)
adb shell screenrecord /sdcard/recording.mp4
adb pull /sdcard/recording.mp4 ./recording.mp4

# Dump the UI hierarchy (XML snapshot of all views)
adb shell uiautomator dump /sdcard/ui.xml
adb pull /sdcard/ui.xml ./ui.xml
cat ./ui.xml   # inspect bounds and resource-ids for tap targeting
```

Use the UI dump to find exact coordinates or resource IDs when you're not sure where to tap:

```xml
<!-- Example node from ui.xml -->
<node resource-id="com.example.myapp:id/btn_submit"
      bounds="[100,500][980,620]"
      clickable="true" />
<!-- Centre tap = x:(100+980)/2=540, y:(500+620)/2=560 -->
```

---

## Step 7 — Common Debugging Scenarios

### App crashes on launch

```bash
adb logcat -c
adb shell am start -n <package>/<activity>
adb logcat -s AndroidRuntime:E
# Look for the exception type and stack trace
```

### Network requests failing

```bash
# Check if the emulator has network access
adb shell ping -c 3 google.com

# Proxy traffic through Charles / mitmproxy
adb shell settings put global http_proxy <your-machine-ip>:8888
# To remove proxy:
adb shell settings delete global http_proxy
```

### Database inspection

```bash
# Pull the app's database (debug builds only)
adb shell run-as <your.package.name> cp /data/data/<your.package.name>/databases/app.db /sdcard/
adb pull /sdcard/app.db ./app.db
# Open with DB Browser for SQLite or Android Studio's Database Inspector
```

### SharedPreferences / DataStore inspection

```bash
adb shell run-as <your.package.name> cat /data/data/<your.package.name>/shared_prefs/<prefs-name>.xml
```

---

## Step 8 — Clean Up

```bash
# Uninstall the app
adb uninstall <your.package.name>

# Kill a running app
adb shell am force-stop <your.package.name>

# Reboot the emulator
adb reboot
```

---

## Checklist

- [ ] Emulator booted and `adb devices` shows `device` (not `offline`)
- [ ] Logcat cleared before reproducing the issue
- [ ] App installed with `installDebug` or `adb install -r`
- [ ] Stack trace captured and saved if a crash occurred
- [ ] Screenshot or screen recording taken to document the issue
- [ ] Network proxy removed after debugging session if it was set
