# Android Dynamic Analysis Guide
## Using mitmweb + ADB Proxy + Frida + Objection (A–Z)

> ⚠️ **Legal & Ethical Notice**
>
> The techniques in this guide are intended **only for security testing, reverse engineering, or debugging applications you own or have explicit permission to test**. Intercepting or modifying traffic of third‑party apps without authorization may violate laws or terms of service.

---

# Table of Contents

1. Overview
2. Prerequisites
3. Environment Setup
4. Installing mitmproxy
5. Starting mitmweb Proxy
6. Configuring Android Emulator Proxy via ADB
7. Installing mitmproxy Certificate on Emulator
8. Verifying Traffic Interception
9. Installing Frida
10. Running Frida Server on Android
11. Using Frida Tools for Runtime Instrumentation
12. Installing Objection
13. Disabling SSL Pinning with Objection
14. Typical Workflow
15. Troubleshooting
16. Cleanup

---

# 1. Overview

This workflow is commonly used in **mobile application security testing**.

Tool | Purpose
---|---
mitmproxy / mitmweb | HTTP/HTTPS traffic interception
ADB | Android debugging and device communication
Frida | Dynamic instrumentation toolkit
Objection | Runtime mobile exploration toolkit built on Frida

**High level flow**

```
Android App
     ↓
Android Emulator Proxy
     ↓
mitmweb (intercept traffic)
     ↓
Frida Injection
     ↓
Objection (disable SSL pinning)
```

---

# 2. Prerequisites

System requirements:

- Linux / macOS / Windows
- Python 3.8+
- Android Emulator or rooted device
- Android SDK (ADB installed)
- Internet connection

Tools required:

```
mitmproxy
frida-tools
objection
adb
```

Verify ADB:

```bash
adb version
```

---

# 3. Environment Setup

Create a Python virtual environment (recommended):

```bash
python3 -m venv mobile-lab
source mobile-lab/bin/activate
```

Upgrade pip:

```bash
pip install --upgrade pip
```

---

# 4. Installing mitmproxy

Install mitmproxy:

```bash
pip install mitmproxy
```

Verify installation:

```bash
mitmproxy --version
```

mitmproxy provides three interfaces:

Command | Description
---|---
mitmproxy | Terminal UI
mitmdump | CLI tool
mitmweb | Web interface

---

# 5. Starting mitmweb Proxy

Start the proxy:

```bash
mitmweb --listen-port 8080
```

Default interface:

```
http://127.0.0.1:8081
```

Proxy address:

```
127.0.0.1:8080
```

Open the web UI in browser:

```
http://127.0.0.1:8081
```

---

# 6. Configuring Android Emulator Proxy via ADB

First ensure emulator is running:

```bash
adb devices
```

Example output:

```
emulator-5554 device
```

Set global HTTP proxy:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
```

Explanation:

```
10.0.2.2 = host machine from emulator
8080    = mitmproxy port
```

Verify proxy setting:

```bash
adb shell settings get global http_proxy
```

Expected:

```
10.0.2.2:8080
```

---

# 7. Installing mitmproxy Certificate on Emulator

Intercepting HTTPS requires installing the mitmproxy CA certificate.

Open in emulator browser:

```
http://mitm.it
```

Download:

```
Android Certificate
```

Install certificate:

```
Settings → Security → Encryption & Credentials
Install certificate → CA Certificate
```

Name it:

```
mitmproxy
```

---

# 8. Verifying Traffic Interception

Open an app or website inside emulator.

Example:

```
https://example.com
```

In **mitmweb UI** you should see:

```
HTTP requests
HTTPS requests
Headers
Response data
```

If requests appear → proxy works.

---

# 9. Installing Frida

Install Frida tools:

```bash
pip install frida-tools
```

Verify:

```bash
frida --version
```

---

# 10. Running Frida Server on Android

Download matching frida-server:

```
https://github.com/frida/frida/releases
```

Steps:

### Push to device

```bash
adb push frida-server /data/local/tmp/
```

### Make executable

```bash
adb shell chmod 755 /data/local/tmp/frida-server
```

### Start server in emulator with root

```bash
adb shell
su
/data/local/tmp/frida-server &
```

Verify connection:

```bash
frida-ps -U
```

If working you will see running apps.

---

# 11. Using Frida Tools for Runtime Instrumentation

List installed applications:

```bash
frida-ps -Uai
```

Attach to running app:

```bash
*Recommended*: frida -U -p 1234 [PID you get when app is running on emulator and you run the frida-ps -Uai command]
frida -U -n com.example.app
```

Spawn and inject script:

```bash
frida -U -f com.example.app -l script.js --no-pause
```

Example simple Frida script:

```javascript
Java.perform(function() {
    console.log("Frida injected successfully");
});
```

---

# 12. Installing Objection

Install Objection:

```bash
pip install objection
```

Verify:

```bash
objection --help
```

---

# 13. Disabling SSL Pinning with Objection

Start Objection by attaching to the app.

Example:

```bash
objection -n com.example.app explore start
```

Inside the Objection shell run:

```
android sslpinning disable
```

This patches many common pinning implementations at runtime.

If successful you should see:

```
SSL pinning bypassed
```

Now the app's HTTPS traffic should appear inside **mitmweb**.

---

# 14. Typical Workflow

Complete workflow example:

### 1 Start proxy

```
mitmweb --listen-port 8080
```

### 2 Start emulator

```
adb devices
```

### 3 Configure proxy

```
adb shell settings put global http_proxy 10.0.2.2:8080
```

### 4 Start Frida server

```
adb shell /data/local/tmp/frida-server &
```

### 5 Launch Objection

```
objection -g com.target.app explore
```

### 6 Disable pinning

```
android sslpinning disable
```

### 7 Monitor traffic

Open:

```
http://127.0.0.1:8081
```

Observe intercepted API calls.

---

# 15. Troubleshooting

## Emulator cannot connect to proxy

Check:

```
10.0.2.2 instead of localhost
```

---

## Frida cannot detect device

Run:

```
adb devices
```

Restart ADB:

```
adb kill-server
adb start-server
```

---

## SSL traffic not appearing

Possible reasons:

- Certificate not installed
- App uses certificate pinning
- Proxy not configured
- TLS inspection blocked

Use:

```
android sslpinning disable
```

---

## Frida version mismatch

Ensure **frida-tools version = frida-server version**

Check:

```
frida --version
```

---

# 16. Cleanup

Remove proxy:

```bash
adb shell settings put global http_proxy :0
```

Kill Frida server:

```bash
adb shell pkill frida-server
```

Deactivate Python environment:

```bash
deactivate
```

---

# Conclusion

This setup enables **dynamic mobile application analysis**, allowing you to:

- Intercept API traffic
- Analyze HTTPS requests
- Instrument apps at runtime
- Test SSL pinning implementations

Common use cases include:

- Mobile penetration testing
- API debugging
- Reverse engineering
- Security research

---
