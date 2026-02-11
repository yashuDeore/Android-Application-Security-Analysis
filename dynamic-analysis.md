# Dynamic Analysis Report

## Executive Summary

This report documents the dynamic analysis performed on the target Android application during runtime. The analysis revealed critical vulnerabilities in session management, insecure data storage, and information disclosure through application logs.

**Analysis Date:** February 2026  
**Application:** Sample Banking App (com.example.insecurebank)  
**Test Environment:** Android Emulator (Pixel 4 API 30)  
**Analysis Duration:** 4 hours  

---

## Methodology

### Test Environment Setup
- **Device:** Android Emulator (rooted)
- **Android Version:** 11 (API 30)
- **Tools:** ADB, Logcat, Frida, SQLite Browser
- **Monitoring:** Real-time traffic and storage analysis

### Testing Approach
1. Install application on rooted emulator
2. Monitor runtime behavior using ADB logcat
3. Inspect file system for sensitive data
4. Analyze application memory
5. Test authentication and session management
6. Database and storage inspection

---

## Findings

### 🔴 CRITICAL: Credentials Logged in Plaintext

**Vulnerability ID:** DA-001  
**Severity:** Critical  
**OWASP Category:** M2 - Insecure Data Storage

**Description:**  
User credentials and sensitive information are logged to Android Logcat in plaintext during authentication.

**Testing Steps:**
```bash
# Start logcat monitoring
adb logcat | grep -E "password|token|credential"

# Launch app and login with test credentials
# Username: test@example.com
# Password: Test@123
```

**Log Output:**
```
02-11 10:15:23.456 12345 12345 D LoginActivity: Attempting login
02-11 10:15:23.457 12345 12345 D LoginActivity: Username: test@example.com
02-11 10:15:23.458 12345 12345 D LoginActivity: Password: Test@123
02-11 10:15:24.123 12345 12345 D ApiClient: Bearer Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
02-11 10:15:24.456 12345 12345 I AccountManager: User ID: 12345, Account: ****1234
02-11 10:15:24.789 12345 12345 D TransactionService: PIN entered: 4521
```

**Impact:**
- Any app with READ_LOGS permission can capture credentials
- Logs may be included in bug reports
- Credentials exposed in crash dumps
- Easy attack vector on rooted/debugging devices

**Recommendation:**
- Remove all sensitive logging from production builds
- Use ProGuard to strip debug code
- Implement log filtering for sensitive fields

**CVSS Score:** 9.3 (Critical)

---

### 🔴 CRITICAL: Insecure SQLite Database

**Vulnerability ID:** DA-002  
**Severity:** Critical  
**OWASP Category:** M2 - Insecure Data Storage

**Description:**  
Application stores sensitive user data in an unencrypted SQLite database accessible on rooted devices.

**Testing Steps:**
```bash
# Access app's private directory
adb shell
su
cd /data/data/com.example.insecurebank/databases

# Copy database to analyze
adb pull /data/data/com.example.insecurebank/databases/banking.db
sqlite3 banking.db
```

**Database Schema:**
```sql
sqlite> .tables
users  transactions  cards  beneficiaries

sqlite> SELECT * FROM users;
id|username|password|full_name|ssn|phone|balance
1|test@example.com|Test@123|John Doe|123-45-6789|555-0100|25000.00
2|admin@bank.com|Admin@2024|Jane Admin|987-65-4321|555-0200|50000.00

sqlite> SELECT * FROM cards;
id|user_id|card_number|cvv|expiry|pin
1|1|4532123456789012|123|12/26|4521
2|2|5425233430109903|456|03/27|7890
```

**Sensitive Data Found:**
- Plaintext passwords
- Social Security Numbers
- Complete credit card details (PAN, CVV, PIN)
- Account balances
- Personal identification information

**File Permissions:**
```bash
ls -la /data/data/com.example.insecurebank/databases/
-rw-rw---- 1 u0_a123 u0_a123 40960 banking.db
-rw-rw---- 1 u0_a123 u0_a123 32768 banking.db-wal
```

