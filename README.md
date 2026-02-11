# Android Application Security Analysis

> A comprehensive security assessment project demonstrating static analysis, dynamic analysis, and penetration testing of Android applications.

![Android Security](https://img.shields.io/badge/Platform-Android-green)
![Security](https://img.shields.io/badge/Category-Security%20Analysis-red)
![Tools](https://img.shields.io/badge/Tools-Burp%20Suite%20%7C%20APKTool%20%7C%20Jadx-blue)

## 📋 Project Overview

This project demonstrates a complete security analysis workflow for Android applications, including:
- Static code analysis and reverse engineering
- Dynamic runtime analysis using emulators
- Network traffic interception and API testing
- Identification of common security vulnerabilities

## 🎯 Objectives

- Perform comprehensive security assessment of Android applications
- Identify OWASP Mobile Top 10 vulnerabilities
- Analyze insecure data storage mechanisms
- Test for exposed APIs and misconfigurations
- Document findings with detailed reports and proof-of-concepts

## 🔧 Tools & Technologies

### Analysis Tools
- **JADX** - APK decompilation and static analysis
- **APKTool** - APK reverse engineering and resource extraction
- **Burp Suite** - Traffic interception and API testing
- **Android Studio** - Emulator and debugging
- **ADB (Android Debug Bridge)** - Device interaction and debugging
- **MobSF** - Mobile Security Framework for automated analysis

### Testing Environment
- Android Emulator (API Level 28-34)
- Genymotion (alternative emulator)
- Rooted device/emulator for deep analysis
- Proxy configuration for traffic interception

## 🏗️ Project Structure

```
android-security-analysis/
├── reports/                    # Security analysis reports
│   ├── static-analysis.md
│   ├── dynamic-analysis.md
│   ├── network-analysis.md
│   └── final-report.md
├── screenshots/               # Evidence and proof-of-concepts
│   ├── burp-intercept/
│   ├── insecure-storage/
│   └── api-exposure/
├── tools/                     # Custom scripts and utilities
│   ├── setup-environment.sh
│   ├── certificate-pinning-bypass.js
│   └── data-extraction.py
├── vulnerable-app/            # Sample vulnerable app (if created)
│   └── InsecureBank.apk
└── README.md
```

## 🚀 Getting Started

### Prerequisites

```bash
# Install required tools (Linux/macOS)
sudo apt-get update
sudo apt-get install openjdk-11-jdk android-tools-adb

# Install JADX
wget https://github.com/skylot/jadx/releases/download/v1.4.7/jadx-1.4.7.zip
unzip jadx-1.4.7.zip -d jadx

# Install APKTool
wget https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/linux/apktool
wget https://bitbucket.org/iBotPeaches/apktool/downloads/apktool_2.9.1.jar
chmod +x apktool
sudo mv apktool apktool_2.9.1.jar /usr/local/bin/
```

### Environment Setup

1. **Configure Android Emulator**
   ```bash
   # Launch Android emulator
   emulator -avd Pixel_4_API_30 -writable-system
   
   # Root the emulator (if needed)
   adb root
   adb remount
   ```

2. **Setup Burp Suite Proxy**
   ```bash
   # Configure proxy on emulator
   adb shell settings put global http_proxy <YOUR_IP>:8080
   
   # Install Burp CA certificate
   adb push burp-cert.der /sdcard/
   ```

3. **Install Test Application**
   ```bash
   adb install vulnerable-app.apk
   ```

## 🔍 Analysis Methodology

### 1. Static Analysis

**Objectives:**
- Decompile APK and analyze source code
- Identify hardcoded credentials and API keys
- Review AndroidManifest.xml for misconfigurations
- Analyze insecure data storage patterns

**Process:**
```bash
# Decompile APK
jadx -d output/ target-app.apk

# Extract resources
apktool d target-app.apk -o extracted/

# Search for sensitive data
grep -r "password\|api_key\|secret" output/
```

**Key Findings:**
- Hardcoded API keys in source code
- Insecure shared preferences usage
- Exported activities without proper permissions
- Weak cryptographic implementations

### 2. Dynamic Analysis

**Objectives:**
- Monitor runtime behavior
- Analyze memory for sensitive data
- Test authentication mechanisms
- Identify insecure data transmission

**Process:**
```bash
# Launch app and monitor logs
adb logcat | grep -i "password\|token\|api"

# Inspect app databases
adb shell
run-as com.example.app
cd databases/
sqlite3 app.db
```

**Key Findings:**
- Passwords stored in plaintext in SQLite
- Session tokens logged in clear text
- Sensitive data in application logs
- Insecure file permissions

### 3. Network Traffic Analysis

**Objectives:**
- Intercept HTTP/HTTPS traffic
- Analyze API endpoints and parameters
- Test for authentication bypass
- Identify exposed sensitive data

**Process:**
- Configure device to use Burp Suite proxy
- Install SSL certificate on device
- Bypass certificate pinning (if implemented)
- Intercept and modify requests

**Key Findings:**
- API endpoints exposed without authentication
- Sensitive data transmitted over HTTP
- Missing rate limiting on critical endpoints
- Weak session management

## 📊 Vulnerability Categories

### OWASP Mobile Top 10 Coverage

| Vulnerability | Severity | Found | Description |
|--------------|----------|-------|-------------|
| M1: Improper Platform Usage | High | ✅ | Exported components without protection |
| M2: Insecure Data Storage | Critical | ✅ | Plaintext passwords in SharedPreferences |
| M3: Insecure Communication | High | ✅ | HTTP traffic for sensitive operations |
| M4: Insecure Authentication | High | ✅ | Weak session management |
| M5: Insufficient Cryptography | Medium | ✅ | Hardcoded encryption keys |
| M6: Insecure Authorization | Medium | ✅ | Missing authorization checks |
| M7: Client Code Quality | Low | ⚠️ | Some code quality issues |
| M8: Code Tampering | Medium | ⚠️ | No integrity checks |
| M9: Reverse Engineering | Medium | ✅ | Easy to decompile |
| M10: Extraneous Functionality | Low | ⚠️ | Debug features in production |

## 🎓 Key Learnings

### Technical Skills Developed
- APK reverse engineering and decompilation
- Traffic interception and analysis
- Mobile penetration testing methodologies
- Security vulnerability identification
- Report writing and documentation

### Security Best Practices Identified
- Always encrypt sensitive data at rest
- Use certificate pinning for critical connections
- Implement proper authentication and authorization
- Remove debug code from production builds
- Follow principle of least privilege
- Validate and sanitize all inputs

## 📝 Sample Findings

### Critical: Plaintext Password Storage
**Description:** User credentials stored in SharedPreferences without encryption

**Location:** `shared_prefs/user_prefs.xml`

**Evidence:**
```xml
<string name="username">admin@example.com</string>
<string name="password">Admin@123</string>
```

**Impact:** Any app with backup permissions can access credentials

**Recommendation:** Use Android Keystore for secure credential storage

---

### High: Exposed API Endpoints
**Description:** Admin API endpoints accessible without authentication

**Request:**
```http
GET /api/admin/users HTTP/1.1
Host: api.example.com
```

**Response:**
```json
{
  "users": [
    {"id": 1, "email": "admin@example.com", "role": "admin"}
  ]
}
```

**Recommendation:** Implement proper authentication and authorization

## 🛡️ Remediation Guidelines

1. **Secure Data Storage**
   - Use Android Keystore for cryptographic keys
   - Encrypt sensitive data with AES-256
   - Use SQL Cipher for database encryption

2. **Secure Communication**
   - Use HTTPS for all network communications
   - Implement certificate pinning
   - Validate SSL certificates properly

3. **Code Obfuscation**
   - Enable ProGuard/R8 in release builds
   - Remove debug logs and comments
   - Implement root detection

## 📚 References & Resources

- [OWASP Mobile Security Testing Guide](https://owasp.org/www-project-mobile-security-testing-guide/)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
- [Android Security Documentation](https://developer.android.com/topic/security)
- [Burp Suite Mobile Testing](https://portswigger.net/burp/documentation/desktop/mobile)

## 🤝 Contributing

This is a learning project, but suggestions and improvements are welcome!

