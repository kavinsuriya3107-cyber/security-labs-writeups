# OWASP UnCrackable Level 1 — Android Reverse Engineering Writeup

## Overview

UnCrackable Level 1 is part of OWASP's Mobile Security Testing Guide crackme
series — a deliberately vulnerable Android app built to practice static and
dynamic analysis. The app has a single input field: type in the "correct
secret" and it tells you if you got it right. The secret isn't stored as
plaintext anywhere obvious — it's recovered by decrypting a hardcoded
AES-encrypted value, which means the actual challenge is reconstructing the
validation logic from decompiled Java and reproducing it yourself.

This writeup walks through how I got from "unknown APK" to "recovered secret,
confirmed in-app" — static analysis first, then an independent Python
reproduction, then a runtime check to prove the two agree.

## Environment

| Tool | Purpose |
|---|---|
| EndeavourOS (Linux) | Host OS |
| JADX-GUI | Decompiling the APK to readable Java |
| Android Studio Emulator (API 37) | Running the app for dynamic testing |
| Python 3 + `cryptography` | Reproducing the AES decryption offline |
| `unzip` | Initial APK archive inspection |

## 1. Inspecting the APK

Before touching a decompiler, I checked what was actually inside the archive:

```bash
unzip -l UnCrackable-Level1.apk | head -40
```

Standard structure: `classes.dex`, `AndroidManifest.xml`, resources, and the
usual `META-INF` signing files. No `lib/` directory — so no native code to
worry about, which meant JADX alone would be enough to read the whole app.

## 2. Finding the Entry Point

Opened the APK in JADX-GUI and checked the manifest for the launcher
activity:

- **Package:** `owasp.mstg.uncrackable1`
- **Launcher Activity:** `sg.vantagepoint.uncrackable1.MainActivity`
- **minSdkVersion:** 19 · **targetSdkVersion:** 28

## 3. Reading `onCreate()`

```java
if (c.a() || c.b() || c.c()) {
    a("Root detected!");
}
if (b.a(getApplicationContext())) {
    a("App is debuggable!");
}
super.onCreate(bundle);
setContentView(R.layout.activity_main);
```

Three root-detection calls and one debuggable-configuration check run before
anything else. Worth noting: `b.a()` only checks whether the app is *built*
debuggable — it doesn't confirm a debugger is actually attached at runtime,
which is a meaningfully different thing. When any of these checks fire, a
shared helper `a(String)` pops an alert and calls `System.exit(0)`.

## 4. Tracing the Secret Check

The `verify()` method tied to the input field doesn't do any comparison
itself — it just hands the user's input off to a validation function:

```java
byte[] key = b("8d127684cbc37c17616d806cf50473cc");
byte[] ciphertext = Base64.decode(
    "5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=", 0
);
return input.equals(new String(decryptedBytes));
```

Both the AES key and the ciphertext are sitting right there in the decompiled
source — hex-encoded and Base64-encoded respectively, but neither is actually
*hidden*. That's the whole vulnerability in one line.

## 5. The AES Helper — and a Detail Worth Getting Right

```java
SecretKeySpec secretKeySpec = new SecretKeySpec(keyBytes, "AES/ECB/PKCS7Padding");
Cipher cipher = Cipher.getInstance("AES");
cipher.init(2, secretKeySpec); // 2 = DECRYPT_MODE
return cipher.doFinal(ciphertext);
```

This is where it's easy to get sloppy. The `SecretKeySpec` is labeled
`"AES/ECB/PKCS7Padding"`, but that label is just a string tag on the key
object — it isn't what tells the cipher how to operate. The actual
transformation requested is `Cipher.getInstance("AES")`, a bare algorithm
name. In most JCA/Android providers that resolves to AES in ECB mode with
PKCS5 padding by default — and since PKCS5 and PKCS7 padding are identical
for AES's 16-byte block size, the label isn't wrong, just imprecise about
*why* it's correct. I didn't want to just assume this from the source,
though — the Python reproduction below is what actually confirms it, not the
source code alone.

## 6. Reproducing the Decryption in Python

Rather than trust my reading of the Java, I rebuilt the decryption
independently:


```python
import base64
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import padding

key = bytes.fromhex("8d127684cbc37c17616d806cf50473cc")
ciphertext = base64.b64decode("5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=")

cipher = Cipher(algorithms.AES(key), modes.ECB())
decryptor = cipher.decryptor()
padded_plaintext = decryptor.update(ciphertext) + decryptor.finalize()

unpadder = padding.PKCS7(algorithms.AES.block_size).unpadder()
plaintext = unpadder.update(padded_plaintext) + unpadder.finalize()

print("Recovered secret:", plaintext.decode("utf-8")) 

```

AES-ECB with PKCS7 unpadding, explicitly specified — no guessing involved
on the Python side.

## 7. Confirming It in the App

Two tests in the emulator, to make sure this wasn't a false positive:

| Input | Result |
|---|---|
| `hello123` | ❌ "That's not it. Try again." |
| `I want to believe` | ✅ "This is the correct secret." |

The recovered value matched the app's own validation — the static analysis
and the independent decryption agreed with runtime behavior, which is really
the only way to call this confirmed rather than just "probably right."

## Security Findings

**Hardcoded cryptographic material.** The AES key and ciphertext both ship
inside the APK. Embedding them client-side doesn't make them secret — it
just adds one decompilation step between an attacker and the plaintext.
*Fix: don't ship long-term secrets in a client binary at all; validate
sensitive values server-side.*

**Client-side-only validation.** The entire pass/fail decision happens on
the device, which means the device can be made to lie about the outcome.
*Fix: security-critical checks need a trusted server in the loop — the
client should never be the sole authority on whether something is valid.*

**Root/debug checks as the only integrity signal.** The detection logic is
plain Java, callable and patchable like any other method. *Fix: treat these
as one weak signal among several, never as a standalone security boundary.*

## What This Reinforced

- Full trace from a UI event down to the exact cryptographic primitive
  deciding pass/fail — not just "I found strings that looked like a key."
- A `SecretKeySpec` label and a `Cipher.getInstance()` transformation are two
  different things, and conflating them is an easy, common mistake.
- Static analysis tells you what *should* happen; only running the app tells
  you it actually *does*. I didn't call this done until both agreed.```
