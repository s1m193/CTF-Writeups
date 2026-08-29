# 🔐 AppLock — Android Reverse Engineering Write-up

## 🧭 Overview

The goal of this challenge was to analyze the Android application **AppLock (`com.domobile.applock`)** and bypass its pattern-based authentication.

The application stores the lock pattern in a way that allows it to be recovered from the application's local data. By combining **ADB**, **jadx**, static analysis, Base64 decoding, and a pattern-cracking tool, the original lock pattern can be recovered and used to bypass the application's authentication.

---

## 📱 Target Application

**Application:** AppLock
**Package:** `com.domobile.applock`
**Platform:** Android
**Environment:** LDPlayer 9.2.0.1
<img width="289" height="499" alt="image" src="https://github.com/user-attachments/assets/ed6fc679-7055-498c-b070-ea9a37b48805" />

---

## 🔎 Step 1 — Identifying the Target Activity

The first step was identifying the currently active Android activity using ADB.

The output revealed the application package and the lock screen activity:

```text
com.domobile.applock
com.domobile.applock.VerifyActivity
```

<img width="1000" height="52" alt="image" src="https://github.com/user-attachments/assets/0bc6b324-98bb-45de-a71a-822a603edaf8" />

The `VerifyActivity` appeared to be responsible for handling the pattern verification process, making it a good starting point for reverse engineering.

---

## 🧩 Step 2 — Reverse Engineering with jadx

The APK was loaded into **jadx** to inspect the application's Java code.

The `VerifyActivity` class was located, and the `onKeyDown` method was identified as part of the pattern verification flow.

<img width="739" height="249" alt="image" src="https://github.com/user-attachments/assets/115bed06-2940-4252-9ea7-51cd1efca0da" />

The method passed the input to an object responsible for displaying and processing the pattern verification screen.
<img width="924" height="60" alt="image" src="https://github.com/user-attachments/assets/4df103cd-cdd2-4e7d-92ed-149a6f2d04a0" />

Tracing this object revealed that it was an instance of:

```text
ConfirmLockPattern
```

This class appeared to handle the pattern input and validation logic.

---

## 🔬 Step 3 — Tracing the Pattern Listener

Inside `ConfirmLockPattern`, method `e()` was found to configure the pattern listener for the lock pattern view.

<img width="1117" height="249" alt="image" src="https://github.com/user-attachments/assets/69aa7d75-dbf8-4514-9670-e645f55027a2" />

The listener was implemented through an inner class named `h`.

<img width="451" height="58" alt="image" src="https://github.com/user-attachments/assets/368603f0-f430-4ab0-a75e-7945fda9af18" />

Following the execution flow led to another class responsible for performing the actual pattern comparison.

---

## 🔐 Step 4 — Finding the Pattern Validation Logic

Inside class `a`, a field named `c` was identified.

<img width="247" height="43" alt="image" src="https://github.com/user-attachments/assets/320516ae-61ff-44a4-b5b8-83404f0eadb5" />

The field referenced class `d`, which contained the following validation method:

```text
a(List list)
```

This method was responsible for checking whether the entered pattern matched the stored pattern.
The important part of the logic was the comparison between:

* The pattern entered by the user
* The stored pattern retrieved through `gb.f()`


<img width="555" height="154" alt="image" src="https://github.com/user-attachments/assets/e2ac8744-6419-42d6-8ba3-445a5f608331" />


The application decoded the stored Base64 value and compared the resulting bytes with the bytes generated from the entered pattern.

The comparison was performed using:

```text
Arrays.equals()
```

This confirmed that the application's authentication depended directly on a locally stored value.

---

## 🗂️ Step 5 — Locating the Stored Pattern

Following the `gb.f()` function revealed that the application retrieves the stored pattern from **SharedPreferences**.

The relevant key was:

```text
image_lock_pattern
```

<img width="478" height="69" alt="image" src="https://github.com/user-attachments/assets/90f02794-ded2-4d32-b7fc-551e281594f2" />

This was an important finding because SharedPreferences is not an appropriate place to store a sensitive authentication secret in a trivially reversible format.

---

## 📂 Step 6 — Extracting the Stored Value

