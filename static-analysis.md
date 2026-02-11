# Static Analysis Report

## Executive Summary

This report documents the static analysis performed on the target Android application. The analysis revealed multiple security vulnerabilities including hardcoded credentials, insecure data storage configurations, and exposed application components.

**Analysis Date:** February 2026  
**Application:** Sample Banking App (com.example.insecurebank)  
**APK Version:** 1.0.0  
**Target SDK:** 30  

---

## Methodology

### Tools Used
- **JADX v1.4.7** - Decompilation and source code analysis
- **APKTool v2.9.1** - Resource extraction and manifest analysis
- **grep/ripgrep** - Pattern matching for sensitive data
- **Android Studio** - Code review and navigation

### Analysis Steps
1. APK decompilation using JADX
2. AndroidManifest.xml security review
3. Source code analysis for hardcoded secrets
4. Resource file inspection
5. Library and dependency analysis
6. Code quality assessment

---

## Findings

### 🔴 CRITICAL: Hardcoded API Credentials

**Vulnerability ID:** SA-001  
**Severity:** Critical  
**OWASP Category:** M5 - Insufficient Cryptography

**Description:**  
API credentials and secret keys are hardcoded directly in the source code, making them easily accessible through reverse engineering.

**Location:**  
`com/example/insecurebank/api/ApiConfig.java`

**Code Evidence:**
```java
public class ApiConfig {
    public static final String API_KEY = "sk_live_51234567890abcdef";
    public static final String API_SECRET = "whsec_abc123def456ghi789";
    public static final String BASE_URL = "https://api.example.com";
    
    private static final String ENCRYPTION_KEY = "MySecretKey12345"; // AES key
}
```

**Impact:**
- Attackers can extract API keys through APK decompilation
- Unauthorized API access possible
- Potential data breaches and financial loss

**Recommendation:**
- Store API keys in secure backend services
- Use OAuth 2.0 for authentication
- Implement certificate pinning
- Use Android Keystore for key management

**CVSS Score:** 9.1 (Critical)

---

### 🔴 CRITICAL: Insecure SharedPreferences Usage

**Vulnerability ID:** SA-002  
**Severity:** Critical  
**OWASP Category:** M2 - Insecure Data Storage

**Description:**  
User credentials and sensitive data stored in SharedPreferences without encryption.

**Location:**  
`com/example/insecurebank/utils/PreferenceManager.java`

**Code Evidence:**
```java
public void saveUserCredentials(String username, String password) {
    SharedPreferences prefs = context.getSharedPreferences("UserPrefs", MODE_PRIVATE);
    SharedPreferences.Editor editor = prefs.edit();
    editor.putString("username", username);
    editor.putString("password", password);  // Plaintext password!
    editor.putString("account_number", accountNumber);
    editor.apply();
}
```

**Impact:**
- Credentials accessible to any app with backup permissions
- Easy extraction through ADB on rooted devices
- Data leakage through cloud backups

**Recommendation:**
```java
// Use EncryptedSharedPreferences
import androidx.security.crypto.EncryptedSharedPreferences;
import androidx.security.crypto.MasterKeys;

String masterKeyAlias = MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC);

SharedPreferences sharedPreferences = EncryptedSharedPreferences.create(
    "secure_prefs",
    masterKeyAlias,
    context,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
);
```

**CVSS Score:** 8.8 (High)

---

### 🟠 HIGH: Exported Components Without Protection

**Vulnerability ID:** SA-003  
**Severity:** High  
**OWASP Category:** M1 - Improper Platform Usage

**Description:**  
Multiple Android components are exported without proper permission checks, allowing unauthorized access from other applications.

**Location:**  
`AndroidManifest.xml`

**Vulnerable Configuration:**
```xml
<activity
    android:name=".activities.AdminActivity"
    android:exported="true">  <!-- No permission required! -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>

<receiver
    android:name=".receivers.TransactionReceiver"
    android:exported="true">  <!-- Accessible by any app -->
</receiver>

<provider
    android:name=".providers.UserDataProvider"
    android:authorities="com.example.insecurebank.provider"
    android:exported="true"
    android:grantUriPermissions="true">  <!-- Database exposed! -->
</provider>
```

**Impact:**
- Unauthorized access to admin functionality
- Data exposure through content provider
- Intent spoofing attacks possible

**Recommendation:**
```xml
<!-- Add custom permissions -->
<permission
    android:name="com.example.insecurebank.permission.ADMIN_ACCESS"
    android:protectionLevel="signature" />

<activity
    android:name=".activities.AdminActivity"
    android:exported="true"
    android:permission="com.example.insecurebank.permission.ADMIN_ACCESS">
</activity>

<!-- Or set exported to false if not needed by other apps -->
<activity
    android:name=".activities.AdminActivity"
    android:exported="false">
</activity>
```

**CVSS Score:** 7.5 (High)

---

### 🟠 HIGH: Insecure Database Storage

**Vulnerability ID:** SA-004  
**Severity:** High  
**OWASP Category:** M2 - Insecure Data Storage

**Description:**  
SQLite database stored without encryption, containing sensitive user data.

**Location:**  
`com/example/insecurebank/database/DatabaseHelper.java`

