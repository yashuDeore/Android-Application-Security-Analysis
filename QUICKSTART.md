# Quick Start Guide

## Getting Started with Android Security Analysis

This guide will help you set up and start using this Android security analysis project.

---

## Prerequisites

Before you begin, ensure you have:

- [ ] A computer running Linux, macOS, or Windows (with WSL)
- [ ] Basic understanding of Android applications
- [ ] Familiarity with command-line tools
- [ ] (Optional) A rooted Android device or emulator

---

## Installation

### Step 1: Clone the Repository

```bash
git clone https://github.com/yourusername/android-security-analysis.git
cd android-security-analysis
```

### Step 2: Run the Setup Script

```bash
cd tools
chmod +x setup-environment.sh
./setup-environment.sh
```

This will install:
- JADX (APK decompiler)
- APKTool (resource extractor)
- Python dependencies
- ADB tools

### Step 3: Verify Installation

```bash
# Check JADX
jadx --version

# Check APKTool
apktool --version

# Check ADB
adb version

# Check Python packages
pip3 list | grep frida
```

---

## Basic Usage

### 1. Static Analysis

Analyze an APK file for hardcoded credentials and misconfigurations:

```bash
# Decompile APK
jadx -d output/myapp target-app.apk

# Extract resources
apktool d target-app.apk -o extracted/

# Search for secrets
grep -r "api_key\|password\|secret" output/myapp/
```

### 2. Dynamic Analysis

Extract and analyze runtime data:

```bash
# Root the emulator (if needed)
adb root
adb remount

# Extract app data
python3 tools/data-extraction.py com.example.app

# View the analysis report
cat extracted_data/com.example.app/security_analysis_report.json
```

### 3. Network Traffic Analysis

Intercept and analyze HTTP/HTTPS traffic:

```bash
# Setup Burp Suite proxy (first install Burp Suite)
cd tools
./setup-proxy.sh 192.168.1.100 8080

# Bypass SSL pinning with Frida
frida -U -f com.example.app -l tools/certificate-pinning-bypass.js --no-pause
```

---

## Example Workflow

Here's a complete analysis workflow:

### Step 1: Obtain the APK

```bash
# Option A: Download from device
adb shell pm path com.example.app
adb pull /data/app/com.example.app-xxx/base.apk

# Option B: Use a test APK
# Download InsecureBankv2 or DIVA for practice
```

### Step 2: Static Analysis

```bash
# Decompile
jadx -d output/analysis target.apk

# Check AndroidManifest.xml
cat output/analysis/resources/AndroidManifest.xml

# Search for vulnerabilities
grep -ri "hardcoded\|password\|secret" output/analysis/
```

### Step 3: Install and Run on Device

```bash
# Install APK
adb install target.apk

# Launch the app
adb shell monkey -p com.example.app 1
```

### Step 4: Dynamic Analysis

```bash
# Start capturing logs
adb logcat | grep -i "password\|token" > logs.txt

# Extract app data
python3 tools/data-extraction.py com.example.app

# Inspect databases
sqlite3 extracted_data/com.example.app/databases/app.db
```

### Step 5: Network Analysis

```bash
# Start Burp Suite
# Configure device proxy
adb shell settings put global http_proxy 192.168.1.100:8080

# Bypass SSL pinning if needed
frida -U -f com.example.app -l tools/certificate-pinning-bypass.js

# Capture and analyze traffic in Burp Suite
```

### Step 6: Document Findings

Use the templates in `/reports/` to document your findings:

- `static-analysis.md` - Code and configuration issues
- `dynamic-analysis.md` - Runtime behavior
- `network-analysis.md` - Traffic interception findings
- `final-report.md` - Comprehensive summary

---

## Practice Applications

Test your skills with these intentionally vulnerable Android apps:

1. **DIVA (Damn Insecure and Vulnerable App)**
   - Download: https://github.com/payatu/diva-android
   - Great for beginners

2. **InsecureBankv2**
   - Download: https://github.com/dineshshetty/Android-InsecureBankv2
   - Banking app with multiple vulnerabilities

3. **OWASP GoatDroid**
   - Download: https://github.com/jackMannino/OWASP-GoatDroid-Project
   - Comprehensive vulnerable app

4. **AndroGoat**
   - Download: https://github.com/satishpatnayak/AndroGoat
   - Modern vulnerable app

---

## Common Tasks

### Find Hardcoded API Keys

```bash
jadx -d output app.apk
grep -r "api.*key\|API.*KEY" output/ --include="*.java"
```

### Check for Insecure Data Storage

```bash
# Pull SharedPreferences
adb pull /data/data/com.example.app/shared_prefs/

# Check databases
adb pull /data/data/com.example.app/databases/
sqlite3 app.db "SELECT * FROM users;"
```

### Test for SQL Injection

```python
# In Burp Suite Repeater, try:
username=admin' OR '1'='1
password=anything
```

### Check Certificate Pinning

```bash
# Try to intercept HTTPS without bypassing
# If you see traffic = No pinning
# If you don't see traffic = Pinning implemented

# Bypass with Frida
frida -U -f com.example.app -l certificate-pinning-bypass.js
```

---

## Troubleshooting

### Issue: "adb: device unauthorized"

**Solution:**
```bash
# On device: Accept the RSA key fingerprint prompt
# Then:
adb kill-server
adb start-server
adb devices
```

### Issue: "Cannot pull /data/data - Permission denied"

**Solution:**
```bash
# Root is required
adb root
adb remount
# Then try again
```

### Issue: "JADX decompilation failed"

**Solution:**
```bash
# Try with different options
jadx --no-res --no-debug-info -d output app.apk

# Or use online decompiler:
# http://www.javadecompilers.com/apk
```

### Issue: "Frida server not running"

**Solution:**
```bash
# Download Frida server for your device architecture
# adb shell getprop ro.product.cpu.abi

# Push and run Frida server
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"
```

---

## Learning Resources

### Beginner
- OWASP Mobile Security Testing Guide
- Android Security Documentation
- YouTube: "Android App Security Testing" tutorials

### Intermediate
- Mobile Security Framework (MobSF)
- Frida scripting tutorials
- Burp Suite Academy - Mobile Testing

### Advanced
- Android Internals book
- Custom Frida scripts
- Binary exploitation

---

## Tips for Success

1. **Always get authorization** before testing any app
2. **Start with practice apps** before testing real applications
3. **Document everything** - screenshots, commands, findings
4. **Learn gradually** - master one technique at a time
5. **Join communities** - Reddit r/AskNetsec, HackerOne, BugCrowd
6. **Keep learning** - Android security evolves rapidly

---

## Next Steps

After completing this guide:

1. ✅ Test 2-3 practice vulnerable apps
2. ✅ Write detailed reports for each finding
3. ✅ Learn to write exploits/PoCs
4. ✅ Join a bug bounty program
5. ✅ Contribute to open-source security tools

---

## Support

For questions or issues:
- Check the main README.md
- Review the reports/ directory for examples
- Open an issue on GitHub

---

**Happy hacking! 🔒**

Remember: *With great power comes great responsibility. Always test ethically.*