With ADB access to the application's data, the SharedPreferences directory was inspected.

The key:

```text
image_lock_pattern
```
contained the following Base64-encoded value:

```text
2wTGRwcsTKXnuFMR4DEIRZrSt0k=
```

<img width="712" height="169" alt="image" src="https://github.com/user-attachments/assets/cb639979-cf83-4f39-b8fb-1585883c6382" />

At this point, the pattern itself was not directly visible, but the value was clearly Base64-encoded.

---

## 🔓 Step 7 — Decoding the Base64 Value

The Base64 value was decoded and saved for further analysis.

<img width="649" height="52" alt="image" src="https://github.com/user-attachments/assets/c23dfc7d-acad-404f-ab46-fc5012f07876" />

The resulting bytes represented the data used internally by the application during pattern verification.

---

## 🧨 Step 8 — Cracking the Android Pattern

The decoded bytes were then supplied to an Android pattern-lock cracking tool.
https://github.com/niejuhu/android_lock_cracker

The tool tested possible Android pattern combinations against the recovered data.

<img width="1056" height="634" alt="image" src="https://github.com/user-attachments/assets/8861ddc5-bfd6-47cf-b5b5-242af537702f" />

The matching pattern was recovered:


This represents the following gesture sequence on the standard 3×3 Android pattern grid.

<img width="470" height="332" alt="image" src="https://github.com/user-attachments/assets/3acee618-7c1b-4c7f-96f1-d3de1a41031b" />

---

## ✅ Step 9 — Authentication Bypass

The application accepted the pattern successfully.

<img width="540" height="961" alt="image" src="https://github.com/user-attachments/assets/d17ec04a-d9c7-466c-8907-17d62ddec7b8" />

This confirmed that the stored authentication value could be recovered and used to bypass the application's lock mechanism.

---

## 🎯 Result

The complete attack chain was:

```text
ADB
 ↓
Identify VerifyActivity
 ↓
Reverse engineer APK with jadx
 ↓
Trace ConfirmLockPattern
 ↓
Find pattern comparison logic
 ↓
Locate SharedPreferences
 ↓
Extract image_lock_pattern
 ↓
Decode Base64
 ↓
Crack recovered pattern
 ↓
3 → 6 → 4 → 2
 ↓
Authentication bypass
```

---

## 🛡️ Security Impact

The vulnerability allows an attacker with sufficient access to the device's application data to recover the AppLock authentication pattern.

Once the pattern is recovered, the attacker can unlock features and applications protected by AppLock without knowing the original pattern.

The main issue is that the authentication secret is stored locally in a trivially encoded form rather than being protected using a secure key derivation and storage mechanism.

---

## 🔧 Recommended Mitigations

A secure implementation should:

1. Avoid storing the raw pattern or a trivially reversible representation such as Base64.
2. Use a strong, salted password-based key derivation function such as **PBKDF2** for pattern-derived secrets.
3. Use the **Android Keystore System** for protecting cryptographic keys.
4. Avoid relying solely on client-side storage for security-sensitive authentication.
5. Consider appropriate root/debugging detection and tamper-resistance mechanisms.

---

## 🧠 Key Takeaways

* **ADB** can be extremely useful for Android application enumeration and runtime analysis.
* **jadx** can reveal authentication logic hidden inside an APK.
* Obfuscated code can still be traced by following data flow and method calls.
* **SharedPreferences** should not be used to store sensitive authentication data in plaintext or trivially encoded formats.
* **Base64 is encoding, not encryption.**
* Following the validation flow from the UI to the final comparison is often the fastest way to understand an application's authentication mechanism.

---

## 🧰 Tools Used

| Tool                        | Purpose                                                |
| --------------------------- | ------------------------------------------------------ |
| **ADB**                     | Android device interaction and application data access |
| **jadx**                    | APK decompilation and static analysis                  |
| **Base64**                  | Decoding the stored value                              |
| **Android Pattern Cracker** | Recovering the original lock pattern                   |
| **LDPlayer**                | Android testing environment                            |

---

**The challenge was ultimately solved by tracing how AppLock validates the pattern, recovering the locally stored authentication value, decoding it, and reproducing the correct pattern.**