**Code Evidence:**
```java
public class DatabaseHelper extends SQLiteOpenHelper {
    private static final String DATABASE_NAME = "banking.db";
    // No encryption implemented!
    
    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE users (" +
                   "id INTEGER PRIMARY KEY," +
                   "username TEXT," +
                   "password TEXT," +  // Plaintext!
                   "account_balance REAL," +
                   "ssn TEXT)");  // PII stored without encryption!
    }
}
```

**Database File Location:**  
`/data/data/com.example.insecurebank/databases/banking.db`

**Impact:**
- All user data accessible on rooted devices
- Easy extraction through ADB backup
- Compliance violations (PCI-DSS, GDPR)

**Recommendation:**
```java
// Use SQLCipher for database encryption
import net.sqlcipher.database.SQLiteDatabase;

SQLiteDatabase.loadLibs(context);
SQLiteDatabase db = SQLiteDatabase.openOrCreateDatabase(
    databaseFile,
    password,  // Strong encryption password from Android Keystore
    null
);
```

**CVSS Score:** 7.8 (High)

---

### 🟡 MEDIUM: Weak Cryptographic Implementation

**Vulnerability ID:** SA-005  
**Severity:** Medium  
**OWASP Category:** M5 - Insufficient Cryptography

**Description:**  
Custom encryption implementation using weak algorithms and hardcoded keys.

**Location:**  
`com/example/insecurebank/crypto/CryptoUtil.java`

**Code Evidence:**
```java
public class CryptoUtil {
    private static final String ALGORITHM = "DES";  // Weak algorithm!
    private static final String KEY = "12345678";   // Hardcoded key!
    
    public static String encrypt(String data) {
        SecretKeySpec keySpec = new SecretKeySpec(KEY.getBytes(), ALGORITHM);
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        cipher.init(Cipher.ENCRYPT_MODE, keySpec);
        return Base64.encodeToString(cipher.doFinal(data.getBytes()), Base64.DEFAULT);
    }
}
```

**Issues:**
- DES is deprecated and cryptographically broken
- Hardcoded encryption key
- No IV (Initialization Vector) used
- ECB mode (implicit) - patterns leak

**Recommendation:**
```java
// Use AES-256 with GCM mode
public class SecureCrypto {
    private static final String ALGORITHM = "AES/GCM/NoPadding";
    
    public static byte[] encrypt(byte[] data, SecretKey key) throws Exception {
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec spec = new GCMParameterSpec(128, generateIV());
        cipher.init(Cipher.ENCRYPT_MODE, key, spec);
        return cipher.doFinal(data);
    }
    
    // Get key from Android Keystore
    private static SecretKey getKey() {
        KeyStore keyStore = KeyStore.getInstance("AndroidKeyStore");
        return (SecretKey) keyStore.getKey("MyKeyAlias", null);
    }
}
```

**CVSS Score:** 6.5 (Medium)

---

### 🟡 MEDIUM: Debug Features in Production

**Vulnerability ID:** SA-006  
**Severity:** Medium  
**OWASP Category:** M10 - Extraneous Functionality

**Description:**  
Debug functionality and logging enabled in production builds.

**Location:**  
`AndroidManifest.xml` and various source files

**Evidence:**
```xml
<application
    android:debuggable="true"  <!-- Should be false in production! -->
    android:allowBackup="true">  <!-- Security risk -->
```

```java
public class LoginActivity extends AppCompatActivity {
    private static final String TAG = "LoginActivity";
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        Log.d(TAG, "User credentials: " + username + ":" + password);  // Logging sensitive data!
        
        if (BuildConfig.DEBUG) {
            // Debug backdoor
            if (username.equals("debug") && password.equals("admin123")) {
                loginSuccess();
            }
        }
    }
}
```

**Impact:**
- App can be debugged in production
- Sensitive data logged and accessible
- Debug backdoors present
- Backup allows data extraction

**Recommendation:**
```xml
<application
    android:debuggable="false"
    android:allowBackup="false">
```

```groovy
// In build.gradle
buildTypes {
    release {
        debuggable false
        minifyEnabled true
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt')
    }
}
```

**CVSS Score:** 5.3 (Medium)

---

## Summary Statistics

| Severity | Count | Percentage |
|----------|-------|------------|
| Critical | 2 | 33% |
| High | 2 | 33% |
| Medium | 2 | 34% |
| **Total** | **6** | **100%** |

## Compliance Impact

### OWASP Mobile Top 10
- ✅ M1: Improper Platform Usage
- ✅ M2: Insecure Data Storage
- ✅ M5: Insufficient Cryptography
- ✅ M10: Extraneous Functionality

### Regulatory Concerns
- **PCI-DSS:** Non-compliant (plaintext card data)
- **GDPR:** High risk (unencrypted PII)
- **CCPA:** Data protection violations

## Recommendations Priority

1. **Immediate Actions:**
   - Remove hardcoded API keys
   - Encrypt SharedPreferences and database
   - Disable debug mode in production

2. **Short-term (1-2 weeks):**
   - Implement proper encryption (AES-256-GCM)
   - Add permissions to exported components
   - Remove debug logging

3. **Long-term:**
   - Implement certificate pinning
   - Add root detection
   - Enable ProGuard/R8 obfuscation
   - Security code review process

---

**Analyst:** Security Researcher  
**Report Generated:** February 11, 2026