**Impact:**
- Complete data breach on rooted devices
- PCI-DSS compliance violation
- GDPR violation (inadequate data protection)
- Identity theft risk

**Recommendation:**
```java
// Implement SQLCipher encryption
dependencies {
    implementation 'net.zetetic:android-database-sqlcipher:4.5.4'
}

// Encrypt database with strong key from Keystore
String password = getEncryptionKeyFromKeystore();
SQLiteDatabase db = SQLiteDatabase.openOrCreateDatabase(
    dbFile, password, null
);
```

**CVSS Score:** 9.8 (Critical)

---

### 🟠 HIGH: Insecure SharedPreferences Storage

**Vulnerability ID:** DA-003  
**Severity:** High  
**OWASP Category:** M2 - Insecure Data Storage

**Description:**  
Session tokens and user preferences stored in plaintext XML files.

**Testing Steps:**
```bash
adb shell
su
cd /data/data/com.example.insecurebank/shared_prefs
cat UserPrefs.xml
```

**File Content:**
```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="username">test@example.com</string>
    <string name="password">Test@123</string>
    <string name="session_token">eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...</string>
    <boolean name="remember_me">true</boolean>
    <string name="device_id">android_device_123456</string>
    <long name="last_login">1707648923000</long>
    <string name="account_number">1234567890</string>
</map>
```

**Additional Files:**
```bash
cat AppConfig.xml
<string name="api_endpoint">https://api.insecurebank.com</string>
<string name="api_key">sk_live_51234567890abcdef</string>
```

**Impact:**
- Session hijacking possible
- Credentials recoverable from backups
- Cloud backup exposure (if enabled)

**Evidence of Backup Vulnerability:**
```bash
# Backup app data
adb backup -f backup.ab com.example.insecurebank
# Convert and extract
dd if=backup.ab bs=24 skip=1 | openssl zlib -d > backup.tar
tar -xvf backup.tar
# All SharedPreferences now accessible!
```

**Recommendation:**
Use EncryptedSharedPreferences from Android Security library

**CVSS Score:** 8.1 (High)

---

### 🟠 HIGH: Weak Session Management

**Vulnerability ID:** DA-004  
**Severity:** High  
**OWASP Category:** M4 - Insecure Authentication

**Description:**  
Sessions do not expire, and session tokens are not invalidated on logout.

**Testing Process:**
1. Login to application
2. Capture session token: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`
3. Logout from application
4. Reuse old session token → **Still valid!**

**Token Analysis:**
```bash
# Decode JWT token
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." | cut -d'.' -f2 | base64 -d

{
  "user_id": "12345",
  "username": "test@example.com",
  "role": "user",
  "iat": 1707648923
  // No 'exp' field - token never expires!
}
```

**Additional Issues:**
- No session timeout
- Tokens not invalidated server-side
- No token refresh mechanism
- Session survives app reinstall (stored in SharedPreferences)

**Test Proof:**
```bash
# Day 1: Get token after login
Token: abc123...

# Day 7: Token still works
curl -H "Authorization: Bearer abc123..." https://api.example.com/user/profile
# Response: 200 OK (Should have expired!)

