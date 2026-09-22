# Tanix TX3 — JW Library WebView 52 Scrolling Problem

## Overview

The Tanix TX3 Android TV box is being used to run **JW Library** on a 1920×1080 display.

The device runs an old manufacturer-provided Android 8.1 firmware. During testing of JW Library, a problem was identified with the application's handling of document scrolling:

> JW Library does not reliably trigger/perform the expected scrolling to the end of a document.

The problem appears to be related to the very old WebView implementation included in the TX3 firmware.

The firmware contains:

```text
Package:      com.android.webview
Version:      52.0.2743.100
Version code: 275610000
Location:     /system/app/webview/webview.apk
Android:      8.1
API level:    27
```

Chromium/WebView 52 is extremely old compared with the WebView versions for which modern Android applications were developed.

The working hypothesis is therefore:

**JW Library is interacting correctly with Android, but WebView 52 does not implement the relevant scrolling/document behavior sufficiently for the current JW Library application.**

The purpose of the investigation is to verify this experimentally by running JW Library with a substantially newer WebView, without permanently modifying the firmware.

---

## Device information

The relevant device information is:

```text
Device:       Tanix TX3
Android:      8.1
API level:    27
Display:      1920 × 1080
CPU ABI:      armeabi-v7a
ABI list:     armeabi-v7a,armeabi
```

The architecture was determined with:

```sh
getprop ro.product.cpu.abi
getprop ro.product.cpu.abilist
```

Output:

```text
armeabi-v7a
armeabi-v7a,armeabi
```

Therefore the appropriate WebView APK architecture for this device is **ARMv7 / armeabi-v7a**, not ARM64.

---

## Original WebView installation

The firmware's WebView is installed as a system application:

```sh
pm path com.android.webview
```

Output:

```text
package:/system/app/webview/webview.apk
```

The package information is:

```sh
dumpsys package com.android.webview | grep -E 'versionCode|versionName'
```

Output:

```text
versionCode=275610000 minSdk=21 targetSdk=25
versionName=52.0.2743.100
```

The complete package location is:

```text
/system/app/webview/webview.apk
```

This means the original WebView is part of the TX3 firmware rather than an ordinary user-installed application.

---

## Why WebView is relevant to JW Library

JW Library contains content and interfaces that depend on Android's WebView/Chromium rendering stack.

The application can therefore depend on WebView behavior for operations involving HTML/document content, scrolling, JavaScript, layout, and interaction with rendered documents.

The relevant symptom is that the expected operation that should cause the document to scroll to its end does not behave correctly with the TX3's WebView 52.

This is particularly suspicious because:

1. The TX3 is running Android 8.1.
2. Its WebView is version 52 from the same general era as the original Android 7/8 firmware.
3. The currently installed JW Library application is substantially newer.
4. Modern WebView versions contain many years of Chromium fixes and behavioral changes.
5. The problem appears to involve interaction with rendered document content rather than a basic Android input operation.

The correct diagnostic approach is therefore to test a newer WebView before modifying JW Library or the underlying firmware.

---

## Important distinction: Android version vs. WebView version

Android 8.1 itself does not necessarily require WebView 52.

The Android operating system and the WebView implementation are related but are not the same thing.

The TX3 firmware happens to contain:

```text
Android 8.1
WebView 52
```

but a compatible Android 8.x device can run considerably newer Chromium/WebView releases.

Consequently, replacing or overriding the old WebView is a reasonable diagnostic experiment.

---

## Initial hypothesis

The investigation began with the following hypothesis:

```text
JW Library
     │
     ▼
Android WebView
     │
     ▼
Chromium 52
     │
     └── old document/scroll behavior
```

The proposed test is:

```text
JW Library
     │
     ▼
Android WebView
     │
     ▼
Chromium/WebView 112
     │
     └── test whether scrolling works correctly
```

If the problem disappears when WebView 112 is active, this provides strong evidence that the original WebView implementation is responsible.

This does not by itself prove that WebView 52 is the only possible cause, but it provides a controlled A/B test.

---

## Why WebView 112 was selected

A Google Android System WebView build was selected that supports:

```text
Android 7.0+
armeabi-v7a
```

The test version is:

```text
Android System WebView
112.0.5615.100
```

This is dramatically newer than:

```text
52.0.2743.100
```

while still being suitable for Android 8.1.

A conservative intermediate WebView is preferable to immediately attempting the newest possible WebView because Android 8.1 is an old platform and manufacturer firmware can have compatibility limitations.

---

## Important package-name discovery

The original firmware WebView uses:

```text
com.android.webview
```

Google's Android System WebView APK uses:

```text
com.google.android.webview
```

These are different Android packages.

This turned out to be important during testing.

The original firmware package remains:

```text
com.android.webview
```

at:

```text
/system/app/webview/webview.apk
```

while the newly installed Google WebView appears separately as:

```text
com.google.android.webview
```

at:

```text
/data/app/com.google.android.webview-1/base.apk
```

Therefore, installing the Google APK did **not** replace the firmware WebView.

Instead, the device now contains two WebView providers:

```text
com.android.webview
    52.0.2743.100
    /system/app/webview/webview.apk

com.google.android.webview
    112.0.5615.100
    /data/app/com.google.android.webview-1/base.apk
```

This is useful because it allows the newer provider to potentially be tested without immediately modifying the system partition.

---

## Current state

The device now has:

```text
Original provider:
com.android.webview
52.0.2743.100
/system/app/webview/webview.apk

Additional provider:
com.google.android.webview
112.0.5615.100
/data/app/com.google.android.webview-1/base.apk
```

The original system WebView remains untouched.

The next diagnostic step is to determine whether Android 8.1's `WebViewUpdateService` recognizes and permits the newly installed Google provider.

The relevant commands are:

```sh
settings get secure webview_provider
```

and:

```sh
dumpsys webviewupdate
```

These commands should be used before making any changes to `/system`.

---

## Conclusion

The TX3's JW Library scrolling problem is being investigated as a potential compatibility problem between the modern JW Library application and the firmware's extremely old Chromium/WebView 52 implementation.

The safest diagnostic path is:

1. Preserve the original WebView.
2. Install a compatible newer WebView as an additional package.
3. Determine whether Android recognizes it as a valid WebView provider.
4. Switch the active provider if the framework permits it.
5. Test JW Library.
6. Only consider modifying `/system` if provider switching is impossible and the newer WebView demonstrably fixes the problem.

At the current stage, **the original WebView has not been replaced**.