# After logout: Token still works
# After password change: Old token still works!
```

**Impact:**
- Session hijacking attacks
- Unauthorized access after logout
- Inability to revoke compromised sessions

**Recommendation:**
- Implement token expiration (exp claim)
- Invalidate sessions on logout
- Use refresh tokens
- Implement server-side session management

**CVSS Score:** 7.4 (High)

---

### 🟡 MEDIUM: Application Files World-Readable

**Vulnerability ID:** DA-005  
**Severity:** Medium  
**OWASP Category:** M2 - Insecure Data Storage

**Description:**  
Some application files are created with insecure permissions allowing access by other apps.

**Testing Steps:**
```bash
adb shell
cd /data/data/com.example.insecurebank/files
ls -la
```

**Output:**
```
-rw-rw-rw- 1 u0_a123 u0_a123  4096 temp_transactions.json
-rw-rw-rw- 1 u0_a123 u0_a123  2048 user_cache.txt
-rw-rw---- 1 u0_a123 u0_a123  8192 encrypted_data.bin
```

**Vulnerable File Content:**
```bash
cat temp_transactions.json
{
  "transactions": [
    {
      "id": "TXN001",
      "amount": 1500.00,
      "account": "****6789",
      "timestamp": "2026-02-11T10:30:00Z"
    }
  ]
}
```

**Impact:**
- Information disclosure to malicious apps
- Data leakage through file access

**Recommendation:**
```java
// Set secure file permissions
FileOutputStream fos = context.openFileOutput(
    "sensitive_data.txt",
    Context.MODE_PRIVATE  // Only this app can access
);
```

**CVSS Score:** 5.5 (Medium)

---

### 🟡 MEDIUM: Sensitive Data in Memory

**Vulnerability ID:** DA-006  
**Severity:** Medium  
**OWASP Category:** M2 - Insecure Data Storage

**Description:**  
Sensitive data persists in memory and can be extracted via memory dumps.

**Testing with Frida:**
```javascript
// Frida script to hook password field
Java.perform(function() {
    var EditText = Java.use("android.widget.EditText");
    EditText.getText.implementation = function() {
        var text = this.getText();
        console.log("EditText value: " + text);
        return text;
    };
});
```

**Output:**
```
EditText value: Test@123
EditText value: 4532123456789012
EditText value: 4521
```

**Memory Dump Analysis:**
```bash
# Take memory dump
adb shell am dumpheap com.example.insecurebank /data/local/tmp/heap.dump
adb pull /data/local/tmp/heap.dump

# Analyze with MAT (Memory Analyzer Tool)
# Found in memory:
- Passwords as String objects
- Credit card numbers
- Session tokens
- API keys
```

**Impact:**
- Data extraction through memory dumps
- Sensitive data not cleared after use

**Recommendation:**
- Use char[] instead of String for passwords
- Clear sensitive data after use
- Implement memory scrubbing

**CVSS Score:** 6.0 (Medium)

---

## Runtime Behavior Analysis

### Application Flow
```
Launch → Login Screen → Enter Credentials
    ↓
Credentials logged to Logcat (CRITICAL)
    ↓
Stored in SharedPreferences (HIGH)
    ↓
API Authentication → Token received
    ↓
Token stored unencrypted (HIGH)
    ↓
Database access → Unencrypted SQLite (CRITICAL)
```

### Security Events Logged
- All authentication attempts
- Transaction details
- Account balances
- PIN entries
- API endpoints accessed

---

## Summary Statistics

| Severity | Count | Percentage |
|----------|-------|------------|
| Critical | 2 | 33% |
| High | 2 | 33% |
| Medium | 2 | 34% |
| **Total** | **6** | **100%** |

## Attack Scenarios

### Scenario 1: Rooted Device Attack
1. User installs app on rooted device
2. Attacker gains root access
3. Extracts database file → Gets all user data including passwords, SSN, cards
4. Reads SharedPreferences → Gets session token
5. Uses token to access API → Complete account takeover

**Likelihood:** High  
**Impact:** Critical

### Scenario 2: Malicious App Attack
1. User installs malicious app with READ_LOGS permission
2. Malicious app monitors logcat
3. User logs into banking app
4. Malicious app captures credentials from logs
5. Credentials used for unauthorized access

**Likelihood:** Medium  
**Impact:** Critical

---

## Recommendations

### Critical Priority
1. **Remove sensitive logging immediately**
2. **Encrypt SQLite database with SQLCipher**
3. **Implement EncryptedSharedPreferences**
4. **Add session expiration and invalidation**

### High Priority
1. Implement proper session management
2. Add token refresh mechanism
3. Clear sensitive data from memory
4. Fix file permissions

### Medium Priority
1. Implement root detection
2. Add jailbreak detection
3. Use Android Keystore for key management
4. Implement certificate pinning

---

**Analyst:** Security Researcher  
**Report Generated:** February 11, 2026
